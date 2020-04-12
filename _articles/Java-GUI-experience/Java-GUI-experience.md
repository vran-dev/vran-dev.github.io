---
title: "一次用 Java 写 GUI 的历程回顾"
date: 2020-04-12 12:30:37 +0800
tag: [Java, JavaFX]
---


# 一次用 Java 写 GUI 的历程回顾

## 起因

[elastic-job-lite]( https://github.com/elasticjob/elastic-job-lite)  是公司使用的一款定时任务调度框架，该框架将所有的任务调度信息都注册进了 [zookeeper](https://zookeeper.apache.org/) 中。

为了方便定位相关的问题，我去网上搜了 zookeeper 相关的图形化客户端，结果没有一款符合自己的需求，于是就干脆自己写一个算了。

> 该客户端是去年国庆假期写的，经历了从 Java8 到 Java11 的重构，该文章主要是对整个过程的一个回顾



## 从需求分析到实现

**面向用户**：zookeeper 用户

**软件名称**：PrettyZoo

**功能**：

	- 实时展示节点的变化
	- 支持对节点进行增删改查

交互与原型：

交互分两部，第一步启动页面要求用户输入 zookeeper 的服务地址，连接成功后会跳到节点操作页面

​	![原型](imgs/image-20200404222657485.png)

**技术选型**：

	- 语言采用 Java
	- UI 框架采用 Swing
	- 采用传统的分层架构
	- Zookeeper Client 采用 Apache Curator

**实现**：

![image-20200404223849266](imgs/image-20200404223849266.png)

​	![image-20200404223906151](imgs/image-20200404223906151.png)

**不足**：

	- 一次只能管理一个 zookeeper server
	- 由于是 Java8，运行需要安装额外的 JRE 或者 JDK
	- 颜值不足
	- 架构方面，内部膨胀导致层与层之间的边界不清晰

## 界面与架构的重构规划

重构主要是为了解决上一版的不足，而第一步就是分析产生这些问题的根本原因，再提出对应的解决方案

- 一次只能管理一个 server

  这是因为交互和设计上导致的，要解决该问题需要重新思考交互和软件的布局。

  经过网上的调研，最终决定采用 “三栏布局”，即解决了以前交互模式的分割感，又满足了需求上一次性管理多个 server。

  ![](imgs/image-20200404224726599.png)

  

- 需要额外安装 JDK

  既然选择了 Java 平台自然是逃离不掉 JRE 或 JDK 的，此时换其他平台的话成本太高，好在 Java9 的模块化（Jigsaw）为此提供了一定的解决方案，总之就是模块化方案使得最总打包的软件不再需要用户安装 JRE 或 JDK 都可以运行了。

  由于最新的 JDK LTS 版本是 11，所以直接升级到 Java11

- 颜值不足

  Java Swing 太过于重量级，干脆将 UI 方案切换到 JavaFX。

  JavaFX 可以通过 CSS 来调整软件的整体样式，这样在颜值方面可以用更轻量级的方式进行调整

- 分层架构导致的模块边界不清晰

  架构的改造对用户的感知几乎没有，但依然是一个重中之重的事情，因为随着功能的增加，每次改动的成本也会成倍的增加，最终的结果可能导致该软件无法维护下去。

  架构的整体框架依然是分层，不过加入了 `SPI` 层用于反转层与层之间的依赖关系，可以参见下图

  ![](imgs/prettyzoo-arch.jpg)

  

  这样就将传统的自上而下的依赖关系改成了一个插件式的架构。

## 重构的实现

整个软件重构的难点在于从 Java8 迁移到 Java11 的模块化平台，因为 Java 的模块化平台并不是兼容的，很多依赖的库还不支持模块化，这就需要依赖到第三方插件，将其转换为模块。

最终重构完成后，打包出来的软件接近 50M，不需要安装 Java 运行环境就可以运行，还是相当满意的。

下图展示了布局和交互

​	![](imgs/add-server.png)

![](imgs/add-node.png)

![](imgs/search-view.jpg)



## **总结**

该软件最终的成品我开源在了 Github上，详情点击 [ https://github.com/vran-dev/PrettyZoo]( https://github.com/vran-dev/PrettyZoo)，欢迎 star 和 issue。



## 参考

1. [PrettyZoo](https://github.com/vran-dev/PrettyZoo)
2. [OpenJFX](https://openjfx.io/)