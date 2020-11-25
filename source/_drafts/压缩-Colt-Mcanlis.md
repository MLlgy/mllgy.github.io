---
title: 压缩-Colt Mcanlis
tags:
---


HandlerThread、AsyncTask、Executor、Service、IntentService


## AsyncTask

小任务、短任务时，如果觉的方便就可以使用。

为什么是小而短的呢？因为通过 AsyncTask 执行长任务容易引发内存泄漏。

不过现在有了各种方案后，AsyncTask 不太常用，因为 API 使用起来较为繁琐。考虑使用 AsyncTask 的前提：需要将后台任务推到前台，或者使用 Handler。

## Executor

能用就用，如果没有就后台任务推到前台，需要做的只是将任务扔到后台，那么就使用 Executor 就可以了，特别好，没有什么缺点。

## HandlerThread

Android 提供了该 API，但是使用场景特别少，其实 HandlerThread 本质上就是一个 **在后台不断执行的单线程**，我们可以将 任务 推到这个后台单线程中去执行，其实这个工作 Executors.singleThreadExecutor() 就可以做。

HandlerThread 的使用场景：**需要在指定的 线程 中去执行任务**，而不是通过 Executors.singleThreadExecutor() 任意创建一个线程去执行任务，比如在 Android 中的主线程中去执行任务，其实本质上就是 HandlerThread。但是在开发中很少存在必须要在指定的已有的线程中去执行任务，所以 HandlerThread 的使用场景很少。

Executor 和 HandlerThread 的一点：终止任务，Executor 无法取消已经添加到队列中的任务，而 HandThread 可以通过 Handler 取消已经添加的任务。

## Service 和 IntentService

两者都不是线程的概念，**Service 是后台线程的活动空间**。

一个场景： 比如音乐播放器需要在后台线程播放音乐，但是用户可能在任务的界面操作：播放 、暂停、停止、下一首等，那么就需要有地方存储状态：线程播放到哪了？目前的状态等等，那么这个存放状态的地方就是 Service。

IntentService 首先是一个 Service，另外它是一个执行一遍任务就会挂掉的 Service，它在处理线程任务本身没有比 Executor 有任何优势，那么 IntentService 什么时候用？有时候线程需要用到 Context 时，可以使用 IntentService，其他场景没有使用的必要。

Service 和 IntentService 两者都是在使用多线程之外有别的需要，才会使用两者。


IntentService 在创建时会创建 HandlerThread 对象 handlerThread ，并开启 handlerThread.start();，并且获得 handlerThread 的 Looper 对象用以创建 Handler 对象 handler，执行 IntentService 的 onStartCommand 会调用 start(),在 start 中通过 handler 发送任务，在 handler 的 handleMessage 执行任务，任务执行完毕调用 stopSelf 结束 IntentService。


## 总结

能用 Executor 就用 Executor，
需要后台切前台，考虑 AsyncTask 和 Handler；
HandlerThread 很少会用到；
Service 和 IntentService，Service 为后台线程的活动空间，IntentService 为可以获取 Context 的线程工具（注意它是一个线程工具）；



