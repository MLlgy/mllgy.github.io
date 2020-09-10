---
title: Kotlin 协程视频
tags:
---

使用 C 编写代码，看起来是阻塞,但是不会，

协程使用 suspend 修改 fun

suspend 会告诉 Kotin 编译器该函数需要在 coroutines 中执行


suspicious



scope 可以携带 Job，Job 可以定义 scope 和 协程的生命周期，如果向 scope 传递 job，那么这意味着它会以一种特定的方式处理异常