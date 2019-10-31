---
title: ' Kotlin 核心编程(八):Kotlin 协程(第几篇待定)'
tags:
---



---


**one**

```blic fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit// 真正的协程
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```
launch 并是声明一个协程(或者说不是立刻去启动一个协程)，而是会运行一个 lambda 闭包，真正的协程参数时 Lambda 闭包。


**two**

协程是将负载的异步操作放入底层中，程序逻辑可以在协程中按照顺序来表达，一次简化异步编程，该底层库可以将用户代码包装为回调/订阅事件，在不同线程或者不同机器调度执行。



协程是由程序直接实现的一种轻量级线程。

协程主要还是运行在线程中，因此协程不能替代线程。

协程中执行挂起函数，只是挂起函数本身，当协程在等待时，线程将返回到线程池中，当协程等待完成时，使用线程池中的空闲线程来恢复程序继续执行。



使用同步的代码格式去发起一次异步请求。
