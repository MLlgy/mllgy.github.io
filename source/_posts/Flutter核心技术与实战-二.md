---
title: Flutter核心技术与实战(二)：Widget 是什么？
date: 2020-01-06 16:01:28
tags:
---
## Widget 是什么？


 Flutter 的核心思想是：一切皆为 Widget，那么什么是 Widget 呢？

 Widget 是 Flutter **功能的抽象描述**，是 **视图的配置信息**，是数据的映射。


 ## Widget 的渲染过程


 Flutter 如何结构化的组织数据，提供给渲染引擎，最终完成界面显示。

 Flutter 将视图树的概念进行扩展，把视图数据的 **组织** 和 **渲染** 分为三部分，即 `Widget、Element、RenderObject。`


![Widget、Element、RenderObject 的关系](/source/images/2020_01_06_01.png)


### Widget 

Widget 是在 Flutter 世界里对视图的一种结构化的描述，可以类比 Android 中的控件的概念，Widget 里面存储的是有关视图渲染的配额制信息，包括布局、渲染属性、事件响应信息等。

Flutter 将 Widget 设计成不可变的，当视图渲染的配置信息发生改变时，Flutter 会选择重建 Widget 树的方式进行数据更新，以数据驱动 UI 的构建方式控制视图显示。 Flutter
这样的设计会存在大量的数据的销毁和重建，会对垃圾回收造成压力。





