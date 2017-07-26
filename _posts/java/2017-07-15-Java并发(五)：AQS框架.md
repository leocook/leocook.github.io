---
layout: post
comments: true
date: 2017-07-15
categories: java
tags: AQS
---

* content
{:toc}

关键词：AQS CLH 

<code>AQS</code>指的是<code>java.util.concurrent.locks.AbstractQueuedSynchronizer</code>这个类，在阅读Jdk源码时，你会发现这个类是<code>java.util.concurrent</code>包的核心。
例如在<code>ReentrantLock</code>、<code>ReentrantReadWriteLock</code>、<code>CountDownLatch</code>等类中存在内部类<code>Sync</code>，都是用了AQS。如果想清楚整个Java的并发体系，这个类必读不可。




### AQS概括

先说说几个概念

- 独占锁&共享共享锁  
    - 独占锁  
    资源最多同时只能被一个线程占用
    - 共享共享锁  
    资源同时可以被多个线程占用

- 公平锁&非公平锁  
    - 公平锁  
    线程按照提交的顺序依次去获取资源，按照一定的优先级来，FIFO
    - 非公平锁  
    线程获取资源的顺序是无序的

AQS中的线程阻塞队列是基于<code>CLH lock queue</code>来实现的，后边会重点说明一下。

#### AQS的关键点有三个

- 提供变量<code>state</code>来维护状态
- 维护线程阻塞队列
- 阻塞和唤醒线程

#### AQS中的关键字段

- private volatile int state;  
同步状态，通过设置state来达到同步的效果.<code>该字段在不同的子类中，代表的意义是不同的</code>
- private transient volatile Node tail;  
线程等待队列的尾部，只有<code>enq</code>方法在向队列添加新线程时，才会修改该值
- private transient volatile Node head;  
线程等待队列的头部

下面是offset相关的变量，主要是为了提供CAS操作而设立的变量：

- private static final long stateOffset;  
同步状态的offset
- private static final long headOffset;  
等待队列头的offset
- private static final long tailOffset;  
等待队列尾的offset
- private static final long waitStatusOffset;  
当前节点的等待状态的offset
- private static final long nextOffset;  
当前节点的下一个节点offset

#### AQS中的关键方法
已经实现的方法如下

- public final void acquire(int arg)  
获取锁，一般会调用acquire(1)来获取。
- public final boolean release(int arg)  
释放锁，一般会调用release(1)来释放。

需要被子类实现的方法

- protected boolean tryAcquire(int arg)  
尝试获取独占锁,在获取前会检查同步状态是否允许获取独占锁。获取成功则返回true，返回false则把线程加入到等待队列。

- protected boolean tryRelease(int arg)  
尝试通过设置同步状态，来释放独占锁。

- protected int tryAcquireShared(int arg)  
尝试获取共享锁,在获取前会检查同步状态是否允许获取共享锁。获取成功则返回true，返回false则把线程加入到等待队列。

- protected boolean tryReleaseShared(int arg)  
尝试通过设置同步状态，来释放共享锁。

### CLH lock queue和自旋锁

#### CLH lock queue
CLH lock queue是一个存放线程的FIFO队列，队列中的每个线程都在等待它前一个线程释放锁，前面的线程释放了锁之后，该线程将会开始回解除锁并开始执行线程。

#### Spin Lock（自旋锁）
是线程通过循环来等待而不是睡眠。

#### java.util.concurrent.locks.AbstractQueuedSynchronizer.Node类

该类实现了<code>CLH lock queue</code>的一个变种，它的数据结构大致如下：
```
      +------+  prev +-----+       +-----+
 head |      | <---- |     | <---- |     |  tail
      +------+       +-----+       +-----+
```
>图中没有把next指向描述出来。

