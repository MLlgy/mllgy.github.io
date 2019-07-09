---
title: Hprof 拾遗
date: 2019-07-09 17:43:11
tags: [Hprof,内存泄漏,工具]
---


### 0x0001 引用一

以下内容来自：[Different Ways to Capture Java Heap Dumps](https://www.baeldung.com/java-heap-dump-capture)

A heap dump is a snapshot of all the objects that are in memory in the JVM at a certain moment. They are very useful to troubleshoot memory-leak problems and optimize memory usage in Java applications. **Heap dumps are usually stored in binary format hprof files.** 

<!-- more -->

### 0x0002 引用二


以下内容来自：[Understanding heap dumps](https://www.ibm.com/support/knowledgecenter/en/SS3KLZ/com.ibm.java.diagnostics.memory.analyzer.doc/heapdump.html)

**A heap dump is a snapshot of the memory of a Java™ process**.

The snapshot contains information about the Java objects and classes in the heap at the moment the snapshot is triggered. Because there are different formats for persisting this data, there might be some differences in the information provided. Typically, a full garbage collection is triggered before the heap dump is written, so the dump contains information about **the remaining objects in the heap**.

The Memory Analyzer works with HPROF binary heap dumps, IBM® system dumps, and IBM portable heap dumps (PHD) from various platforms. See Supported dump file types.

Typical information in a heap dump, depending on the heap dump type, includes(堆转储文件中包含的信息):

**All Objects**
    
    Class, fields, primitive values, and references.
**All Classes**

    Class loader, name, super class, and static fields.

**Garbage collection roots**

    Objects defined to be reachable by the JVM.
**Thread Stacks and Local Variables**

    Call-stacks of threads at the moment of the snapshot, and information about local objects on a frame by frame basis.


### 0x0003 Android 中的 Hprof

```
 Debug.dumpHprofData(String filePath);
```
在 Android 中使用以上代码将对转储文件生成到指定的文件上。

在 AS 中如何分析 Hprof 见官方文档：[使用 Memory Profiler 查看 Java 堆和内存分配](https://developer.android.com/studio/profile/memory-profiler)

