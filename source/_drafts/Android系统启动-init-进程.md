---
title: Android系统启动-Init篇
tags:
---


init进程是Linux系统中用户空间的第一个进程，进程号固定为1。Kernel启动后，在用户空间启动init进程，并调用init中的main()方法执行init进程的职责。对于init进程的功能分为4部分：

* 解析并运行所有的init.rc相关文件
* 根据rc文件，生成相应的设备驱动节点
* 处理子进程的终止(signal方式)
* 提供属性服务的功能


init进程(pid=1)是Linux系统中 **用户空间的第一个进程**，主要工作如下：

* 创建一块共享的内存空间，用于属性服务器;
* 解析各个rc文件，并启动相应属性服务进程;
* 初始化epoll，依次设置signal、property、keychord这3个fd可读时相对应的回调函数;
* 进入无限循环状态，执行如下流程：
    * 检查action_queue列表是否为空，若不为空则执行相应的action;
    * 检查是否需要重启的进程，若有则将其重新启动;
    * 进入epoll_wait等待状态，直到系统属性变化事件(property_set改变属性值)，或者收到子进程的信号SIGCHLD，再或者keychord 键盘输入事件，则会退出等待状态，执行相应的回调函数。
    * 
可见init进程在 _开机之后_ 的核心工作就是 **响应property变化事件** 和 **回收僵尸进程**。

* **响应property变化事件**:当某个进程调用property_set来改变一个系统属性值时，系统会通过socket向init进程发送一个property变化的事件通知，那么property fd会变成可读，init进程采用epoll机制监听该fd则会 触发回调handle_property_set_fd()方法。
* **回收僵尸进程**:在Linux内核中，如父进程不等待子进程的结束直接退出，会导致子进程在结束后变成僵尸进程，占用系统资源。为此，init进程专门安装了SIGCHLD信号接收器，当某些子进程退出时发现其父进程已经退出，则会向init进程发送SIGCHLD信号，init进程调用回调方法handle_signal()来回收僵尸子进程。




----


[Android系统启动-Init篇](http://gityuan.com/2016/02/05/android-init/)