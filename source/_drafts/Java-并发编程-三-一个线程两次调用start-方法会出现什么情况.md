---
title: 'Java 并发编程(三): 一个线程两次调用start()方法会出现什么情况'
tags:
---


## 概述


Java 线程是不允许两个


## 线程是什么


在操作系统的角度，线程是系统调度的最小单位，一个进程可以包含多个线程，线程作为任务的真正运作者，有自己的栈、寄存器、本地存储等，线程会和本进程中的其他线程共享文件操作符、虚拟地址空间等。


在具体实现中，线程还分为内核线程、用户线程，**Java 的线程实现其实是与虚拟机相关的**。在 JDK1.2 之后，JDK 已经抛弃了所谓的Green Thread，也就是用户调度的线程，现在的模型是 **一对一映射到操作系统内核线程**。

Thread 的源码，你会发现其基本操作逻辑大都是以 JNI 形式调用的本地代码。

```
private native void start0();
private native void setPriority0(int newPriority);
private native void interrupt0();
```

## 线程的基本操作


1. 继承 Thread
2. 实现 Runnable

Runnable 的好处是，不会受 Java 不支持类多继承的限制，重用代码实现，当我们需要重复执行相应逻辑时优点明显。而且，也能更好的与现代 Java 并发库中的 Executor 之类框架结合使用,这样我们就不用操心线程的创建和管理，也能利用 Future 等机制更好地处理执行结果。

```
Runnable task = () -> {System.out.println("Hello World!");};
Future future = Executors.newFixedThreadPool(1)
.submit(task)
.get();
```


### 守护线程


有的时候应用中需要一个长期驻留的服务程序，**但是不希望其影响应用退出**，就可以将其设置为守护线程，如果 JVM 发现只有守护线程存在时，将结束进程，具体可以参考下面代码段。注意，**必s须在线程启动之前设置**。
```
Thread daemonThread = new Thread();
daemonThread.setDaemon(true);
daemonThread.start();
```

## 线程的状态

* 新建(NEW)

线程被创建出来还没有真正启动的状态。

* 就绪(RUNNABLE)

线程已经在 JVM 中执行，当然由于执行需要计算资源，它可能是正在运行，也可能还在等待系统分配给它 CPU 片段，在就绪队列里面排队。

* 运行(RUNNING)

在 Java API 层面没有体现这一状态。


* 阻塞(BLICKED)

和同步相关，阻塞表示线程在等待 Monitor lock。比如，线程试图通过 synchronized 去获取某个锁，但是其他线程已经独占了，那么当前线程就会处于阻塞状态。

* 等待(WAITING)

正在等待其他线程采取某些操作,正在等待其他线程采取某些操作。

* 终止（TERMINATED）

不管是意外退出还是正常执行结束，线程已经完成使命，终止运行。


![](/source/images/2020_02_09_01.png)


## 影响线程状态的方法


### Thread 的一些方法

线程自身的方法，除了 start，还有多个 join 方法，等待线程结束；yield 是告诉调度器，主动让出 CPU；另外，就是一些已经被标记为过时的 resume、stop、suspend 之类，据我所知，在 JDK 最新版本中，destory/stop 方法将被直接移除。


### Object 的一些方法


基类 Object 提供了一些基础的 wait/notify/notifyAll 方法。

如果我们持有某个对象的 Monitor 锁，调用 wait 会让 **当前线程** 处于等待状态，直到其他线程 notify 或者 notifyAll。所以，本质上是提供了 Monitor 的获取和释放的能力，是线程间的基本通信方式。


### 并发工具类的方法

比如 CountDownLatch.await()。




## TIP

慎用 ThreadLocal。

Java 提供的一种保存线程私有信息的机制，因为其在整个线程生命周期内有效，所以可以方便地在一个线程关联的不同业务模块之间传递信息，比如事务 ID、Cookie 等上下文相关信息。

当 Key 为 null 时，该条目就变成“废弃条目”，相关“value”的回收，往往依赖于几个关键点，即 set、remove、rehash。

通常弱引用都会和引用队列配合清理机制使用，但是 ThreadLocal 是个例外，它并没有这么做。


废弃项目的回收依赖于显式地触发，否则就要等待线程结束，进而回收相应 ThreadLocalMap！这就是很多 OOM 的来源，所以通常都会建议，应用一定要自己负责 remove，并且不要和线程池配合，因为 worker 线程往往是不会退出的