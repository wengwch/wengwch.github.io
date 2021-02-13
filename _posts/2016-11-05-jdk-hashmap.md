---
title: JDK源码学习之HashMap
date: 2016-11-05 20:45:01
categories:
  - Programing Language
tags:
  - Java
math: true
---

### 1. 概述
HashMap是Java开发中最常用的数据结构，阅读源码有助于我们了解其工作原理及实现。关于HashMap的特性，可以参考官方文档：

> Hash table based implementation of the Map interface. This implementation provides all of the optional map operations, and permits null values and the null key. (The HashMap class is roughly equivalent to Hashtable, except that it is unsynchronized and permits nulls.) This class makes no guarantees as to the order of the map; in particular, it does not guarantee that the order will remain constant over time.

HashMap实现了Map接口，允许键/值为`null`，非线程安全的，也不能保证顺序。

<!-- more -->

HashMap基于数组实现，并使用拉链法解决哈希碰撞的问题。在开始之前，先介绍一下HashMap中的`capacity`和`loadFactor`。

* `capacity`表示容量，即hashmap中`table`数组的长度，该值一直为$2^n$(`1<<n`)，默认为16。

* `loadFactor`表示负载因子，默认为0.75，当hashmap中元素超过`capacity*loadFactor`这个阈值，就会进行扩容`resize`。

HashMap的散列函数采用除留余数法，使用key的hash值被table数组的长度除后所得的余数为散列地址。即\\(f(k) = key.hash\mod capacity\\)

下面看一下在Java8中HashMap的代码实现（主要介绍`put`, `get`, `resize`的实现）。



### 2. put


put函数的逻辑大概如下：

1. 令`n = table.length`, `hash = hash(key)`
2. 计算结点散列地址`index = hash % n`
3. 检测是否与`table[index]`发生碰撞，有碰撞跳转`5`
4. 直接将结点放入`table[index]`，跳转`7`
5. 检查`table[index]`处的链表是否已经树化，是就将结点插入树中，如果key值存在，直接覆盖旧值，跳转`8`，如果不存在，跳转`7`
6. 遍历链表，如果key值存在，直接覆盖旧值，跳转`8`。如果不存在，将结点插入链表尾部，并检查链表长度是否超过树化的阈值（默认8），超过就将链表转化为红黑树，提高查找效率
7. 检查元素数量是否超过`capacity*loadFactor`，是就进行扩容并且`rehash`
8. 结束


现在来看一下put的代码：

```java

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K, V>[] tab;
    Node<K, V> p;
    int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
      // 初始化table
      n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
      // 无碰撞
      tab[i] = newNode(hash, key, value, null);
    else {
      // 碰撞
      Node<K, V> e;
      K k;
      if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
        e = p;
      else if (p instanceof TreeNode)
        // 已经树化，使用putTreeVal插入结点
        e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value);
      else {
        // 遍历链表
        for (int binCount = 0; ; ++binCount) {
          if ((e = p.next) == null) {
            // 将结点插入链表尾部
            p.next = newNode(hash, key, value, null);
            if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
              // 将链表转化为红黑树，提高效率
              treeifyBin(tab, hash);
            break;
          }
          if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
            break;
          p = e;
        }
      }
      if (e != null) { // existing mapping for key
        // key值已经存在
        V oldValue = e.value;
        if (!onlyIfAbsent || oldValue == null)
          // 覆盖旧值
          e.value = value;
        afterNodeAccess(e);
        return oldValue;
      }
    }
    // 修改计数，在使用迭代器的过程中有其他线程修改了map，那么将抛出ConcurrentModificationException
    ++modCount;
    if (++size > threshold)
      // 扩容
      resize();
    afterNodeInsertion(evict);
    return null;
  }  
    
```

### 3. resize

resize做了两件事情：

* 将table容量扩大到两倍大小
* rehash计算元素在新table中的散列地址，存入新table

扩容很简单，直接创建一个原table大小2倍的新table即可。

对于rehash计算的新散列地址，只有两种情况：

