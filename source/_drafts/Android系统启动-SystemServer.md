---
title: Android系统启动-SystemServer
tags:
---
SystemServer由Zygote fork生成的，进程名为system_server，该进程承载着framework的核心服务。

Zygote 启动过程中会调用 startSystemServer(),是进程 system_server 的启动起点。



1. 从 zygote 进程 fork 新进程 
2. 关闭新进程中开启的 socket 