---
title: Kotlin 核心编程(一)：Kotlin 生态的基本了解
tags:
---






JVM 平台的语言
Kotlin  和 Java 一样，为在 JVM 平台上运行的语言。

JVM 平台下的语言比较火的有：Java、Scala、Kotlin 等。

### Kotlin 产生的背景

Kotlin 和 Scala 的设计理念不同：

* Scala： 
* Kotlin： 更好的 java
  




Java8 的探索：
* 高阶函数和 Lambda
* Stream API
* Optional 类

Java 的前景展望

* 数据类
* vi

---

* Scala 发明者：在 Java 中引入泛型的大佬。
* Scala 在设计之初就集成了面向对象和函数式。

Scala 彻底拥抱了函数式

Scala 现在主要的使用领域：大数据。Spark 用 Scala 开发的。

Scala 简单而不容易的哲学相应的代价：
* 更加抽象的编程范式，学习成本大。
* 建立了与 Java 完全不同的思维模式。


基于此 Kotlin 作为 JVM 上新兴的编程语言，慢慢的走入开发者的视野。



### Kotlin 的设计理念：改良的 Java

2010 年，看到了 Java 相比与新语言的滞后性，JetBrains 产生创造了 Kotlin 这种改良 Java 的想法。


Kotlin 与现有的 Java 代码完全兼容,。

* 兼容其语法和生态，Java 程序员更容易掌握。
* 更加实用，创造扩展的语法。
* Java 中没有的语法糖，改善了开发中的体验，比如 Smart Cast:

```
// Smart Cast 语法
if(view is ViewGroup){
    view.addView(childView)
}
```


**强大的生态**

* Android开发，现在 Kotlin 已经为 Google Android 的官方开发语言。
* 服务端开发，Spring FrameWork 5 已经拥抱了 Kotlin。
* 前端开发
* 原生开发，Kotlin Native 项目使 Kotlin 离开了 JVM，直接编译成机器代码提供系统环境运行，单处于早期阶段。

Scala ：more than Java
Kotlin: a better Java





----
什么是函数式编程？


