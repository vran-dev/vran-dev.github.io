---
title: " Scala 与 Algebraic data type"
date: 2020-08-18 20:50:00 +0800
tag: [Scala, 编程语言]
---



# Scala 与 Algebraic data type

## 前言

在计算机编程中，数据类型（Data type）是一个基本概念，它是数据的一个属性，编译器或解释器可以根据该属性知道开发者打算如何使用该数据。

常见的数据类型有**基本数据类型**（Primitive data types）和**组合数据类型**（Composite Types）。

基本数据类型一般都是由编程语言内置的，比如 Integer、Float、Boolean、Char 等，而组合数据类型是从基本类型派生出来的，一个组合类型可以由基本类型组成，也可以由「其他组合类型 + 基本类型」组成。

比如在 Java 中通过 class 关键字就可以创建一个组合类型

```java
public class User {
  private String name; // 姓名
  private Integer age; // 年龄
  private List<User> familyMembers; // 家庭成员
}
```



Scala 中的 `case class` 也是组合类型

```scala
case class User(name: String, age: Integer, familyMembers: List[User])
```



组合类型又可以分为很多种，而本文要聊的 **Algebraic data type** （以下简称 ADT 或 ADTs）就是其中之一

注：Algebraic data type，一般译作*代数数据类型*



## Algebraic data type

Algebraic data type 在函数式编程语言中是一个很常见的概念，要理解 ADT ，得先明白 Algebra（代数）的含义。

所谓“代数”，指的就是用符号代替元素和运算，在 ADT 中，代数就是 **product** 和 **sum** （即积与和）。

这里的 product 和 sum 是一种**组合既有类型从而产生新类型的运算**：

- 和：表示选择，即 A ｜ B，读作 A 或 B
- 积：表示组合，即  (A, B)，读作 A 和 B

基于这两种组合方式，ADT 又被划分为了 Sum type 和 Product type。

为了理解 Sum type 和 Product type，我们需要先了解一下**类型计数**从而找到代数与类型之间的联系。



### 类型计数

类型计数就是将类型当作一个集合，然后去统计该集合中可能的元素数量，比如

| 类型                                    | 值                       | 计数     |
| --------------------------------------- | ------------------------ | -------- |
| boolean                                 | true \| false            | 2        |
| int                                     | -2147483648 ~ 2147483647 | 2^32     |
| Unit                                    | Unit                     | 1        |
| Nothing                                 |                          | 0        |
| String                                  |                          | infinite |
| 注：Unit 和 Nothing 都是 Scala 中的类型 |                          |          |



### Sum type

Sum type，也称之为 Tagged Union，引用一下 wiki 的定义

