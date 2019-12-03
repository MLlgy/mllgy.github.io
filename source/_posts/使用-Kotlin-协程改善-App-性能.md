---
title: 使用 Kotlin 协程改善 App 性能
tags: [Kotlin 协程]
date: 2019-10-29 11:50:16
---


### 0x0001 概述

协程是一种并发设计模式，您可以在 Android上 使用它来简化异步执行的代码。协程在 1.3 版中添加到 Kotlin，基于其他语言的既定概念进行设计 Kotlin 中的协程。

在Android上，协程可帮助解决两个主要问题：
* 管理长时间运行的任务，这些任务可能会阻塞主线程并导致您的应用程序冻结。
* 提供主安全性，或从主线程安全地调用网络或磁盘操作。
  
<!-- more -->
本主题介绍如何使用Kotlin协程来解决这些问题，从而使您能够编写更简洁，更简洁的应用程序代码。

###  0x0002 管理行长时间运行的任务

在 Android上，每个应用程序都有一个主线程来处理用户界面并管理用户交互。如果在 应用程序给主线程分配了过多的工作，则该应用程序 UI 界面会出现丢帧、卡顿的现象。同时像网络请求、JSON解析、读写数据库、甚至只是遍历大列表都可能导致您的应用运行缓慢，以至于 UI 界面会出现丢帧、卡顿的现象，所以长时间运行的操作应在主线程之外运行。


下面示例展示了通过协程运行长时间任务：

```
suspend fun fetchDocs() {                             // Dispatchers.Main
    val result = get("https://developer.android.com") // Dispatchers.IO for `get`
    show(result)                                      // Dispatchers.Main
}

suspend fun get(url: String) = withContext(Dispatchers.IO) { /* ... */ }
```

除了 call（或 invoke）和 return 外，协程通过在常规的函数上添加两个操作来完成构建，依次可以处​​理长时间运行的任务，这两个操作分别是：
* suspend(挂起): 会暂停当前协程的执行，并且保存所有的本地局部变量。
* resume(恢复): 会从挂起的位置继续执行挂起的协程。

需要注意的是，只能可以在挂起函数中或者协程构建器中才可以调用挂起函数。


在上面的示例中，**get() 仍在主线程上运行**，但是它 **在启动网络请求之前挂起了协程**。网络请求完成后，get 会恢复挂起的协程，而不是使用回调将结果通知给主线程中的调用者。


Kotlin 使用 **栈帧** 来管理函数运行时所需要的局部变量。**当挂起一个协程，当前函数的栈帧会被拷贝下来，并且保存。当协程恢复时，栈帧会从其保存位置拷贝回来，方法重新运行**。所以使用协程，即使代码看起来像是普通的顺序阻塞请求，协程也可以确保网络请求避免阻塞主线程。


###  0x0003 Use coroutines for main-safety、 使用 Kotlin 协程来保证主线程调用安全

Kotlin 协程使用 Dispatchers API 去指定哪一个线程用来执行协程代码。为了在主线程以外的线程中运行代码，应该指定 Kotlin 协程执行的 **调度器**(Dispatchers.Default 或 Dispatchers.IO)。**在 Kotlin 中所有的协程必须运行在调度器中**，即使协程运行在主线程中也需要指定调度器。协程可以挂起自己，调度器用来恢复挂起的协程。

为了指定协程运行的位置，Kotlin 提供了如下调度器：

* Dispatchers.Main
    
    使用此调度器，用来指定协程运行在主线程中。此调度器应该仅在 **与 UI 交互或者执行短时间任务** 时使用，否则会造成主线程阻塞，比如：调用挂起函数，运行 Android UI 中的操作、更新 LiveData 对象。

* Dispatchers.IO

    此调度器主要可以用来优化 **在主线程中运行磁盘或者网络 IO 的情况**，比如：使用 Room 组件、读写文件、网络操作。

* Dispatchers.Default 

    此调度器主要用来优化 **在主线程之外执行 CPU 密集型工作** 的情况，比如：对列表排序、解析 Gson。

继续上面的示例，现在可以使用 Dispatchers 去重新定义 get 函数。在 get 的方法内，调用 `withContext(Dispatchers.IO)` 创建运行在 IO 线程池中的闭包，该闭包中的所有代码都会在调度器 Dispatchers.IO 指定的线程中运行。由于 `withContext` 本身也是一个挂起函数，所以 get 也要定义为挂起函数。

