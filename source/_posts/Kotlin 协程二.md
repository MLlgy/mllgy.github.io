---
title: Kotlin 协程二
date: 2019-10-07 17:12:00
tags: [Kotlin 官方文档,Coroutines(协程)]
---


### 协程上下文和调度器

协程总是要运行在 CoroutineContext 类型为代表的协程上下文中，协程上下文是各种不同元素的集合，其中主元素为 Job，同事我们也会使用他的调度器。


#### 调度器与线程

协程上下文包括了一个协程调度器，它确定了协程执行时使用的一个或多个线程。协程调度器可以指定协程运行在指定线程，也可以调度它运行在线程池中或不受限的运行。


所有的协程构建器(比如：launch、async) 会接收一个可选的 CoroutineContext 参数，它可以被显示的为一个新的协程或其他上下文元素指定一个调度器。

<!-- more -->
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

由上面可知协程可以在一个线程中挂挂起在另外一个线程中恢复。在单一线程中也难弄清楚协程在何时何地在做什么事，所以这时我们需要打印正在执行代码的线程名。

```
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() = runBlocking<Unit> {
    val a = async {
        log("I'm computing a piece of the answer")
        6
    }
    val b = async {
        log("I'm computing another piece of the answer")
        7
    }
    log("The answer is ${a.await() * b.await()}")    
}
//打印日志：
[main @coroutine#2] I'm computing a piece of the answer
[main @coroutine#3] I'm computing another piece of the answer
[main @coroutine#1] The answer is 42
```

可以很清楚的知道协程执行的线程以及协程上下文。


#### 在不同线程间切换

```
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() {
    newSingleThreadContext("Ctx1").use { ctx1 ->
            newSingleThreadContext("Ctx2").use { ctx2 ->
                    runBlocking(ctx1) {
                        log("Started in ctx1")
                        withContext(ctx2) {
                            log("Working in ctx2")
                        }
                        log("Back to ctx1")
                    }
            }
    }    
}
打印日志：
[Ctx1 @coroutine#1] Started in ctx1
[Ctx2 @coroutine#1] Working in ctx2
[Ctx1 @coroutine#1] Back to ctx1
```

从打印日志中可知 可以为 runBlocking() 显式的设置线程上下文，同时可以使用 withContext 函数来改变协程的线程上下文，从每条日志的`@coroutine#1`知它们仍运行在相同的协程中，以上函数只是改变了协程运行的线程，但是并没有改变协程的执行。


#### 上下文中的 Job

Job 是它上下文中的一部分，协程可以在它所属的上下文中使用 coroutineContext[Job] 表达式来获取 Job 实例对象：

```
fun main() = runBlocking<Unit> {
    val job: Job? = coroutineContext[Job]
    println("My job is ${coroutineContext[Job]}")
    println("My job is $job")
    println("Job is active?: $isActive")
}
// 打印日志：
My job is BlockingCoroutine{Active}@5fa7e7ff
My job is BlockingCoroutine{Active}@5fa7e7ff
Job is active?: true
```
进一步验证了 `coroutineContext[Job]` 取回的为所属上下文的 Job 对象。

CoroutineScope 中的 isActive 只是 coroutineContext[Job]?.isActive == true 的一种方便的快捷方式。


#### 子协程


当一个协程被其它协程在 CoroutineScope 中启动的时候， 它将通过 CoroutineScope.coroutineContext 来承袭上下文，并且这个新协程的 Job 将会成为父协程的子 Job。**当一个父协程被取消的时候，所有它的子协程也会被递归的取消**。

```
fun main() = runBlocking<Unit> {
    println("***[${Thread.currentThread().name}] runBlocking above")
    // 启动一个协程来处理某种传入请求（request）
    val request = launch {
        // 孵化了两个子作业, 其中一个通过 GlobalScope 启动
        println("***[${Thread.currentThread().name}] parent scope(父协程)")
        GlobalScope.launch {
            println("job1: I run in GlobalScope and execute independently!")
            delay(1000)
            println("***[${Thread.currentThread().name}] GlobalScope(全局协程)")
            println("job1: I am not affected by cancellation of the request")
        }
        // 另一个则承袭了父协程的上下文
        launch {
            println("***[${Thread.currentThread().name}] child scope(子协程)")
            delay(100)
            println("job2: I am a child of the request coroutine")
            delay(1000)
            println("job2: I will not execute this line if my parent request is cancelled")// 由于父协程 cancel，所以子协程中断执行，该语句无法打印
        }
    }
    delay(500)
    request.cancel() // 取消请求（request）的执行
    delay(1000) // 延迟一秒钟来看看发生了什么
    println("main: Who has survived request cancellation?")
    println("***[${Thread.currentThread().name}] runBlocking Blew")
}
// 打印日志：
***[main @coroutine#1] runBlocking above(runBlocking 外层协程)
***[main @coroutine#2] parent scope(父协程)
job1: I run in GlobalScope and execute independently!
***[main @coroutine#4] child scope(子协程)
job2: I am a child of the request coroutine
***[DefaultDispatcher-worker-2 @coroutine#3] GlobalScope(全局协程)
job1: I am not affected by cancellation of the request
main: Who has survived request cancellation?
***[main @coroutine#1] runBlocking Blew(runBlocking 外层协程)
```


