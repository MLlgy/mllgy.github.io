---
title: JobScheduler 和 WorkManager 的基本使用
date: 2019-11-14 16:37:39
tags: [JobScheduler,WorkManager]
---

### 0x0001 JobScheduler 的使用

> 对于满足网络、电量、时间等一定预定条件而触发的任务，那么jobScheduler便是绝佳选择。JobScheduler主要用于在未来某个时间下满足一定条件时触发执行某项任务的情况，那么可以创建一个JobService的子类，重写其onStartJob()方法来实现这个功能。

JobScheduler 的使用步骤：

<!-- more -->

* 创建 ComponentName 对象。
* 构建 JobInfo 对象。
* 构建 JobScheduler 对象。
* 调用 JobScheduler 对象 schedule() 调度任务。


1. 创建 ComponentName 对象

```
ComponentName mServiceComponent = new ComponentName(this, MyJobService.class);
```

2. 构建 JobInfo 对象

```
// 可以使用 setXX 方法，为 JobInfo 设置条件
JobInfo jobInfo = new JobInfo.Builder(123, jobService) //任务Id等于123
        .setMinimumLatency(5000)// 任务最少延迟时间 
        .setOverrideDeadline(60000)// 任务deadline，当到期没达到指定条件也会开始执行 
        .setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED)// 网络条件，默认值NETWORK_TYPE_NONE
        .setRequiresCharging(true)// 是否充电 
        .setRequiresDeviceIdle(false)// 设备是否空闲
        .setPersisted(true) //设备重启后是否继续执行
        .setBackoffCriteria(3000，JobInfo.BACKOFF_POLICY_LINEAR) //设置退避/重试策略
        .build(); 
```

3. 构建 JobScheduler 对象

```
JobScheduler scheduler = (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);  

```


4.  调用 JobScheduler 对象 schedule() 调度任务

```
scheduler.schedule(jobInfo);
```

5. Service 中的关键方法

```
class MyService extends JobService{
    // 在任务开始执行时，执行该方法
    @Override
    public boolean onStartJob(final JobParameters params) {
        .....
        .....
        // 需要在 Handler 中执行 Job，这一点是十分重要的
        // true 代表执行任务后，定义的 Job 会继续运行，直到调用 jobFinished 
        return true;
    }

    // 调用 jobFinished 后执行
    @Override
    public boolean onStopJob(JobParameters params) {
        
        return false;
    }
}
```
6. 取消 Job

```
// 取消所有
JobScheduler jobScheduler = (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);
jobScheduler.cancelAll();
// 取消指定 Job
int jobId = job.getId();
jobScheduler.cancel(jobId);
```

具体代码参见文末 DEMO 链接。

### 0x0002 WorkManager 使用

WorkManager 为 JetPack 框架中的一个组件，WorkManager 用来执行那些不需要立刻执行、不需要可靠的执行的任务，甚至在 App 退出或者手机重启时运行任务。


例如可以执行以下任务：

* 上传日志到服务器。
* 定期与服务器同步数据。

使用 WorkManager 相应 API 可以创建任务，当满足任务的执行条件是，把任务发送到 WorkManager 中执行。

WorkManager 不适用于正在进行中的后台任务，比如：

*  应用退出，后台任务取消。
*  需要立即执行的任务。

如果使用到上面的功能，请参见L[后台任务处理指南](https://developer.android.google.cn/guide/background/?hl=en)。




1. 创建一个后台任务

在 WorkManager 中通过 Work 来定义一个任务， doWork 方法在 WorkManager 提供的后台线程中执行。

```
class BackgroundWork(appContext: Context, workParams: WorkerParameters) : Worker(appContext, workParams) {
    override fun doWork(): Result {
        Log.e("doWork","time start")
        Thread.sleep(3000L)
        Log.e("doWork","time end")
        return Result.success(workDataOf("name" to "success"))
    }
}
```
返回的 Result 通知 WorkManager 任务是否成功执行。


2. 定义任务执行的的条件和时机：WorkRequest 

在 WorkManager 存在两种 WorkRequest ：

* OneTimeWorkRequest(一次性)

* PeriodicWorkRequest(周期式)


 2.1 构建一个 OneTimeWorkRequestBuilder ：

```
private val backWorkRequest by lazy {
    OneTimeWorkRequestBuilder<BackgroundWork>()
            .build()
}
```

* 2.2 构建一个 PeriodicWorkRequest:

```
val oneTimeRequest = PeriodicWorkRequestBuilder<BackgroundWork>(15, TimeUnit.MINUTES)// 定义周期性间隔
        .build()
```

不过周期性执行的时间最小为 15 分钟，这一点需要注意，具体可以参见：[Recurring work](https://developer.android.google.cn/topic/libraries/architecture/workmanager/how-to/recurring-work?hl=zh)

2.3 为 WorkRequest 添加约束条件:

```
val constraints = Constraints.Builder()
        .setRequiresDeviceIdle(true)
        .setRequiresCharging(true)
        .build()

val oneTimeRequest = OneTimeWorkRequestBuilder<CompressWorker>()
        .setConstraints(constraints)
        .build()
val oneTimeRequest = PeriodicWorkRequestBuilder<BackgroundWork>(15, TimeUnit.MINUTES)// 定义周期性间隔
        .setConstraints(constraints)
        .build()
```
定义的 Work 只有在所有的约束条件满足时，才会执行。如果在任务运行时，约束添加变更，不再满足添加的约束条件，WorkManager 将停止执行 Worker，当满足约束条件时，将重试该任务。  

不过自己在 DEMO 中为 oneTimeRequest 添加约束条件后，在任务执行过程中，手机不满足约束条件时，任务也会继续执行下去。


再者还需要注意一点的是，系统检测约束条变化需要一定的时间，并不是约束条件变化系统都会马上通知 WorkManager 去执行相应的 Work 的，必须上例中的充电状态的变化后，在经过 20s 左右后， WorkManager 才会通知执行相应的 Work。

3. 向系统发送任务

```
WorkManager.getInstance(myContext).enqueue(uploadWorkRequest)
```



### 0x0003 两者的对比

官方文档给出关于两者的描述：

>**JobScheduler**: This is an API for scheduling various types of jobs against the framework that will be executed in your application's own process.

>**WorkManager**: With WorkManager, you can easily set up a task and hand it off to the system to run under the conditions you specify.


其实两者所能实现的功能是相似的：定义任务，交给 Android 系统执行。但是 WorkManager 作为 Jetpack 的一部分，拥有更多的特性，更推荐使用 WorkManager。


----
[JobScheduler 官方 DEMO](https://github.com/googlearchive/android-JobScheduler)

[WorkManager 官方 DEMO](https://github.com/android/background-tasks-samples)


[JobScheduler 官方文档](https://developer.android.google.cn/reference/android/app/job/JobScheduler?hl=en)

[WorkManager 官方文档](https://developer.android.google.cn/topic/libraries/architecture/workmanager?hl=zh)

[理解JobScheduler机制](http://gityuan.com/2017/03/10/job_scheduler_service/)