- 关键字段  
    - SHARED  
    共享模式的节点常量，供其方式使用
    - EXCLUSIVE  
    独占模式的节点常量，供其方式使用
    - waitStatus  
    当前节点/线程的等待状态,该变量会有下面几个值：CANCELLED/SIGNAL/CONDITION/PROPAGATE
    - CANCELLED  
    由于超时或者中断线程被中断，节点/线程的状态变为该值时，状态将不会发生变化，可以理解线程被中断之后，就停止了，状态自然不会发生什么变化了。
    - SIGNAL  
    表示当前节点以及成功执行,等待unpark
    - CONDITION  
    标识线程在condition queue中处于等待状态,等待某一个条件
    - PROPAGATE  
    后续结点会传播唤醒的操作，共享锁可执行,独占锁不可
    - prev  
    等待队列中，该节点的上一个节点
    - next  
    等待队列中，该节点的下一个节点
    - thread  
    当前的线程

- 关键方法  
其实没有什么重要的方法需要在这里提，<code>Node</code>这个类就是个用来实现双向队列的数据结构，它是双向队列中的一个节点。

### AQS详解
在“AQS概括”中已经简单的描述了一些关键的字段和方法，相信在了解了<code>CLH lock queue</code>、<code>自旋锁</code>，以及<code>LockSupport</code>类的设计和实现之后，再回过头来查看AQS内部的实现细节将会很轻松。
AQS重要实现了这几个重要的功能，CLH lock queue的维护、独占模式的获取/释放、共享模式的获取/释放。

#### CLH lock queue的维护
我们知道AQS维护了一个FIFO的队列，那肯定就有入队和出队的操作。

- CLH lock queue入队

```
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node); //把节点放入队列
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

private final boolean compareAndSetTail(Node expect, Node update) {
    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}

private final boolean compareAndSetHead(Node update) {
    return unsafe.compareAndSwapObject(this, headOffset, null, update);
}
```

