---
title: Kotlin 中的结构化并发是什么?
date: 2019-10-11 18:41:20
tags: [Coroutines(协程),Kotlin]
---




> 在接触到 Kotlin 的协程后，在官方文档知道，通过结构化的并发可以避免很多问题，具体参见官方文档 [结构化的并发](https://www.kotlincn.net/docs/reference/coroutines/basics.html#%E7%BB%93%E6%9E%84%E5%8C%96%E7%9A%84%E5%B9%B6%E5%8F%91)，但是什么是结构化的并发呢？通过搜索引擎没有查询到有用的信息。此处只是记录自己通过网上博客的学习，为自己现阶段的记录和理解，这个话题也会持续更新。


Kotlin 协程中如何实现结构化并发呢？ 先看一下官方文档中是基于什么问题使用结构化的并发？首先看一下这段代码：

```
fun main() = runBlocking {
    val job = GlobalScope.launch { // 启动一个新协程并保持对这个作业的引用
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join() // 等待直到子协程执行结束
}
```

线程的启动总是全局的，一般是在程序的上下文启动的，对应协程，就如上面例子一样通过 GlobalScope 创建一个全局协程。不过官方文档上针对 GlobalScope 所阐述的确定，抱歉没看懂。。。
<!-- more -->
但是协程是可以 **在指定作用域内启动协程的**。协程构建器都将 CoroutineScope 的实例添加到其代码块所在的作用域中，外部协程（示例中的 runBlocking）直到在其作用域中启动的所有协程都执行完毕后才会结束，此时就不用再像上面一样通过调用 join 来等待协程的执行。

```

fun main() = runBlocking { // this: CoroutineScope
    launch { // 在 runBlocking 作用域中启动一个新协程
        delay(1000L)
        println("World!")
    }
    println("Hello,")
}
```

不过这个例子似乎不太形象，在博客上存在这样一个例子:


```
suspend fun loadAndCombine(name1: String, name2: String): Image { 
    val deferred1 = async { loadImage(name1) }
    val deferred2 = async { loadImage(name2) }
    return combineImages(deferred1.await(), deferred2.await())
}
```

这个例子是 [KotlinConf 2017 - Introduction to Coroutines by Roman Elizarov](https://www.youtube.com/watch?v=_hfBv0a09Jc)，但是这个例子目前自己没能实现(参看了原博文说是此例子有很多处错误，不知是否该例是伪代码))，但是有这个例子又能够延伸什么是结构化的并发，先放在这吧(我尽力了)。

首先在外部协程中调用函数，如果此操作被取消了，那么这两个 loadImage 动作是不会被取消的。

进一步改善的结果：

```
suspend fun loadAndCombine(name1: String, name2: String): Image { 
    val deferred1 = async(conroutineContext) { loadImage(name1) }
    val deferred2 = async(conroutineContext) { loadImage(name2) }
    return combineImages(deferred1.await(), deferred2.await())
}
```
这样在父协程被取消后，子协程也会被取消。但是这样也存在一定的问题，如果 deferred1.await() 抛出异常，但是 deferred2 不会受到影响，这样是不符合逻辑的。

这时就可以使用结构化的并发了，将 deferred1、deferred2 置于同一个作用域上，作用域内的协程称为外部协程的子协程，那么当其中外部协程被取消或者子协程异常时，作用域中的所有子协程都会被取消。

```
suspend fun loadAndCombine(name1: String, name2: String): Image =
    coroutineScope { 
        val deferred1 = async { loadImage(name1) }
        val deferred2 = async { loadImage(name2) }
        combineImages(deferred1.await(), deferred2.await())
    }
```
---

**知识来源：**

[结构化的并发](https://www.kotlincn.net/docs/reference/coroutines/basics.html#%E7%BB%93%E6%9E%84%E5%8C%96%E7%9A%84%E5%B9%B6%E5%8F%91)

[[译] Kotlin 、协程、结构化并发](https://blog.csdn.net/weixin_33755847/article/details/91366063)

[Structured Concurrency Kickoff](https://trio.discourse.group/t/structured-concurrency-kickoff/55)