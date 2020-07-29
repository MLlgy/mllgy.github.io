---
title: Kotlin 协程官方文档学习(一)
date: 2019-10-07 17:11:30
tags: [Kotlin,Coroutines协程]
---


### 1. 协程的基本介绍

协程，本质上是轻量级的线程。

它们在某些 `CoroutineScope 上下文中` 与 `launch 协程构建器` 一起启动。

在 GlobalScope 中启动了一个新的协程，这意味着新协程的生命周期只受整个应用程序的生命周期限制。



delay 是挂起函数不会造成线程阻塞，但是会挂起协程，并且挂起函数只能在协程中使用。



### 2. 阻塞与非阻塞

阻塞与非阻塞都是针对于是否阻塞主线程来说的。

<!-- more -->

```
// 示例 1
fun main() {
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(1000L)
        println("World!")
    }
    println("Hello") // 主线程中的代码会立即执行
    runBlocking {     // 但是这个表达式阻塞了主线程
        delay(2000L)  // 我们延迟 2 秒来保证 JVM 的存活
    } 
}
```

例如上程序中 GlobalScope.launch{} 中的 delay(1000L) 只会阻塞协程，但是 **不会阻塞主线程的执行**，所以以上代码会首先打印: Hello ，然后打印: World!。

runBlocking{} 代码块为阻塞式的。

### 3. 定义 Job

示例 1 中通过阻塞主线程一段时间:
`(runBlocking{delay(2000L)})`，
从而等待协程的完成，这不是一个好的方式，可以通过 Job 来改善上述方法。

```
fun main() = runBlocking {
    val job = GlobalScope.launch { // launch a new coroutine and keep a reference to its Job
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join() // wait until child coroutine completes    
}
```

调用 `Job#join()` 主线程会一直阻塞，直到指定协程执行完毕。

### 4. 结构化并发


使用 GlobalScope.launch 会创建顶部协程，它会消耗一定的资源，如此的话启动多个协程会导致内存不足，此时使用 **结构化并发** 可以解决这个问题。

定义一个外部协程，在其内部也可以定义新的协程，包括外部协程在内的每个协程构建器都将 CoroutineScope 的实例添加到其代码块所在的作用域中。 我们可以在这个作用域中启动协程而无需显式调用 join ，因为 **外部协程（示例中的 runBlocking）直到在其作用域中启动的所有协程都执行完毕后才会结束**。

```
fun main() = runBlocking { //this:CoroutineScope

    // 在 runBlocking 作用域中启动一个新协程
    launch {// this: CoroutineScope
        delay(1000L)
        println("World!")

        launch {// this: CoroutineScope
            delay(5000L)
            println("hahahah")
        }
    }
    println("Hello,")
}
```



### 5. 构建作用域

除了使用构建器(如: launch、async 等)提供协程作用域之外，还可以使用 coroutineScope 构建器声明自己的作用域。

使用  coroutineScope 可以构建协程作用域，**构建的协程作用域在所有子协程执行完毕之前不会结束**。


```
fun main() = runBlocking { // this: CoroutineScope
    launch { 
        delay(200L)
        println("Task from runBlocking")
    }
    
    coroutineScope { // 创建一个协程作用域
        launch {
            delay(500L) 
            println("Task from nested launch")
        }
    
        delay(100L)
        println("Task from coroutine scope") // 这一行会在内嵌 launch 之前输出
    }
    
    println("Coroutine scope is over") // 这一行在内嵌 launch 执行完毕后才输出
}
```
以下为打印日志:
```
Task from coroutine scope
Task from runBlocking
Task from nested launch
Coroutine scope is over
```

注意打印顺序，从打印顺序中可以看出协程的 **非阻塞**，因为 `Task from coroutine scope ` 最先打印出来。



### 协程的取消与超时

协程是可以被取消的。

```
fun main() = runBlocking {
    val job = launch {
        repeat(1000) { i ->
                println("job: I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // 延迟一段时间
    println("main: I'm tired of waiting!")
    job.cancel() // 取消该作业
    job.join() // 等待作业执行结束
    println("main: Now I can quit.")
}
```

