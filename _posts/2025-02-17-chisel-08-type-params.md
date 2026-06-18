---
title: "敏捷硬件开发语言Chisel与数字系统设计（八）：Scala的类型参数化"
date: 2025-02-17 09:00:00 +0800
description: "深入Scala泛型编程：var字段的getter/setter、类型构造器（泛型类）、型变注解（协变+、逆变-、不变）、型变检查规则、类型构造器的继承关系以及上界/下界。"
categories:
  - FPGA
home_category: reading-notes
series: chisel-and-scala-study
tags:
  - Chisel
  - Scala
  - 泛型
  - 型变
  - 类型构造器
  - 上界下界
cover_image: /assets/images/chisel-scala/cover.jpg
article_kicker: CHISEL NOTE
word_count: 5.5k 字
read_time: 约 16 分钟
---

> 📖 **本书来源：**《敏捷硬件开发语言Chisel与数字系统设计》，梁峰、吴斌、张国和、雷冰洁、雷绍充 编著，电子工业出版社，2022年
>
> 上一篇我们完成了模式匹配的全部内容，从本篇开始进入Scala类型系统的深层——类型参数化（泛型）。

## 8.1 var类型的字段

对于可重新赋值的字段，如果在类中定义了一个`var`类型的字段，编译器会把它限制为`private`，同时隐式定义一个名为`变量名`的**getter**和名为`变量名_=`的**setter**。

在Scala 3中，未初始化的var字段需要导入：

```scala
scala> import scala.compiletime.uninitialized

scala> class A {
     | var aInt: Int = uninitialized
     | }
// defined class A
```

> **注意：** 原书中使用下划线`_`赋初值的方式在Scala 3中已被废除。

可以自定义getter和setter来改变字段的读写行为：

```scala
scala> class A {
     | private var a: Int = uninitialized
     | def originalValue: Int = a
     | def originalValue_=(x:Int) = a = x
     | def tenfoldValue: Int = a * 10
     | def tenfoldValue_= (x: Int) = a = x / 10
     | }

scala> val a = new A
scala> a.originalValue = 1
scala> a.originalValue
val res32: Int = 1
scala> a.tenfoldValue
val res33: Int = 10
scala> a.tenfoldValue = 1000
scala> a.originalValue       // 底层存储已被修改
val res35: Int = 100
```

两组getter/setter操作的是同一个底层字段，但对外提供了不同的读写视图。

## 8.2 类型构造器（Type Constructor）

```scala
scala> abstract class A[T] {
     | val a: T
     | }
// defined class A
```

`A`被称为**类型构造器**——它接收一个类型参数来构造一个类型，也可以说`A`是一个**泛型的类**。泛型的类和特质需要在方括号中加上类型参数。

## 8.3 型变注解（Variance Annotation）

类型构造器的类型参数可以是**协变**的、**逆变**的或者**不变**的：

- `A[+T]`：**协变**——`Sub`是`Super`的子类，则`A[Sub]`也是`A[Super]`的子类型
- `A[-T]`：**逆变**——`Sub`是`Super`的子类，则`A[Sub]`是`A[Super]`的超类型
- `A[T]`（无注解）：**不变**——两者是没有任何关系的不同类型

**协变的典型例子：** `List[+T]`——`List[Cat]`是`List[Animal]`的子类型（`Cat`是`Animal`的子类）。

**逆变的典型例子：** `Function1[-S, +T]`——函数的入参类型是逆变的，返回类型是协变的。

## 8.4 检查型变注解

标记了型变注解的类型参数不能随便使用，类型系统设计要满足**里氏替换原则**。

**核心原则：** 生产者产生的值的类型应该是**子类**（协变），消费者接收的值的类型应该是**超类**（逆变）。基于此：
- **方法的入参**的类型应该是**逆变**类型参数
- **方法的返回类型**应该是**协变**的

Scala编译器把类或特质中任何出现类型参数的地方都当作一个"点"，从声明类型参数的类和特质作为**顶层**开始逐步往内层深入：

