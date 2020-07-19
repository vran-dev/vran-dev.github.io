---
title: "真的学不动了：刚学会 Type classes Pattern,  就过时了？"
date: 2020-07-19 10:45:00 +0800
tag: [Scala, 编程语言]
---

# 真的学不动了：刚学会 Type classes Pattern,  就过时了？



## 引言

Type classes pattern 可谓是 Scala 中的屠龙技之一，然而这一招式随着 Scala3 的发布也产生了巨大的变化......

如果你是 Java 开发者，即使不会 Scala，我也希望你继续阅读下去，起码你能知道 JVM 上的语言原来也能玩出这么多花样。

> 如果你不知道 Type classes 是什么，推荐你阅读上一篇文章[《真的学不动了: 除了 class , 也该了解 Type classes 了》](https://blog.cc1234.cc/articles/typeclasses-1/typeclasses-1.html)，本文主要是描述了 Type classes 的实现在 Scala3 中的变化。



## 回顾 Scala2 与 Type classes

在 Scala2 中要实现 Type classes，得借助于强大的**隐式系统**，而由于没有直接的概念和语法支持，在 Scala 中也被称之为 Type classes Pattern。

下面用 Scala 中的 Type classes Pattern 来改造一下  `Comparator`

```scala
trait Comparator[T] {
  def compare(a: T, b: T): Int
}

object ComparatorInstances {
  implicit val intComparator = new Comparator[Int] {
    override def compare(a: Int, b: Int) = a.compareTo(b)
  }
  
  implicit def listComparator[T](implicit cmp: Comparator[T]) = new Comparator[List[T]] {
    override def compare(a: List[T], b: List[T]): Int = {
      (a, b) match {
        // 连个都为空
        case (Nil, Nil) => 0
				// 某一个为空
        case (Nil, _) => 1
        case (_, Nil) => -1
        // 两个都不为空，就比较头结点
        case (l, r) =>
          cmp.compare(l.head, r.head) match {
            // List.tail 表示除了头结点以外的结点
            case 0 => compare(l.tail, r.tail)
            case num => num
          }
      }
    }
  }
}

object Same {
  def apply[T](a: T, b: T)(implicit cmp: Comparator[T]) = cmp.compare(a, b) == 0
}
```

稍微要说一下的就是  `implicit def listComparator[T](implicit cmp: Comparator[T])`  这个函数了，称之为隐式转换方法。它的作用就是当你需要 Comparator[List[T]] 的实例时，编译器会在作用域内找到 Comparator[T] 的实例，然后将其转换为 Comparator[List[T]] 的实例。

如果没有这个方法

- 处理 List[Int] 时需要定义一个 Comparator[List[Int]] 实例
- 处理 List[String] 时需要定义一个 Comparator[List[String]]  的实例
- ......

有了该隐式方法以后

- 处理 List[Int] 时，编译器会找 Comparator[Int] 的实例， 然后根据该方法得到 Comparator[List[Int]] 实例进行处理
- 处理 List[String] 时， 编译器会找 Comparator[String] 的实例， 然后根据该方法得到 Comparator[List[String]] 实例进行处理
- ......

其实最主要的作用就是在处理高阶类型时能复用已有的 Type class 实例。

最后再来测试一下， 调用前需要先将 Type class 实例都导入（正常的 import 语法）到作用域内

```scala
import ComparatorInstances._
Same(1, 2)
Same(List(1, 2), List(1, 2))
```

在实现 Scala 中实现 type classes，很多时候还会使用**函数扩展**来简化调用，Scala 中的函数扩展是基于隐式类来实现的。

比如下面的代码就是在不修改 T 类型的情况下为 其扩展了 `<` 、 `>` 和 `isSameTo`  三个个函数

```scala
object ComparatorInstances {
  // ...

  implicit class Ops[T](a: T)(implicit cmp: Comparator[T]) {
    
    def isSameTo(b: T) = cmp.compare(a, b) == 0
    
    def <(b: T) = cmp.compare(a, b) < 0

    def >(b: T) = cmp.compare(a, b) > 0
  }

}
```

结合 Type classes Pattern 和函数扩展，使得调用更加自然

```scala
import ComparatorInstances._
Same(List(1, 2), List(1, 2))
// 函数扩展为 List 扩展了 < 函数
List(1,2).<(List(1, 3))
// 也可以省略 . 和参数括号
List(1, 2) < List(1, 3)
// 函数扩展
List(1,2).isSameTo(List(2, 3))
List(1, 2) isSameTo List(2, 3)

```



虽然借助于隐式系统实现了 Type classes, 但还是有刺可以挑的

- 语言层面的支持不友好

  引入了晦涩的隐式转换等概念，而这也是导致 Scala 的 Type classes 难以掌握的原因之一。

  难道就不能有更友好的语法支持吗？

- 实例导入冲突

  我明明只是导入了某些依赖，但这个依赖里面却有 type class 实例，它覆盖了我期望调用的实例，造成了非常隐秘的 BUG。

  （从另一个角度来说，这个**实例可覆盖**又是一个灵活性的体现）

  难道就不能区分普通的 import 和 type class 实例的 import 吗？

> 对于使用隐式系统实现 Type classes 的缺点，还可以参考  《[The Limitations of Type Classes as Subtyped Implicits》](https://www.adelbertc.com/publications/typeclasses-scala17.pdf)

下面就来看看这些 Scala3 是如何解决这些痛点的吧。



## 探秘 Scala3 与 Type classes

在  Scala3 中，再也不用为 Type classes 和 implicit 的概念而感到混乱了， implicit 被全新的 `Given`、`Using` 、`Extension Method` 等特性所替代。

> 要使用 Scala3 的语法的话，需要到 https://dotty.epfl.ch/ 下载 Dotty 编译器，用以替代 Scalac 编译器。

Talk is Cheap，我们用 Scala3 的语法再次实现一遍 Comparator  的 Type classes。

首先是 Type class 的定义， 这一块仍是基于 trait 和泛型，没有变化。

```scala
trait Comparator[T] {
  def compare[T](a: T, b: T): Int
}
```



### Given

然后就是实现 Type class 实例了，用全新的语法 `given` 来实现。

```scala
given intCompare as Comparator[Int] {
  override def compare(a: Int, b: Int) = a.compareTo(b)
}
```

given 后面跟实例名称， as 可以指定类型。

> 你可以认为就是它一种新的初始化语法，只不过是特地为了和后面的 using、given import 配合使用的

given 支持命名构造和匿名构造，当使用匿名构造时编译器会为实例自动生成一个可读的名称，下面展示了一个完整的实例构造

```scala
object ComparatorInstances {

  // 命名构造
  given intComparator as  Comparator[Int] {
    override def compare(a: Int, b: Int) = a.compareTo(b)
  }
  
  // 匿名构造 自动生成可读的实例名称： given_Comparator_String
  given Comparator[String] {
    override def compare(a: String, b: String) = a.compareTo(b) 
  }

  // 高阶函数命名构造
  given doubleComparator as Comparator[Double] = instance[Double]((a, b) => a.compareTo(b))
  
  // 高阶函数匿名构造
  given Comparator[Float] = instance[Float]((a, b) => a.compareTo(b))
  
  private def instance[T](func: (T, T) => Int) = new Comparator[T] {
    override def compare(a: T, b: T) = func(a, b)
  }
}
```



### Using

对外暴露的 API 定义也不用再使用隐式参数了，转而使用 using 替代。

被 using 修饰的参数称之为**上下文参数**，该参数可以不用手动传入，编译器会在作用域内寻找匹配的 given 实例传入。

```scala
object Same {
  // 命名上下文参数
  def apply[T](a: T, b: T)(using cmp: Comparator[T]) = cmp.compare(a, b) == 0
}
```



### Given Import

现在再通过 `import ComparatorInstances._` 导入实例已经行不通了，在 Scala3 中已经将普通的导入和 given 实例的导入区分开了。

given 实例的导入称之为  `given import`, 它的语法也很简单

```scala
// 导入单个实例
import xxx.{given aInstance}
// 通配符 导入所有实例
import xxx.{given _}
```

最后验证一下前面的实现

```scala
// 导入所有 given 实例就用占位符 _
// 也可以只导入某一个实例
import ComparatorInstances.{given _}

Same(1, 2)
Same(1.2F, 2.2F)
Same(1.2D, 2.2D)
Same("ok", "ok")

```

结合  given、using 和 given import 已经实现了 Type classes，相信你也发现了，相较于 Scala2 并没有减少多少代码量，但是语法含义却是更加清晰了，再也不用面对晦涩的隐式系统了。



### Extension Method

最后再来看一下  `implicit class` ，光从名字上我们很难和**函数扩展**关联起来，在 Scala3 中，你可以完全忘记 implicit class 这样的概念了，现在统一称之为 `Extension method`，翻译过来就是函数扩展。

`Extension method`  是一种新的函数定义语法，参考下面的代码

```scala
trait Comparator[T] {
  def compare[T](a: T, b: T): Int
  
  def (a: T) >(b: T) = compare(a, b) > 0
  def (a: T) <(b: T) = compare(a, b) < 0
}
```

 `>` 和 `<` 就是全新的函数扩展语法，方法名前的参数 `a: T` 表示为 类型 T 扩展函数，该函数的参数为 `b:T`。

```scala
// 为字符串扩展了 > 函数
"ok".>("hello")
// 也可以省略 . 和括号
"ok"  > "hello"
```

再基于全新的`函数扩展`和 using 来实现一个 List[T] 的 Type class 实例吧

```scala
object ComparatorInstances {
  
  // ......
  
  def [T](a: List[T]) listInstance(using cmp: Comparator[T]): Comparator[List[T]] = new Comparator[List[T]] {
    override def compare(a: List[T], b: List[T]): Int = {
      (a, b) match {
        case (Nil, Nil) => 0
        case (Nil, _) => 1
        case (_, Nil) => -1
        case (l, r) =>
          cmp.compare(l.head, r.head) match {
            case 0 => compare(l.tail, r.tail)
            case num => num
          }
      }
    }
  }
}
```



### 旧瓶装新酒？

如果你已经跨过了隐式系统那道门槛，这样的语法改变并不会给你带来多大的惊喜，感觉就像酒瓶装新酒一样，但实际上能把复杂的东西做的简单一些，这才是最难得的。



## 比较 Scala2 与 Scala3

其实在 Scala3 中 `given`、`using`、`extension method` 这些被统一称之为**抽象上下文**（Contextual Abstractions），而这些概念我们都可以和 Scala2 做一个对比，形成概念迁移。

1. `given` 可以当做无参的 `implicit object`

```scala
given intComparator as Comparator[INT] { ... }
```

```scala
implicit object intComparator Comparator Ord[Int] { ... }
```



2. using 可以当做以前的**隐式参数**

```scala
def same[T](a: T, b: T)(using cmp: Comparator[T]) {...}
```

```scala
def same[T](a: T, b: T)(implicit cmp: Comparator[T]) {...}
```



3. extension method 可以当做以前的**隐式类**

```scala
trait Comparator[T] {
  def compare(a: T, b: T): Int

  def (a: T) >(b: T) = compare(a, b) > 0
  def (a: T) <(b: T) = compare(a, b) < 0
}
```

```scala
implicit class ComparatorOps[T](a: T)(implicit cmp: Comparator[T]) {
    def <(b: T) = cmp.compare(a, b) < 0

    def >(b: T) = cmp.compare(a, b) > 0
}
```





## 一些期待

开发者们一直喜欢拿 Scala 和 Haskell、Java 作比较，甚至在社区都形成了这样的印象

- A worse Haskell for the JVM
- A better Java

而对于 Scala 来说，它就想做自己，这样的愿景在 Scala3 中似乎也已经触手可及了。



## 参考

1. [Dotty: A next-generation compiler for Scala](https://dotty.epfl.ch/)
2. [The Limitations of Type Classes as Subtyped Implicits](https://www.adelbertc.com/publications/typeclasses-scala17.pdf)
3. [Scala Type Classes comparison](https://medium.com/se-notes-by-alexey-novakov/scala-type-classes-comparison-28b76ce1f37a)
4. [真的学不动了: 除了 class , 也该了解 Type classes 了](https://blog.cc1234.cc/articles/typeclasses-1/typeclasses-1.html)
5. [preparing_for_scala_3_PPT](https://www.slidestalk.com/u39/preparing_for_scala_3_light_bend_vsjjhm)