---
title: "敏捷硬件开发语言Chisel与数字系统设计（七）：Scala的模式匹配"
date: 2025-02-13 09:00:00 +0800
description: "深度学习Scala最强大的特性之一——模式匹配：样例类、八种模式类型、模式守卫、密封类、可选值（Option）、提取器以及偏函数（PartialFunction）。"
categories:
  - FPGA
home_category: study-notes
series: chisel-and-scala-study
tags:
  - Chisel
  - Scala
  - 模式匹配
  - 样例类
  - 偏函数
  - Option
article_kicker: CHISEL NOTE
word_count: 7.5k 字
read_time: 约 22 分钟
---

> 上一篇我们学习了内建控制结构，最后提到了match表达式。本篇正式进入Scala最强大的特性之一——模式匹配。

## 7.1 样例类（case class）和样例对象

定义类时，如果在最前面加上关键字`case`，则这个类就被称为**样例类**。Scala编译器自动对样例类添加一些语法便利：

1. **工厂方法：** 可以通过`类名(参数)`构造对象，无需`new`；
2. **公有字段：** 参数列表的每个参数都隐式获得`val`前缀；
3. **自然实现：** 自动实现`toString`、`hashCode`和`equals`方法；
4. **copy方法：** 用于构造与旧对象只有某些字段不一样的新对象。

```scala
scala> case class Students(name: String, score: Int)
// defined case class Students

scala> val stu1 = Students("Alice", 100)       // 无需new
val stu1: Students = Students(Alice,100)

scala> stu1.name
val res4: String = Alice

scala> val stu2 = stu1.copy()                  // 完整复制
scala> stu2 == stu1
val res6: Boolean = true

scala> val stu3 = stu1.copy(name = "Bob")      // 只修改name
scala> stu3 == stu1
val res7: Boolean = false
```

样例对象类似样例类，在定义单例对象时加上`case`关键字。

## 7.2 模式匹配（Pattern Matching）

语法：**`选择器 match {可选分支}`**

可选分支的定义：`case 模式 => 表达式`

**三条重要规则：**
1. `match`是一个**表达式**，可以返回值；
2. 可选分支存在**优先级**，按编写顺序匹配，只有第一个匹配成功的被选中；
3. 要确保**至少有一个模式**匹配成功，否则报错。

## 7.3 模式的种类

### 7.3.1 通配模式

用下划线`_`表示，匹配**任何**对象，通常放在末尾用于缺省，相当于`switch`中的`default`。

```scala
scala> def test(x: Any) = x match {
     | case List(1, 2, _) => true
     | case _ => false
     | }
scala> test(List(1, 2, 3))
val res8: Boolean = true
```

### 7.3.2 常量模式

使用常量作为模式，只能匹配自己。任何字面量、`val`类型的变量、单例对象都可以作为常量模式。如`Nil`可用于匹配空列表。

### 7.3.3 变量模式

变量模式是一个变量名，匹配任何对象，并且与成功匹配的输入对象**绑定**，可以在表达式中进一步使用。

**词法区分规则：** 以小写字母开头的简单名称被当作**变量模式**，其他引用都是**常量模式**。如果想用小写名称引用常量，可以用反引号包起来：`` `somethingElse` ``

### 7.3.4 构造方法模式

把样例类的构造方法作为模式，形式为`名称(模式)`。**Scala的模式支持深度匹配**——括号中的模式可以是任何模式，包括嵌套的构造方法模式：

```scala
scala> def test5(x: Any) = x match {
     | case B("abc", e, A(10)) => e + 1
     | case _ =>
     | }
```

- `"abc"`是常量模式
- `e`是变量模式，绑定第二个参数
- `A(10)`是嵌套的构造方法模式——第三个参数必须是以10为参数构造的A的对象

### 7.3.5 序列模式

序列类型（如`List`或`Array`）可用于模式匹配。`_*`放在最后可以匹配任意个元素：

```scala
scala> def test6(x: Any) = x match {
     | case Array(1, _*) => "OK"
     | case _ => "Oops!"
     | }
scala> test6(Array(1, 2, 3))
val res24: String = OK
```

### 7.3.6 元组模式

元组可用于模式匹配，在圆括号中可以包含任意模式：

