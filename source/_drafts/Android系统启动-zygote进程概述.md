---
title: Android系统启动-zygote进程概述
tags:
---

## Android 系统概述

Android 系统中存在两个截然不同的两个世界：

* Java 世界： Google 针对这个世界提供 SDK，在这个世界运行的程序为基于 Dalvik 虚拟机的 Java 程序。
* Native 世界：使用 Native 语言（C或者 C++）开发的程序。

在这样的 Android 世界中存在两个比较重要的进程：Zygote 进程 和 system_server 进程：

* Zygote 进程：和 Android 系统中的 Java 世界有重要的关系。
* system_server 进程：系统重要的服务（Service）都驻留在 Java 世界中。

这两个进程是在 Android 系统是 Java 世界的半边天，任何一个进程死亡，都会导致系统崩溃。


## Native世界中的 zygote 

zygote 本身是一个 native 的应用程序，与内核、驱动无关，在 init 进程其中过程中，解析 init.rc 文件创建。

zygote 对应的文件为 App_main.cpp，通过源码可以看到重要工作是由 AppRuntime 的 start 函数完成。

在 AppRuntime::start 方法中，涉及关键的点如下：
* 创建虚拟机-startVM
* 注册JNI函数-startReg
* 通过 JNI 调用 Java 函数，具体为调用 ZygoteInit#main 方法


基于以上3点，它们开创了 Android 系统的 Java 世界，至此进入了 Android 系统中的 Java 世界。


## Java 世界中 Zygote

ZygoteInit#main 涉及的主要流程：


1. 建立 IPC 通信服务端——registerZygoteSocket

在 Android 系统中主要通过 Binder 进行 IPC 通信，但是 zygoet 与系统中其他程序没有使用 Binder，而是使用了 Socket，而 registerZygoteSocket 函数就是为了建立这个 Socket

2. 预加载类和资源——preloadClasses、preloadResources

    * 预加载通用类(`/system/etc/preloaded-classes`)
    * drawable和color资源(`com.android.internal.R.array.preloaded_drawables`、`com.android.internal.R.array.preloaded_color_state_lists`、`com.android.internal.R.array.preloaded_freeform_multi_window_drawables`)
    * openGL:`EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY)`
    * 共享库:`System.loadLibrary("android");
        System.loadLibrary("compiler_rt");
        System.loadLibrary("jnigraphics");`

    preloaded-classes 包含的类有很多，所以预加载类执行的时间较长，这是 Android 系统启动比较慢的原因之一。preResources 加载 framework-res.apk 中的资源。


3. 启动 system_serve——startSystemServer


这样函数会创建在 Java 世界中 **系统 Service** 所 **驻留的进程 system_server**，如果该进程死了，会导致 zygote 自杀或者重启。

在该函数中 zygote 进程会执行 fork 操作——`Zygote.forkSystemServer`，产生一个 sysytem_server 进程。


4. 有求必应之等待请求——runSelectLoopMode


在关键点一中，`registerZygoteSocket` 主注册了一个用于 IPC 的 Socket，此 Socket 在runSelectLoopMode发挥作用。


runSelectLoopMode 的具体工作：

1. 处理客户连接和客户请求，客户在 ZygoteServer 中用 ZygoteConnection 表示。
2. 客户的请求由 ZygoteConnection 的 processOneCommand 处理。



## SystemServer 进程


SystemServer 进程的名字为 system_server，为 Zygote 进程 fork 产生的第一个子进程。


## SystemServer 进程的诞生