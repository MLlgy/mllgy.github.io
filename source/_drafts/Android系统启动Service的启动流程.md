---
title: Android系统启动Service的启动流程
tags:
---


### 新进程中的 Service 的启动流程

涉及进程：调用者进程、system_server 进程、Zygote 进程、新创建的目标进程


创建过程中 Service 会发送以下 delay 信息：create（Service 的 onCreate 超时检测）、start（Service 的 onStartCommon 的超时检测）

1. 由调用者进程发起启动 Service 的请求 =》 system_server 进程启动 Service

动作发生的进程：发起者进程。
交互手段：Binder。

IPC 涉及到的类：AcitivyManagerNative、ActivityManagerProxy、ActivityManagerServer、IActivityManger（前三者均是该接口的实现类）。


两进程间通过 Binder 实现跨进程实现请求的传递，接下来动作将发生在 system_server 进程中。


2. system_server 进程与 Zygote 进程间交互，fork 产生新的进程

动作发生的进程：system_server 进程。
交互手段：Socket。
创建新进程的动作：AMS#startProcessLocked。


通过 AMS#startProcessLocked 创建新进程后，经过层层调用后，最后会调用 AMS#attachApplicationLocked，至于为什么会调用，请看 [Android四大组件与进程启动的关系](http://gityuan.com/2016/10/09/app-process-create-2/) 中 3.1~3.7 的概述。接下来调用 ActiveServer#realStartServiceLocked 在 system_server 进程中发起真正开启 Service 的请求，比如为了执行 onCreate 而执行的一系列动作。


3. 发起执行 Service 的具体的生命周期函数的动作：onCreate、onStartCommon

执行进程的变化：由 system_server 发起动作 =》最终在目的进程执行 Service 的生命周期函数
交互的手段：Binder
涉及类：ApplicationThreadProxy、ApplicationThreadNative、ApplicationThread、IApplicationThread

最终通过 Android 系统中的消息机制，将发起 Service 的动作发送到主线程的消息队列中，等待该消息的具体执行。


4. 在目的进程执行 Service 的具体生命周期函数。

1. 通过反射的技术，加载具体的 Serice 类，创建相应的对象
2. 创建 ContextImpl、Application 对象，Service 关联上下文对象（context、Application）
3. 调用服务的生命周期函数
4. 发起完成 Service 创建成功的动作，





根据前后台服务，有不同的延时时间：

前台服务：20s
后台服务：200s


![](http://gityuan.com/images/android-service/start_service/start_service_processes.jpg)



---

---

[startService启动过程分析](http://gityuan.com/2016/03/06/start-service/)

[Android四大组件与进程启动的关系](http://gityuan.com/2016/10/09/app-process-create-2/)