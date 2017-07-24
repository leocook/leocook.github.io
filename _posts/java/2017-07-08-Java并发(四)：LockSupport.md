---
layout: post
comments: true
date: 2017-07-08
categories: java
tags: LockSupport
---

* content
{:toc}

关键词：LockSupport

前面我们讨论了<code>sun.misc.Unsafe</code>类，该类提供了面向操作系统直接操作内存和CPU的方法，例如分配内心和阻塞线程等等。但是该类在使用时是不安全的，所以jdk在不同的场景下对它做了不同的包装。

<code>java.util.concurrent.locks.LockSupport</code>类就是对<code>sun.misc.Unsafe</code>类进行了一些封装，主要提供一些锁的基础操作。





LockSupport阻塞线程的机制与<code>Object</code>的wait和notify是不一样的：

- 调用API层面的区别  
<code>LockSupport</code>的park和unpark可以用“线程”作为该方法的参数，语义更合乎逻辑。
<code>Object</code>的wait和notify是由<code>监视器对象</code>来调用的，对线程来说，它的阻塞和唤醒是被动的，不能准确的控制某个指定的线程，要么随机唤醒（notify）、要么唤醒全部（notifyAll）。
- 实现原理的区别  
<code>Object</code>的wait和notify以及<code>synchronized</code>都是通过占用和释放该<code>对象的监视器</code>来实现锁的获取和释放。
<code>LockSupport</code>不使用对象的监视器，每次执行<code>park</code>时会消耗1个<code>许可</code>，每次执行<code>unpark</code>时会获得1个<code>许可</code>。

>如果查看Unsafe的C++[源码](http://hg.openjdk.java.net/jdk7/jdk7/hotspot/file/81d815b05abb/src/os/linux/vm/os_linux.cpp)会发现，这个<code>许可</code>，其实就是一个<code>_counter</code>变量。
当执行<code>park</code>的时候，若<code>_counter</code>值大于0则立马返回并把<code>_counter</code>的值设置为0，线程不会阻塞；若<code>_counter</code>值等于0，则阻塞当前线程。可以理解这个过程将会消耗一个许可，若没有许可被消耗，则阻塞。
当执行<code>unpark</code>的时候，将会把<code>_counter</code>的值设置为1。可以理解这个过程是是给线程添加一个许可，且多次调用也只会添加一个许可。
<code>_counter</code>为1的时候表示许可可用，为0的时候表示许可不可用。如果能够很好的理解这个“许可”的设计，在查看LockSupport类源码的时候，将会轻松很多。

#### LockSupport类的核心字段

```
//unsafe，用于CAS操作
private static final Unsafe unsafe = Unsafe.getUnsafe();

//Thread中parkBlocker的内存偏移量
private static final long parkBlockerOffset;
```

在<code>java.lang.Thread</code>类中有个字段<code>parkBlocker</code>用来存放该线程阻塞时是被哪个对象阻塞的。

#### LockSupport类的核心方法

- public static void park()  
如果许可可用，则使用该许可，并立刻返回；否则阻塞当前线程。

- public static void park(Object blocker)  
如果许可可用，则使用该许可，并立刻返回；否则阻塞当前线程，并告诉线程是谁阻塞了它。

- public static void parkNanos(long nanos)  
如果许可可用，则使用该许可，并立刻返回；否则阻塞当前线程nanos纳秒。

- public static void parkNanos(Object blocker, long nanos)  
如果许可可用，则使用该许可，并立刻返回；否则阻塞当前线程nanos纳秒，并告诉线程是谁阻塞了它。

- public static void parkUntil(long deadline)  
如果许可可用，则使用该许可，并立刻返回；否则阻塞当前线程直到时间deadline。与parkNanos相比，这里的时间是绝对时间戳。

- public static void parkUntil(Object blocker, long deadline)  
如果许可可用，则使用该许可，并立刻返回；否则阻塞当前线程直到时间deadline，并告诉线程是谁阻塞了它。

- public static void unpark(Thread thread)  
若许可不可用，则使许可可用。若线程因为调用了park而阻塞，则它将解除阻塞状态。否则保证下一次调用park不会受阻。

- private static void setBlocker(Thread t, Object arg)  
给线程t设置阻塞对象，告诉t是谁阻塞了它。

- public static Object getBlocker(Thread t)  
获取是谁阻塞了线程t

#### Thread.interrupt()方法

Thread.interrupt()方法不会中断一个正在运行的线程。当线程被Object.wait、Thread.join和Thread.sleep三种方法阻塞时，若调用了Thread.interrupt()方法，线程将退出阻塞状态，并会抛出一个<code>InterruptedException</code>中断异常。

><code>LockSupport.park()</code>也能够响应中断信号，但是它不会抛出<code>InterruptedException</code>中断异常。

- Thread.interrupted() & Thread.isInterrupted()方法  
测试线程是否已经中断。前者是静态方法，后者不是。当我们需要停止一个线程的时候，一般有两种方式：
    - 通过共享变量；
    - 通过调用线程的<code>interrupt</code>方法。

前者在线程阻塞的时候不能够被中断，只有当线程执行的时候才可以;而后者只能在线程处于阻塞的时候才能够被中断，线程执行时不可以中断。

- interrupted status  
<code>interrupted status</code>是线程的中断状态，被JVM的C++代码维护着。
    - 在调用Thread.join和Thread.sleep之后，将会设置interrupted status；
    - 当退出阻塞、抛出<code>InterruptedException</code>异常的时候，interrupted status将会被清除；
    - 调用<code>public static boolean interrupted()</code>方法后，会清除interrupted status

#### 总结

<code>java.util.concurrent.locks.LockSupport</code>类是对<code>sun.misc.Unsafe</code>类的二次封装，主要提供了一些线程阻塞的工具。
该类在AQS中被大量的运用，在阅读AQS的源码时，需要对该类有所了解。