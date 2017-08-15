---
layout: post
comments: true
date: 2017-07-22
categories: java
tags: ConcurrentHashMap
---

* content
{:toc}


Jdk本身给我实现了很多数据结构，例如List、Set、Queue以及Map等。但是这些类都不是线程安全的，在Jdk1.0的时候提供了<code>Hashtable</code>、<code>Vector</code>以及<code>Stack</code>这些线程安全集合，但由于它们只是在原来集合的基础上通过关键字<code>synchronized</code>来保证功能的**原子性和可见性**，这样做的缺点就是效率地下，当有一个线程执行get的时候，其它所有线程都被阻塞。



### 什么是线程安全

线程安全是一个很难一言表达清楚的，通俗的说就是：一个程序在单线程的环境下可以正常运行，在多线程环境下也能正确的运行，并且可以返回确定的结果，那么这个程序就是线程安全的。
一个线程安全的类，在多线程环境下不会因为执行的顺序导致对象成为无效状态，同样也不会违反任何先验条件和后验条件。

关于无效状态、先验条件、后验条件，下面是一些说明：

- 无效状态

在做选择排序的过程中，前边已经排序好的那段数据顺序被其它线程给打乱了，这时候就可以说这个被排序的类处于无效状态。

- 先验条件

在做二分查找的时候，被查找的集合必须是已经排好序的，这就是执行二份查找的条件，也就是先验条件。

- 后验条件

指的代码的业务目标是否正确。


### fail-fast机制

当迭代器实现了fail-fast机制，创建了该集合的迭代器后，使用该集合迭代器遍历或者修改集合元素的之前，若该集合被直接调用方法并产生了修改（size的变化），那么将会抛出<code>java.util.ConcurrentModificationException</code>异常。  
>例如<code>ArrayList</code>对象，线程1在用迭代器遍历里边元素的同时，线程2put了一个新的元素，那么就会抛出异常<code>java.util.ConcurrentModificationException</code>。

fail-fast机制是使用了<code>modCount</code>变量（集合类持有的字段）和<code>expectedModCount</code>（迭代器持有的字段）来实现。
在迭代器创建的时候将exceptedModCount被初始化为modCount的值，任何对集合的修改都会改变modCount的值.每当调用next等方法的时候，会判断expectedModCount和当前的modCount是否相等，不相等的话则抛出CME异常.

- modCount  
<code>modCount变量是集合类中的变量</code>，在集合被修改时，modCount变量将会加1；当调用迭代器的方法改变了集合size的方法时，modCount变量也会加1。

- expectedModCount  
<code>expectedModCount变量是迭代器类中的变量</code>，在创建的迭代器对象时，会使用modCount的值来对其进行初始化。只有在调用迭代器改变了集合size的时候，才会加1。

- ArrayList、LikedList  
它们都是AbstractList的子类，它们的modCount都是从AbstractList类中继承来的，当执行会使集合size发生变化的方法时，例如：add、remove等，modCount会加1。执行set、get操作不会使modCount发生变化，因为他们没有使集合的size发生变化。
当调用迭代器的next、add、remove、previous、set等访问集合元素的方法时，都会校验exceptedModCount和modCount是否相等，如果不相等，则抛出CME异常。

- HashMap、LinkedHashMap  
当调用Map的方法或者Map的迭代器使得Map集合被修改（值被修改，或者顺序发生变化）时，modCount加1。
在调用Map的迭代器访问、修改Map集合元素的时候，都会判断expectedModCount和当前的modCount是否相等，不相等的话则抛出CME异常.

>很显然，只使用modCount和expectedModCount并不能保证集合的线程安全。  
>- modCount是集合类的成员变量，expectedModCount是迭代器的成员变量
>- 在使用迭代器遍历集合元素时，会判断modCount和expectedModCount是否相等来校验集合是否被修改过，很显然这个办法是存在bug的。
>- 对于Map结构，数据只要变化modCount就会加1；对于List结构，当集合size变化时，modCount才会加1。  
>
>这些规则表述的可能不是很准确，具体可以查看相关源码。Set结构是什么样的呢？留给读者自己查看源码吧！

### Collections.synchronizedXXX

发展到Jdk1.2时，Jdk提供了<code>Collections.synchronizedXXX</code>系列的方法：

- Collections.synchronizedCollection  
包装Collection接口的实现类，使其具备线程安全

- Collections.synchronizedList  
包装List接口的实现类，使其具备线程安全

- Collections.synchronizedMap  
包装Map接口的实现类，使其具备线程安全

- Collections.synchronizedSet  
包装Set接口的实现类，使其具备线程安全

