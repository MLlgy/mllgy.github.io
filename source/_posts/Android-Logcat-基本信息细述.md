---
title: Android Logcat 基本信息细述
date: 2020-02-24 14:51:48
tags:
---


## Logcat 信息格式


> date time PID-TID/package priority/tag: message


具体为：

> 12-10 13:02:50.071 1901-4229/com.google.android.gms V/AuthZen: Handling delegate intent.

PID 代表进程标识符，TID 则为线程标识符；如果仅有一个线程，两者可以相同。

## 读取垃圾回收消息

### Dalvik 日志消息-


在 Dalvik（而不是 ART）中，每个 GC 都会将以下信息输出到 logcat：

> D/dalvikvm(PID): GC_Reason(GC 原因) Amount_freed(释放内存量), Heap_stats(堆统计数据), External_memory_stats(外部内存统计数据), Pause_time(暂停时间)

实例：

> D/dalvikvm( 9050): GC_CONCURRENT freed 2049K, 65% free 3571K/9991K, external 4703K/5261K, paused 2ms+2ms


常见的 GC 原因：


* GC_CONCURRENT
  
在堆 **开始占用内存时** 释放内存的并发 GC。

* GC_FOR_MALLOC

您的 **堆已满** 而系统不得不停止您的应用并回收内存时，应用 **尝试分配内存** 而引起的 GC。

* GC_HPROF_DUMP_HEAP

**请求创建 HPROF 文件**，来分析堆时发生的 GC。

* GC_EXPLICIT

**显式 GC**，例如当您调用 gc() 时（您应避免调用它，而应信任 GC 会根据需要运行）。

* GC_EXTERNAL_ALLOC

这仅适用于 API 级别 10 及更低级别（更新的版本会在 Dalvik 堆中分配任何内存）。外部分配内存的 GC（例如存储在原生内存或 NIO 字节缓冲区中的像素数据）。


#### 暂停时间

堆越大，暂停时间越长。并发暂停时间显示两个暂停：一个出现在 **回收开始时**，另一个出现在 **回收快要完成时**。

在此类日志消息积聚时，请注意 **堆统计数据**（上面示例中的 3571K/9991K 值）的增大情况。如果此值继续增大，可能会出现内存泄露。


### ART 日志消息


ART 不会为未明确请求的 GC 记录消息。只有在系统认为 GC 速度较慢时才会输出 GC 消息。更确切地说，仅在 **GC 暂停时间超过 5 毫秒** 或 **GC 持续时间超过 100 毫秒时**，才会输出 GC 记录日志。


输出格式：

>  I/art: GC_Reason GC_Name Objects_freed(Size_freed) AllocSpace Objects,
        Large_objects_freed(Large_object_size_freed) Heap_stats LOS objects, Pause_time(s)
    
GC 原因查看官方链接
[使用 Logcat 写入和查看日志](https://developer.android.google.cn/studio/debug/am-logcat?hl=zh_cn#format)