---
title: Flutter 介绍
date: 2019-03-27 10:25:00
tags:
---



* main函数使用了(=>)符号, 这是Dart中单行函数或方法的简写
* 在 Flutter 中大部分都是 widget，包括对齐(alignment)、填充(padding)、布局(layout)。
* Scaffold(脚手架) 是Material library 中提供的一个widget, 它提供了默认的导航栏、标题和包含主屏幕widget树的body属性。
* widget 的主要工作是提供一个build()方法来描述如何根据其他较低级别的widget来显示自己。
* 变量以下划线(_) 为前缀时，在 Dart 会被强制将其变为私有的。

### 路由
* 在 Flutter 中页面称为路由(Router)。
* 在 Flutter 中，**导航器** 管理应用程序的 **路由栈**，将路由推入 (push) 到导航器的栈中，将会显示该路由界面，从导航器中弹出 (pop) 路由，将返回到前一个路由(页面)。