这里我们查看<code>java.util.Collections.SynchronizedCollection</code>类的源码，可以看出，这也是在原来集合的基础上包装了一层<code>synchronized</code>关键字，只是提供了并发集合使用的便利性，并没有从本质上提高并发集合的效率！

```
static class SynchronizedCollection<E> implements Collection<E>, Serializable {
    private static final long serialVersionUID = 3053995032091335093L;

    final Collection<E> c;  // Backing Collection
    final Object mutex;     // Object on which to synchronize

    SynchronizedCollection(Collection<E> c) {
        if (c==null)
            throw new NullPointerException();
        this.c = c;
        mutex = this;
    }
    SynchronizedCollection(Collection<E> c, Object mutex) {
        this.c = c;
        this.mutex = mutex;
    }

    public int size() {
        synchronized (mutex) {return c.size();}
    }
    public boolean isEmpty() {
        synchronized (mutex) {return c.isEmpty();}
    }
    public boolean contains(Object o) {
        synchronized (mutex) {return c.contains(o);}
    }
    public Object[] toArray() {
        synchronized (mutex) {return c.toArray();}
    }
    public <T> T[] toArray(T[] a) {
        synchronized (mutex) {return c.toArray(a);}
    }

    public Iterator<E> iterator() {
        return c.iterator(); // Must be manually synched by user!
    }

    public boolean add(E e) {
        synchronized (mutex) {return c.add(e);}
    }
    public boolean remove(Object o) {
        synchronized (mutex) {return c.remove(o);}
    }

    public boolean containsAll(Collection<?> coll) {
        synchronized (mutex) {return c.containsAll(coll);}
    }
    public boolean addAll(Collection<? extends E> coll) {
        synchronized (mutex) {return c.addAll(coll);}
    }
    public boolean removeAll(Collection<?> coll) {
        synchronized (mutex) {return c.removeAll(coll);}
    }
    public boolean retainAll(Collection<?> coll) {
        synchronized (mutex) {return c.retainAll(coll);}
    }
    public void clear() {
        synchronized (mutex) {c.clear();}
    }
    public String toString() {
        synchronized (mutex) {return c.toString();}
    }
    private void writeObject(ObjectOutputStream s) throws IOException {
        synchronized (mutex) {s.defaultWriteObject();}
    }
}
```

### java.util.concurrent概述
Jdk1.5开始，Java的并发模型发生了翻天覆地的变化,具体可以查看<code>java.util.concurrent</code>包下的类。

- CAS的实现  
目前大多数CPU硬件级别都只是CAS了，jdk1.5后封装了CAS的实现，并以Unsafe类供jdk中其它类调用。这个设计的初衷是给Jdk的类库使用，都不是给开发人员。

- 锁和原子类  
在CAS的基础上，实现了AQS，并实现了ReentrantLock锁，一般很少创建ReentrantLock对象，而是继承ReentrantLock类并在此基础上构建。
<code>java.util.concurrent.atomic</code>包下提供了一系列的原子变量，例如在并发环境下计算自增值，就可以使用该包下面的类。

- 高级的工具类  
例如下面三个：
    - Executor框架：线程池
    - TimerTask类：定时器
    - Semaphore类：资源计数器
    - CountdownLatch类：倒计时计数器

上面简单的介绍了<code>java.util.concurrent</code>包，那么该包下关于集合线程安全的实现体现在哪些方面呢？

- 阻塞的线程安全集合  
集合被其它线程访问时，例如使用迭代器迭代、get或者put的时候，当其它线程在访问该资源时，会进入阻塞状态。这种集合在并发执行时，可以保证数据的强一致性。

- 非阻塞的线程安全集合  
集合被其它线程访问时，例如使用迭代器迭代、get或者put的时候，当其它线程在访问该资源时，不会进入阻塞状态。这种集合在并发执行时，可以保证数据的弱强一致性，更新的数据在经过某一段时间之后才会被访问到。

#### ConcurrentXXX
以<code>Concurrent</code>开头的集合类，都是<code>非阻塞的线程安全类</code>。
非阻塞的线程安全集合在使用时只能保证数据的弱一致性，无法保证数据的强一致性。如果必须要保证数据的强一致性，可以使用阻塞的线程安全类。前边提到的使用synchronized来保证线程安全的集合类，就是阻塞的线程安全类。

这里号称是非阻塞的线程安全类，只是理想状态下非阻塞，使用时阻塞的概率较低，效率较高，但还是会可能线程被阻塞的。

- ConcurrentHashMap  
非阻塞线程安全的HashMap

