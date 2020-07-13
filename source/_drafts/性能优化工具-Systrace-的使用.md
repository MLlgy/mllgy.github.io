---
title: '性能优化工具:Systrace 的使用'
tags:
---

### 0x00001 Systrace 是什么？



Systrace 是平台提供的一款工具，用于 记录 **短期内** 的 **设备活动**。该工具会生成一份报告，其中汇总了 **Android 内核中的数据**，**提供一段时间内的系统进程的总体情况**，例如 `CPU 调度程序`、`磁盘活动`和 `应用线程`。还会检查所捕获的跟踪信息，以突出显示它所观察到的问题（例如界面卡顿或耗电量高。也可查看谷歌系列官方文档 [Systrace 概览](https://developer.android.google.cn/studio/profile/systrace.html?hl=zh-cn)、[了解 Systrace](https://source.android.google.cn/devices/tech/debug/systrace?hl=zh-cn)。


### 0x0001 Systrace 基本原理


systrace 利用了 Linux 的 `ftrace` 调试工具，相当于 **在系统各个关键位置都添加了一些性能探针**，也就是在代码里加了一些性能监控的埋点。Android 在 ftrace 的基础上封装了 `atrace`，并增加了更多特有的探针，例如 Graphics、Activity Manager、Dalvik VM、System Server 等。

### 0x0003 Systrace 可以做什么

Android 4.1 新增的属于 sample 类型的分析工具，不能实现类似于 TraceView 的统计应用时间的耗时分析，但是系统预留了 `Trace.beginSection` 接口来监听应用程序的调用耗时，所以可以通过编译期给每个函数插桩的方式实现，在重要的函数的入口和出口分别增加 `Trace.beginSection` 和 `Trace.endSection`，在其原有的基础上 **增加了应用程序耗时的监控**。

通过对函数插桩，可以看到整个流程 **系统** 和 **应用程序** 的调用流程，包括系统关键线程，例如渲染线程、线程锁、线程锁、GC 耗时等。

常用于 **跟踪系统** 的 I/O 操作、CPU 负载、Surface 渲染、GC 等事件。

### 0x0004 Systrace 命令

进入到 Android SDK 相应目录(platform-tools/systrace)，执行如下命令：

```
python systrace.py -h
```
可以查看 Systrace 的全部参数。

Systrace 的使用方法为：

```
 systrace.py [options] [category1 [category2 ...]]
```


Option 常用参数：


* -o: 指定 treace 的输出路径
* -t N，-time=N：执行时间，默认 5s
* -l：展示连接设备的支持的 category
* -a <packageName>,-app=<packageName>: 开启自定义 Trace 功能，如果 App 中有自定义的 Trace Lable，一定要指明该参数
* -b:限制 trace 文件的大小，默认为无限制

设备支持的 category 可以使用 -l 参数获得，具体含义查看知识链接。

### 0x0005 获得 Trace 文件





### 0x0006

* 在 trace 页面搜索关键字：

在页面的右上方直接输入关键字，或者使用快捷键 / 来开启输入框，点击右上角的左、右箭头，切换搜索结果。如何快速跳转到指定的区域，可以使用如下快捷键组合：

1. 点击 m 键，用于标记当前选择区域，再次点击 m 键，可用于取消当前选择区域；
2. 点击 f 键，用于放大当前选取；


----


https://www.cnblogs.com/blogs-of-lxl/p/10926824.html

[再忙也要学Systrace 之 app 冷启动](https://www.jianshu.com/p/6451538fb38b) 本文作者博文也很好看，可以学习