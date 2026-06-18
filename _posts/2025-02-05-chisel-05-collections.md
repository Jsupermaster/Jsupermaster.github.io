---
title: "敏捷硬件开发语言Chisel与数字系统设计（五）：Scala的集合"
date: 2025-02-05 09:00:00 +0800
description: "系统学习Scala集合库：数组（Array）、列表（List）、数组缓冲/列表缓冲（ArrayBuffer/ListBuffer）、元组（Tuple）、映射（Map）、集（Set）、序列（Seq）及高阶方法（map/foreach/zip/reduce/fold/scan）。"
categories:
  - FPGA
home_category: study-notes
series: chisel-and-scala-study
tags:
  - Chisel
  - Scala
  - 集合
  - 数组
  - 列表
  - Map
  - 高阶函数
article_kicker: CHISEL NOTE
word_count: 8.0k 字
read_time: 约 22 分钟
---

> 上一篇学习了包与导入，本篇进入Scala丰富的集合库。Scala的集合包括数组、列表、集、映射、序列、元组、数组缓冲和列表缓冲等。

## 5.1 数组（Array）

### 5.1.1 数组的定义

数组是计算机内一片**地址连续**的内存空间。数组元素类型可以是任意的，不同元素类型会导致每个元素的内存大小不一样，但**所有元素类型必须一致**。数组对象是**定长**的，构造完毕就不能再更改长度。构造语法：`new Array[T](n)`

Array的伴生对象中还定义了一个`apply`工厂方法：

```scala
scala> val charArray = Array('a','b','c')
val charArray: Array[Char] = Array(a, b, c)
```

### 5.1.2 数组的索引与元素修改

数组的下标写在**圆括号**中，数组元素是可变的：

```scala
scala> val intArray = new Array[Int](3)
val intArray: Array[Int] = Array(0, 0, 0)

scala> intArray(0) = 1
scala> intArray(1) = 2
scala> intArray(2) = 3

scala> intArray
val res0: Array[Int] = Array(1, 2, 3)
```

## 5.2 列表（List）

### 5.2.1 列表的定义

列表是一种基于**链表**的数据结构，也是**定长**的，每个元素类型相同，**不可再重新赋值**。

```scala
scala> val intList = List(1, 1, 10, -5)
val intList: List[Int] = List(1, 1, 10, -5)

scala> intList(0)
val res1: Int = 1
```

### 5.2.2 列表添加数据

| 操作符 | 作用                       |
| ------ | -------------------------- |
| `::`   | 向列表的头部添加元素或列表 |
| `:::`  | 用于拼接左右两个列表       |
| `+:`   | 向列表的头部添加元素或列表 |
| `:+`   | 向列表的尾部添加元素或列表 |

注意，这些操作都会返回一个**新的列表**。以冒号结尾的操作符（`::`、`:::`、`+:`）的调用对象在右侧：

```scala
scala> val x = List(1)
scala> val y = 2 +: x
val y: List[Int] = List(2, 1)

scala> val z = x :+ 3
val z: List[Int] = List(1, 3)
```

### 5.2.3 列表子对象Nil

`Nil`表示空列表，类型是`List[Nothing]`。因为List的类型参数是协变的，所以`List[Nothing]`是所有列表的子类，即**Nil兼容所有类型的元素**：

```scala
scala> 1 :: 2 :: 3 :: Nil
val res5: List[Int] = List(1, 2, 3)
```

数组和列表元素不仅可以是值类型，也可以是自定义的类，甚至是数组和列表本身，构成嵌套的结构。

## 5.3 数组缓冲与列表缓冲（ArrayBuffer & ListBuffer）

定义在`scala.collection.mutable`包中的`ArrayBuffer`和`ListBuffer`，支持在头部和尾部动态增删元素，且耗时固定。

| 操作符 | 作用                                 |
| ------ | ------------------------------------ |
| `+=`   | 向缓冲的尾部添加元素                 |
| `+=:`  | 向缓冲的头部添加元素                 |
| `-=`   | 从缓冲的尾部开始删去第一个符合的元素 |

```scala
scala> import scala.collection.mutable.{ArrayBuffer, ListBuffer}

scala> val ab = new ArrayBuffer[Int]()
scala> ab += 10
val res8: ArrayBuffer[Int] = ArrayBuffer(10)

scala> -10 +=: ab
val res9: ArrayBuffer[Int] = ArrayBuffer(-10, 10)

scala> val lb = new ListBuffer[String]()
scala> lb += "one"
scala> lb ++= Seq("abc", "oops", "good")
scala> lb -= "abc"
val res12: ListBuffer[String] = ListBuffer(one, oops, good)

scala> lb.toArray
val res13: Array[String] = Array(one, oops, good)

scala> lb.toList
val res14: List[String] = List(one, oops, good)
```

## 5.4 元组（Tuple）

元组也是不可变的，但**可以包含不同类型的对象**，使用圆括号表示。用`._n`访问元素，**第一个元素是`._1`**：

```scala
scala> val t = ("God", 'A', 2333)
val t: (String, Char, Int) = (God, A, 2333)

scala> t._1
val res15: String = God

scala> t._2
val res16: Char = A

scala> t._3
val res17: Int = 2333
```

元组是一系列类`Tuple1`~`Tuple22`，最多包含22个元素。二元组也称为**对偶**，在映射中会用到。元组的遍历需要先调用`productIterator`方法获取迭代器：