- ConcurrentLinkedDeque  
非阻塞线程安全的双端队列

- ConcurrentLinkedQueue  
非阻塞线程安全的队列

- ConcurrentSkipListMap  
非阻塞线程安全的TreeMap（有序）

- ConcurrentSkipListSet  
非阻塞线程安全的TreeSet（有序）

#### CopyOnWriteArrayXXX
这一类集合在数据发生变化时，会通过内存拷贝来创建一个新的集合。若在迭代器遍历过程中集合的元素发生变化，迭代器会继续遍历“老”的集合，其它操作会访问“新”的集合。  
这类集合适合“数据量小”、“读多写少”的场景。

- CopyOnWriteArrayList  
线程安全的ArrayList，基于内存拷贝实现的。

- CopyOnWriteArraySet  
线程安全的ArrayList，基于内存拷贝实现的。其实就是对CopyOnWriteArrayList的包装，调用add方法的时候会调用CopyOnWriteArrayList类的addIfAbsent方法，当元素不存在的时候才会被加入到集合中。

#### BlockingXXX
阻塞线程安全类，其中BlockingDeque接口继承了BlockingQueue接口。这一类集合在并发过程中，可以保证数据的强一致性。

- ArrayBlockingQueue  
阻塞的线程安全队列，它是基于数组实现的，创建的时候就需要指定队列的长度。

- LinkedBlockingQueue  
阻塞的线程安全队列，是基于链表实现的。

- LinkedBlockingDeque  
阻塞的线程安全双端队列，是基于链表实现的。

- PriorityBlockingQueue  
阻塞的线程安全优先队列，队列中的元素必须实现Comparable接口，在插入元素时，会调用接口的<code>compareTo</code>方法来判断元素的大小，从而确定元素的位置。

### ConcurrentHashMap
这里把<code>ConcurrentHashMap</code>单独拿出来说一下，因为ConcurrentXXX系列集合实现的核心思想和它基本上一样。CopyOnWriteArray系列和BlockingXXX系列实现手法比较简单，自己浏览一下源码就可以了。

ConcurrentHashMap的类部结构大概如下图这样：

