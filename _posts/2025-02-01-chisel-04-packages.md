---
title: "敏捷硬件开发语言Chisel与数字系统设计（四）：Scala的包和导入"
date: 2025-02-01 09:00:00 +0800
description: "掌握Scala中包（package）的定义与层次结构、灵活的import导入语法（重命名、隐藏）、访问修饰符（private/protected）及包对象。"
categories:
  - FPGA
home_category: reading-notes
series: chisel-and-scala-study
tags:
  - Chisel
  - Scala
  - 包
  - import
  - 访问修饰符
cover_image: /assets/images/chisel-scala/cover.jpg
article_kicker: CHISEL NOTE
word_count: 2.2k 字
read_time: 约 7 分钟
---

> 上一篇我们学习了面向对象编程的完整内容，本篇进入代码组织相关的主题——包与导入机制。

## 4.1 包（Package）

包以关键字`package`开头，可以用花括号把包的范围括起来，这样一个文件可以包含多个不同的包；也可以不用花括号，这样包的声明必须在最前面，表明整个文件的内容都属于这个包。

对于包的命名方法，可以使用**反转域名法**，即`com.xxx.xxx`。在包中可以定义class、object和trait，也可以定义别的package。

编译一个包文件，会在当前路径下生成一个与包名相同的文件夹，文件夹中是编译后生成的文件。如果多个文件的顶层包的包名相同，编译后的文件会放在同一个文件夹内。一个包的定义可以由多个文件的源代码组成，**包名和源文件所在路径不一定相同**。

## 4.2 包的层次和精确代码访问

包中可以定义包，所以包也有层次结构。包可以通过句点符号按路径层次访问：

```
package one.two
```

等效于：

```scala
package one
  package two
```

这样会先编译出一个名为`one`的文件夹，在里面再编译出一个名为`two`的文件夹。

**访问规则：**

1. 访问同一个`package`内的class、object、trait**不需要**增加路径前缀；
2. 访问同一个包内更深一层的包所包含的内容，只需写出更深的那一层包；
3. 使用花括号表示包的作用范围时，包外所有可访问的内容在包内也可以直接访问。

以上规则在同一个文件内显式嵌套才可以生效，如果包分散在多个文件中通过包名带句点来嵌套则无法生效。

Scala还定义了一个隐式的顶层包 **`_root_`**，它是所有自定义包的顶层包。

下面是一个完整的包使用示例：

```scala
package bobsrockets {
  package navigation {
    class Navigator {
      val map = new StarMap
    }
    class StarMap
  }

  class Ship {
    val nav = new navigation.Navigator   // 使用同层深一层包的类
  }

  package fleets {
    class Fleet {
      def addShip() = { new Ship }       // 可访问包外的类
    }
  }
}
```

## 4.3 import导入

Scala的`import`非常灵活，体现在以下三点：

1. **可以出现在代码的任意位置**；
2. 除了导入包内所有内容，还可以导入对象（单例对象和new构造的对象都可以）和包自身，甚至函数的参数也可以当作对象来导入；
3. **可以重命名或隐藏某些成员**。

### 导入语法示例

```scala
import A.B              // 导入包B，访问时写B.M
import A.B.M            // 导入类M，直接写M即可
import A.C.N            // 导入对象N
import A.{B, C}         // 同时导入B和C
import A._              // 导入A的所有内容（通配导入）
import A.{B => packageB}    // 导入B并重命名为packageB
import A.{B => _, _}        // 隐藏B，导入A的其他所有元素
```

## 4.4 自引用

Scala中用关键字`this`指代对象自己。如果`this`用在类的方法中，则指代正在调用方法的那个对象；如果用在类的构造方法中，则指代当前正在构建的对象。

## 4.5 访问修饰符

包、类和对象的成员都可以标上访问修饰符：

- **`private`**：私有的，只能被包含它的包、类或对象的内部代码访问
- **`protected`**：受保护的，除了能被内部访问，还能被子类访问

还可以加入限定词：`private[X]`和`protected[X]`把访问权限扩大到X的内部。

**限定词`this`：**
- `private[this]`比`private`更严格，只能在当前对象内部访问
- `protected[this]`在此基础上扩展到定义时的子类

伴生对象和伴生类共享访问权限，两者可以互访对方的所有私有成员。在伴生对象中使用`protected`没有意义，特质使用`private`和`protected`也没有意义。

## 4.6 包对象

包中可直接包含的元素有类、特质和单例对象，但字段和方法不能直接定义在包中。Scala把字段和方法放在一个**"包对象"**中，每个包都允许有一个包对象。

包对象使用关键字组合`package object`来定义，其名称与关联的包名相同，类似伴生类和伴生对象的关系。

> 下一篇：Scala的集合
