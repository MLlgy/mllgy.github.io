---
title: Binder基本原理
date: 2019-06-05 13:33:02
tags: [Binder]
---


Binder 机制是 Android 特有的跨进程通信机制，本文为 [Android 插件化开发]() 相关章节的阅读笔记。


#### 00x01

Binder 分为 Client 端和 Server 端，在进行跨进程通信时，两端分别为两个不同的进程。发消息者为 Client，相应的接收消息者为 Server。根据消息的流向，一端可以同时为 Client 和 Server。

#### 00x02

Binder 组成：

* Client
* Server
* ServerManager(管理 Server )

<!-- more -->

#### 00x03

为了更好的帮助读者理解，我们可以看一个场景：**打电话**。

* ServerManager: 相当于电话局，储存每个用户的电话。
* Client: 用户 A
* Server: 用户 B

`用户 A 向用户 B 拨打电话`

1. A 拨打电话
2. 电话会被转接到电话局
3. 电话局查询 A 拨打的电话，若电话存在，则接通电话；若电话不存在，则提示该号码不存在。

在这个过程中还有一个十分重要的角色：**接线员**，它做了很多事情，承担了十分重要的角色，映射到 Binder 机制，接线员相当于 **Binder 驱动**。

#### 00x04

Binder 基本运行机制如下图：

![Binder 运行机制](/../images/2019_06_05_01.jpg)


#### 00x05 Binder 的通信过程

假如 Client 进程调用 Server 进程中的方法 add(),因为两者隶属于不同进程，此时需要在 Binder 的协助下完成 IPC(Inter-process Communication) 通信。

1. Server 端在 SM(ServerManager) 中完成注册。
2. Client 要想调用 Server 的 add 方法，必须获得 Server 对象，但是 SM 不会吧真正的 Server 对象返回给 Client，而是把 Server 的代理对象 Proxy 返回给 Client。
3. Client 调用 Proxy 中的 add 方法，SM 会帮助它调用 Server 的 add 方法，并把结果返回给 Client。

此时 Client 和 Server 实现了进程间通信，在此过程中 Binder 驱动做了许多事情，但是我们目前不需要关心。


![Binder 通信过程](/../images/2019_06_05_02.png)


#### 00x06

在研究系统框架或第三方库时，我们不要陷入细节中，要先熟悉基本的运行机制、运行流程，根据自我需求，选择性的查看实现细节，但是需要注意的还是不能陷入细节，要有的放矢的跳出细节，把控全局。