![结构](http://7xriy2.com1.z0.glb.clouddn.com/ConcurrentHashMap.png)

默认情况下Segment数组的大小是16(DEFAULT_INITIAL_CAPACITY),Segment下的HashEntry数组的默认大小是2(MIN_SEGMENT_TABLE_CAPACITY)。

#### 原理相关

- 锁的范围  
ConcurrentHashMap中读数据是没有锁的，写数据时会有锁。在向同一个Segment元素下写数据时，会使用同一把锁。

- 非阻塞锁  
这里号称是非阻塞的线程安全类，只是理想状态下非阻塞，使用时阻塞的概率较低，效率较高，但还是会可能线程被阻塞的。

- 弱一致性原理  
ConcurrentHashMap的get是没有锁的，但是put是有锁的。可能会出现一种情况是：线程1先执行了put，但是在数据没还没完全写入时，线程2读取了集合里的数据，此时是读取不到线程1刚刚执行写操作写入的数据，但是在很短暂的一个时间段后，线程2就能读到线程1写入的数了。
这就是读写分离产生的弱一致性。

#### 内部类

- HashEntry  
  该类实现的就是单纯的单向链表结构，和HashMap中的Entry类似。
  - 关键字段
  
  ```   
  /**
   * hash值
   */
  final int hash;
  final K key;
  volatile V value;

  /**
   * 下一个节点
   */
  volatile HashEntry<K,V> next;
  ```

  - 关键方法  

  ```
  /**
   * 使用UNSAFE来设置next节点
   */
  final void setNext(HashEntry<K,V> n) {
      UNSAFE.putOrderedObject(this, nextOffset, n);
  }

  // Unsafe相关的初始化
  static final sun.misc.Unsafe UNSAFE;
  static final long nextOffset;
  static {
      try {
          UNSAFE = sun.misc.Unsafe.getUnsafe();
          Class k = HashEntry.class;
          nextOffset = UNSAFE.objectFieldOffset
              (k.getDeclaredField("next"));
      } catch (Exception e) {
          throw new Error(e);
      }
  }
  ```
    
    
- Segment  
该类就是ConcurrentHashMap中分桶的实现。
  - 关键字段  

  ```
  /**
   * 处理器个数
   */
  static final int MAX_SCAN_RETRIES =
      Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;
      
  /**
   * 基于数组实现的Hash表
   */
  transient volatile HashEntry<K,V>[] table;
  
  /**
   * 分桶内的元素个数。
   */
  transient int count;
  ```
  - 关键方法 
   
  ```
  final V put(K key, int hash, V value, boolean onlyIfAbsent) {
      /**
       * 1. 写数据前,先拿到当前的节点元素,并执行park
       */
      HashEntry<K,V> node = tryLock() ? null :
          scanAndLockForPut(key, hash, value);
      V oldValue;
      try {
          //拿到分桶数组
          HashEntry<K,V>[] tab = table;

          //拿到该节点应该在的哪个
          int index = (tab.length - 1) & hash;

          //拿到分桶中HashEntry的起始位置HashEntry链表下
          HashEntry<K,V> first = entryAt(tab, index);

          //往下迭代
          for (HashEntry<K,V> e = first;;) {

              // 当分桶不为空时,持续遍历,找到节点,并修改它的value
              if (e != null) {
                  K k;
                  if ((k = e.key) == key/*基本类型,或同一对象*/ ||
                      (e.hash == hash && key.equals(k))/*+自定义对象*/) {
                      oldValue = e.value;
                      if (!onlyIfAbsent) {
                          e.value = value;
                          ++modCount;
                      }
                      break;
                  }
                  e = e.next;
              }
              //当分桶非空的时候
              else {

                  //若node已经获取到了,设置node为first元素
                  if (node != null)
                      node.setNext(first);
                  else
                      node = new HashEntry<K,V>(hash, key, value, first);
                  int c = count + 1;

                  //集合元素超过阈值后,扩充
                  if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                      rehash(node);
                  else
                      //否则直接把节点的值放在分桶的第一个位置
                      setEntryAt(tab, index, node);
                  ++modCount;
                  count = c;
                  oldValue = null;
                  break;
              }
          }
      } finally {
          unlock();
      }
      return oldValue;
  }
  
  /**
   * 获取锁,并返回对应的 HashEntry 节点。
   *
   * 1.若对应的分桶数组不存在,表示该hash值还未出现过,则直接创建出一个node,并在尝试次数达到MAX_SCAN_RETRIES时,阻塞
   * 2.若对应的分桶数组存在,则阻塞,接触阻塞后返回节点
   *
   * @return a new node if key not found, else null
   */
  private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
      //拿到分桶数组的首元素
      HashEntry<K,V> first = entryForHash(this, hash);
      HashEntry<K,V> e = first;
      HashEntry<K,V> node = null;

      int retries = -1; // negative while locating node

      //尝试获取锁,若获取不到锁,就一直轮询下去
      while (!tryLock()) {
          HashEntry<K,V> f; // to recheck first below
          if (retries < 0) {
              if (e == null) {    //首元素不存在,意思是hash还没有碰撞过
                  if (node == null) // speculatively create node
                      node = new HashEntry<K,V>(hash, key, value, null);
                  retries = 0;
              }
              else if (key.equals(e.key)) //查看key和首元素的key是否相等
                  retries = 0;
              else
                  //如果key的位置不在首元素出
                  e = e.next;
          }
          else if (++retries > MAX_SCAN_RETRIES) {
              lock(); //独占
              break;
          }
          else if ((retries & 1) == 0/*retries为偶数时*/ &&
                   (f = entryForHash(this, hash)) != first) {
              /**
               * 如果在次过程中HashEntry数组的first节点发生了变化,则重新遍历
               */
              e = first = f; // re-traverse if entry changed
              retries = -1;
          }
      }
      return node;
  }
  
  /**
   * 获取数据，key不能为null
   */
  public V get(Object key) {
      Segment<K,V> s; // manually integrate access methods to reduce overhead
      HashEntry<K,V>[] tab;
      int h = hash(key);

      //确定数据是在哪个分桶中的
      long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;

      //确定数据是在哪个分桶中的
      if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
          (tab = s.table) != null) {

          /**
           * 1.先确定数据是在哪个链表中
           * 2.遍历下去,直到读到对应的key的值
           */
          for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                   (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
               e != null; e = e.next) {
              K k;
              if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                  return e.value;
          }
      }
      return null;
  }
  ```

### 总结
1. 老的线程安全集合是基于<code>synchronized</code>关键字实现的，效率低下，但是具备强一致性的特性；
2. 基于AQS实现的新版线程安全集合效率较高，强一致性和弱一致性的集合都有部分的实现；
3. 个人觉得<code>ConcurrentHashMap</code>这个类源码研究透了，整个Java集合并发这部分知识就很好理解了；
4. 原理级别的东西看下源码就知道了，不用每一步都刻意去记忆，理解并记住其中的设计思想就可以了。

