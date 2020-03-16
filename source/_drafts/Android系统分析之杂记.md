---
title: Android系统分析之杂记
tags:
---



Binder 用于不同进程之间通信，有一个进程的 Binder 客户端向另外一个进程的 Binder 服务端发送事件。

Handler 用于同一个进程中的不同线程间的通信。



## Activity 的生命周期


Activity 的生命周期，其实都是其他线程通过 handler 发送消息给主线程，主线程中的 ActivityThread 中的内部类 H 控制整个核心消息处理机制，通过 H.sendActivity() 来执行相关的函，控制 Activity 的生命周期。


[](http://gityuan.com/2016/03/18/start-activity-cycle/)  作者站在上帝视角，俯视源码，这样更容易把握宏观流程、执行步骤，思路变得很清晰，特别牛。

