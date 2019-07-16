---
title: Kotlin 协程 二
tags:
---


### 协程上下文和调度器

协程总是要运行在 CoroutineContext 类型为代表的协程上下文中，协程上下文是各种不同元素的集合，其中主元素为 Job，同事我们也会使用他的调度器。


#### 调度器与线程

协程上下文包括了一个协程调度器，它确定了协程执行时使用的一个或多个线程。协程调度器可以指定协程运行在指定线程，也可以调度它运行在线程池中或不受限的运行。


所有的协程构建器(比如：launch、async) 会接收一个可选的 CoroutineContext 参数，它可以被显示的为一个新的协程或其他上下文元素指定一个调度器。

```
fun main() = runBlocking<Unit> {
    launch { // 运行在父协程的上下文中，即 runBlocking 主协程
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) { // 不受限的——将工作在主线程中
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) { // 将会获取默认调度器
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(newSingleThreadContext("MyOwnThread")) { // 将使它获得一个新的线程
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }
}
// 打印日志：
Unconfined            : I'm working in thread main
Default               : I'm working in thread DefaultDispatcher-worker-1
newSingleThreadContext: I'm working in thread MyOwnThread
main runBlocking      : I'm working in thread main
```

#### 非受限调度器 vs 受限调度器

在上面的例子中有 Dispatchers.Unconfined ，此为非受限调度器。

```
fun main() = runBlocking<Unit> {
    launch(Dispatchers.Unconfined) { // 非受限的——将和主线程一起工作
        println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
    }
    launch { // 父协程的上下文，主 runBlocking 协程
        println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
    }    
}
//打印日志：
Unconfined      : I'm working in thread main
main runBlocking: I'm working in thread main
Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor
```
Dispatchers.Unconfined协程调度器会在程序运行到第一个挂起点时，在调用者线程中启动,所以日志打印为：
`Unconfined      : I'm working in thread main`,此时在主线程中挂起。

挂起后，它将在挂起函数执行的线程中恢复，恢复的线程完全取决于该挂起函数在哪个线程执行，挂起函数 delay() 运行的线程为 
`Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor`

。非受限调度器适合协程不消耗 CPU 时间也不更新任何限于特定线程的共享数据（如 UI）的情境。


#### 调试协程与线程 

由上面可知协程可以在一个线程中挂挂起在另外一个线程中恢复。如果在一个线程中也可以