当队列不为空时，节点添加到队列前，队列状态如下图所示：
![queue-notnull-before](http://7xriy2.com1.z0.glb.clouddn.com/queue-notnull-before.png)

节点加入队列的过程如下图所示：
![queue-notnull-after](http://7xriy2.com1.z0.glb.clouddn.com/queue-notnull-after.png)

当队列为空时，首次加入节点会进行一些初始化，具体的操作过程可以看下图：
![queue-null-init](http://7xriy2.com1.z0.glb.clouddn.com/queue-null-init.png)

- CLH lock queue出队  

```
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
} 
```


#### 独占模式的实现

##### 独占模式的获取

```
public final void acquire(int arg) {
    if (!tryAcquire(arg)/*尝试获取锁*/ &&
        acquireQueued(addWaiter(Node.EXCLUSIVE/*设置为独占锁状态,并把当前线程放入了队列中*/), arg)/*放入队列后,阻塞该线程*/)
        selfInterrupt(); //若park的原因是线程被interrupt掉了,则中断线程
}
```
这块的原理大概如下：

- 先通过<code>tryAcquire</code>来尝试获取锁，如果能获取到锁的话就返回；否则加入到队列，并阻塞线程；
- 线程退出阻塞的时候，通过<code>Thread.interrupted()</code>来判断线程是否是因为被中断才退出park的，若是的话，则中断当前线程。

具体我们可以查阅下面这几个方法的源码：

- acquireQueued  
线程节点之前以及加入到队列中了，该方法将要阻塞该线程，在退出阻塞的时候，返回该线程是否是被中断退出的。
```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;

        //自旋锁,double check
        //等待前面的节点释放锁
        for (;;) {
            final Node p = node.predecessor(); //获取该节点的上一个节点

            //如果前继节点就是head,现在就可以直接去尝试获取锁,如果没有其它线程的干扰,肯定是能够获取到的
            if (p == head/*前继节点就是head*/ && tryAcquire(arg)/*尝试获取锁*/) {

                //前继节点出队,当前的node设置为head
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }

            //去除前面被取消的(CANCELLED)的线程
            //若前继节点没有没被取消,则表示当前线程可以被park
            //park当前线程,park结束时检查是否被interrupt,若是则设置interrupted为true,跳出循环后中断线程
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        //若线程失败,则取消获取锁
        if (failed)
            cancelAcquire(node);
    }
}
```

- shouldParkAfterFailedAcquire  
去除前面被取消的(CANCELLED)的线程,若前继节点没有没被取消,则表示当前线程可以被park
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * 前继节点已经成功执行,等待释放锁,所以当前的线程可以安全的park。
         */
        return true;
    if (ws > 0) {
        /*
         * 前继节点的线程已经被取消,所以跳过已经被取消的节点,并一直往前跳过所有连续的CANCELLED节点
         *
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * 当前继节点的waitStatus非SIGNAL & 非CANCELLED,需要设置waitStatus的值为SIGNAL,并返回false,继续自旋
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

- parkAndCheckInterrupt  
park当前线程,并返回线程是否被中断
```
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);

    //阻塞结束时,调用Thread.interrupted()来检查结束阻塞的原因是什么
    return Thread.interrupted();
}
```

##### 独占模式的释放
释放独占模式，解除线程阻塞；该过程中会唤醒队列中后继节点的线程。

- release  
该方法一般会被子类调用，例如<code>java.util.concurrent.locks.ReentrantLock#unlock</code>这个方法，release的实现如下：
```
public final boolean release(int arg) {
    if (tryRelease(arg)/*该方法被子类实现*/) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h)/*唤醒后继节点的线程*/;
        return true;
    }
    return false;
}
```

- unparkSuccessor  
唤醒后继节点的线程   

```
private void unparkSuccessor(Node node) {
    /*
     * node节点以及执行完成,不用加锁了,所以可以把等待状态设置为0。
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * 唤醒node节点的后继节点线程。
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

#### 共享模式的实现

##### 共享模式的获取

- acquireShared  
```
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg)/*尝试获取共享模式,若无许可则进入等待*/ < 0)
        doAcquireShared(arg);
}
```

- doAcquireShared  
该方法和独占模式的<code>acquireQueued</code>方法很像，主要的区别就是在<code>setHeadAndPropagate</code>中，如果当前节点获取到了许可，且还有多余的许可，则继续让后继节点获取许可并唤醒他们，并一直往后传递下去。

```
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED)/*添加节点到队列*/;
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //获取到前继节点
            final Node p = node.predecessor();
            if (p == head) { //如果前继节点就是head节点的话
                int r = tryAcquireShared(arg);//尝试获取共享模式
                if (r >= 0) {
                    setHeadAndPropagate(node, r); //设置当前节点为head节点
                    p.next = null; // help GC
                    if (interrupted)    //判断退出xxxx的时候,是否是被interrupt掉的
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }

            //去除前面被取消的(CANCELLED)的线程
            //若前继节点没有没被取消,则表示当前线程可以被park
            //park当前线程,park结束时检查是否被interrupt,若是则设置interrupted为true,跳出循环后中断线程
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

- setHeadAndPropagate  

```
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) { //若队列里有线程节点
            int ws = h.waitStatus;

            //线程状态为SIGNAL的时候,才可以唤醒后继节点的线程
            if (ws == Node.SIGNAL) {

                //重置Node的状态为0,unpark该节点
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h); //unpark
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                //如果线程状态为0则更新状态为PROPAGATE，让后继节点传播唤醒操作,失败重试
                continue;                // loop on failed CAS
        }

        //head变化了,说明该节点被唤醒了,则继续唤醒后边的节点
        if (h == head)                   // loop if head changed
            break;
    }
}
```

- 共享模式的释放  
当tryReleaseShared返回true的时候,会把一个或多个线程从共享模式中唤醒。

```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)/*该方法由子类实现*/) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

#### 总结
AQS是<code>java.util.concurrent</code>中的核心，它的内部类<code>java.util.concurrent.locks.AbstractQueuedSynchronizer.Node</code>通过实现<code>CLH lock queue</code>的一个变种来轮询执行CAS操作来调用<code>Unsafe.park()</code>获取锁。
AQS理解了，下一篇会聊聊它的一个具体的实现<code>java.util.concurrent.locks.ReentrantLock</code>类，这样会更直观的展现AQS的精妙。
