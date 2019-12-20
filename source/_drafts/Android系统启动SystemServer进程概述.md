---
title: Android系统启动SystemServer进程概述
tags:
---


## SystemServer进程概述

Zygote 通过Zygote.forkSystemServer 函数克隆出  SystemServer进程，那么  SystemServer 进程的主要职责是什么呢？fork 成功后，通过调用 handleSystemServerProcess 函数来承担自己的职责，在该函数中进行一系列操作，比如关闭从zygote fork 而来的 socket、设置进程的参数、调用 RuntimeInit.zygoteInit函数。


在 RuntimeInit.zygoteInit 函数中存在以下几个关键点：

1. native 层的初始化——zygoteInitNative

具体实现在 AndroidRuntime.cpp 中，具体工作为启动了一个线程，该线程主要用于 Binder 通信。

zygoteInitNative 在调用后，将与 Binder 通信系统建立联系，这样 SystemServer 进程就可以使用了 Binder 了。

2. 调用 invokeStaticMain 函数，通过反射调用 Java 世界的 SystemServer#main 函数。

而最后之所以能够顺利的执行 SystemServer#main 函数，是因为在代码中建立的异常机制，具体在梳理代码详述。


注意这一系列动作的执行是由 handleSystemServerProcess 触发的，而 handleSystemServerProcess 执行的线程为 fork Zygote 进程产生的新进程，即 system_servert：

```
if(pid = 0){
    handleSystemServerProcess(XXX);
}
```

3. SystemServer 


在第二步中，最终调用了 SystemServer 的 main 函数，而这一切还都是发生在 system_server 进程中，记住这一点是十分重要的。


在 SystemServer#main 的主要工作是：

* 创建各种 SystemContext，比如：ActivityThread、Instrumentation、ContextImpl、LoadedApk、Application
* 加载 libandroid_server.so 库，通过 SystemServerManager 对象启动各种服务，比如 Installer、AMS、PMS 等服务


**Java 世界核心的 Service 都在这里启动**，至此 system_server 进程的启动工作总算完成了，进入了 Looper.loop 状态，等待其他线程通过 Handler 发送消息到主线程中在进行处理。



---

此处遗留的疑问：


* SystemServer 运行在 system_server 进程中，那么如何能够调用 Loop.loop(),

此时的线程为何物，不是运行在 进程中吗？


导致进程中的主线程加入到 Binder 的通信大潮中，此处的主线程是 system_server 中的主线程吗，还是 Android 系统层面的主线程。
