---
title: LeakCanary 是如何工作的
date: 2019-07-04 15:40:01
tags:
---




#### 如何使用 Leakmemory 泄漏链路


> [19:30] You look for objects that you know you try to ask the question throught subject should be in memory or not. If you can anwser the question, it's greate because you're going to help you reduce the space of the problem. If the anwser is yes that it should be in memory, the problem is in blow, if the anwser is no, the problem is above.

> [12:37] Method: 
> Find an object and ask should this object should be alive or should be in garbage collection?


#### 如何使用 package:leakmomory 进程进行问题定位


should this object should be alive at this point in time? how we know that?

视频中 15：00 分钟左右

不只是更具 leakcanary 在手机上的 trace 图，leakcanary 可以在 package:leakmomory 进程显示详细的 trace 日志。

![展示](/../images/2019_07_03_05.jpg)

图中包名显示为 package:leakmomory 线程，以下为 trace 的详细信息。



[16：00]
we will dump the states of every singleInstance and that's where you can find the problem .

可以在这里可以根据实例对象或其属性值判断每个实例是否应该被回收，当然这需要相应有关该类的一些知识。如果该实例对象应该被回收，那么说明内存泄漏的对象在上面。


在上面的视频中的一个例子：

    根据 LeakCanary 的内存泄漏 trace 图，发现存在 ChangeHandlerFramLayout 对象泄漏，在 leakmomory 进程中可以详细获知该对象的各个属性值，从而可以判断该对象的状态。在此里中 ChangeHandlerFramLayout 对象的 mAttachInfo = null，说明该对象不再与屏幕关联，即该对象在此时应该被 GC 回收掉，不应该存活，说明内存泄漏的原因在 Trace 图上面的对象。

![trace 图](/../images/2019_07_05_01.jpg)

![ChangeHandlerFramLayout 对象](/../images/2019_07_05_02.png)

![ChangeHandlerFramLayout 对象的 mAttachInfo 值](/../images/2019_07_05_03.png)



接下来分析了位于 ChangeHandlerFramLayout 上的 MainActivity 实例对象，同理，在 leakmomory 查看该实例的状态，与ChangeHandlerFramLayout 对象不同的是此时我们关注属性值为 mDestory = true，说明 MainActivity 对象已经销毁，说明该对象也应该被 GC 回收。

如此往复，结合 trace 图和 leakmomory 进程信息，判断对象是否应该存在，不过根据对象的不同，判断标准不同，如例子中的 mAttachInfo、mDestory。


### LeakCanary 实践

使用官方 [LeakCanary](https://github.com/square/leakcanary/tree/jrod/2019-04-15/hprof) 库的 demo 进行展示如何定位内存泄漏。此处演示的为 LeakCanary 1.6.3 ，master 分支已于 2019-05-21 变更为 由 1.x 变更为 2.x ，具体查看 [Chnage Log](https://github.com/square/leakcanary/blob/4bbc0f6f2e3c9a25ca890ece6770f81cf9059510/docs/changelog.md)。


LeakCanary 2.x 功能更加全面，需要使用。

Demo 的 Trace 图为：

![Trace 图](/../images/2019_07_05_04.png)

点击 MainActivity 所在行，显示详细信息。

![MainActivity 相关信息](/../images/2019_07_05_05.png)

可以发现 MainActivity 的 mDestoryed = true,说明 MainActivity 应该被 GC 回收，那么内存泄漏的应该发生在 **之上**。


点击 MainActivity$2.this$0 显示具体信息，

![MainActivity$2.this.0 相关信息](/../images/2019_07_05_06.png)



从图中得知 MainActivity$2.this$0 为 anonymous implent Runnable ，匿名对象。this$0 为 com.example.MainActivity,此时 this$0 在旋转屏幕后需要被回收，但是该对象依旧存在，说明匿名对象 this$0 导致了 MainActivity 对象的泄漏。

匿名对象的具体代码如下：
```
 Runnable work = new Runnable() {
    @Override
    public void run() {
        SystemClock.sleep(20000);
    }
};
new Thread(work).start();
```
这在 Java 中是一个经典的内存泄漏的案例，原因是匿名对象持有外部类的引用引起的，我们要做的就是将匿名对象静态化。

```
private static class CustomRunnable implements Runnable {
    @Override
    public void run() {
        SystemClock.sleep(20000);
    }
}
CustomRunnable work = new CustomRunnable();
new Thread(work).start();
```




---




---

[leakcanary 官方网站](https://square.github.io/leakcanary/)

[leakcanary 基本原理](https://square.github.io/leakcanary/fundamentals/)

[Youtube 视频：[Uber Mobility] Memory Leak Hunt: LeakCanary - Pierre-Yves Ricau](https://www.youtube.com/watch?v=KwArTJHLq5g)

[LeakCanary 2: Leaner, Better, Faster, Kotliner! by Pierre-Yves Ricau, Square, Inc EN](https://www.youtube.com/watch?v=LEX8dn4BLUw)

[Detect all memory](https://medium.com/square-corner-blog/leakcanary-detect-all-memory-leaks-875ff8360745)

https://mp.weixin.qq.com/s/uwDk5D986OdMzKgtjpHdHg

https://mp.weixin.qq.com/s/idjFaJsLpVLw52RSYHA_Vg