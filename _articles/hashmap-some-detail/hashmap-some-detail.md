---
title:  "HashMap 的细节笔记"
date:   2020-04-27 21:19:45 +0800
tags: [Java, HashMap]
---

# HashMap 的细节笔记



### **HashMap 的节点转为红黑树以后，树形节点之间是基于什么来比较的呢？**

首先会根据 key 的 Hash 值来进行比较，如果相等的话，会再判断 key 是否实现了 `Comparable` 接口，如果是的话，可以直接通过 `compareTo` 方法进行比较。

如果 hash 值相等，而且也没有实现Comparable 接口呢？此时会对 key 对象使用 `System.identityHashCode(key)` 进行再次 hash 运算，最后再根据新的 hash 值进行比较。

那么 `System.identityHashCode` 又是什么操作呢？

这是一个 native 的方法，实现的原理也很简单，就是不管你对象有没重写 hashCode 函数，它都只返回对象默认的 hashCode 值。



### **为什么 HashMap 的链表节点冲突数达到 8 才会转为红黑树？**

在 hashmap 的源码注释中有一段 Implementation notes， 里面有提到具体的决策原因。

虽然红黑树的查询事件复杂度是 logN,  但是红黑树节点的 size 大概是普通节点的 2 倍，而且在插入效率方面，红黑树的插入时间复杂度为 logN, 链表的插入节点时间复杂度是 O (1)。

综上所述，实则是一个时间和空间的决策，那么为什么是 8 呢？

如果 key 的 hash 值分布的足够均匀，几乎不会转换成树形节点。

假设使用随机的 hash 算法，理想情况下，通过泊松分布的概率函数可以计算某个桶位冲突节点达到 K 个的概率，设 `λ = 0.75`, 带入以下公式



![](img/poisson-distribution.svg)



| 节点数(K) |  概率(P)   |
| :-------: | :--------: |
|     0     | 0.60653066 |
|     1     | 0.30326533 |
|     2     | 0.07581633 |
|     3     | 0.01263606 |
|     4     | 0.00157952 |
|     5     | 0.00015795 |
|     6     | 0.00001316 |
|     7     | 0.00000094 |
|     8     | 0.00000006 |

节点数大于 8 的概率少于一千万分之一 



### **HashMap 的 capacity 为什么是 2 的次幂？**

我们都知道在构造 HashMap 的时候可以指定初始容量

```java
public HashMap(int initialCapacity)

public HashMap(int initialCapacity, float loadFactor)
```

如果传入的 initialCapacity 不是 2 的 n 次方的话，HashMap 会调用 `tableSizeFor` 方法，该方法会返回大于 cap 的最小 2 次幂的值

```java
static final int tableSizeFor(int cap) {
  int n = -1 >>> Integer.numberOfLeadingZeros(cap - 1);
  return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

为什么需要这样做呢？

- 在计算索引时，可以用 & 来替代 %
- 在 resize 时，可以更方便的对冲突节点进行处理

先来看第一点

我们指的 capacity 实际就是 bucket.length，每一个 key 对应的索引是根据下面的公式进行计算的

```matlab
   index = (bucket.length - 1) & hash(key)
```

假设 capacity 保证为 2 的次幂，那么 `capacity - 1` 就可以保证该数的二进制低位全部为 1

比如 

```java
(16 - 1) = 0000_0000_0000_1111
(32 - 1) = 0000_0000_0001_1111
(64 - 1) = 0000_0000_0011_1111
```

那么 在与 `hash(key) ` 进行 `&` 运算，可以保证最终得到的结果范围符合 [0, buket.length - 1]，即

```java
 0 <= index <= buket.length - 1
```



再来看 resize

众所周知，HashMap 在节点达到阈值时，会按当前容量的两倍进行扩容，也就是相当于当前容量值左移一位。

比如当前容量为 16，向左移位 1 位就是 32，对应的二进制位

```java
16 = 0000_0000_0001_0000
32 = 0000_0000_0010_0000  
```

扩容以后，节点需要进行 REHASH，结合前面提到的 hash 算法

```java
 index = (bucket.length - 1) & hash(key)
```

由于低位不变，只是高位变成了 1，这样的话，就可以把一段冲突的链表节点分为两部分，高位为 1 与 高位为 0。

假设新的容量为 32，两个节点 A 和 B 的 Hash 值如下

```properties
A = 0000_0000_0000_1011
B = 0000_0000_0001_1011
```

那么新的索引

```math
index(A) = 11111 & A = 1011 即 11
index(B) = 11111 & B = 11011 即 27
```

很明显如果 index(B) = (old capacity - 1) + old index =  (16 - 1) + 11

而这个也是得益于 capacity 是 2 的 n 次方这一特性