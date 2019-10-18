---
title: Rxjava 源码学习(二):基本流程流程图
tags:
---


在 [Rxjava 源码学习(一):基本流程分析]() 分析了基本流程，并且通过 Map 操作符一窥 Rxjava 操作符的特色，而本篇主体只有一张 Rxjava 流程图，但是这张流程图基本上可以概括 Rxjava 的框架，整个流程可以看做是一个 “横向 S” 模型。


该图共涉及 Rxjava 事件流的以下几个方面：
1. Observable 的创建
2. Observer 的创建、产生订阅关系
3. 订阅关系的传递
4. 取消订阅的流程


<!-- more -->
具体看图上的标记。

![](/../images/2019_10_18_01.png)



从图上可以看到，在最终订阅 Observer 之前，执行每一个操作符并不会同时生成相应的 Observable 和 Observer，以调用 subscribe 为分界线，将整个事件流分成两部分：
1. 调用 subscribe 之前，生成相应操作符的 Observable。
2. 调用 subscribe 之后，生成相应操作符的 Observer，并产生订阅关系。

需要注意的一点是在查看源码会看到 upstream、downstream，具体的 up 和 down 不是有相应对象的生成顺序决定的，而是有 Rxjava 相应操作符的调用先后决定。




