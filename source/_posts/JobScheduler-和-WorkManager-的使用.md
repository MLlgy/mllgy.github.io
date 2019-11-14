---
title: JobScheduler 和 WorkManager 的使用
date: 2019-11-14 16:37:39
tags:
---

### JobScheduler 的使用

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

### WorkManager 使用

WorkManager 为 JetPack 框架中的一个组件，WorkManager 为


----
[JobScheduler 官方 DEMO](https://github.com/googlearchive/android-JobScheduler)
[WorkManager 官方 DEMO](https://github.com/android/background-tasks-samples)