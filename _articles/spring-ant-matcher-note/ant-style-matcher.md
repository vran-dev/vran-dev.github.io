---
title:  "Spring Ant-Style Matcher 笔记"
date:   2020-05-04 11:40:46 +0800
tags: [Java, Spring]
---

# Spring Ant-Style Matcher 笔记

在使用 Spring 的时候经常会写路径相关的配置，比如

- `@RequestMapping` 中配置请求路径
- `@ComponentScan` 中配置包路径
- 为 Interceptor 配置规则
- ......

而这些路径的配置规则中，我们经常会看见 *、** 等类似于正则表达式的符号，这实际上就是 Spring 的路径匹配规则。

Spring 默认有一个 `AntPathMatcher` 类用于处理这类路径的规则匹配， 它借鉴了 [Ant](https://ant.apache.org/) 的实现（如果你感兴趣的话，可以在[点击这里](https://ant.apache.org/manual/dirtasks.html#patterns)进行了解），这也是为什么称之为 **Ant-Style** 的原因。

> 题外话：Ant 是一个自动化构建工具，最开始是为了解决 Tomcat 的构建问题而诞生的

Ant 本身是定义了三种匹配规则，而 spring 在此基础之上额外扩展了一种，参考下面的表格

|                  |                                                         |
| ---------------- | ------------------------------------------------------- |
| ?                | 匹配单个字符                                            |
| *                | 匹配 0 个或多个字符                                     |
| **               | 匹配 0 个或多个目录                                     |
| {spring: [a-z]+} | 匹配正则表达式 [a-z]+, 将匹配后的路径作为 spring 的变量 |

下面的表格展示了一个具体的实例

|                                |                                                              |
| ------------------------------ | ------------------------------------------------------------ |
| com/t?st.jsp                   | 匹配 com/test.jsp，com/tast.jsp，com/txst.jsp                |
| com/*.jsp                      | 匹配 com 目录下所有以 .jsp 结尾的文件                        |
| com/**/test.jsp                | 匹配 com 所有层级的子目录下的 test.jsp 文件                  |
| org/springframework/*\*/\*.jsp | 匹配所有 org/springframework 的子目录下的 jsp 文件           |
| com/{filename:\\w+}.jsp        | 匹配 com 目录下的以字母命名的 jsp 文件，并将文件名赋值给变量 filename |

注意：pattern 和 path 的路径要么都是绝对路径，要么都是相对路径



## 参考

1、[org.springframework.util.AntPathMatcher](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/util/AntPathMatcher.html)

2、[Ant patterns](https://ant.apache.org/manual/dirtasks.html#patterns)

