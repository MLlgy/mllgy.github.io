---
title: Android系统分析之杂记
tags:
---



Binder 用于不同进程之间通信，有一个进程的 Binder 客户端向另外一个进程的 Binder 服务端发送事件。

Handler 用于同一个进程中的不同线程间的通信。



## Activity 的生命周期


Activity 的生命周期，其实都是其他线程通过 handler 发送消息给主线程，主线程中的 ActivityThread 中的内部类 H 控制整个核心消息处理机制，通过 H.sendActivity() 来执行相关的函，控制 Activity 的生命周期。


在 [Activity 生命周期](http://gityuan.com/2016/03/18/start-activity-cycle/) 一文中，  作者站在上帝视角，俯视源码，这样更容易把握宏观流程、执行步骤，思路变得很清晰，特别牛。

## 理解 Android 进程创建流程


[理解 Android 进程创建流程](http://gityuan.com/2016/03/26/app-process-create/)


进程的创建流程：

![](/source/images/2020_03_17_02.jpg)

![](/source/images/2020_03_17_01.jpg)


**system_server 进程**

通过 Process.start()发起创建新进程的需求，通过 socket 通道发送给 zygote 进程。


**zygote 进程**

系统中存在多个 zygote 进程，在 fork zygote 进程时，需要将几个特定的 zygote 子进程关闭（Zygote的4个Daemon线程），只留下 zygote 或者 zygote64 进程（其 pid 为 1）。

fork zygote 产生 子进程（pid = 0），然后在子进程中进行相关的操作，比如设置 selinux 上下文、设置子进程进程名等操作。


在子进程中进行一系列的操作后，重启之前关闭的 zygote 进程（Zygote的4个Daemon线程）。

创建新进程的以上过程经历了三步操作：

1. 停止 zygote 的四个子线程运行。
2. fork  zygote 产生子进程，并对子进程进行相关参数、函数的设置。
3. 启动 zygote 的四个子线程。

自此，新进程已经创建完毕。

**新进程的操作**

那些接下来就是在新进程的操作了。

通用初始化

* zygote 初始化

打开 /dev/binder  设备驱动，使用 mmap() 映射内存地址空间，开启 binder 线程池。

应用初始化

handleChildProc 方法

通过抛异常调整执行流程，以反射的方式进入 ActivityThread 的 main 方法，从而进入新进程的主进程的循环状态。

接下来就是创建 Activity/Service 的过程。



copy-on-write 原理：


## 杀进程的实现原理

kill进程其实是通过发送signal信号的方式来完成的。

在 Linux 中信号是跨进程通信的一种方式。


### 用户 kill 进程的流程

 


最终调用 kill(pid,sig) 方法,进行接下来的操作。

![](/source/images/2020_03_17_03.jpg)

![](/source/images/2020_03_17_04.jpg)


### 内核态 kill 动作

* Process.killProcess(int pid): 杀pid进程
* Process.killProcessQuiet(int pid)：杀pid进程，且不输出log信息
* Process.killProcessGroup(int uid, int pid)：杀同一个uid下同一进程组下的所有进程
  
    最终调用 killProcessGroup、killProcessGroupOnce **杀死该进程组中所有的进程**，这也意味着如果 kill <pid>，当 pid 为某个进程的线程时，最终还是会杀死进程。


###   内核态 kill
  
用户端操作 api 杀死进程，最终都会调用到 kill(pid,sig) 方法，该方法位于用户空间的 native 层，通过 Linux 的系统调用进入 Linux 内核的 sys_kill  方法，此时 sig 信号为 9。


根据 pid 和信号杀死相应的进程。

具体流程

![](/source/images/2020_03_17_05.jpg)

在 Process 中定义了 3 个信号：

```
public static final int SIGNAL_QUIT = 3;  //用于输出线程trace
public static final int SIGNAL_KILL = 9;  //用于杀进程/线程
public static final int SIGNAL_USR1 = 10; //用于强制执行GC
```

这三个信号会匹配 Linux 系统中的信号，对于 kill -9 在 Linux 中为 SIGKILL的处理过程，该信号不能被忽略也不能被捕获，所以不会有目标线程的 sigal catcher 线程来处理，而是直接交由 Linux 内核完成。

但对于信号3和10，则是交由目标进程(art虚拟机)的SignalCatcher线程来捕获完成相应操作的，接下来进入目标线程来处理相应的信号。


### sigal catcher 

* kill -3 <pid>：该pid所在进程的SignalCatcher接收到信号SIGNAL_QUIT，则挂起进程中的所有线程并dump所有线程的状态。
* kill -10 <pid>： 该pid所在进程的SignalCatcher接收到信号SIGNAL_USR1，则触发进程强制执行GC操作。

对于 3、10 信号的捕获由 sigalcatcher 线程来捕获完成相关操作的。

对于信号在 Android 进程创建流程中的 ForkAndSpecializeCommon 方法涉及到 sigal 的操作，在这一系列操作中，主要涉及：

* 启动信号捕获 

startSigalCather()

SignalCatcher是一个守护线程，用于捕获SIGQUIT、SIGUSR1信号，并采取相应的行为。

* 处理信号

SignalCatcher::Run，监听相应的信号类型，做出 **输出线程 trace** 和 **强制 GC 的动作**。