打印日志：

```
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

当协程中在执行计算任务是协程是不能被取消的。


超时 ：

withTimeout(1300L) {
    repeat(1000) { i ->
            println("I'm sleeping $i ...")
        delay(500L)
    }
}

在执行超过 1300ms 会报出错误。


### 6. 挂起函数

使用 `suspend` 修饰的函数，排列的挂起函数 **默认顺序执行**。

```
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // 假设我们在这里做了一些有用的事
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // 假设我们在这里也做了一些有用的事
    return 29
}

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")    
}

//执行结果：
The answer is 42
Completed in 2012 ms
```
通过打印时间可以得知默认挂起函数为顺序执行的。
### 7. lauch 、 async

async 与 lauch 一样，开启了一个单独的协程，与其他协程一起进行并行工作。
不同的 launch 返回一个 Job 不附带任何结果值，而 async 返回 Deffered ，它是一个轻量级的非阻塞 future， 这代表了一个将会在稍后提供结果的 promise。你可以使用 **.await() 在一个延期的值上得到它的最终结果**， 但是 Deferred 也是一个 Job，所以如果需要的话，你可以取消它 。

同样是上面的例子，我们使用 async 并发修饰函数。

```
val time = measureTimeMillis {
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")// 使用 await() 函数获得他的最终结果
    }
println("Completed in $time ms") 

//执行结果：
The answer is 42
Completed in 1029 ms
```
从执行时间可知使用 async 修饰函数为 **并行执行** 的。


### 8. 惰性 async

如果懒加载一样，惰性 async 只有在使用时才会执行，执行 start() 方法执行该方法。

```
fun main() = runBlocking<Unit> {
val time = measureTimeMillis {
    val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
    val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }
    // 执行一些计算
    one.start() // 启动第一个
    two.start() // 启动第二个
    println("The answer is ${one.await() + two.await()}")
}
println("Completed in $time ms")
}
//执行结果：
The answer is 42
Completed in 1017 ms
```
以上 one、two 只是定义的两个协程，由于 `start = CoroutineStart.LAZY` 的存在没有真正的执行，只有在执行 start 方法后两个协程才会真正的执行


### 9. async 风格函数

在 Kotlin 中不推荐使用此类型的函数,故不详述.

### 10. 结构化 async 函数

结构化 async 函数 **就是使多个 async 函数执行时如果一个函数发生异常,则其他未执行的 async 函数也不会得到执行**,此功能使用  coroutineScope 来实现.

```
fun main() = runBlocking<Unit> {
    try {
        failedConcurrentSum()
    } catch (e: ArithmeticException) {
        println("Computation failed with ArithmeticException")
    }
}

suspend fun failedConcurrentSum(): Int = coroutineScope {
    val one = async<Int> {
        try {
            delay(Long.MAX_VALUE) // 模拟一个长时间的运算
            42
        } finally {
            println("First child was cancelled")
        }
    }

    val two = async<Int> {
        delay(2000L)
        println("Second child throws an exception")
        throw ArithmeticException()
    }

    val three = async {
        println("Test third run or not")
        12
    }
    one.await() + two.await() + three.await()
}

// 打印日志"
Test third run or not
Second child throws an exception
First child was cancelled
Computation failed with ArithmeticException
```

如果在上面函数执行过程中 two 发生异常,那么此时正在等待执行的 one 将中断执行,但是之前的 three 则可以正常的执行。

----

**知识链接**


[Improve app performance with Kotlin coroutines](https://developer.android.com/kotlin/coroutines)


[Coroutines on Android (part I): Getting the background](https://medium.com/androiddevelopers/coroutines-on-android-part-i-getting-the-background-3e0e54d20bb)


[Kotlin-24.协程和线程(Coroutine & Thread)](https://blog.csdn.net/qq_32115439/article/details/74018755)

[使用协程进行 UI 编程指南](https://github.com/hltj/kotlinx.coroutines-cn/blob/master/ui/coroutines-guide-ui.md)
