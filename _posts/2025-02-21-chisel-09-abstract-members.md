---
title: "敏捷硬件开发语言Chisel与数字系统设计（九）：Scala的抽象成员"
date: 2025-02-21 09:00:00 +0800
description: "学习Scala中四种抽象成员（抽象类型type、抽象方法、抽象val/var字段），掌握抽象val字段的初始化顺序陷阱与lazy val惰性求值方案，以及Scala枚举的使用。"
categories:
  - FPGA
home_category: study-notes
series: chisel-and-scala-study
tags:
  - Chisel
  - Scala
  - 抽象成员
  - 枚举
  - lazy val
  - type关键字
article_kicker: CHISEL NOTE
word_count: 4.5k 字
read_time: 约 13 分钟
---

> 上一篇我们完成了类型参数化，从本篇开始学习Scala中抽象成员的概念。

## 9.1 抽象成员

Scala有**4种抽象成员**：抽象`val`字段、抽象`var`字段、抽象方法和抽象类型。声明如下：

```scala
scala> trait Abstract {
     | type T                          // 抽象类型
     | def transform(x: T): T          // 抽象方法
     | val initial: T                  // 抽象val字段
     | var current: T                  // 抽象var字段
     | }
// defined trait Abstract
```

### 抽象类型

抽象类型指的是用关键字 **`type`** 声明的一种类型，它是某个类或特质的成员但未给出定义。抽象类型允许子类在实现时指定具体类型，这是Scala类型系统的又一灵活之处。

### 抽象val字段

在不知道某个字段正确的值，但是明确知道在当前类的每个实例中该字段都会有一个**不可变更**的值时，可以使用抽象`val`字段。

抽象`val`字段与无参方法的区别：
- **抽象`val`字段**保证每次使用都返回一个**相同的值**
- **抽象方法**每次可能返回**不一样的值**

### 抽象var字段

抽象`var`字段和抽象`val`字段类似，但可以被**重新赋值**。

抽象类和特质不能直接使用`new`构造实例，只能由子类继承后实现它们。这种"在超类中声明、在子类中具体化"的模式是Scala众多设计模式的基础，也被Chisel广泛使用。

## 9.2 初始化抽象val字段

抽象`val`字段有时会承担**超类参数**的作用，它们允许程序员在子类中提供在超类中缺失的细节。

### 初始化顺序陷阱

在构造子类的实例对象时，**首先构造超类/超特质的组件，然后才轮到子类的剩余组件**。在这个过程中，如果需要访问超类/超特质的抽象`val`字段，会交出相应类型的**默认值**，而不是子类中的定义。例如：

```scala
scala> trait RationalTrait {
     | val numerArg: Int
     | val denomArg: Int
     | require(denomArg != 0)
     | }
// defined trait RationalTrait

scala> new RationalTrait {
     | val numerArg = 1
     | val denomArg = 1
     | }
java.lang.IllegalArgumentException: requirement failed
```

因为超特质构造时`denomArg`还是`Int`类型的默认值`0`，`require(denomArg != 0)`判断失败！

### 解决方案：惰性的val字段（lazy val）

把`val`字段定义成**惰性的**，可以让程序自己确定初始化顺序。在`val`字段前加上关键字 **`lazy`**——该字段只有在**首次被使用时才会进行初始化**：

```scala
scala> trait LazyRationalTrait {
     | val numerArg: Int
     | val denomArg: Int
     | lazy val numer = numerArg / g
     | lazy val denom = denomArg / g
     | override def toString = numer + "/" + denom
     | private lazy val g = {
     |   require(denomArg != 0)
     |   gcd(numerArg, denomArg)
     | }
     | private def gcd(a: Int, b: Int): Int =
     |   if (b == 0) a else gcd(b, a % b)
     | }
// defined trait LazyRationalTrait

scala> val x = 2
scala> new LazyRationalTrait {
     | val numerArg = 1 * x
     | val denomArg = 2 * x
     | }
val res37: LazyRationalTrait = 1/2
```

### `lazy val`的行为特点

- 如果是用表达式初始化，那就对表达式**求值并保存**
- 后续使用字段时都**复用保存的结果**，而不是每次都求值表达式
- 只有首次被使用时才会触发初始化，绕开了构造顺序问题

> **注意：** 原书中的"预初始化字段"（`new {定义} with 超类`）语法在Scala 3中已被废弃，`lazy val`是推荐的替代方案。

在Chisel中，`lazy val`广泛用于延迟硬件模块的实例化，在需要时才构建子模块。

## 9.3 Scala的枚举

Scala在标准库中提供了一个枚举类 **`scala.Enumeration`**，通过创建一个继承自这个类的子对象可以创建枚举。

### 基本枚举定义

```scala
scala> object Color extends Enumeration {
     | val Red, Green, Blue = Value
     | }
// defined object Color
```

`Enumeration`类定义了一个名为`Value`的内部类，以及同名的无参方法。该方法每次都返回内部类`Value`的全新实例。枚举值的真正类型是 **`Color.Value`**。

### 带名称的枚举值

方法`Value`有一个重载版本，接收字符串参数给枚举值关联特定名称：

```scala
scala> object Direction extends Enumeration {
     | val North = Value("N")
     | val East = Value("E")
     | val South = Value("S")
     | val West = Value("W")
     | }
// defined object Direction

scala> Color.values
val res38: Color.ValueSet = Color.ValueSet(Red, Green, Blue)

scala> Direction.values
val res39: Direction.ValueSet = Direction.ValueSet(N, E, S, W)
```

### 通过编号索引枚举值

枚举值从**0**开始编号，可以通过`对象名(编号)`返回对应枚举值的名称：

```scala
scala> Color(2)
val res40: Color.Value = Blue

scala> Direction(0)
val res41: Direction.Value = N
```

在Chisel中，枚举经常用于表示硬件的**状态机状态**、**指令操作码**等有限集合。

> 下一篇（最后一篇）：隐式转换与隐式参数