1. 在顶层的点都是**协变点**（如方法的返回类型）
2. 默认情况下，更深一层嵌套的点与外一层的点归为一类
3. **例外①：** 方法的值参数所在的点根据方法外的点进行**翻转**
4. **例外②：** 方法的类型参数也会根据方法外的点进行翻转
5. **例外③：** 如果类型也是类型构造器（如`C[T]`），T有`-`注解时根据外层翻转，有`+`注解时保持与外层一致，否则变成不变点

**规则：** 协变点只能用`+`注解，逆变点只能用`-`注解。没有型变注解的类型参数可以用在任何点。

### 完整示例

```scala
scala> abstract class Cat[-T, +U] {
     |   def meow[W-](volume: T-, listener: Cat[U+, T-]-): Cat[Cat[U+, T-]-, U+]+
     | }
// defined class Cat
```

正号表示协变点，负号表示逆变点。编译器检查`T`是否都出现在逆变点、`U`是否都出现在协变点。

## 8.5 类型构造器的继承关系

以常用的单参数函数为例，其特质`Function1`的部分定义如下：

```scala
scala> trait Function1[-S, +T] {
     | def apply(x: S): T
     | }
// defined trait Function1
```

- 类型参数`S`代表函数入参的类型，是**逆变**的
- 类型参数`T`代表函数返回值的类型，是**协变**的

假设`类A`是`类a`的超类，`类B`是`类b`的超类，定义了一个函数的类型为`Function1[a, B]`，那么这个函数的子类型应该是`Function1[A, b]`。

**结论：对于含有多个类型参数的类型构造器，要构造子类型，就是把逆变类型参数由子类替换成超类、把协变类型参数由超类替换成子类。**

## 8.6 上界和下界

### 下界（Lower Bound）

对于类型构造器`A[+T]`，倘若没有别的手段，方法的参数不能泛化（因为协变的类型参数不能用作函数的入参类型）。Scala提供了下界语法，形式为 **`U >: T`**，表示`U`必须是`T`的**超类**或者是`T`本身：

```scala
scala> abstract class A[+T] {
     | def funcA[U >: T](x: U): U
     | }
// defined class A
```

### 上界（Upper Bound）

与下界对应，形式为 **`U <: T`**，表示`U`必须是`T`的**子类**或本身。可以在`A[-T]`这样的逆变类型构造器中泛化方法的返回类型：

```scala
scala> abstract class A[-T] {
     | def funcA[U <: T](x: U): U
     | }
// defined class A
```

### 上下界总结

| 语法      | 含义          | 典型用途                     |
| --------- | ------------- | ---------------------------- |
| `U >: T`  | U是T的超类    | 协变类型的方法入参泛化       |
| `U <: T`  | U是T的子类    | 逆变类型的方法返回类型泛化   |

## 8.7 方法的类型参数

除了类和特质可以声明类型参数，**方法也可以带有类型参数**：
- 如果方法仅仅使用了包含它的类/特质已声明的类型参数，不用自己写出
- 如果出现了未声明的类型参数，则必须写在方法的类型参数中
- **方法的类型参数不能有型变注解**

```scala
scala> abstract class A[-T] {
     | def funcA[U](x: T, y: U): Unit   // U是方法自己的类型参数
     | }
```

> **注意：** 方法的类型参数不能与包含它的类和特质已声明的类型参数一致，否则会将它们覆盖。

## 8.8 对象私有数据

`var`类型的字段，其类型参数**不能是协变的或逆变的**（因为`var`既有getter又有setter，同时处于协变和逆变位置）。但如果`var`字段是对象私有的（用`private`修饰），外部无法访问，就可以**忽略对型变注解的检查**。

```scala
class A[+T] {
  private var data: T = _    // OK：对象私有，外部不可见
}
```

这一规则在Chisel的硬件模块实现中非常实用——内部可变状态用`private`保护，对外保持泛型的型变安全性。

> 下一篇：Scala的抽象成员
