# 你不知道的多态 - Typeclass



## 回顾

在上一篇文章 [《这些多态，了解一下》](https://blog.cc1234.cc/articles/polymorphism/polymorphism.html) 中有提到一个 ad-hoc polymorphism（特设多态），最典型的实例就是函数重载和操作符重载，而除了这两个之外，typeclass 也是属于 ad-hoc polymorphism 这一分类，它也正是本文要讨论的主题。

typeclass 一般译作**类型类**，Haskell 是最早实现该特性的编程语言，



## 从 Java 的 Interface 开始

  



## Haskell 与 Typeclass





## Scala 与 Typeclass

在 Scala 中， typeclass 不是一个语言特性，而是一个模式（类似于设计模式），通过 trait 和隐式参数来实现。

> Trait 类似于 Java 的接口，但更加强大



而在 Scala 中实现 typeclass Pattern 的步骤也很简单

```scala
trait Paint[T] {
  def paint(t: T)
}
```



```scala
object Paint {

  implicit val strPaint = instance[String](str => println(s"string = $str"))
  
  implicit val intPaint = instance[Int](int => println(s"int = $int"))

  def instance[A](f: A => Unit): Paint[A] = new Paint[A] {
    override def paint(t: A): Unit = f(t)
  }
  
  def paint[T: Paint](content: T) = implicitly[Paint[T]].paint(content)
}
```



```scala
Paint.paint("hello world");
Paint.paint(123456);
```



```scala
// 编译错误：no implicits found for parameter
Paint.paint(1.23);
```











## 参考

1. [Type class](https://en.wikipedia.org/wiki/Type_class)
2. 