```
suspend fun fetchDocs() {                      // Dispatchers.Main
    val result = get("developer.android.com")  // Dispatchers.Main
    show(result)                               // Dispatchers.Main
}

suspend fun get(url: String) =                 // Dispatchers.Main
    withContext(Dispatchers.IO) {              // Dispatchers.IO (main-safety block)
        /* perform network IO here */          // Dispatchers.IO (main-safety block)
    }                                          // Dispatchers.Main
}
```

由于 withContext() 让我们无需引入回调，就可以控制任何代码行所运行的线程池，并且可以将其应用到很小的功能上，例如从数据库读取或执行网络请求，可见使用 Kotlin 协程，可以通过**细粒度的控制来调度线程**。

Kotlin 协程中一个好的实践是：使用 withContext（）来确保每个函数都是主安全(主线程安全的)的，这意味着 **可以从主线程调用该函数**。


正是 withContext 的作用，在上面的示例中，fetchDocs 运行在主线程，但是该方法可以安全的调用在后台执行网络请求的 get 方法。由于 Kotlin 协程支持 suspend 和 resume，当 whitContext 闭包执行完毕后，在主线程中相应的协程会携带 get 方法的结果恢复运行。


> 重要：在 Kotlin 中仅仅使用 suspend 是不能指定方法在后台线程中运行的。 在 Kotlin 挂起函数运行在主线中是十分场景的。在需要安全调用时，应该使用 withContext 而不是 suspend 函数，就像读写磁盘、执行网络操作、运行 CPU 密集型任务。


###  0x0004 withContext 的性能表现

与实现相同效果的回调的实现相比，withContext（）不会增加额外的开销。此外，在某些情况下，除了回调(效果与 withContext 相同)的实现之外，还可以使用 withContext() 来优化调用。例如，如果一个函数对网络进行了十次调用，则可以通过使用外部 withContext() 告诉 Kotlin 仅切换一次线程。即使 网络库多次使用 withContext 指定不同的调度器，但是在程序中网络请求仍保留在同一调度程序上，避免切换线程。另外，Kotlin优化了Dispatchers.Default和Dispatchers.IO之间的切换，以尽可能避免线程切换。

> 重要：使用使用诸如 Dispatchers.IO 或 Dispatchers.Default 之类的线程池的调度器，并不能保证闭包中代码从上到下在同一线程上执行。在一些情况下，在 suspend-resume 动作后，Kotlin 协程可能会从一个线程移到另一个线程中执行，这意味着对于整个 withContext（）块，线程局部变量可能会执行不同的值。


###  0x0005 Designate a CoroutineScope(指定协程范围)

当定义个协程时，同时也会定义它的 CoroutineScope 对象(协程范围)。一个 CoroutineScope 对象管理一个或多个相关的协程，在其范围内，可以使用 CoroutineScope 对象开启一个新的协程。与调度器(Dispatcher)不同，CoroutineScope 不会运行协程。

CoroutineScope 需要保证在用户离开应用程序中的内容区域时停止协程执行，通过使用 CoroutineScope，可以确保任何正在运行的操作都能够正确的停止。

####  0x0006 将 CoroutineScope 与 Android Architecture 组件一起使用

在 Android 中，可以将 CoroutineScope 的实现与组件生命周期相关联，这样的话，可以避免内存泄漏或为与用户不再相关的 Activity 或 Fragment 进行额外的工作。使用 Jetpack 组件，它们自然地适合 ViewModel，因为 ViewModel 对象在 Activity 配置(比如屏幕旋转)更改时不会被销毁，不用担心 Kotlin 协程会被取消或重启。

Scope 对象知道它们开启的每一个协程，这就意味着，在任何时候，你都可以在取消该作用域内开启的所有动作。作用域可以自行传播，因此，如果协程开始另一个协程，则两个协程都具有相同的作用域。这意味着，即使其他库从你的作用域(Scope)中启动协程，你也可以随时取消它们。当你在 ViewModel 中使用协程时，这一点是十分重要的，如果 ViewModel 对象被销毁了，那么所有正在执行的异步任务会被取消，否则的话会浪费资源，严重的话会导致内存泄漏。如果有异步工作需要在销毁 ViewModel 对象之后继续进行，则应在应用程序体系结构的较低层进行。

