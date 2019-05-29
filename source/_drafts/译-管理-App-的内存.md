---
title: '[译]管理 App 的内存'
tags:
---

[官方 Doc](https://developer.android.com/topic/performance/memory.html)


由于手机的物理内存是有限的，所以 RAM(Random Access Memory,随机访问内存) 在手机上的作用更加重要。虽然 ART 和 Davlik 虚拟机会定时进行 GC 操作，释放无用的内存，但是这并不意味着你可以忽略手机关于内存的分配和释放。我们仍然需要避免引入内存泄漏，比如保持静态变量的对象引用等，我们需要在适当的生命周期的回调函数中释放它们。



### 监控可用的内存和内存用途

在解决内存泄漏问题之前，我们需要先找到问题发生的位置， AS 中的  [Memory Profiler](https://developer.android.com/studio/profile/memory-profiler.html#record-allocations) 可以通过以下几个角度来寻找问题。

    * 查找随着时间 App 内存的分配情况。
    * 在 App 运行过程中，手动触发 GC ，并且可以获得 Java 堆栈的快照。
    * 记录 App 的内存分配情况，可以查看每一个分配的对象，并且可以跳转到源代码中。

#### Release memory in response to events

在 []() 的描述中，App 可以以多种方式回收内存，在内存紧张时甚至可以通过杀死 App 来获得内存空间。可以通过 Activity 实现 ComponentCallbacks2 接口，在其回调函数 --  onTrimMemory() 中监听内存相关的事件，在指定的事件时回收对象。