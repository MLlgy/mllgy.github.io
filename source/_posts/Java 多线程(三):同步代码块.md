---
title: 'Java 多线程(三):同步代码块'
date: 2019-08-19 15:12:40
tags: [Java 多线程,Java]
---

### 0x0001 synchronize(this) 同步代码块

`synchronize` 同步方法在某些情况下会有一些弊端：比如 A 线程调用同步方法执行一个长时间的任务，那么 B 线程必须等待比较长的时间才可以获得对象锁。在这种情况下可以使用 `synchronize` 同步代码块可以解决。

**synchronize 方法**是对 **当前对象** 进行加锁，而 **synchronize 代码块** 是对 **某一个对象** 进行加锁，当然也包括当前对象。

[SynchronizedMethodBlock](https://github.com/leeGYPlus/JavaCode/blob/master/src/thread/SynchronizedMethodBlock.kt) 中的方法 one、two、three 分别展示了非同步情况、同步方法、同步代码块，通过日志打印可知同步代码块可以有效的避免同步方法执行的低效率。

<!-- more -->
同样的，**同步代码块 synchronize(this) 持有的是当前调用对象的锁**，具体验证可以查看代码：[SynchronizedMethodBlockTest](https://github.com/leeGYPlus/JavaCode/blob/master/src/thread/SynchronizedMethodBlockTest.kt) 中可以验证该结论。

### 0x0002 synchronize(this) 同步代码块间的同步性
在使用同步 `synchronize(this)` 同步代码块需要注意的是，当一个线程访问 object 的一个 `synchronize(this)` 同步代码块时，其他线程对 **同一个 object** 中的所有其他 `synchronize(this)` 同步代码块的访问将被阻塞，这说明 synchronize 使用的 “对象监视器“ 是同一个对象，代码验证：[SynchronizedMethodBlockTestTwo](https://github.com/leeGYPlus/JavaCode/blob/master/src/thread/SynchronizedMethodBlockTestTwo.kt)。

### 0x0003 synchronized 同步方法与synchronzied(this) 同步代码块的同步性
多个线程调用同一个对象中的不同名称的 synchronized 同步方法或 synchronzied(this) 同步代码块时，为同步执行，阻塞执行的，**因为它们的锁对象是同一个**。

这说明 synchronized 同步方法或 synchronized(this) 同步代码块分别有两种作用：

synchronized 同步方法：

1. 对其他 synchronized 同步方法或 synchronized(this) 同步代码块调用呈阻塞状态。
2. 同一时间只有一个线程可以执行 synchronize 同步方法中的代码。

synchronized(this) 同步代码块：

1. 对其他 synchronized 同步方法或 synchronized(this) 同步代码块调用呈阻塞状态
2. 同一时间只有一个线程可以执行 synchronized(this) 同步代码块中的代码。

### 0x0004 将任意对象作为同步代码块的对象监视器


Java 支持将任意对象作为 =对象监视器，**任意参数一般为实例变量或者是方法的参数**，使用格式为：synchronzied(非 this 对象)。

synchronzied(非 this 对象) 同步代码块的作用为：

    在多个线程持有对象监视器为同一个对象的前提下，同一时间只有一个线程可以执行synchronzied(非 this 对象) 同步代码块中的代码。

验证：[SynchronizedMethodBlockTestThree](https://github.com/leeGYPlus/JavaCode/blob/master/src/thread/SynchronizedMethodBlockTestThree.kt) 中 two 方法展示synchronzied(非 this 对象) 的同步效果。

`synchronzied(非 this 对象)` 同步代码块为对象锁，
线程执行 synchronize(非this对象 object)，获得该对象的锁，对于其他线程的访问，存在以下几种情况：
* 当其他线程访问 synchronize(非this对象 object){} 同步代码块时，呈同步效果。
* 当其他线程访问对象 object 中的 synchronize 同步方法时，呈同步效果。
* 当其他线程访问对象 object 中的 synchronize(this) 同步代码方法时，呈同步效果。

----
**知识链接：**

[Java多线程编程核心技术](http://product.dangdang.com/23711315.html)