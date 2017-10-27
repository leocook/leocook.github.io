---
layout: post
comments: true
date: 2017-06-24
categories: java
tags: CAS Unsafe
---

* content
{:toc}

关键词：CAS、Unsafe

CAS是JSR-166和核心思想，Java中的CAS思想被C/C++实现，并被sun.misc.Unsafe类包装，供Java调用。查看其对应的C/C++源码可以[点击这里](https://github.com/aeste/gcc/blob/master/libjava/sun/misc/natUnsafe.cc)。  
Java中的非阻塞锁是基于AQS实现的，而AQS的设计就是建立在CAS和Unsafe类上的，所以学习本文还是很有必要性的。

在学习CAS、sun.misc.Unsafe类之前，我们需要知道一些基础的概念：

- [Java并发：原子性、可见性、有序性](http://www.leocook.org/2017/06/12/Java%E5%B9%B6%E5%8F%91-%E5%8E%9F%E5%AD%90%E6%80%A7-%E5%8F%AF%E8%A7%81%E6%80%A7-%E6%9C%89%E5%BA%8F%E6%80%A7/)
- [Java并发：volatile关键字](http://www.leocook.org/2017/06/17/Java%E5%B9%B6%E5%8F%91-volatile%E5%85%B3%E9%94%AE%E5%AD%97/)




## CAS
CAS是<code>Compare and Swap</code>的缩写，即比较并转换。在设计并发算法的时候会用到的技术，JSR-166特性是完全建立在CAS的基础上的，可见其重要性。
维基百科对CAS的解释可以看这里：[CAS比较并交换](https://zh.wikipedia.org/wiki/%E6%AF%94%E8%BE%83%E5%B9%B6%E4%BA%A4%E6%8D%A2)

Java就是通过Unsafe类的compareAndSwap系列方法实现的CAS，当前的绝大多是CPU都是支持CAS的，不同厂商的CPU的CAS指令可能是不同的。
#### CAS原理
CAS有三个操作数：内存位置V，预期值A和新值B（将要被修改成的值）。
在修改值的时候，若内存位置V存的值和预期值A相等，那么就把内存位置的V的值修改为B，返回true；否则，什么都不做，并返回false。
在Java的实现中，V可以是一个存储A地址的long整数，A是一个使用了volatile修饰的基础数据类型或者对象，B的类型和A的类型一致。

#### AtomicInteger示例
java.util.concurrent.atomicAtomicInteger是Jdk提供的一个类，如果你需要一个读写有原子性的整数类型，使用它就对了！
我们可以趴一下AtomicInteger的源码，可以观察到下面这个方法:

```
/**
 * 在当前值上原子性的自增1
 */
public final int getAndIncrement() {
    for (;;) {
        int current = get();
        int next = current + 1;
        if (compareAndSet(current, next))
            return current;
    }
}

public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```
我们能够看到，为了保证操作的原子性，调用了<code>compareAndSet</code>方法，而<code>compareAndSet</code>方法又调用了Unsafe的<code>compareAndSwapInt</code>方法。

#### CAS的ABA问题
维基百科上的说明是如下：

- 1.进程P1读取了一个数值A
- 2.P1被挂起(时间片耗尽、中断等)，进程P2开始执行
- 3.P2修改数值A为数值B，然后又修改回A
- 4.P1被唤醒，比较后发现数值A没有变化，程序继续执行。
对于线程P1来说，数值一直是A未变化过，但实际上数值发生过变化的。关于这个维基百科里说的很清楚。

## sun.misc.Unsafe

我们在日常开发的时候，Java是无法直接做操作系统级别的访问，如果想访问操作系统，我们可以使用C/C++来开发，然后使用JNI或者JNA来调用C/C++的库。
JVM中存在<code>sun.misc.Unsafe</code>这样的一个类，该类中提供了一系列的底层方法，这些方法可以直接操作操作系统的内存等。该类在设计的时候，默认是不让一般的开发人员不可以使用，只有授信代码才可以使用。

#### Unsafe类是单例的
从下面的代码中，我们可以看出Unsafe是单例的。且只能通过类加载器来获取，不能直接使用new来创建。

```
private static final Unsafe theUnsafe;

private Unsafe() {
}

@CallerSensitive
public static Unsafe getUnsafe() {
    Class var0 = Reflection.getCallerClass();
    if(var0.getClassLoader() != null) {
        throw new SecurityException("Unsafe");
    } else {
        return theUnsafe;
    }
}
```

#### 创建Unsafe对象

如果你直接调用getUnsafe方法来创建Unsafe对象，那么在编译的时候你将会得到警告提示，例如对于下面这段代码：

```
import sun.misc.Unsafe;

public class ThreadDemo{
    public static void main(String[] args){
        Unsafe unsafe = Unsafe.getUnsafe();
    }
}
```

我们编译时将会看到如下的警告信息：

```
ThreadDemo.java:1: 警告: Unsafe是内部专用 API, 可能会在未来发行版中删除
import sun.misc.Unsafe;
               ^
ThreadDemo.java:6: 警告: Unsafe是内部专用 API, 可能会在未来发行版中删除
Unsafe unsafe = Unsafe.getUnsafe();
^
ThreadDemo.java:6: 警告: Unsafe是内部专用 API, 可能会在未来发行版中删除
Unsafe unsafe = Unsafe.getUnsafe();
                ^
3 个警告
```

但是我可以对代码进行授信处理，Java是通过内加载器是否为根类加载器判断是否授信的，我们可以使用JVM参数bootclasspath：

```
java -Xbootclasspath:/usr/java/jdk1.7.0/jre/lib/rt.jar:. com.mishadoff.magic.UnsafeClient
```

**Jdk不建议开发者直接创建Unsafe对象**，如果我们必须要创建，那么我们可以使用**反射**的方式来创建：

```
import java.lang.reflect.Field;
import sun.misc.Unsafe;
import sun.reflect.Reflection;

Field field = Unsafe.class.getDeclaredField("theUnsafe");
field.setAccessible(true);
Unsafe unsafe = (Unsafe) field.get(null);
System.out.println(unsafe);
```

#### Unsafe类的方法介绍
Unsafe类中的方法有很多，其实我们需要关注只有这么几个。

- 内存分配  
allocateMemory：分配内存（非堆内存）
reallocateMemory：重新分配内存
freeMemory：释放内存

- 线程操作  
park：锁定当前的线程
unpark：解锁指定的线程

- CAS操作  
public final native boolean compareAndSwapXXX(Object o,long offset,K expected,K x)
例如：compareAndSwapInt、compareAndSwapLong、compareAndSwapObject等等。
该方法也就是Java中的CAS的实现，这个方法会比较expected的值和内存地址在offset位置的值是否一样，如果一样则会更新expected的值为x，并返回true，否则返回false。

还有其它的相关方法可以看下面：

- public native int addressSize()  
本地指针所占用的存储大小，值为4或8.

- public native int pageSize()  
返回内存页信息

- public native Object allocateInstance(Class cls)  
分配一个指定的对象，但是不执行任何构造方法。

- public native int arrayBaseOffset(Class arrayClass)  
返回数组的内存起始位置

- copyMemory  
把某段内存块中的数据copy到另外一段内存块中。

- defineAnonymousClass  
定义一个匿名类。

- defineClass  
让JVM定义一个类，不进行安全性检查。

- ensureClassInitialized  
确定类已经被初始化了。这个经常在访问类的静态成员变量时会结合访问。

- fieldOffset  
返回字段在对象中的内存offset

- freeMemory  
释放内存

- getXXX(Object o,long offset)   
获取对象o中内存偏移量为offset的值。

- getUnsafe  
单例，获取Unsafe对象

- monitorEnter(Object o)  
锁定对象，直到调用了monitorExit(Object o)后才会被解锁

- monitorExit(Object o)  
解锁对象，该对象必须是之前就已经被锁定了(被执行过monitorExit)

详情可以查看：<code>http://www.docjar.com/docs/api/sun/misc/Unsafe.html</code>

#### Examples for Unsafe  
- sun.misc.Unsafe#allocateInstance()  
不使用类的构造方法，来生成一个类的对象。

```
import java.lang.reflect.Field;
import sun.misc.Unsafe;
import sun.reflect.Reflection;

Field field = Unsafe.class.getDeclaredField("theUnsafe");
field.setAccessible(true);
Unsafe unsafe = (Unsafe) field.get(null);

Person person1 = (Person)unsafe.allocateInstance(Person.class);
System.out.println(person1); //null: 0

Person person2 = new Person();
System.out.println(person2);//test: 22
```
我们对比person1和person2，可以发现person1是没有使用构造方法的，而person2是使用了构造方法的。

- objectFieldOffset、putXXX  
操作对象成员的值：
<code>objectFieldOffset(Field var1)</code>是获取成员变量的相对于对象内存的偏移量。
<code>putLong，putInt，putDouble，putChar，putObject</code>等，可以直接修改对象内存中的数据，可以突破访问修饰符(private/protected)的限制。可以查看下面的例子：

```
import java.lang.reflect.Field;
import sun.misc.Unsafe;
import sun.reflect.Reflection;

Field field = Unsafe.class.getDeclaredField("theUnsafe");
field.setAccessible(true);
Unsafe unsafe = (Unsafe) field.get(null);

Person person1 = (Person)unsafe.allocateInstance(Person.class);
System.out.println(person1);    //null: 0

Class clazz = person1.getClass();
Field name = clazz.getDeclaredField("name");
Field age = clazz.getDeclaredField("age");

unsafe.putObject(person1, unsafe.objectFieldOffset(name),"张三");
unsafe.putInt(person1, unsafe.objectFieldOffset(age),18);

System.out.println(person1);    //张三: 18
```

关于Unsafe类里的方法，感兴趣的网上查找阅读相关的C/C++实现。

- 大数组操作  
Java最大只能创建长度为<code>Integer.MAX_VALUE</code>的数组，我们知道当JVM消耗内存过高的时候发生GC，性能上会大打折扣。那么当我们需要创建一个大数组，且不想因为GC而引起大的性能损耗时，我们能想到的就是使用堆外内存。例如下边的代码就实现了一个大的字节数组：

```
class BigByteArray {
    private final static int BYTE = 1;

    private long size;
    private long address;

    public BigArray(long size) {
        this.size = size;
        address = getUnsafe().allocateMemory(size * BYTE);
    }

    public void set(long i, byte value) {
        getUnsafe().putByte(address + i * BYTE, value);
    }

    public int get(long idx) {
        return getUnsafe().getByte(address + idx * BYTE);
    }

    public long size() {
        return size;
    }
}
```

#### 总结
学习完Unsafe和CAS之后，我们再来学习Java中的AQS的实现，就好理解了。

