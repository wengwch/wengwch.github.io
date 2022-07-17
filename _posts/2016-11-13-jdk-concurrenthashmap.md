---
title: JDK源码学习之ConcurrentHashMap
date: 2016-11-13 11:17:40
categories: 
  - Programing Language
tags:
  - Java
---

### 概述

上一篇介绍了HashMap的代码实现。众所周知，HashMap不是线程安全的，那么怎样让HashMap变得线程安全呢，最简单的方法就是给HashMap的方法加上`synchronized`，使其变为同步的，HashTable就是这么干的。这样虽然能解决并发的问题，但是由于锁粒度太大，多线程时竞争激烈，效率很低，因此并不推荐使用。在并发环境下，有更好的选择`ConcurrentHashMap`

ConcurrentHashMap在java7中使用分段锁的机制减小锁粒度，提高并发效率。但是在java8中，已经抛弃了这种实现，使用更加高效的CAS自旋的方式进一步提升HashMap在并发环境中的效率。

这篇主要介绍一下put和resize过程的实现。

<!-- more -->

### 关键字段和方法

在看源代码之前，先了解一下ConcurrentHashMap的几个关键字段和方法。

先看一下几个关键的字段。

```java

/**
 * Node数组，长度为2的指数次方，用于存储数据。在第一次插入时初始化
 */
private transient volatile Node<K,V>[] table;

/**
 * resize过程中用到，只有在resize过程中非空
 */
private transient volatile Node<K,V>[] nextTable;

/**
 * 计数器，主要在没有竞争的时候使用
 * 有竞争时，使用类似LongAdder的机制提高并发环境下计数器的效率
 */
private transient volatile long baseCount;

/**
 * 用于控制table的初始化和resize过程
 * -1 表示正在初始化过程中
 * 其他负数表示正在resize过程中
 * 除此之外，值为0.75*table.length
 */
private transient volatile int sizeCtl;

/**
 * 在resize过程中，分割table，使多线程可以同时对table的不同区间进行resize操作，提高resize的效率
 */
private transient volatile int transferIndex;
```
注意到，这些字段都有`volatile`关键字，主要用于保证内存可见性和顺序性。关于`volatile`关键字的作用，有兴趣的可以自己去研究一下。

再看一下三个关键方法

```java
// 获取table[i]的值
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i);
// 使用cas设置table[i]的值，成功返回true，失败返回false
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i, Node<K,V> c, Node<K,V> v)
// 设置 table[i]的值
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v)
```
这三个操作都是原子的，可以确保对table的修改都是原子操作。

### put实现

ConcurrentHashMap的put实现和HashMap的很相似，主要区别就是使用CAS使对表的操作原子化，以确保线程安全。直接看代码

```java
  final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null)
      throw new NullPointerException();
    // 计算key的hash值
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K, V>[] tab = table; ; ) {
      Node<K, V> f;
      int n, i, fh;
      if (tab == null || (n = tab.length) == 0)
        // 初始化table
        tab = initTable();
      else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
        // 散列位置为空，存入Node，如果返回false，说明有多个线程尝试修改改位置，自旋后重新尝试修改
        if (casTabAt(tab, i, null, new Node<K, V>(hash, key, value, null)))
          break;                   // no lock when adding to empty bin
      } else if ((fh = f.hash) == MOVED)
        // 如果哈希值为MOVED，表明这是一个ForwardingNode，其他线程正在进行resize操作，去帮助resize
        tab = helpTransfer(tab, f);
      else {
        // 以下过程和HashMap类似，如果key值存在，就覆盖旧值，不存在，就插入到链表尾部
        V oldVal = null;
        synchronized (f) {
          if (tabAt(tab, i) == f) {
            if (fh >= 0) {
              binCount = 1;
              for (Node<K, V> e = f; ; ++binCount) {
                K ek;
                if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                  oldVal = e.val;
                  if (!onlyIfAbsent)
                    e.val = value;
                  break;
                }
                Node<K, V> pred = e;
                if ((e = e.next) == null) {
                  pred.next = new Node<K, V>(hash, key, value, null);
                  break;
                }
              }
            } else if (f instanceof TreeBin) {
              Node<K, V> p;
              binCount = 2;
              if ((p = ((TreeBin<K, V>) f).putTreeVal(hash, key, value)) != null) {
                oldVal = p.val;
                if (!onlyIfAbsent)
                  p.val = value;
              }
            }
          }
        }
        if (binCount != 0) {
          if (binCount >= TREEIFY_THRESHOLD)
            treeifyBin(tab, i);
          if (oldVal != null)
            return oldVal;
          break;
        }
      }
    }
    // 计数计加1，并会检查是否需要resize
    addCount(1L, binCount);
    return null;
  }
```


### resize实现

resize的大概流程是这样的。从旧表的尾部往前寻找未处于resize过程的区间，将区间开始位置的结点都转移到新表中，并将该位置设置为ForwardingNode，表明该区间处于resize过程。然后从后往前遍历该区间，将所有位置都设置为ForwardingNode，并所有结点都转移到新表中。完成该区间的resize操作后，再对下一区间进行resize操作，直到旧表中所有结点都转移到新表中，用新表取代旧表。由于resize是以区间为单位进行的，因此多线程可以同时对不同的区间进行resize操作，不会有数据冲突，可以提高整个表resize的速度。下面是简单的图例。

