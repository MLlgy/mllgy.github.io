---
title: '性能优化工具:Systrace 的使用'
tags:
---

关于 Systrace 是什么，可以查看谷歌系列官方文档 [Systrace 概览](https://developer.android.google.cn/studio/profile/systrace.html?hl=zh-cn)、[了解 Systrace](https://source.android.google.cn/devices/tech/debug/systrace?hl=zh-cn)。

Systrace 是平台提供的一款工具，用于 记录 **短期内** 的 **设备活动**。该工具会生成一份报告，其中汇总了 **Android 内核中的数据**，**提供一段时间内的系统进程的总体情况**，例如 `CPU 调度程序`、`磁盘活动`和 `应用线程`。还会检查所捕获的跟踪信息，以突出显示它所观察到的问题（例如界面卡顿或耗电量高）。








----


https://www.cnblogs.com/blogs-of-lxl/p/10926824.html