> 通过抛出 CancellationException 协同地取消协程。在协程取消期间触发捕获Exception或Throwable的异常处理程序。

借助适用于 Android 的 KTX 库，您还可以使用扩展属性 viewModelScope 创建协程，这些协程可以一直运行到 ViewModel 被销毁为止。


###  0x0007 Start a coroutine(开启一个协程)


开启协程的两种方法：
* launch
  
    启动一个新的协程，并且不将结果返回给调用方。可以使用发布启动任何被认为是“即发即弃”的工作。
* async

    启动一个新的协程，并使用 await 方法并返回挂起函数的结果。

通常，您应该从常规函数启动新的协程，因为常规函数无法调用 await。仅当在另一个协程内部 或在 suspend 函数内部并执行 `并行分解` 任务时才使用 async 开启协程。

在前面的示例的基础上，下面是一个带有 viewModelScope KTX 扩展属性的协程，该属性使用 launch 从常规函数切换到协程：

```
fun onDocsNeeded() {
    viewModelScope.launch {    // Dispatchers.Main
        fetchDocs()            // Dispatchers.Main (suspend function call)
    }
}
```

> launch 和 async 协程构造器处理异常的方式有所不同。由于 async 期望在某个时刻最终调用 await ，因此它会保留异常并将其作为 await 调用的一部分重新抛出。这意味着，如果使用 await 从常规函数中启动新的协程，则可能会静默删除异常。这些丢弃的异常不会出现在崩溃指标中，也不会在logcat中记录。


###  0x0008 Parallel decomposition(并发分解)


由 suspend 函数启动的所有协程必须在该函数返回时停止，因此您可能需要确保这些协程在返回之前完成。使用 Kotlin中 的结构化并发，您可以定义一个 coroutineScope 来启动一个或多个协程。然后，使用 await()（用于单个协程）或 awaitAll()（用于多个协程），可以保证这些协程在从函数返回之前完成。

作为示例，定义一个 coroutineScope，它异步获取两个文档。通过延迟引用上调用 await（），我们保证两个异步操作在返回值之前都已完成：

```
suspend fun fetchTwoDocs() =
    coroutineScope {
        val deferredOne = async { fetchDoc(1) }
        val deferredTwo = async { fetchDoc(2) }
        deferredOne.await()
        deferredTwo.await()
    }
```

也可以使用 awaitAll ：

```
suspend fun fetchTwoDocs() =        // called on any Dispatcher (any thread, possibly Main)
    coroutineScope {
        val deferreds = listOf(     // fetch two docs at the same time
            async { fetchDoc(1) },  // async returns a result for the first doc
            async { fetchDoc(2) }   // async returns a result for the second doc
        )
        deferreds.awaitAll()        // use awaitAll to wait for both network requests
    }
```

即使 fetchTwoDocs() 使用 async 启动新的协程，该函数仍使用 awaitAll() 在启动的协程完成后再返回。但是请注意，即使我们没有调用 awaitAll()，coroutineScope 生成器也不会在所有新的协程完成之后才恢复调用 fetchTwoDocs 的协程。----存疑？？？？

此外，coroutineScope会捕获协程抛出的所有异常，并将其路由回调用方。


###  0x0009 Architecture components with built-in support(具有内置支持的架构组件)

一些架构组件，包括 ViewModel 和 Lifecycle，包含了对协程的内置支持，通过其自己的 CoroutineScope 成员实现的。

比如，ViewModle 内部包含了 viewModelScope，这提供了一种 **在 ViewModel 范围内启动协程的方法**，如以下示例所示：

```
class MyViewModel : ViewModel() {

    fun launchDataLoad() {
        viewModelScope.launch {
            sortList()
            // Modify UI
        }
    }

    /**
    * Heavy operation that cannot be done in the Main Thread
    * 主线程中不能运行繁琐的运算
    */
    suspend fun sortList() = withContext(Dispatchers.Default) {
        // Heavy work
    }
}
```

而 LiveData 通过 liveData 闭包 支持协程：


```
liveData {
    // runs in its own LiveData-specific scope
}
```

----
**知识链接：**

[Google 官方文档:Improve app performance with Kotlin coroutines](https://developer.android.com/kotlin/coroutines)