---
title: Android系统启动SystemServiceManager概述
tags:
---



## 概述


ServiceManager 是 Binder IPC 通信过程中的守护进程，


## 启动

ServiceManager 是nit 进程通过解析 init.rc 文件而创建的，对应的源代码为 service_manager.c，进程名为 servicemanatger