* 新地址不变
* 新地址 = 旧地址 + 扩容前表容量大小

以上结论很容易推导：
假设原表容量为n，\\(n=2^k\\)， 扩容后新表容量为\\(m = 2n =2^{k + 1}\\)，key的hash值为h

rehash前：

* 令\\(a = \lfloor{\frac h n}\rfloor = \lfloor{\frac h {2^k}}\rfloor\\)
* 散列地址为\\(f_1(h) = h\mod n = h - an\\)

rehash后：

* 散列地址为\\(f_2(h) = h\mod m = h\mod 2n = h -\lfloor \frac a 2 \rfloor m\\)

如果a为偶数，\\(f_2(h) = h - \lfloor \frac a 2 \rfloor m = h - \frac a 2 2n = h - an = f_1(h)\\)

如果a为奇数，\\(f_2(h) = h - \lfloor \frac a 2 \rfloor m = h - \frac {a - 1} 2 2n = h - an + n = f_1(h) + n\\)

了解以上结论，再看resize的代码就很容易了。

```java
// 创建新表，容量为旧表的两倍
Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
table = newTab;
if (oldTab != null) {
  //rehash
  for (int j = 0; j < oldCap; ++j) {
    Node<K, V> e;
    if ((e = oldTab[j]) != null) {
      oldTab[j] = null;
      if (e.next == null)
        // 该散列地址只有一个结点，没有碰撞，直接将结点rehash后存入新表
        newTab[e.hash & (newCap - 1)] = e;
      else if (e instanceof TreeNode)
        // 对于已经树化的，通过split将结点重新rehash，有兴趣的可以自己研究一下
        ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
      else { // preserve order
        Node<K, V> loHead = null, loTail = null;
        Node<K, V> hiHead = null, hiTail = null;
        Node<K, V> next;
        do {
          next = e.next;
          if ((e.hash & oldCap) == 0) {
            // rehash后散列地址不变的结点组成新链表loHead
            if (loTail == null)
              loHead = e;
            else
              loTail.next = e;
            loTail = e;
          } else {
            // rehash后散列地址改变的结点组成新链表hiHead
            if (hiTail == null)
              hiHead = e;
            else
              hiTail.next = e;
            hiTail = e;
          }
        } while ((e = next) != null);
        if (loTail != null) {
          // 将loHead链表放入新表原地址中
          loTail.next = null;
          newTab[j] = loHead;
        }
        if (hiTail != null) {
          // 将hiHead链表放入新表原地址 + oldCap位置中
          hiTail.next = null;
          newTab[j + oldCap] = hiHead;
        }
      }
    }
  }
}
```

### 4. get

了解了put和resize，get就很容易理解了，直接看代码:

```java
final Node<K, V> getNode(int hash, Object key) {
    Node<K, V>[] tab;
    Node<K, V> first, e;
    int n;
    K k;
    if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
      // 检查第一个结点，如果key相等，直接返回value
      if (first.hash == hash && // always check first node
          ((k = first.key) == key || (key != null && key.equals(k))))
        return first;
      if ((e = first.next) != null) {
        if (first instanceof TreeNode)
          // 已经树化的，在树中查找
          return ((TreeNode<K, V>) first).getTreeNode(hash, key);
        do {
          // 遍历链表查找，查找key值命中的value
          if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
        } while ((e = e.next) != null);
      }
    }
    return null;
  }
```

### 5. 结论

HashMap是java开发中最常用的数据结构，也是java面试中常见的问题，了解其源码不管从哪方面看都是有好处的。另外，在平时的使用中，需要注意HashMap并不是线程安全的，在并发环境中使用会出现很多问题，比如resize可以造成死循环等。在并发环境中一定要使用`ConcurrentHashMap`。

### 参考

[疫苗：Java HashMap的死循环](http://coolshell.cn/articles/9606.html)

[Java HashMap工作原理及实现](http://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)

[Java HashMap工作原理](http://www.importnew.com/16599.html)