```scala
scala> def test7(x: Any) = x match {
     | case (1, e, "OK") => "OK, e = " + e
     | case _ => "Oops!"
     | }
scala> test7(1, 10, "OK")
val res26: String = OK, e = 10
```

### 7.3.7 带类型的模式

模式定义时可以声明具体的数据类型，替代类型测试和类型转换：

```scala
scala> def test8(x: Any) = x match {
     | case s: String => s.length
     | case m: Map[_, _] => m.size
     | case _ => -1
     | }
scala> test8("OK")
val res27: Int = 2
```

> **注意：** 由于Scala采用擦除式的泛型，运行时无法保留类型参数的具体信息，所以只能匹配到`Map[_, _]`这个级别。

### 7.3.8 变量绑定模式

形式是 **`变量名 @ 模式`**，匹配规则与绑定前相同，匹配成功后把输入对象的相应部分与变量进行绑定：

```scala
scala> def test9(x: Any) = x match {
     | case (1, 2, e @ 3) => e
     | case _ => 0
     | }
scala> test9(1, 2, 3)
val res29: Any = 3
```

### 八种模式总结

| 模式         | 语法                  | 用途                     |
| ------------ | --------------------- | ------------------------ |
| 通配模式     | `_`                   | 匹配任意、忽略局部       |
| 常量模式     | 字面量/val/object     | 精确匹配特定值           |
| 变量模式     | 小写变量名            | 匹配任意并绑定           |
| 构造方法模式 | `类名(模式...)`       | 匹配样例类，支持深度嵌套 |
| 序列模式     | `List(模式, _*)`      | 匹配定长或不定长序列     |
| 元组模式     | `(模式, 模式)`        | 匹配元组                 |
| 带类型模式   | `变量: 类型`          | 类型测试与转换           |
| 变量绑定模式 | `变量 @ 模式`         | 匹配并绑定子结构         |

## 7.4 模式守卫（Pattern Guard）

模式守卫出现在模式之后，是一条用`if`开头的语句。模式守卫可以是任意的布尔表达式，只有守卫返回`true`时才能匹配成功：

```scala
case i: Int if i > 0 => ...
case s: String if s(0) == 'a' => ...
case (x, y) if x == y => ...
```

## 7.5 密封类（sealed class）

如果在`class`前面加上关键字`sealed`，那么这个类是**密封类**。密封类只能在**同一个文件**中定义子类，不能在文件之外被别的类继承。要使用模式匹配，最好把顶层的基类做成密封类——这样编译器可以检查`match`是否覆盖了所有可能子类。

## 7.6 可选值（Option）

可选值就是类型为 **`Option[T]`** 的值。`Option`是一个密封抽象类，有两个子类：
- **`Some(x)`**：表示"有值"
- **`None`**：表示"无值"

`Option[T]`与`T`是两个完全无关的类型，赋值时需要做类型转换（最常见的方式就是模式匹配），在编译期就进行判空处理。常用方法：
- `isDefined`：`None`返回`false`，`Some`返回`true`
- `get`：把`Some(x)`中的`x`返回，`None`则报错

## 7.7 模式匹配的另类用法

对于提取器，可以通过`var/val 对象名(模式) = 值`的方式使用模式匹配，常用于定义变量：

```scala
scala> val Array(x, y, _*) = Array(-1, 1, 233)
val x: Int = -1
val y: Int = 1

scala> val a :: 10 :: _ = List(999, 10)
val a: Int = 999
```

## 7.8 偏函数（PartialFunction）

偏函数的作用是划分一个输入参数的可行域，在可行域内对入参执行一种操作。偏函数有两个抽象方法：`isDefinedAt`判断入参是否合法，`apply`是函数体。

**用case语句定义偏函数：**

```scala
scala> val isInt1: PartialFunction[Any, String] = {
     | case x: Int => x + " is a Int."
     | case _ => "else."
     | }
val isInt1: PartialFunction[Any, String] = <function1>

scala> isInt1(1)
val res30: String = 1 is a Int.

scala> isInt1("1")
val res31: String = else.
```

广义上**一个`case`语句就是一个偏函数**，所以才可以用于模式匹配。偏函数在Scala集合库中被广泛使用，比如`collect`方法就接收一个偏函数作为参数。

> 下一篇：Scala的类型参数化