*寻找未处于resize过程的区间*

![1](/assets/img/dc93f038-ae75-11e6-9b2a-b2008c079c5e.png)

*转移区间开始位置的结点到新表，并设置该位置为ForwardingNode，表明区间正处于resize过程*

![2](/assets/img/e3692f0e-ae75-11e6-88bd-77f12eea55b9.png)

*遍历该区间，将所有结点都转移到新表中*

![3](/assets/img/e3a0f7cc-ae75-11e6-817d-d228824e02ae.png)

下面看一下代码的实现细节

```java
private final void transfer(Node<K, V>[] tab, Node<K, V>[] nextTab) {
  int n = tab.length, stride;
  // 设置区间长度，最小为16
  if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
    stride = MIN_TRANSFER_STRIDE; // subdivide range
  // nextTab为空，则创建新表。已经有其他线程正在进行resize操作，则不为空
  if (nextTab == null) {            // initiating
    try {
      @SuppressWarnings("unchecked")
      Node<K, V>[] nt = (Node<K, V>[]) new Node<?, ?>[n << 1];
      nextTab = nt;
    } catch (Throwable ex) {      // try to cope with OOME
      sizeCtl = Integer.MAX_VALUE;
      return;
    }
    nextTable = nextTab;
    transferIndex = n;
  }
  int nextn = nextTab.length;
  ForwardingNode<K, V> fwd = new ForwardingNode<K, V>(nextTab);
  boolean advance = true;
  boolean finishing = false; // to ensure sweep before committing nextTab
  for (int i = 0, bound = 0; ; ) {
    Node<K, V> f;
    int fh;
    // 寻找没有处于resize过程的区间
    while (advance) {
      int nextIndex, nextBound;
      // --i，用于遍历区间，或者在resize完成时，遍历整个表检查是否将所有结点都转移
      if (--i >= bound || finishing)
        advance = false;
      else if ((nextIndex = transferIndex) <= 0) {
        i = -1;
        advance = false;
      } else if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex,
          nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
        // 设置区间的开始结束位置bound 和 nextIndex
        bound = nextBound;
        i = nextIndex - 1;
        advance = false;
      }
    }
    if (i < 0 || i >= n || i + n >= nextn) {
      int sc;
      if (finishing) {
        // resize 完成，使用新表取代旧表
        nextTable = null;
        table = nextTab;
        // 设置sizeCtl为0.75倍的新表长度
        sizeCtl = (n << 1) - (n >>> 1);
        return;
      }
      if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
        // 完成resize，线程退出。最后一个完成resize的线程会进行rechek，查看所有结点是否全部转移，并将没有转移的都转移到新表，然后使用新表取代旧表
        if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
          return;
        finishing = advance = true;
        i = n; // recheck before commit
      }
    } else if ((f = tabAt(tab, i)) == null)
      // 该位置没有结点，直接设置为ForwardingNode
      advance = casTabAt(tab, i, null, fwd);
    else if ((fh = f.hash) == MOVED)
      // 该结点所处区间已经处于resize过程，寻找下一个区间
      advance = true; // already processed
    else {
      synchronized (f) {
        // rehash过程，将结点转移到新表上，和HashMap基本一致
        if (tabAt(tab, i) == f) {
          Node<K, V> ln, hn;
          if (fh >= 0) {
            int runBit = fh & n;
            Node<K, V> lastRun = f;
            for (Node<K, V> p = f.next; p != null; p = p.next) {
              int b = p.hash & n;
              if (b != runBit) {
                runBit = b;
                lastRun = p;
              }
            }
            if (runBit == 0) {
              ln = lastRun;
              hn = null;
            } else {
              hn = lastRun;
              ln = null;
            }
            for (Node<K, V> p = f; p != lastRun; p = p.next) {
              int ph = p.hash;
              K pk = p.key;
              V pv = p.val;
              if ((ph & n) == 0)
                ln = new Node<K, V>(ph, pk, pv, ln);
              else
                hn = new Node<K, V>(ph, pk, pv, hn);
            }
            // 部分结点转移的新表不变的位置
            setTabAt(nextTab, i, ln);
            // 部分结点转移到新表原位置+旧表长度的位置
            setTabAt(nextTab, i + n, hn);
            // 将旧表该位置设置为ForwardingNode，表明已经完成结点转移
            setTabAt(tab, i, fwd);
            advance = true;
          } else if (f instanceof TreeBin) {
            //树化的相关操作，省略
            ...
          }
        }
      }
    }
  }
}

```

### 总结

这篇主要介绍了一下ConcurrentHashMap的实现方式，并对ConcurrentHashMap在并发环境中为什么高效做了一些简单的分析。另外从ConcurrentHashMap的实现也能简单了解一些lock free算法的设计和实现思路。


### 参考

[无锁HashMap的原理与实现](http://coolshell.cn/articles/9703.html)

[无锁队列的实现](http://coolshell.cn/articles/8239.html)

[聊聊并发（四）——深入分析ConcurrentHashMap](http://www.infoq.com/cn/articles/ConcurrentHashMap)


