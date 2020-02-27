---
title: Gradle 学习优秀博客集锦
tags:
---


[深入理解Android之Gradle](https://blog.csdn.net/innost/article/details/48228651)



Gradle另外一个特点就是它是一种DSL，即Domain Specific Language，领域相关语言。什么是DSL，说白了它是某个行业中的行话。还是不明白？徐克导演得《智取威虎山》中就有很典型的DSL使用描述.

Gradle中也有类似的行话，比如sourceSets代表源文件的集合等。


基于行话，我们甚至可以建立一个模板，使用者只要往这个模板里填必须要填的内容，Gradle就可以非常漂亮得完成工作，得到想要的东西。

Groovy

* 动态语言


* 脚本语言


Groovy有时候又像一种脚本语言。前文也提到过，当我执行 Groovy 脚本时， **Groovy 会先将其编译成 Java 类字节码**，然后通过 Jvm 来执行这个Java类。图1展示了Java、Groovy和Jvm之间的关系。

* 闭包

闭包，是一种数据类型，它代表了一段可执行的代码。



Gradle 既是一种脚本工具，也是一种变编程框架。

### 基本组件

Project

Task


一个 Project 到底包含多少个 Task，其实是由编译脚本指定的 **插件** 决定。插件是什么呢？**插件就是用来定义Task，并具体执行这些Task的东西**。

Gradle是一个框架，作为框架，它负责 **定义流程和规则**。而 **具体的编译工作则是通过插件的方式来完成的**。比如编译Java有Java插件，编译Groovy有Groovy插件，编译Android APP有Android APP插件，编译Android Library有Android Library插件。

一个Project是由若干tasks来组成的，当gradlexxx的时候，实际上是要求gradle执行xxx任务。这个任务就能完成具体的工作。

具体的工作和不同的插件有关系。编译Java要使用Java插件，编译Android APP需要使用Android APP插件。

每种插件定义的Task都不尽相同，这就是所谓的Domain Specific。


Task 之间存在依赖关系，依赖关系对我们使用gradle有什么意义呢？

如果知道Task之间的依赖关系，那么开发者就可以添加一些定制化的Task。比如我为assemble添加一个SpecialTest任务，并指定assemble依赖于SpecialTest。当assemble执行的时候，就会先处理完它依赖的task。自然，SpecialTest就会得到执行了。



### 扩展属性 
2019_08_22_01就不需要ext前缀了。ext属性支持Project和Gradle对象。即Project和Gradle对象都可以设置ext属性。                                                                                                                         





[Andorid DSL 预览](http://google.github.io/android-gradle-dsl/current/)

[【Android 修炼手册】Gradle 篇 -- Gradle 的基本使用](https://zhuanlan.zhihu.com/p/65249493)

关于 implementation、api、compileOnly、runtimeOnly、Android Transform 、插件表现步骤、调试插件。

android gradle plugin 提供了 transform api 用来在 .class to dex 过程中对 class 进行处理，可以理解为一种特殊的 Task，因为 transform 最终也会转化为 Task 去执行。在 transform 中的处理，一般会涉及到 class 文件的修改，操纵字节码的工具一般是 javasist 和 asm 居多。


--- 


[Groovy API](http://www.groovy-lang.org/api.html)

[Gradle API](https://docs.gradle.org/current/javadoc/index.html?overview-summary.html)

[Gradle 用户手册](https://docs.gradle.org/current/userguide/userguide.html)

[Gradle Index](https://docs.gradle.org/current/javadoc/index-all.html)
