---
title: 如何使用 LeakCanary(1.6.3之前版本) 寻找内存泄漏位置
date: 2019-07-04 15:40:01
tags: [LeakCanary,内存泄漏,内存优化,工具]
---

### 如何使用 Leakmemory 泄漏链路(Trace 图)

以下英文部分摘自 [Youtube 视频：[Uber Mobility] Memory Leak Hunt: LeakCanary - Pierre-Yves Ricau](https://www.youtube.com/watch?v=KwArTJHLq5g)

> [19:30] You look for objects that you know you try to ask the question throught subject should be in memory or not. If you can anwser the question, it's greate because you're going to help you reduce the space of the problem. If the anwser is yes that it should be in memory, the problem is in blow, if the anwser is no, the problem is above.

> [个人译] 如果你知道在 Trace 图中出现的对象是否应该存在于内存中，这会帮助你很好的分析内存泄漏位置。如果对象不应该存在于内存中，那么内存泄漏的位置应该在Trace 中该对象上面的位置；如果对象应该存在于内存中，那么内存泄漏的位置应该在Trace 中该对象下面的位置。

> [12:37] Method: 
> Find an object and ask should this object should be alive or should be in garbage collection?

<!-- more -->

### 如何使用 package:leakmomory 进程进行问题定位

以下英文部分摘自 [Youtube 视频：[Uber Mobility] Memory Leak Hunt: LeakCanary - Pierre-Yves Ricau](https://www.youtube.com/watch?v=KwArTJHLq5g)

> [15：00] Should this object should be alive at this point in time? how we know that?

不只是根据 leakcanary 在手机上的 Trace 图，leakcanary 可以在 package:leakmomory 进程显示详细的 Trace 日志。其实现在手机上的 Trace 图也可以显示出关键的 Trace 日志，只是手机品牌不同显示的详细程度不同，如果手机上信息语言简略，推荐查看 AS 中 package:leakmomory 中的相关日志。


<!-- ![展示](/../images/2019_07_03_05.jpg) -->
**package:leakmomory 中关于对象的详细信息**：

<img src="/../images/2019_07_03_05.jpg" height="70%" width="70%">

图中包名显示为 package:leakmomory 线程，以下为 Trace 的详细信息。

> [16：00] We will dump the states of every singleInstance and that's where you can find the problem.

可以在这里可以根据实例对象或其属性值判断每个实例是否应该被回收，当然这需要相应有关该类的一些知识。如果该实例对象应该被回收，那么说明内存泄漏的对象在上面。


在上面的视频中的一个例子：

    根据 LeakCanary 的内存泄漏 Trace 图，发现存在 ChangeHandlerFramLayout 对象存在泄漏链路中，在 leakmomory 进程中可以详细获知该对象的各个属性值，从而可以判断该对象的状态。
    
    在此处中 ChangeHandlerFramLayout 对象的 mAttachInfo = null，说明该对象不再与屏幕关联，该对象在此时应该被 GC 回收掉，不应该存活，说明内存泄漏是 Trace 图上面的对象引起的，进一步定位内存泄漏位置。

<!-- ![Trace 图](/../images/2019_07_05_01.jpg) -->

**Trace** 图：

<img src="/../images/2019_07_05_01.jpg" height="50%" width="50%">

<!-- ![ChangeHandlerFramLayout 对象](/../images/2019_07_05_02.png) -->
**ChangeHandlerFramLayout** 对象：

<img src="/../images/2019_07_05_02.png" height="70%" width="70%">


<!-- ![ChangeHandlerFramLayout 对象的 mAttachInfo 值](/../images/2019_07_05_03.png) -->
**ChangeHandlerFramLayout 对象的 mAttachInfo 值**：

<img src="/../images/2019_07_05_03.png" height="70%" width="70%">

接下来分析了位于 ChangeHandlerFramLayout 上的 MainActivity 实例对象，同理，在 leakmomory 查看该实例的状态，与ChangeHandlerFramLayout 对象不同的是此时我们关注属性值为 mDestoryed = true，说明 MainActivity 对象已经销毁，说明该对象也应该被 GC 回收。

如此往复，结合 trace 图和 leakmomory 进程信息，判断对象是否应该存在，**不过根据对象的不同，判断该对象是否应该存活标准不同**，如例子中的 mAttachInfo、mDestoryed。


### LeakCanary 实践一

使用官方 [LeakCanary 1.6.3 ](https://github.com/square/leakcanary/tree/v1.6.3) 库的 demo 进行展示如何定位内存泄漏。此处演示的为 LeakCanary 1.6.3 ，master 分支已于 2019-05-21 变更为 由 1.x 变更为 2.x ，具体查看 [Chnage Log](https://github.com/square/leakcanary/blob/4bbc0f6f2e3c9a25ca890ece6770f81cf9059510/docs/changelog.md)。


LeakCanary 2.x 功能更加全面，定位难度更加简单，但是方法基本一致，这里以 1.6.3 版本为主展示如何寻找内存泄漏位置。



<!-- ![Trace 图](/../images/2019_07_05_04.png) -->
**Demo 的 Trace 图为**：

<img src="/../images/2019_07_05_04.png" height="50%" width="50%">



<!-- ![MainActivity 相关信息](/../images/2019_07_05_05.png) -->
点击 MainActivity 所在行，显示详细信息。
**详细信息**：

<img src="/../images/2019_07_05_05.png" height="50%" width="50%">

可以发现 MainActivity 的 mDestoryed = true,说明 MainActivity 应该被 GC 回收，那么内存泄漏的应该发生在 **之上**。




<!-- ![MainActivity$2.this.0 相关信息](/../images/2019_07_05_06.png) -->

点击 `MainActivity$2.this$0` 显示具体信息:

<img src="/../images/2019_07_05_06.png" height="70%" width="70%">


从图中得知 `MainActivity$2.this$0` 为 **anonymous implent Runnable(继承 Runnble 的匿名对象)** 。从截图中值 `this$0` 为 `com.example.MainActivity` 实例对象,此时 `this$0` 所指向的 MainActivity 在旋转屏幕后会被销毁、被回收，但是 `Runnable 对象` 执行后台任务导致 `MainActivity$2` 对象依旧存在,即该对象此时应该存在于内存中，那么导致其所持有的 MainActivity 引用不能被回收，从而导致了 MainActivity 对象的泄漏。

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


针对匿名内部类具体分析一下过程：


```
public class Main {

    private void test() {

        Runnable work = new Runnable() {
            @Override
            public void run() {
                // Do some slow work in background
//                SystemClock.sleep(20000
            }
        };
        new Thread(work).start();
    }
}
```

通过 Java 命令编译该 Java 文件：

```
javac Main.java
```

编译出两个文件：
```
# Main.class
public class Main {
    public Main() {
    }

    private void test() {
        Runnable var1 = new Runnable() {
            public void run() {
            }
        };
        (new Thread(var1)).start();
    }
}

# Main$1.class
class Main$1 implements Runnable {
    Main$1(Main var1) {
        this.this$0 = var1;
    }

    public void run() {
    }
}
```
此处的 `this$0` 就为上面截图中的 `com.example.MainActivity` 实例对象，所以匿名内部类持有外部类的引用。

### LeakCanary 实践二

这个例子更能够体现一步一步寻找内存泄漏点：

首先此次内存泄漏产生的 Trace 图如下：

<!-- ![Trace 图](/../images/2019_07_06_01.png) -->

**步骤一： Trace 图**

<img src="/../images/2019_07_06_01.png" height="50%" width="50%">

首先我们通过截图 Title( MainActivity Leaked)明确 **产生泄漏的对象为 MainActivity 的实例对象**，Title 下面展示产生泄漏的 Trace 图。

我们根据 Trace 图一步一步分析:

**步骤二**：

<img src="/../images/2019_07_06_02.png" height="50%" width="50%">

**步骤三**：

<img src="/../images/2019_07_06_03.png" height="50%" width="50%">

**步骤四**：

<img src="/../images/2019_07_06_04.png" height="50%" width="50%">

**步骤五**：

<img src="/../images/2019_07_06_05.png" height="50%" width="50%">

**步骤六**：

<img src="/../images/2019_07_06_06.png" height="50%" width="50%">


<!-- ![步骤一](/../images/2019_07_06_02.png) -->

<!-- ![步骤二](/../images/2019_07_06_03.png) -->

<!-- ![步骤三](/../images/2019_07_06_04.png) -->

<!-- ![步骤四](/../images/2019_07_06_05.png) -->

<!-- ![步骤五](/../images/2019_07_06_06.png) -->

通过以上步骤，我们知道由于反转屏幕后 MainActivity 中的 Button 实例对象需要被回收，但是由于 HttpRequestHelper 对象在反转屏幕后继续存在，同时 HttpRequestHelper 实例对象持有 Button 对象的引用，所以 Button 不能成功被回收，导致 Button 持有的 MainActivity 实例对象在  Destory 后不能成功销毁，从而导致了 MainActivity 内存泄漏。

为了验证以上结论，我们可以继续向上查看 Trace：

<!-- ![验证一](/../images/2019_07_06_07.png)

![验证二](/../images/2019_07_06_08.png) -->


<img src="/../images/2019_07_06_07.png" height="50%" width="50%">

**实例对象详细信息**：

<img src="/../images/2019_07_06_08.png" height="50%" width="50%">

---
**有用的资源：**

[leakcanary 官方网站](https://square.github.io/leakcanary/)

[leakcanary 基本原理](https://square.github.io/leakcanary/fundamentals/)

[Detect all memory](https://medium.com/square-corner-blog/leakcanary-detect-all-memory-leaks-875ff8360745)

[Youtube 视频：[Uber Mobility] Memory Leak Hunt: LeakCanary - Pierre-Yves Ricau](https://www.youtube.com/watch?v=KwArTJHLq5g)

[LeakCanary 2: Leaner, Better, Faster, Kotliner! by Pierre-Yves Ricau, Square, Inc EN](https://www.youtube.com/watch?v=LEX8dn4BLUw)



