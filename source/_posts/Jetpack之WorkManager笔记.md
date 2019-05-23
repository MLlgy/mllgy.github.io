---
title: Jetpack之WorkManager基本了解
date: 2019-05-14 15:15:58
tags:
---


### 概述

通过WorkManager API，可以轻松安排即使应用程序退出或设备重新启动也可以运行的可延迟的异步任务。


### feature

* 向后兼容到 API 14

    * 在 API 23+ 的设备上使用 JobScheduler。
    * 在 API 14-22 的设备上使用 BroadcastReceiver 以及 AlarmManager。
* 为网络或获取充电状态添加约束
* 安排一次性异步任务或定期任务  
* 监控和管理计划任务
* 链接多个任务
* 即使应用程序或设备重新启动，也可确保任务执行
* 坚持Doze模式等省电模式


<!-- more -->

WorkManager 适用于可延迟的任务-即不需要立即运行，即使应用程序退出或设备重新启动也需要可靠运行的情况。例如：

* 将日志或分析发送到后端服务
* 定期将应用程序数据与服务器同步

WorkManager 不适用于在应用程序进程消失时，安全退出的后台工作，也不适用于需要立即执行的任务。


### 使用 Work


1. 通过 Work 来定义 任务

```
class UploadWorker(context:Context,workerParameters: WorkerParameters):Worker(context,workerParameters) {

    override fun doWork(): Result {
        return Result.success()
    }
}
```
2. 通过 WorkRequest 来管理 任务怎样以及何时运行任务。

    * 对于一次性的 WorkRequest ，使用  OneTimeWorkRequest。
    * 对于定时 WorkRequest，使用 PeriodicTimeWorkRequest。
  
    WorkRequest 还可以包含其他信息，例如任务运行的约束条件，工作输入，延迟以及重试工作的退避策略。
```
 val uploadWorkRequest = OneTimeWorkRequestBuilder<UploadWorker>().build()
```

  3. 将您的任务交给系统。
   
 Work 执行的确切时间取决于 WorkRequest 和系统优化中使用的约束。WorkManager 旨在根据这些限制提供最佳行为。

```
WorkManager.getInstance().enqueue(uploadWorkRequest)
```


### Work、WorkRequest、WorkManager 关系

Work 定义任务, WorkRequest 基于 Work 创建 执行任务的请求，WorkManager 作为管理者执行 任务请求。


### 定义 WorkRequest

#### 为 WorkRequest 添加约束条件

```
// Create a Constraints object that defines when the task should run
val constraints = Constraints.Builder()
        .setRequiresDeviceIdle(true)
        .setRequiresCharging(true)
        .build()

// ...then create a OneTimeWorkRequest that uses those constraints
val compressionWork = OneTimeWorkRequestBuilder<CompressWorker>()
        .setConstraints(constraints)
        .build()
```

当所有的约束条件满足时，work 会执行。如果在执行 work 过程中不再满足约束条件，那么 WorkManager 会停止 work 的执行，等待约束条件满足后再次执行 work。

#### 延迟执行


以下是将任务设置为在排队后至少10分钟后运行。
```
val uploadWorkRequest = OneTimeWorkRequestBuilder<UploadWorker>()
        .setInitialDelay(10, TimeUnit.MINUTES)
        .build()
```