---
title: Launcher 相关
tags: [Launcher]
---

### 0x0001 Laucnher 监听 package 的变化过程

涉及知识点： AIDL 跨进程通信

基本流程：
0. 注册回调 CallBack
1. Launcher 获取 launcerapp 服务，用来监听 package 的变化
2. launcerapp 服务内部通过注册广播([PackageMonitor](https://opengrok.pt.xiaomi.com/opengrok3/xref/miui-r-venus-dev/frameworks/base/core/java/com/android/internal/content/PackageMonitor.java))来监听 package 的变化
3. 在接收到广播时，通过 AIDL 获得 0 中注册的 CallBack 执行相关操作

具体可参见 [Android M Launcher3监听packages变化实现过程](https://www.jianshu.com/p/e0f6a8f1dfda)

----

**知识链接：**

[Android M Launcher3监听packages变化实现过程](https://www.jianshu.com/p/e0f6a8f1dfda)