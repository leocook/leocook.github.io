---
layout: post
comments: true
date: 2017-06-17
categories: java
tags: volatile、happen-before、内存屏障
---

* content
{:toc}

关键词：volatile、happen-before、内存屏障

在阅读Java并发编程相关的jdk源码中，有个关键字<code>volatile</code>会经常看到，在并发编程中偶尔也会用到该关键字，但是在使用它的过程中又很容易引起混淆。
<code>volatile</code>关键字一方面通过内存屏障禁止了指令重排，从而保证了有序性；另一方面通过内存屏障实现了可见性。

阅读本文前，你需要理解原子性、可见性、有序性这三个基本的并发特性。可以查看这里：[Java并发：原子性、可见性、有序性](http://www.leocook.org/2017/06/12/Java%E5%B9%B6%E5%8F%91-%E5%8E%9F%E5%AD%90%E6%80%A7-%E5%8F%AF%E8%A7%81%E6%80%A7-%E6%9C%89%E5%BA%8F%E6%80%A7/)

本文会先聊聊CPU从内存中读取值的硬件层面的过程，然后说一说volatile关键字是如何保证可见性、有序性的，以及相关的原理。最后还会列出几个示例来说明volatile不能解决的一些场景。



### CPU读取值的过程

CPU在读取数据进行计算的时候，cpu并不是直接读取内存中的值，而是先把内存中的值读到cpu的高速Cache中，然后cpu直接操作Cache中的值。cpu在完成计算后，把结果写到Cache中，然后再把Cache写回到内存中。具体的过程可以查看下图：

![CPU读取值的过程](http://7xriy2.com1.z0.glb.clouddn.com/volition1.png)

如果修改某个变量的值，大概会有下面几步操作：

- 把值从内存中读到CPU Cache中
- CPU读取Cache中的值执行操作，并把修改后的值写入到CPU Cache中
- 数据从CPU Cache中刷到内存中

所以，只有上面三个步骤在执行的过程中不被其它操作干扰时，这个修改操作才会正常完成。

### volatile变量具有可见性
简言之，被volatile关键字修饰的变量在修改后，将会强制被刷到内存中，且该变量在其它CPU中的Cache将会失效，从而保证线程在修改变量值后，其它线程能立马读到。

下面详细说说使用与不使用volatile关键字的差异：

- 未使用volatile关键字

```
int i=0;//共享变量

//线程1的操作
i=i+1

//线程2的操作
j=i
```

我们假设先执行的线程1，然后执行线程2，一般会想到j=1，但事实上却不一定。

在创建线程的时候，内存结构如下：

![](http://7xriy2.com1.z0.glb.clouddn.com/%E5%B9%B6%E5%8F%911.png)

当执行一次线程1之后，cpu缓存中i变为了1，并把cache中的1刷到了内存中，内存结构如下：

![](http://7xriy2.com1.z0.glb.clouddn.com/%E5%B9%B6%E5%8F%913.png)

由于在线程启动的时候已经把i的值读到了cpu的cache中，所以在执行<code>j=i</code>的时候，给j赋的值是0，而不是1。
很显然，在线程1修改了i的值之后，线程2并没有读到修改后的i值，可以理解i的读取操作不具备可见性。

- 使用volatile关键字

在Java中，我们可以使用<code>volatile</code>关键字来保证变量的可见性，**被volatile修饰后的变量在被修改后，会直接把缓存刷入内存，从而保证下次读取能够读到最新的值**。如下分析：

第一步：线程1直接读取内存的i值

![](http://7xriy2.com1.z0.glb.clouddn.com/vvv1.png)


第二步：执行<code>i=i+1</code>，并把结果刷回内存，CPU2中的缓存失效，然后CPU2更新缓存，把i值赋值给j，并写回内存

![](http://7xriy2.com1.z0.glb.clouddn.com/vvv2.png)

这样就保证了i的操作是具备可见性的了，所以线程1修改了i之后，线程2能立刻读到修改后的值。


### volatile变量不具有原子性
volatile可以理解为一个轻量级的synchronized，但是volatile变量不具备原子性。synchronized对比volatile实现的是锁，锁提供了两个重要的特性：互斥（mutual exclusion） 和可见性（visibility）。正是互斥保证了操作的原子性。

那么什么时候使用volatile，什么时候使用synchronized呢？

- 当变量只需要具备可见性的时候使用volatile，例如：对变量的写操作不依赖于该变量当前值；
- 当变量需要同时具备原子性和可见性的时候，就使用synchronized。
- 在使用volatile和synchronized都可以的时候，优先使用volatile，因为volatile的同步机制性能要高于锁的性能。

### volatile变量一定程度上具有有序性

在使用了volatile关键字之后，将会禁用指令重排，从而保证有序性。
当程序执行到volatile变量的读取或写操作时，将会保证该操作前面的语句都已经执行完成且结果对后边代码具有可见性；该操作后边的代码都没有执行。

我们可以查看下面这个代码块：
```
x = 2;        //语句1
y = 0;        //语句2
flag = true;  //语句3
x = 4;         //语句4
y = -1;       //语句5
```

- flag变量没有被volatile关键字修饰时   
由于指令重排的原因，我们可以得到下面依赖关系：
![](http://7xriy2.com1.z0.glb.clouddn.com/volition11.png)  
我们可以看出：
语句1执行完之后才可以执行语句4；
语句2执行完之后才可以执行语句5；
其它执行的顺序不一定。

- flag变量被volatile关键字修饰后  
由于volatile禁用了指令重排，我们可以得到下面依赖关系：  
![](http://7xriy2.com1.z0.glb.clouddn.com/volition12.png)  
我们可以看出：
语句1和语句2的顺序不一定；
语句1和语句2都执行完之后执行语句3；
语句3执行完之后才执行语句4和语句5；
语句4和语句5谁先执行不一定。

### volatile关键字的实现原理

#### happen-before
如果A happen-before B，那么A的所有操作完成后并产生结果才会执行B操作，可以说A所做的任何操作对B都是可见的。happen-before大概有下面几种场景：

- 1.程序次序规则：在一个单独的线程中，按照程序代码的执行流顺序，（时间上）先执行的操作happen—before（时间上）后执行的操作；
- 2.管理锁定规则：一个unlock操作happen—before后面（时间上的先后顺序，下同）对同一个锁的lock操作；
- 3.volatile变量规则：对一个volatile变量的写操作happen—before后面对该变量的读操作。
- 4.线程启动规则：Thread对象的start()方法happen—before此线程的每一个动作；
- 5.线程终止规则：线程的所有操作都happen—before对此线程的终止检测，可以通过调用Thread.join()方法、获取Thread.isAlive()的返回值等手段检测到线程已经终止执行；
- 6.线程中断规则：对线程interrupt()方法的调用happen—before发生于被中断线程的代码检测到中断时事件的发生；
- 7.对象终结规则：一个对象的初始化完成（构造函数执行结束）happen—before它的finalize（）方法的开始；
- 8.传递性：如果操作A happen—before操作B，操作B happen—before操作C，那么可以得出A happen—before操作C。

其实我们这里主要查看的是第3条，即volatile变量保证的有序性。在代码重排中，主要分为编译器重排和指令重排，为了实现volatile变量的内存语义，JMM会限制这两类重排，下面是JMM针对volatile变量所规定的重排规则表：

| 1st operation | 2st operation | 2st operation | 2st operation |
| --- | --- | --- | --- |
|  | Normal Load<br />Normal Store | Volatile Load | Volatile Store |
| Normal Load |  |  | No |
| Normal Store |  |  | No |
| Volatile Load | No | No | No |
| Volatile Store |  | No | No |


观察上述表格，可以得知Volation变量的happen—before原则：

- 所有的Volatile变量读都happen—before改变量的其它操作
- Volation变量的所有操作都happen—before该变量的写操作
- Volation变量的写操作happen—before该变量的读操作

#### 内存屏障

内存屏障也称之为内存栅栏，是一组处理器指令，用于实现对内存操作顺序限制。



| 1st operation | 2st operation | 2st operation | 2st operation | 2st operation |
| --- | --- | --- | --- | --- |
|  | Normal Load | Normal Store | Volatile Load | Volatile Store |
| Normal Load |  |  |  | LoadStore |
| Normal Store |  |  |  | StoreStore |
| Volatile Load | LoadLoad | LoadStore | LoadLoad | LoadStore |
| Volatile Store |  |  | StoreLoad | StoreStore |



JMM中共有这4种屏障：

- LoadLoad屏障  
执行顺序：Load1—>Loadload—>Load2  
Load2以及后序的Load指令在加载数据之前，都能访问到Load1加载的数据。

- LoadStore屏障    
执行顺序： Load1—>LoadStore—>Store2  
Store2以及后序的Store指令在存储数据之前，都能访问到Load1加载的数据。

- StoreStore屏障  
执行顺序：Store1—>StoreStore—>Store2  
Store2以及后序的Store指令在存储数据之前，都能访问到Store1操作所存储的数据。

- StoreLoad屏障  
执行顺序: Store1—> StoreLoad—>Load2  
Load2以及后序的Load指令在加载数据之前，都可以访问到Store1操作所存储的数据。

#### volatile关键字原理
volatile实际上就是使用内存屏障的来实现可见性和有序性的：
- 有序性：它会确保指令重排时，使用内存屏障保证volatile变量操作前的操作都已经完成了，且在volatile变量操作完成后，才会执行后边的代码
- 可见性：CPU每次在修改volatile变量值之后，它会强制把数据从缓存刷到内存中去，才算本次操作完成
- 可见性：volatile变量值修改后，它会导致其他CPU中的缓存失效

### volatile的几个使用场景

- 状态标记量  

```    
//使用volatile来保证标记变量的可见性
volatile boolean flag = false;
 
while(!flag){
    doSomething();
}
 
public void setFlag() {
    flag = true;
}
```

```
//使用volatile来保证标有序性
volatile boolean inited = false;
//线程1:
context = loadContext();  
inited = true;            
 
//线程2:
while(!inited ){
sleep()
}
doSomethingwithconfig(context);
```

- 双重检查锁  
```
//除了饿汉模式，这种写法创建单例效率最高
class Singleton{
    private volatile static Singleton instance = null;
     
    private Singleton() {
         
    }
     
    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    //一定会把数据刷到内存中
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

### volatile关键字陷阱
Java是一门支持多线程的语言，为了解决线程的并发问题，使用了<code>同步块</code>和<code>volatile关键字</code>机制。
synchronized关键字是同时具备原子性、可见性和有序性；而volatile关键字只具备可见性和有序性，并不具备原子性，问题就出在这里，下面我们举例代码来看看。

- 代码一  
```
public class Counter {

    public static Integer count = 0;
    static CountDownLatch countDownLatch = null;

    public static void inc() {

        //这里延迟1毫秒，使得结果明显
        try {
            Thread.sleep(1);
        } catch (InterruptedException e) {
        }

        count++;
        countDownLatch.countDown();
    }

    public static void main(String[] args) throws InterruptedException {
        int threadCount = 1000;

        countDownLatch = new CountDownLatch(threadCount);
        //同时启动1000个线程，去进行i++计算，看看实际结果

        for (int i = 0; i < threadCount; i++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    Counter.inc();
                }});
            t.start();
        }

        countDownLatch.await();

        //这里每次运行的值都有可能不同,可能为1000
        System.out.println("运行结果:Counter.count=" + Counter.count);
    }
}
```

我这边计算的结果是978，每次运行的结果应该都是不一样的。这段代码应该很好理解，上面代码是很常见的线程不安全实例。

- 代码二  
在上面代码的基础上，使用volatile来修饰count：
```
public volatile static int count = 0;
```

当执行完之后，我们发现结果仍然不是1000，原因是volatile无法保证变量的操作是原子的，只能保证变量的操作是具备可见性的。

- 代码三  
在代码一的基础上，修改了inc方法,给<code>count++;</code>添加同步代码：
```
public synchronized static void inc() {

    //这里延迟1毫秒，使得结果明显
    try {
        Thread.sleep(1);
    } catch (InterruptedException e) {
    }

    count++;
    
    countDownLatch.countDown();
}
```

查看上面的代码，我们可以发现volatile关键字是不具备原子性的。我们再使用的时候，需要避免这个坑。

当我们需要使用具备原子操作的基础类型时，我们除了使用同步代码块，还可以使用<code>java.util.concurrent.atomic</code>包下的原子类型,例如代码一可以这样修改：
```
public class Counter {

    public static AtomicInteger count = new AtomicInteger(0);
    static CountDownLatch countDownLatch = null;

    public static void inc() {

        //这里延迟1毫秒，使得结果明显
        try {
            Thread.sleep(1);
        } catch (InterruptedException e) {
        }

        count.incrementAndGet();
        countDownLatch.countDown();
    }

    public static void main(String[] args) throws InterruptedException {
        int threadCount = 1000;

        countDownLatch = new CountDownLatch(threadCount);
        //同时启动1000个线程，去进行i++计算，看看实际结果

        for (int i = 0; i < threadCount; i++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    Counter.inc();
                }});
            t.start();
        }

        countDownLatch.await();

        //这里每次运行的值都有可能不同,可能为1000
        System.out.println("运行结果:Counter.count=" + Counter.count.get());
    }
}
```

- 总结

一句话总结下，volatile关键字通过内存屏障来保证了变量的可见性和有序性。