```scala
scala> val t1 = (1, "a", "b", true, 2)
scala> for (item <- t1.productIterator) {
     | println("item" + item)
     | }
item1
itema
itemb
itemtrue
item2
```

## 5.5 映射（Map）

### 5.5.1 定义与构造

映射是包含一系列**键-值对**的集合，键和值的类型可以是任意的，但每个键-值对的类型必须一致。映射是一个**特质**，无法通过`new`创建，只能通过伴生对象的`apply`方法构造：

```scala
scala> val map = Map(1 -> "+", 2 -> "-", 3 -> "*", 4 -> "/")
val map: Map[Int, String] = Map(1 -> +, 2 -> -, 3 -> *, 4 -> /)
```

表达式`object1 -> object2`是一个对偶，键-值对也可以写成对偶形式：`Map(('a', 'A'), ('b', 'B'))`

### 5.5.2 三种取值方式

| 方法                          | 行为                                   |
| ----------------------------- | -------------------------------------- |
| `map("键")`                   | 返回对应的值，如果没有则**报错**       |
| `map.get("键")`               | 返回`Option[T]`类型，没有则返回`None`  |
| `map.getOrElse("键", 默认值)` | 返回对应的值，没有则返回**默认值**     |

### 5.5.3 四种遍历方式

| 遍历方式               | 说明             |
| ---------------------- | ---------------- |
| `for((k, v) <- map)`   | 遍历所有的键和值 |
| `for(k <- map.keys)`   | 遍历所有的键     |
| `for(v <- map.values)` | 遍历所有的值     |
| `for(item <- map)`     | 遍历键-值对（元组）|

默认是`scala.collection.immutable`包中的不可变映射，也可以导入`scala.collection.mutable`包中的可变映射来动态增删键-值对。

## 5.6 集（Set）

集也是一个**特质**，也只能通过`apply`工厂方法构建对象。集只能包含**字面值不相同**的同类型元素，构建时传入重复参数会自动去重。集的`apply`方法测试集中是否包含传入的参数，返回`true`或`false`：

```scala
scala> val set = Set(1, 1, 10, 10, 233)
val set: Set[Int] = Set(1, 10, 233)    // 重复元素自动去重

scala> set(10)
val res27: Boolean = true

scala> set(100)
val res26: Boolean = false
```

## 5.7 序列（Seq）

序列`Seq`也是一个特质，**数组和列表都混入了这个特质**。序列可遍历、可迭代，可以用从0开始的下标索引，也可用于循环。序列使用`apply`方法进行构造：

```scala
scala> val seq = Seq(1, 2, 3, 4, 5)
val seq: Seq[Int] = List(1, 2, 3, 4, 5)
```

序列提供了一组统一的操作接口，使得数组和列表在使用`map`、`filter`、`fold`等高阶函数时有完全一致的行为。

## 5.8 集合的常用方法

### 5.8.1 `map`

`map`接收一个无副作用的函数作为入参，对每个元素应用该函数，并将所有结果打包在**新集合**中返回：

```scala
scala> Array("apple", "orange", "pear").map(_ + "s")
val res28: Array[String] = Array(apples, oranges, pears)

scala> List(1, 2, 3).map(_ * 2)
val res29: List[Int] = List(2, 4, 6)
```

### 5.8.2 `foreach`

与`map`类似，但传入的是**有副作用**的函数（不产生新集合）：

```scala
scala> var sum = 0
scala> Set(1, -2, 234).foreach(sum += _)
scala> sum
val res30: Int = 233
```

### 5.8.3 `zip`

把两个可迭代的集合**一一对应**，构成若干对偶，多余的忽略：

```scala
scala> List(1, 2, 3) zip Array('1', '2', '3')
val res31: List[(Int, Char)] = List((1, 1), (2, 2), (3, 3))
```

### 5.8.4 `reduce`

入参是一个**二元操作函数**，对集合中的元素进行归约：

```scala
(1 to 5).reduce(_ + _)        // 1+2+3+4+5 = 15
(1 to 5).reduceLeft(_ + _)    // 1+2+3+4+5 = 15
(1 to 5).reduceRight(_ + _)   // 5+4+3+2+1 = 15
```

### 5.8.5 `fold`

与`reduce`类似，但可以传入一个**初始值**：

```scala
(1 to 5).fold(10)(_ + _)        // 10+1+2+3+4+5 = 25
(1 to 5).foldLeft(10)(_ + _)    // 10+1+2+3+4+5 = 25
(1 to 5).foldRight(10)(_ + _)   // 10+5+4+3+2+1 = 25
```

### 5.8.6 `scan`

与`fold`类似，但会保留**所有中间结果**：

```scala
(1 to 3).scan(10)(_ + _)        // List(10, 11, 13, 16)
(1 to 3).scanLeft(10)(_ + _)    // List(10, 11, 13, 16)
(1 to 3).scanRight(10)(_ + _)   // List(16, 15, 13, 10)
```

### 方法对比总结

| 方法      | 初始值 | 保留中间结果 | 返回类型     |
| --------- | ------ | ------------ | ------------ |
| `map`     | —      | —            | 新集合       |
| `foreach` | —      | —            | Unit         |
| `zip`     | —      | —            | 对偶集合     |
| `reduce`  | 无     | 否           | 单一值       |
| `fold`    | **有** | 否           | 单一值       |
| `scan`    | **有** | **是**       | 中间结果集合 |

这些高阶方法构成了Scala函数式编程的基础，也是后续理解Chisel硬件构造语法的关键。

> 下一篇：Scala的内建控制结构
