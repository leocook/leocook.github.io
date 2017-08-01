---
layout: post
comments: true
date: 2017-07-16
categories: java
tags: ReentrantLock synchronized
---

* content
{:toc}


ReentrantLock是基于AQS设计的可重入锁，synchronized是基于对象监视器实现的可重入锁。使用了它们后，代码都会具有原子性（atomicity）和 可见性（visibility）。

可重入锁也被称为递归锁，指同一个线程内，外层代码锁未被释放时，内层代码也可以获取到锁，递归就是一种很常见的场景。下面就是可重入锁的一种使用场景：

```
public class Demo{
    Lock lock = new Lock();
    public void outer(){
        lock.lock();
        inner();
        lock.unlock();
    }
    public void inner(){
        lock.lock();
        //do something
        lock.unlock();
    }
}
```




ReentrantLock和synchronized在并发编程中，有着相同的语义，但是它们实现的原理存在着较大的差异，在设计的思想上更是有着很多不同之处。

### 实现原理上的区别

#### ReentrantLock
ReentrantLock是基于AQS实现的锁，我们知道AQS是调用了LockSupport类的park和unpark方法来实现阻塞和唤醒的。
如果之前看过[Java并发(四)：locksupport](http://www.leocook.org/2017/07/08/Java%E5%B9%B6%E5%8F%91(%E5%9B%9B)-LockSupport/)这篇文章的话，应该很好理解：ReentrantLock是通过一个<code>_counter</code>变量来标记阻塞状态的。  
阅读ReentrantLock源码可以发现，AQS中的state字段在ReentrantLock中也被用作记录锁的重入次数（也就是同一个线程同时获得锁的次数），当state的值为0时，则表示资源没有被其它锁占用。

#### synchronized
在JVM中，对象（this）或者类（SomeClass.class）都会被分配一个监视器(Monitor)。Monitor可以理解为一种同步工具，也可以理解为伴随着对象实例的一种JVM内部对象。  
synchronized关键字是使用了对象监视器(Monitor)来标识资源是否被锁占用，我们将下边代码反进行反编译：

```
/**
 * 同步代码块
 */
public void method1(){
    synchronized (this){
        System.out.println("method1 start");
    }
}

/**
 * 同步方法
 */
public synchronized void method2(){
    System.out.println("method2 start");
}
```

反编译后，我们可以看到字节码：

```
public void method1();
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: getstatic     #2    // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3    // String method1 start
         9: invokevirtual #4    // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit
        20: aload_2
        21: athrow
        22: return
      Exception table:
         from    to  target type
             4    14    17   any
            17    20    17   any
      LineNumberTable:
        line 12: 0
        line 13: 4
        line 14: 12
        line 15: 22
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
               0      23     0  this   Lcom/xxx/bi/ThreadDemo;
      StackMapTable: number_of_entries = 2
           frame_type = 255 /* full_frame */
          offset_delta = 17
          locals = [ class com/xxx/bi/ThreadDemo, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
           frame_type = 250 /* chop */
          offset_delta = 4


  public synchronized void method2();
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2    // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #5    // String method2 start
         5: invokevirtual #4    // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 19: 0
        line 20: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
               0       9     0  this   Lcom/xxx/bi/ThreadDemo;
```
阅读上面字节码，我们可以从两个方面来看：

- **synchronized同步代码块(method1)**  
我们能看到在进入同步代码块时，会执行<code>monitorenter</code>(占用监视器)，退出代码块时会执行<code>monitorexit</code>(退出监视器)。  
    **关于monitorenter**
    - 若monitor的entry count为0，则该线程进入monitor，然后将entry count设置为1，该线程成为了monitor的所有者；
    - 若monitor的entry count不为0，且该进程已经占用了monitor，则线程只是重新进入该代码块，且entry count加1；
    - 若monitor的entry count不为0，且被其它进程已经占用了monitor，则该线程进入阻塞状态，直到monitor的entry count为0时，再重新尝试成为monitor的所有者；

    **关于monitorexit**
    - 指令执行的时候，monitor的entry count减1，当减1后为0的时候，释放monitor不在占有。其它被这个monitor阻塞的线程开始尝试获取该monitor的所有权。
    - object的wait/notify方法是依赖monitor的，所以只能在同步代码块或者方法中才能调用wait/notify等方法，否则会抛出异常。
执行monitorexit必须是某个已经占用了monitor的线程的对象实例。

- **synchronized同步方法(method2)**  
我们可以看出该方法的access_flags中存在<code>ACC_SYNCHRONIZED</code>标签，该method加上了该标签之后，可以理解进入方法的时候，会做和monitorenter一样的事情，当退出方法的时候将会作出和monitorexit一样的事情。

关于synchronized，有下面几个总结：

>1.使用同步代码块后，JVM会用monitorenter和monitorexit指令完成同步。  
2.使用同步方法后，JVM会使用方法的访问修饰符ACC_SYNCHRONIZED来完成同步。  
3.synchronized只能持有一个对象监视器。  
4.synchronized强制所有锁的获取和释放都在一个代码块中。  
5.synchronized对锁的释放是隐式的。运行超出代码块时，自动释放。  
6.synchronized在释放了monitor之后，随机选取新的线程获取monitor。当线程数多的时候，可能会导致部分线程一直获取不到锁


### API使用上的区别

何时获取锁、何时释放锁，<code>ReentrantLock</code>相对来说更自由，可以由开发者自己来决定，且支持多个条件变量。但<code>synchronized</code>却不行，只能被动的在synchronized代码范围结束时释放锁。

<code>ReentrantLock</code>还支持公平锁和非公平锁，在一些需要保证线程FIFO获取锁的场景下，可以使用ReentrantLock的公平锁，synchronized是没有这个特性的。  
但是为了避免出现死锁，<code>ReentrantLock</code>的锁必须在finally中释放，例如下面代码：

```
Lock lock = new ReentrantLock();
lock.lock();
try { 
  // do something
}
finally {
  lock.unlock(); 
}
```
所以在使用<code>ReentrantLock</code>时需要慎重！

### 锁的范围
关于锁的范围我是这么理解的，如果某个类的对象中存在一把锁可以被该类的所有对象访问，那么这个锁就是类级别的锁；如果某个类的对象中存在一把锁只能被该对象访问，那么这个锁就是对象级别的锁。其实这样解释不是很严谨，下面具体看例子。

#### ReentrantLock
当ReentrantLock对象是静态的时候就是类级别的锁，否则锁就是对象级别的锁。

```
//类级别的锁
static Lock lock1 = new ReentrantLock();

//对象级别的锁
Lock lock2 = new ReentrantLock();
```

#### synchronized
当synchronized同步的变量或者方法是静态的时候，锁就是类级别的锁，否则就是对象级别的锁

- 类级别的锁  
这里使用的是类监视器

```
//变量
static int count = 0;
synchronized(count){
    //...
}

//方法
public synchronized static void method(){
    //...
}

//类加载器
synchronized(Demo2.class){
    //...
}
//或者
synchronized(this.getClass()){
    //...
}


```

- 对象级别的锁  

```
//变量
int count = 0;
synchronized(count){
    //...
}

//方法
public synchronized void method(){
    //...
}

synchronized(this){
    ...
}
```

## ReentrantLock源码解析
如果阅读过看AQS的相关源码，查看ReentrantLock类源码将会很轻松的。阅读ReentrantLock源码时主要就是查看它<code>三个静态内部类的实现</code>，以及<code>公平锁</code>和<code>非公平锁</code>的实现差异。  
>在查看源码的时候需要注意，在实现ReentrantLock的时候，AbstractQueuedSynchronizer类中的<code>state</code>字段的作用是记录重入锁的重入次数，每次获取锁的时候state字段值加一，释放锁的时候state值减一。

### 内部类
主要有这三个静态内部类<code>java.util.concurrent.locks.ReentrantLock.Sync</code>、<code>java.util.concurrent.locks.ReentrantLock.NonfairSync</code>以及<code>java.util.concurrent.locks.ReentrantLock.FairSync</code>。Sync类是另外两个的父类，NonfairSync类实现的是非公平锁，FairSync类实现的是公平锁。

#### Sync
Sync类是一个抽象类，它主要声明了<code>lock抽象方法</code>,实现了获取非公平锁的方法<code>nonfairTryAcquire</code>，以及释放锁的方法<code>tryRelease</code>。

- final boolean nonfairTryAcquire(int acquires)  
尝试获取非公平锁。

```
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        //重入次数为0, 锁未被占用时,直接占用就可以了
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {//锁已经被当前线程占用了
        int nextc = c + acquires;   //重入次数加1
        if (nextc < 0) // overflow  重入次数超过int范围的时候,报错
            throw new Error("Maximum lock count exceeded");
        setState(nextc);    //更新重入次数
        return true;
    }

    //锁已经被其它线程占用了
    return false;
}
```

- protected final boolean tryRelease(int releases)  
尝试释放锁，当重入计数器state值变为0后，表示以及没有锁的占用了。

```
protected final boolean tryRelease(int releases) {
    //重入次数减一
    int c = getState() - releases;

    //当前的线程不可以释放其它线程的锁
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();

    boolean free = false;
    if (c == 0) { //当state为0时,说明锁已经完全释放了
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

#### NonfairSync
非公平锁的实现，主要是实现了lock方法。

```
//获取锁
final void lock() {
    //判断一下锁是否已经被获取:
    //若锁已经被获取,则进入阻塞;否则直接获取
    if (compareAndSetState(0, 1)) //CAS判断锁是否已经被获取
        //符合条件
        setExclusiveOwnerThread(Thread.currentThread()); //直接锁定
    else
        //锁已经被获取,调用AQS尝试获取锁以及进入阻塞
        acquire(1);
}

//尝试获取锁，被AQS调用
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```


#### FairSync
公平锁的实现

```
/**
 * 由于需要判断是否公平,所以和NonfairSync#lock()的实现稍有不同,并没有在AQS的state值为0时,立马获取到锁。
 */
final void lock() {
    acquire(1);
}

/**
 * 除了多调用了hasQueuedPredecessors方法,其它和和nonfairTryAcquire几乎一样
 */
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState(); //获取到重入次数
    if (c == 0) {
        //未重入过
        if (!hasQueuedPredecessors()/*查看是否有比当前线程等待更久的线程,有就返回true(不通过),没有就返回false(通过),和nonfairTryAcquire相比,只多出了这一块*/ &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
对比<code>ReentrantLock.Sync#nonfairTryAcquire</code>方法的实现，我可以看出公平锁中多调用了方法<code>java.util.concurrent.locks.AbstractQueuedSynchronizer#hasQueuedPredecessors</code>。

```
/**
 * 查看是否有比当前线程等待更久的线程,有就返回true,没有就返回false.<br />
 *
 * 若存在比当前线程等待还要久的线程需要满足下面2个条件中的任意一个即可:
 * 1. 线程等待队列中存在等待的线程,且线程不是队列中的第一个线程;  
 * 2. 线程等待队列中存在等待的线程,其它线程正在初始化线程队列,已经修改好了tail指针，但head的next指针还没有修改好，导致head.next为空，可以查看方法java.util.concurrent.locks.AbstractQueuedSynchronizer#enq
 *
 */
public final boolean hasQueuedPredecessors() {
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

这段代码有点饶人，其实简单理解，就是判断线程队列中，当前线程前边有没有其它线程，是的话返回true，没有的话就返回false。

### 关键方法
平时用的比较多的方法就是<code>lock</code>和<code>unlock</code>。截止到jdk1.7时<code>ReentrantLock</code>类的很多方法都是对<code>Sync</code>类的二次封装。

```
//创建非公平锁
Lock lock1 = new ReentrantLock();
Lock lock2 = new ReentrantLock(false);

//创建公平锁
Lock lock3 = new ReentrantLock(true);
```

## 总结
ReentrantLock是一个基于AQS实现的高性能的可重入锁，相比synchronized来说使用时更灵活、且效率更高。  
如果对线程调用的顺序不是很关系，可以使用非公平锁；否则就使用公平锁。非公平锁的性能是优于公平锁的。  
相比synchronized来说，ReentrantLock太过灵活，新手使用很容易出现问题，unlock的代码务必写到<code>finally</code>代码块中。