#### 父协程的职责

父协程总是等待它所有的子协程全部执行完毕，父协程不必显式的跟踪所有子协程的启用，也不必使用 Job.join 在最后的时候等待它们。

```
fun main() = runBlocking<Unit> {
    // 启动一个协程来处理某种传入请求（request）
    val request = launch {
        repeat(3) { i -> // 启动少量的子作业
                launch  {
                    delay((i + 1) * 200L) // 延迟 200 毫秒、400 毫秒、600 毫秒的时间
                    println("Coroutine $i is done")
                }
        }
        println("request: I'm done and I don't explicitly join my children that are still active")
    }
    request.join() // 等待请求的完成，包括其所有子协程
    println("Now processing of the request is complete")
}
//打印日志：
request: I'm done and I don't explicitly join my children that are still active
Coroutine 0 is done
Coroutine 1 is done
Coroutine 2 is done
Now processing of the request is complete
```
如果注释掉 request.join()，执行结果如下：
```
fun main() = runBlocking<Unit> {
    // 启动一个协程来处理某种传入请求（request）
    val request = launch {
        repeat(3) { i -> // 启动少量的子作业
                launch  {
                    delay((i + 1) * 200L) // 延迟 200 毫秒、400 毫秒、600 毫秒的时间
                    println("Coroutine $i is done")
                }
        }
        println("request: I'm done and I don't explicitly join my children that are still active")
    }
    //request.join() // 等待请求的完成，包括其所有子协程
    println("Now processing of the request is complete")
}
//打印日志：
Now processing of the request is complete
request: I'm done and I don't explicitly join my children that are still active
Coroutine 0 is done
Coroutine 1 is done
Coroutine 2 is done
```

所以父协程不调用 join() 函数，也是会等待所有的子协程执行完毕。


#### 为协程命名

```
fun main() = runBlocking(CoroutineName("runBlockingName")) {
    log("Started main coroutine")
    // 运行两个后台值计算
    val v1 = async(CoroutineName("v1coroutine")) {
        delay(500)
        log("Computing v1")
        252
    }
    val v2 = async(CoroutineName("v2coroutine")) {
        delay(1000)
        log("Computing v2")
        6
    }
    log("The answer for v1 / v2 = ${v1.await() / v2.await()}")
}
// 打印日志：
[main @runBlockingName#1] Started main coroutine
[main @v1coroutine#2] Computing v1
[main @v2coroutine#3] Computing v2
[main @runBlockingName#1] The answer for v1 / v2 = 42
```

#### 协程作用域

在日常开发过程中，在 Activity 中我们需要开启多个协程来获取网络数据、后台绘制、执行动画等，这协程动作必须在 Activity 销毁时取消，否则就会引起内存泄漏。

可以手动绑定协程和Activity 的生命周期：

```
class Activity {
    private val mainScope = MainScope()

    fun destroy() {
        mainScope.cancel()
    }
    // 继续运行……
```

也可以在这个 Activity 类中实现 CoroutineScope 接口，最好的方法是使用具有默认工厂函数的委托。 我们也可以将所需的调度器与作用域合并（我们在这个示例中使用 Dispatchers.Default）


**调度器**


```
class Activity : CoroutineScope by CoroutineScope(Dispatchers.Default) {
    // 继续运行……
```

完整执行程序：
```
class Activity : CoroutineScope by CoroutineScope(Dispatchers.Default) {

    fun destroy() {
        cancel() // Extension on CoroutineScope
    }
    // 继续运行……

    // class Activity continues
    fun doSomething() {
        // 在示例中启动了 10 个协程，且每个都工作了不同的时长
        repeat(10) { i ->
            launch {
                delay((i + 1) * 200L) // 延迟 200 毫秒、400 毫秒、600 毫秒等等不同的时间
                println("Coroutine $i is done")
            }
        }
    }
} // Activity 类结束

fun main() = runBlocking<Unit> {
    val activity = Activity()
    activity.doSomething() // 运行测试函数
    println("Launched coroutines")
    delay(500L) // 延迟半秒钟
    println("Destroying activity!")
    activity.destroy() // 取消所有的协程
    delay(1000) // 为了在视觉上确认它们没有工作    
}
// 打印日志：
Launched coroutines
Coroutine 0 is done
Coroutine 1 is done
Destroying activity!
```

#### 线程局部数据