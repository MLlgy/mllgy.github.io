---
title: Android系统分析之帮助理解Android系统的流程图
tags:
---


### 进程创建流程

![](/source/images/2020_03_17_01.jpg)


1. 发起进程

发起进程通过 binder 将创建进程的消息发送给 sysytem_server 进程。

2. system_server 进程

调用 Process.start() 通过 Socket 的跨进程方式，向 Zygote 进程发送消息。


3. Zygote 进程

以 Zygote.main() 为起点，执行 runSelectLoop() 轮询是否有请求，一系列函数调用后，通过 fork zygote 进程产生新进程。

4. 新进程

调用 handleChildProc()，调用 ActivityThread.main()，开启在新进程中的活动。