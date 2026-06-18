---
title: "敏捷硬件开发语言Chisel与数字系统设计（六）：Scala的内建控制结构"
date: 2025-02-09 09:00:00 +0800
description: "系统学习Scala的内建控制结构：if表达式（可返回值）、while循环、函数式风格的for表达式与yield、try-catch-finally异常处理、match表达式等。"
categories:
  - FPGA
home_category: reading-notes
series: chisel-and-scala-study
tags:
  - Chisel
  - Scala
  - 控制结构
  - for表达式
  - 异常处理
  - match
cover_image: /assets/images/chisel-scala/cover.jpg
article_kicker: CHISEL NOTE
word_count: 4.5k 字
read_time: 约 13 分钟
---

> 📖 **本书来源：**《敏捷硬件开发语言Chisel与数字系统设计》，梁峰、吴斌、张国和、雷冰洁、雷绍充 编著，电子工业出版社，2022年
>
> 上几篇我们学习了Scala丰富的集合库，从本篇开始学习Scala的内建控制结构。

## 6.1 if表达式

和大部分编程语言的`if`表达式使用上没有区别，单句话不需要加花括号，多个语句需要加花括号。**Scala中`if`是表达式而非语句**——它总是返回一个值：

```scala
scala> def whichInt(x:Int) = {
     | if(x == 0) "Zero"
     | else if(x > 0) "Positive Number"
     | else "Negative Number"
     | }
def whichInt(x: Int): String

scala> whichInt(-1)
val res33: String = Negative Number
```

## 6.2 while循环

`while`循环的使用方法与C语言类似，有`while`型循环和`do...while`型循环：

```scala
scala> def fac_loop(num: Int): Int = {
     | var res: Int = 1
     | var num1: Int = num
     | while(num1 != 0) {
     |   res *= num1
     |   num1 -= 1
     | }
     | res
     | }
def fac_loop(num: Int): Int

scala> fac_loop(5)
val res34: Int = 120
```

`while`的风格是**指令式**的。`if`被称为**表达式**（可以返回值），而`while`被称为**循环**（返回类型是`Unit`）。实际上所有的`while`循环都可以通过函数式风格的递归来实现：

```scala
scala> def fac(num: Int): Int =
     | if (num == 1) 1 else num * fac(num - 1)
def fac(num: Int): Int

scala> fac(5)
val res35: Int = 120
```

## 6.3 for表达式与for循环

实现循环，推荐使用`for`表达式。`for`表达式是**函数式风格**的。它的一般形式如下：

```
for(seq) yield expression
```

整个`for`表达式算一个语句。`seq`代表一个序列，能放进`for`表达式中的对象必须是一个**可迭代的集合**（列表、数组、映射、区间、迭代器、流和所有的集），它们都混入了特质`Iterable`。`yield`表示把前面序列中符合条件的元素拿出来，逐个应用到后面的表达式中，得到的所有结果按顺序形成一个新的集合对象。

### seq的结构

```scala
for {
  p <- persons               // 生成器（Generator）
  n = p.name                 // 定义（Definition）
  if(n startsWith "To")      // 过滤器（Filter）
} yield n
```

- **生成器** `p <- persons`：右侧是一个可迭代的集合对象，把它的每个元素逐一拿出来与左侧的模式进行匹配；
- **定义** `n = p.name`：一个赋值语句，可有可无；
- **过滤器** `if(...)`：只有表达式为真时元素才会向后传递。

### 完整示例

```scala
class Person(val name: String)

object Alice extends Person("Alice")
object Tom extends Person("Tom")
object Tony extends Person("Tony")
object Bob extends Person("Bob")
object Todd extends Person("Todd")

val persons = List(Alice, Tom, Tony, Bob, Todd)

val To = for {
  p <- persons
  n = p.name
  if(n.startsWith("To"))
} yield n

@main def test() = println(To)
```

编译输出：`List(Tom, Tony, Todd)`

### 嵌套for循环

如果一个`for`表达式中有**多个生成器**，就构成嵌套的`for`循环。以99乘法表为例：

```scala
scala> for {
     | i <- 1 to 9
     | j <- 1 to 9
     | } yield i * j
val res0: IndexedSeq[Int] = Vector(1, 2, 3, ...)
```

### for循环（无yield）

如果只是把每个元素应用到一个返回`Unit`的表达式，则是一个**for循环**而非**for表达式**，关键字`yield`也可以省略：

```scala
scala> var sum = 0
scala> for(x <- 1 to 100) sum += x
scala> sum
val res1: Int = 5050
```

## 6.4 用try表达式处理异常

### 6.4.1 抛出一个异常

可以用`new`构造一个异常对象，并用关键字`throw`手动抛出异常：

```scala
scala> throw new IllegalArgumentException
java.lang.IllegalArgumentException
```

### 6.4.2 try-catch

`try`后面用花括号包含任意条代码，当这些代码产生异常时，被`catch`捕获并进行相应处理：

```scala
scala> def intDivision(x: Int, y: Int) = {
     | try {
     |   x / y
     | } catch {
     |   case ex: ArithmeticException => println("The divisor is Zero!")
     | }
     | }
def intDivision(x: Int, y: Int): Int | Unit

scala> intDivision(10, 0)
The divisor is Zero!
```

### 6.4.3 finally

`try`表达式的完整形式是`try-catch-finally`，无论有没有异常产生，`finally`中的代码都会执行。通常它的作用是执行一些清理工作，比如关闭文件。

## 6.5 match表达式

`match`表达式作用类似于`switch`，是把作用对象与定义的模式逐个比较，按匹配的模式执行相应的操作：

```scala
scala> def something(x: String) = x match {
     | case "Apple" => println("Fruit!")
     | case "Tomato" => println("Vegetable!")
     | case "Cola" => println("Beverage!")
     | case _ => println("Huh?")
     | }
def something(x: String): Unit

scala> something("Cola")
Beverage!
scala> something("Toy")
Huh?
```

## 6.6-6.7 continue/break与变量作用域

Scala不推荐使用`continue`和`break`关键字，需要通过修改代码实现替代方案。

变量的作用域是以花括号为边界的，重名的情况下优先使用内部变量，超出内部变量的范围则使用外部变量。

> 下一篇：Scala的模式匹配