> Sum type is a [data structure](https://en.wikipedia.org/wiki/Data_structure) used to hold a value that could take on several different, but fixed, types

大意就是 Sum type 是从几个固定的类型中选取一个来构造值并进行保存的数据结构。

有点绕，没关系，我们可以通过实际的代码来更直观的感受一下 Sum type。

在 Scala 中要实现 Sum type 得借助与 `sealed` 关键字：sealed 一般用于修饰 trait，编译器会强制要求该 trait 与其实现类在同一个文件下，这样编译器可以获得 trait 的所有子类型信息。

比如下面的代码，Circle、Rectangle 和 Square 类都继承了 Shape 特质：

```scala
sealed trait Shape {}

class Circle extends Shape {}

class Rectangle extends Shape {}

class Square extends Shape {}
```



如果将 Shape 当作一个集合的话，它的元素就是 Circle、Rectangle 和 Square，那么这个集合的 size = 3，这表示了 Shape 可能的值

```scala
Shape = Circle + Rectangle + Square
      = 1 + 1 + 1
      = 3
```

但是对于每一个 Shape 对象而言，它只可能是 Circle、Rectangle 或 Square之一，这就是前面提到的**选择**。



用继承来模拟 Sum type 会让人感觉 Sum type 就是继承，不如再来看一下在没有继承的 Haskell 中 Sum type 是怎样实现的。

下面的 Haskell 代码通过 data 关键字创建了一个新类型 Shape，该类型的值被限定为了  Circle、Rectangle 和 Square 之一，更直观的感受到 Sum type 所表达的选择的含义：

```haskell
data Shape = Circle | Rectangle | Square
```

注：Circle、Rectangle、Square 在这里一般被称之为值构造器



### Product type

相较于 Sum type， Product type 其实更容易理解一些，它就是组合多个类型并同时持有这些类型对应的值的结构。

在 Scala 中，Tuple 和 case class 可以算是很典型的 Product type。

以**二元组** `(Boolean, Boolean) ` 为例，两个元素的类型都是 Boolean，那么这个二元组可能的组合就有 4 种

```scala
(true, true)
(true, false)
(false, false)
(false, true)
```

这个推算很简单，Boolean 的值可能为 true 或 false，即 2 种可能性，那么两个 Boolean 组合就是 2 * 2 = 4 ：

```scala
(Boolean, Boolean) = (true | false, true | false)
                   = (2, 2)
                   = 2 * 2
                   = 4
```

其实就是一个笛卡尔积的计算。

## ADT 与 模式匹配

了解了 ADT 以后，难免会去思考它的作用，其实 ADT 的一个很明显的优势就是结合**模式匹配**，通过模式匹配可以结构 ADT，可以让我们写出更优雅简洁的代码。

> 解构
>
> /jiěgòu/
>
> *动词*
>
> 对事物的内容结构进行剖析。

在模式匹配中，编译器可以检查 ADT 的所有组合情况是否都已经覆盖到了，如果没有覆盖全的话编译器会发出一个警告信息，减少错误的产生（在 Scala 中叫 exhaustiveness checking）。

以 Scala 为例，我们可以使用二元组来自定义 and 和 or 运算，然后通过模式匹配求解，这个过程就是针对 Product type 的解构：

注：_ 代表通配符，表示任意值

```scala
def and(a: Boolean, b: Boolean): Boolean = {
  (a, b) match {
    case (true, true) => true
    case (false, false) => true
    case (true, false) => false
    case (false, true) => false
  }
}

def or(a: Boolean, b: Boolean): Boolean = {
  (a, b) match {
    case (true, _) => true
    case (_, true) => true
    case (_, _) => false
  }
}
```



再来一个复杂点的例子，将前面的 Shape 扩展一下（结合了 Sum type 与 Product type）：

```scala
sealed trait Shape {}

// 通过半径构建 circle
case class Circle(radius: Double) extends Shape

// 通过长和宽构建矩形
case class Rectangle(width: Double, length: Double) extends Shape

// 通过宽构建正方形
case class Square(width: Double) extends Shape
```



再来定义一个将 Shape 转为字符串的方法：

```scala
def showShape(shape: Shape): String = {
  shape match {
    // 匹配半径为 3.14 的圆
    case Circle(3.14) => "perfect small circle"
    // 匹配半径小于 10 的圆
    case Circle(a) if a < 10 => "small circle"
    // 匹配任意半径的圆
    case Circle(_) => "circle"
    // 匹配任意长宽的矩形
    case Rectangle(_, _) => "rectangle"
    // 匹配任意宽度的正方形
    case Square(_) => "square"
  }
}
```



## 最后

学习这样的知识蛮有趣的，这样以后在讨论 A 和 B 语言有些相似特性时，就不用再去纠结 A 抄 B ， B 抄 A 的争论，因为它们可能都是基于同一个理论的不同实现而已。

另外，在知乎（[题：代数数据类型是什么？](https://www.zhihu.com/question/24460419)）上也有大佬认为 Haskell 和 Scala 中的  Algebraic data type 并不算是真正的 Algebraic data type，因为它们都没有 **strict positivity condition** （这已经触及到我的知识盲区了），如果有哪位大佬对此有所了解的话，忘不吝赐教一下。



## 参考

1. [WIKI：Algebraic_data_type](https://en.wikipedia.org/wiki/Algebraic_data_type)
2. [Haskell: Algebraic_data_type](https://wiki.haskell.org/Algebraic_data_type)
3. [如何在 Scala 中利用 ADT 良好地组织业务](http://scala.cool/2017/03/how-to-use-algebraic-data-type-in-scala-development/)，By Shaw
4. [Algebraic Data Type 及其在 Haskell 和 Scala 中的表现 ](http://songkun.me/2018/07/12/2018-07-12-adt-in-haskell-and-scala/)，By Song Kun
5. [The algebra (and calculus!) of algebraic data types](https://codewords.recurse.com/issues/three/algebra-and-calculus-of-algebraic-data-types)，By Joel Burget
6. [Why Sum Types Matter in Haskell](https://medium.com/@willkurt/why-sum-types-matter-in-haskell-ba2c1ab4e372)，By Will Kurt
7. [知乎：什么才是代数？](https://www.zhihu.com/question/50576405)

