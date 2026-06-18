---
title: "敏捷硬件开发语言Chisel与数字系统设计（二）：Scala语言编程基础"
date: 2025-01-24 09:00:00 +0800
description: "从零搭建Scala开发环境，系统学习Scala变量定义与基本类型、函数的多种形式（函数字面量、高阶函数、闭包、柯里化、传名参数等）。"
categories:
  - FPGA
home_category: study-notes
series: chisel-and-scala-study
tags:
  - Chisel
  - Scala
  - 开发环境
  - 函数式编程
  - 闭包
  - 柯里化
article_kicker: CHISEL NOTE
word_count: 8.5k 字
read_time: 约 25 分钟
---

> 上一篇我们介绍了为什么选择Chisel和Scala，本篇正式开始Scala语言编程基础的学习。

## 2.1 Scala的运行

为了更便利地使用Scala和各种编译链，我们最好在Linux环境下进行学习。这里我使用Windows 11的Ubuntu子系统进行学习，使用虚拟机也可以。

环境：ARM64 Windows 11 MatebookEGo Snapdragon (TM) 8cx Gen 3 @ 3.0 GHz / 3.00 GHz / Ubuntu 22.04 WSL2

官方网站：[Install \| The Scala Programming Language](https://www.scala-lang.org/download/)

### 安装过程

首先需要安装Java环境，我的Ubuntu中没有自带Java环境：

```bash
sudo apt install default-jdk
```

之后执行这条指令安装Scala：

```bash
curl -fL https://github.com/VirtusLab/coursier-m1/releases/latest/download/cs-aarch64-pc-linux.gz | gzip -d > cs && chmod +x cs && ./cs setup
```

这时Scala已经被成功安装，但我们需要重启Ubuntu。之后用下面的语句测试：

```bash
scala -version
```

如果能正确显示版本号，没有WARNING则说明已经安装完成。直接输入`scala`便可进入Scala编译器：

```scala
jia@J-MateBookEGo:~$ scala
Welcome to Scala 3.6.2 (21.0.5, Java OpenJDK 64-Bit Server VM).
Type in expressions for evaluation. Or try :help.

scala> 1+2
val res0: Int = 3
```

如果我们希望使用图形化界面编程，可以安装IDEA，网上教程很多，在这里不多说明。

> 本系列代码均基于 **Scala 3.6.2**，部分语法与书中（基于Scala 2.x）有差异，均已标注。

## 2.2 Scala的变量及函数

### 2.2.1 变量定义与基本类型

#### 变量声明

变量首次定义必须使用关键字 `var` 或者 `val`，二者的区别是：

- `val` 修饰的变量禁止被重新赋值，它是一个**只读**的变量。
- `var` 类型的变量可以重新赋值，但新旧必须是同一个类型。

首次定义变量时必须赋值进行初始化。变量定义具有覆盖性，后声明的变量会覆盖前面的变量。

Scala推荐使用 `val` 定义变量，函数式编程的思想之一就是传入函数的参数不应该改变。需要说明的是，使用 `val` 定义的变量并不是不可变，而是这个**指向关系不可变**。当这个变量被声明，变量指向的对象就是唯一确定的，但这个对象本身可以改变，只是变量指向的位置不可变。

#### 基本类型

Scala作为一种静态语言，在编译时会检查每个对象的类型，类型不匹配的非法操作会报错。Scala定义了一些标准类型：

| 类型    | 说明                                               |
| ------- | -------------------------------------------------- |
| Byte    | 8bit有符号整数，补码表示，范围是-2^7~2^7-1          |
| Short   | 16bit有符号整数，补码表示，范围是-2^15~2^15-1       |
| Int     | 32bit有符号整数，补码表示，范围是-2^31~2^31-1       |
| Long    | 64bit有符号整数，补码表示，范围是-2^63~2^63-1       |
| Char    | 16bit无符号字符，Unicode编码表示，范围是0~2^16-1    |
| String  | 字符串类型，属于java.lang包                         |
| Float   | 32bit单精度浮点数                                   |
| Double  | 64bit双精度浮点数                                   |
| Boolean | 布尔值，true or false                               |

定义变量时可以指定类型，也可以让编译器自动推断。

```scala
scala> val x: Int = 123
val x: Int = 123

scala> val y: Long = 123
val y: Long = 123
```

#### 整数字面量

如果单独出现数字，没有任何说明，默认推断为Int类型；结尾有`l`或`L`的推断为Long类型；以`0x`开头的认为是十六进制，不区分大小写。Byte和Short类型需要显式指定：

```scala
scala> val x = 100
val x: Int = 100

scala> val y = 100L
val y: Long = 100

scala> val z: Byte = 200
-- [E007] Type Mismatch Error: ------------------------------------------
1 |val z: Byte= 200
  |             ^^^
  |             Found:    (200 : Int)
  |             Required: Byte
```

Byte类型最大不超过127，超过限制的赋值将报错。

#### 浮点数字面量

浮点数字面量都是十进制的，默认是Double类型。`En`或`en`表示科学计数法10的n次方，末尾加一个`f`或`F`表示Float，`D`或`d`表示Double。Double字面量不能赋值给Float类型变量。

```scala
scala> val x: Float = -3.2
val x: Float = -3.2

scala> val x = -3.2
val x: Double = -3.2

scala> val x: Float = 3.2D
-- [E007] Type Mismatch Error: Found: (3.2d : Double), Required: Float
```

#### 字符字面量和字符串字面量

以单引号括住的一个字符表示一个字符字面量，采用Unicode编码。也可以使用`\u编号`来构建一个字符，但在最新版本Scala编译器中会报错。字符串字面量是用双引号`""`括住的字符序列，长度是任意的。字符和字符串都支持转义字符。

#### 字符串插值

表达式可以被嵌入在字符串字面量中被求值，有三种实现方法：

**s插值器** 形如 `s"...${表达式}..."`。花括号可以不加，但只会识别美元符号到首个非标识字符：

```scala
scala> val name = "Jia"
val name: String = Jia

scala> s"Name = $name"
val res3: String = Name = Jia

scala> s"Result = ${1+2}"
val res4: String = Result = 3
```

**raw插值器** 和s插值器类似，区别是不识别转义字符。

**f插值器** 使用方法更灵活，支持格式控制：

```scala
scala> printf(f"${math.Pi}%.5f")
3.14159
```

### 2.2.2 函数及其几种形式

#### 函数的定义

```scala
scala> def max(x : Int, y: Int): Int = {
     | if(x > y)
     | x
     | else
     | y
     | }
def max(x: Int, y: Int): Int
```

说明：`x`、`y`是输入参数（形参），后面跟的是它们的类型；`Int = {...}`是函数体，表示返回是一个Int类型整数。左侧的`|`只是编译器自动生成的用于多行的代码，并非语法体系中的一部分。

**几条重要规则：**

1. **分号推断：** 语句末尾的分号是可选的，编译器会自动推断分号。如果一行有多条语句，则必须用分号隔开。
2. **函数的返回结果：** `return`关键字是可选的，编译器自动为函数体中的最后一个表达式加上`return`，建议不要显式声明`return`。返回结果的类型也可以自动推断。`Unit`类型表示无返回值，它不是必须的，编译器也可以自动推断；但如果显式声明`Unit`，则即便有可以返回的值也不会返回任何值。
3. **等号与函数体：** 函数体是花括号括起来的部分，里面有多条语句，返回最后一个表达式。当函数的返回类型没有显式声明时等号可以省略，但此时返回类型会变成`Unit`。**建议不要省略等号，并且显式声明返回类型。**
4. **无参函数：** 无参函数可以写空括号作为参数列表，也可以不写；但如果没有空括号，调用的时候禁止写空括号。

#### 方法

方法指定义在`class`、`object`、`trait`中的函数，称为成员函数或方法，这与其他面向对象特性的语言一致。

#### 嵌套函数

函数体内部可以嵌套定义内部函数，但无法被外界访问。

#### 函数字面量

函数式编程认为函数的地位和一个Int值、String值是一样的，因此函数也可以成为一个函数的参数或返回值，也可以把函数赋值给一个变量。

**函数字面量是一种匿名函数**，它可以存储在变量中，成为函数参数，或者当作返回值返回。定义形式是：

```
{参数1: 参数1类型, 参数2: 参数2类型, ...} => {函数体}
```

它可以更精简地使用，用下划线作为占位符替代参数：

```scala
scala> val f = (_: Int) + (_: Int)
val f: (Int, Int) => Int = Lambda/...

scala> f(1,2)
val res5: Int = 3
```

用`def`定义的函数和函数字面量都可以**以函数为参数，返回函数**：

```scala
scala> val add = (x:Int) => {(y:Int) => x + y}
val add: Int => Int => Int = Lambda/...

scala> add(1)(10)
val res6: Int = 11
```

这里是函数的嵌套：第一个函数表示输入是`x`，类型是`Int`，函数体是`(y:Int) => x + y`——这是一个函数字面量。如果调用`add(1)`，则返回的是`(y:Int) => 1 + y`这个函数字面量。

将函数作为参数传递的示例：

```scala
scala> def aFunc(f: Int => Int) = f(1) + 1
def aFunc(f: Int => Int): Int

scala> aFunc(x => x + 1)
val res7: Int = 3
```

在这里我们已经逐渐能够感受到函数式编程的魅力，当理解这种调用方式后将会有很灵活的应用。

#### 部分应用函数

使用`def`定义的函数也具有"函数作为一等值"的功能，只不过需要借助**部分应用函数**的形式来实现。部分应用函数的意思就是给出函数的一部分参数，使其可以赋值给变量或者当作函数参数进行传递：

```scala
scala> def sum(x:Int, y:Int, z:Int) : Int = {x + y + z}
def sum(x: Int, y: Int, z: Int): Int

scala> val a2 = sum(4, _:Int, 6)
val a2: Int => Int = Lambda/...

scala> a2(5)
val res12: Int = 15

scala> val a3 = sum
val a3: (Int, Int, Int) => Int = Lambda/...

scala> a3(4,5,6)
val res9: Int = 15
```

> **注意：** 书中使用的 `sum _` 写法在最新版Scala中不被支持，现在无需写下划线。

```scala
scala> def needSum(f:(Int,Int,Int) => Int) = f(1,2,3)
def needSum(f: (Int, Int, Int) => Int): Int

scala> needSum(sum)
val res13: Int = 6
```

#### 闭包

一个函数除了使用它的参数以外，还可以使用定义在函数以外的其他变量。其中函数的参数称为**绑定变量**，函数以外的变量称为**自由变量**，这样的函数称为**闭包**。

闭包捕获的自由变量是函数定义之前的自由变量，若后面出现新的同名自由变量将前面的覆盖，函数与其无关。但如果自由变量是用`var`创建的可变对象，那么闭包随之改变。

```scala
scala> var a1 = 1
var a1: Int = 1

scala> val a2 = 100
val a2: Int = 100

scala> val add1 = (x : Int) => x + a1
scala> val add2 = (x : Int) => x + a2

scala> add1(1)
val res14: Int = 2

scala> a1 = 100                    // 修改var变量

scala> add1(1)                     // 闭包的值随之改变！
val res16: Int = 101

scala> var a1 = 1                  // 重新定义（覆盖）
scala> add1(1)                     // 闭包不受影响
val res17: Int = 101
```

**结论：** 修改`var`变量的值 → 闭包随之改变；重新定义`var`或`val`（覆盖前面的定义）→ 闭包的值不变。

#### 函数的特殊调用形式

1. **具名参数：** 如果显式声明参数的名字，可以无视参数顺序传递。
2. **默认参数值：** 函数定义时可以给参数一个默认值，调用时缺省该参数则使用默认值。
3. **重复参数：** 允许把函数的最后一个参数标记为重复参数，在类型后面加上星号`*`。

#### 柯里化（Currying）

Scala的特性**柯里化**允许一个函数可以有任意个参数列表。当参数列表中只有一个参数时，调用该函数时允许单个参数用花括号替代圆括号：

```scala
scala> def add(x:Int)(y:Int)(z:Int) = x+y+z
def add(x: Int)(y: Int)(z: Int): Int

scala> add(1)
val res19: Int => Int => Int = Lambda/...

scala> add(1)(2)(3)
val res21: Int = 6

scala> add(1)(2){3}
val res22: Int = 6
```

**柯里化本质上就是把一个多参数函数处理成了嵌套的函数。** 这种写法在函数式编程中非常常见，在Chisel中被大量用于构建硬件描述的DSL语法。

#### 传名参数（By-Name Parameter）

```scala
// 常规调用方法
scala> def add(f: () => Int) = {a1 + a2 + f()}
scala> add(() => 1)
val res24: Int = 4

// 传名参数调用方法
scala> def add2(f: => Int) = {a1 + a2 + f}
scala> add2(1)
val res26: Int = 4
```

**传名参数的核心价值在于"惰性求值"**——参数表达式只在函数体内真正被使用时才计算。这两种版本（`() => Int` 和 `=> Int`）都是在使用到传入的函数时才会计算。但如果把`=>`去掉，就变成了先计算表达式再传入值。

传名参数这一特性在Chisel中被大量运用，用于构建硬件电路的连接关系而不立即执行。

> 下一篇：Scala面向对象编程
