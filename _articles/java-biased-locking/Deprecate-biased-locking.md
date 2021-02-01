---
title: " 你知道为什么 Java 会放弃偏向锁吗？ "
date: 2020-07-25 09:50:00 +0800
tag: [Java, JVM]
---

# 你知道为什么 Java 会放弃偏向锁吗？

## 前言

随着 JDK15 的 Release，大量的新特性也和开发者们正式见面了（虽然不一定能用上......），最引入注目的应该是 Text Block、Pattern Matching for instanceof (Second Preview) 等语法，因为这些特性是实实在在的影响了大家的开发体验。

而本文要讨论的却不是这些特性，而是关于废弃偏向锁的特性

-  Disable and Deprecate Biased Locking（禁用和废弃偏向锁）



## 什么是偏向锁？

在 Java 中，`synchronized` 关键字可以确保同一时间内只有一个线程进入临界区

> 临界区：可以是整个方法体，也可以是方法的局部块

在 JDK1.6 以后，JVM 特地针对 `synchronized` 进行了优化，引入了锁升级的概念，锁状态只可升级不可降级。

```java
无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁
```

这里面我们主要关注**偏向锁**，它是 HotSpot VM 提供的一种用于减少锁开销的优化技术。

它的原理其实很简单：当线程进入第一次同步块以后，锁持有的对象头的偏向锁标记会被置为 1 ，如果未来一段时间内都是同一个线程访问该同步快的话就不用执行加锁操作了。

如果有另一个线程进入同步快的话，偏向锁就会升级到轻量级锁。

由此可以看出来如果同步块只有一个线程访问的话，偏向锁是可以带来显著的性能提升的。

既然如此，为何要废弃掉呢？



## 为什么要废弃？

#### 偏向锁带来的性能收益降低

在现今，偏向锁带来的性能提升远不如过去那么明显了。

在过去，旧的 Java 应用通常使用的都是比较老的集合库，比如 HashTable、Vector 等，这些集合库对应的方法都被 synchronized 修饰了。

如果在单线程使用这些集合库就会造成不必要的加锁操作，导致性能下降。

而新的无锁集合库，如 HashMap、ArrayList 等就没有这样的问题，如果是在多线程情况下，也有 ConcurrentHashMap、CopyOnWriteArrayList 等性能更好的线程安全的集合库。

所以在使用了新的集合库的情况下，偏向锁带来的收益会降低。

在多线程情况下，经常会使用到线程池，关闭偏向锁反而更好。

在明显有竞争的场景下，偏向锁带来了昂贵的撤销操作的代价。

因此，偏向锁其实只适用于那些无竞争的同步操作的应用程序



#### 为 HotSpot 引入了大量复杂度

在 HotSpot 虚拟机中，偏向锁为整个「同步子系统」引入了大量的复杂度，并且对 HotSpot 的其它组件也造成了侵入。

这样的耦合性导致了「同步子系统」很难进行重大设计的变更。



总下下来就是维护成本高、收益越来越低，所以决定废弃它了。



## 后续如何兼容？

默认禁用偏向锁可能会导致一些 Java 应用的性能下降，所以 HotSpot 提供了显示开启偏向锁的命令

```bash
-XX:+UseBiasedLocking
```

以下和偏向锁相关的命令参数仍然可以使用，但是虚拟机会列出对应的警告信息标示其已被废弃掉

```bash
-XX:BiasedLockingStartupDelay=__
-XX:BiasedLockingBulkRebiasThreshold=__
-XX:BiasedLockingBulkRevokeThreshold=__
-XX:BiasedLockingDecayTime=__
// 必须在 C2 下才能使用
-XX:+UseOptoBiasInlining

// 必须配合参数-XX:+UnlockDiagnosticVMOptions使用
// 并且只能加在其后才能生效
-XX:+PrintBiasedLockingStatistics
-XX:+PrintPreciseBiasedLockingStatistics
```

在将来，HotSpot 会慢慢的禁用、废弃直到最终废移除偏向锁。



## 参考

1. [JEP 374: Disable and Deprecate Biased Locking](https://openjdk.java.net/jeps/374)
2. [open JDK15 Release](http://openjdk.java.net/projects/jdk/15/)
3. [Synchronization and Object Locking](https://wiki.openjdk.java.net/display/HotSpot/Synchronization) By Christian Wimmer

