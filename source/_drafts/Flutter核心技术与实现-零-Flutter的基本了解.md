---
title: 'Flutter核心技术与实现(零):Flutter的基本了解'
tags:
---




## 跨平台

1. web

99% 的需要能够得到满足。

问题：性能和体验与原生较大差距。

2. rn

接近于原生的 rn，对业务的支持能不不足浏览器的 5%，适合中低复杂度的低交互页面。

3. flutter

使用 Native 引擎渲染视图，提供丰富的组件和接口。


## 与其他跨平台方案的区别


* React Native 之类的框架，只是通过 JavaScript 虚拟机扩展，调用系统组件，由 Android 和 iOS 系统进行组件的渲染；

* Flutter 则是自己完成了组件渲染的闭环。


图片显示原理：

计算机系统中，图像有 cpu、gpu、显示器共同完成。

CPU 负责图像数据的计算
GPU 负责图像数据的渲染
显示器 负责图像的显示

图片显示机制：

CPU 把计算好的需要显示的显示内容交给 GPU，GPU 完成渲染后放入 **帧缓冲区**，随后 **视频控制器** 根据 **垂直同步信号(Vsync)** 以 **每秒 60 次的速度**，从 **帧缓冲区** 读取 **帧数据** 交给显示器完成显示。


### Flutter 绘制原理

Flutter 绘制原理示意图：



![](https://static001.geekbang.org/resource/image/95/2a/95cb258c9103e05398f9c97a1113072a.png)

Flutter 在两个硬件时针的 VSync 信号之间计算并合成显示视图数据，然后通过 Skia 交给 GPU 进行渲染：UI 线程使用 Dart 语言构建视图结构数据(对应 CPU 计算视图数据)，而这些数据会在 GPU 线程进行图层合成，随后交给 Skia 引擎加工成 GPU 数据，而这些数据会通过 OpenGL 最终提供给 GPU 进行渲染。

### Skia 是什么

Flutter 的底层图像渲染引擎为 Skia