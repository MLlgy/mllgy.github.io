---
title: LeakCanary 是如何工作的
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

    根据 LeakCanary 的内存泄漏 trace 图，发现存在 ChangeHandlerFramLayout 对象泄漏，在 leakmomory 进程中可以详细获知该对象的各个属性值，从而可以判断该对象的状态。在此里中 ChangeHandlerFramLayout 对象的 mAttachInfo = null，说明该对象不再与屏幕关联，即该对象在此时应该被 GC 回收掉，不应该存活，说明内存泄漏的原因在 trace 图上面的对象。

![trace 图](/../images/2019_07_05_01.jpg)

![ChangeHandlerFramLayout 对象](/../images/2019_07_05_02.jpg)

![ChangeHandlerFramLayout 对象的 mAttachInfo 值](/../images/2019_07_05_03.jpg)



接下来分析了位于 ChangeHandlerFramLayout 上的 MainActivity 实例对象，同理，在 leakmomory 查看该实例的状态，与ChangeHandlerFramLayout 对象不同的是此时我们关注属性值为 mDestory = true，说明 MainActivity 对象已经销毁，说明该对象也应该被 GC 回收。

如此往复，结合 trace 图和 leakmomory 进程信息，判断对象是否应该存在，不过根据对象的不同，判断标准不同，如例子中的 mAttachInfo、mDestory。


---
[[Uber Mobility] Memory Leak Hunt: LeakCanary - Pierre-Yves Ricau](https://www.youtube.com/watch?v=KwArTJHLq5g)



---

https://mp.weixin.qq.com/s/uwDk5D986OdMzKgtjpHdHg

https://mp.weixin.qq.com/s/idjFaJsLpVLw52RSYHA_Vg