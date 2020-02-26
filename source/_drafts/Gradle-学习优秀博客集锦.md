---
title: Gradle 学习优秀博客集锦
tags:
---


[深入理解Android之Gradle](https://blog.csdn.net/innost/article/details/48228651)




[Andorid DSL 预览](http://google.github.io/android-gradle-dsl/current/)

[【Android 修炼手册】Gradle 篇 -- Gradle 的基本使用](https://zhuanlan.zhihu.com/p/65249493)

关于 implementation、api、compileOnly、runtimeOnly、Android Transform 、插件表现步骤、调试插件。

android gradle plugin 提供了 transform api 用来在 .class to dex 过程中对 class 进行处理，可以理解为一种特殊的 Task，因为 transform 最终也会转化为 Task 去执行。在 transform 中的处理，一般会涉及到 class 文件的修改，操纵字节码的工具一般是 javasist 和 asm 居多。