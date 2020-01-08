---
title: Flutter核心技术与实战：经典布局
tags:
---

Flutter 中所有的布局列表，没事多查阅：[Widgets 目录](https://flutterchina.club/widgets/)


[原文链接](https://flutter.dev/docs/development/ui/widgets)


# 如何定义子控件在父容器中的排版位置

## 布局 Widget

布局控件与基本控件元素不同，布局类 Widget 不会直接呈现视图内容，而是作为承载其他子 widget 的容器存在。Flutter 中 31 中布局 Widget，布局 Widget 内部会包含一个或者多个子控件，并且提供了布局方式，实现对子控件对齐、嵌套、层叠和缩放等效果。


代表性布局 Widget：单子 Widget 布局、多子 Widget布局、层叠 Widget 布局。


## 单子 Widget布局：Container、Padding、Center

单子 Widget 布局，顾名思义是对其唯一的子 Widget 进行样式包装。


### Container

可以作为单独的控件，也可以作为其他控件的父容器，Container 定义了子 Widget 如何摆放，以及如何展示。


### Padding

可以通过 Padding 设置子 Widget 的间距。


### Center

Center 会将其子 Widget 居中排列。


## 多子 Widget 布局：Row、Column、Expanded

Row：将子 Widget 水平排列
Column：将子 Widget 竖直排列
Expanded：分配子 Widget在布局方向（行/列）中剩余空间


Row、Column 的主轴与纵轴


设置主轴方向上容器的大小与子容器的关系，比如 Row 的 mainAxisSize 的


## 层叠 Widget 布局:Stack、Positioned


Stack 提供层叠布局的容器，而 Positioned 则提供了设置子 Widget 位置的能力。Positioned只能在 Stack 中使用。