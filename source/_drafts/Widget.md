---
title: Widget
tags:
---

## Flutter 中的 Widget 是什么

Widget 是 Flutter 功能的抽象描述，是视图的配置信息，也是数据的映射，是 Flutter 开发框架的最基础的概念。


## Widget 的渲染过程

如何结构化的组织视图数据，提供给渲染引擎，最终完成界面显示。

不同UI框架处理以上问题，无一例外使用了视图树，在 Flutter 中扩展了视图树概念，将视图数据的组织和渲染分为了三部分：Widget、Element、RenderObject。



## Widget

Widget 是 Flutter 世界中对 **视图的一种结构化的描述**，是控件实现的基本逻辑单位，里面存储有关视图渲染的配置信息，比如布局、渲染属性、事件响应等。

**Flutter 中将 Widget 设计成不可变的**，如果视图渲染的配置信息发生改变，此时 Flutter 会选择 **重建** Widget 树的方式更新 UI，从而达到以数据驱动 UI 构建。这样的话就会涉及大量的对象销毁和重建，垃圾回收会有会有一定的压力，但是由于 Widget 本身不涉及实际渲染位图，所以 Widget 只是一份轻量级的数据结构，重建的成本很低，相应的减轻了垃圾回收的压力。

由于 Widget 的不可变性，可以以较低成本进行渲染节点的复用，所以在一个渲染树可能多个 Widget 对应同一个渲染节点，降低了重建 UI 的成本。

## Element

Element 是 Widget 的一个实例化对象，它 **承载了视图构建的上下文数据**，连接结构化配置信息和最终的渲染。

Flutter 渲染的过程，可以分为三步：

1. 通过 Widget 树生成对应的 Element 树。
2. 创建相应的 RenderObject，并关联到 Element.renderObject 属性上。
3. 构建 RenderObject 树，以完成最终的渲染。
   
Element 同时持有 Widget 和 RenderObject，但是 Widget和 Element 只负责发号施令，而最后的工作由 RenderObject 来做，那么为什么不可以直接让　Widget 发号施令，还要添加中间层 Element 呢？其实理论上可以这么做但，但是却会极大的增加渲染带来的性能损耗。

因为 Widget 为不可变的，但是 Element 却是可变的，Element 树这一层将 Widget 树的变化做了抽象，可以只将修改的部分同步到真实的 RenderObject 树中，**最大限度上降低真实渲染视图的修改**，提高渲染效率，**而不用销毁整个渲染视图树进行重建**，这就是 Element树存在的意义。


## RenderObject

RenderObject 主要 **负责实现视图渲染的对象**。

-----








































