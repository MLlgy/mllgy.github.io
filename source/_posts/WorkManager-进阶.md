---
title: WorkManager 进阶
date: 2019-11-18 16:13:49
tags: [WorkManager,Jetpack]
---


> WorkManager 是 Jetpack 的一个可延迟的、基于条件约束的后台进程处理库。


### 0x0001 WorkManager 如何与 OS 进行交互

WorkManager 基本流程：


具体细节可以查看 [JobScheduler 和 WorkManager 的基本使用](https://leegyplus.github.io/2019/11/14/JobScheduler-%E5%92%8C-WorkManager-%E7%9A%84%E4%BD%BF%E7%94%A8/)

<!-- more -->
官方 DEMO：
![](/source/images/2019_11_18_09.png)




**1. 如何持久化请求？**

1. 保存在 WorkManager 数据库中

WorkManager 数据库是后续一切的基础，WorkManager 的任何信息都保存在数据库中：
* 工作程序是否允许
* 是否完成
* 是否失败
* 是否重试 5 次
* and so on

![](/source/images/2019_11_18_10.png)

* 如果 API >= 23

会把这个请求发送到 JobScheduler，

* 22 及以下

如图所示，根据情况不同，会把请求发送到 GcmNetWorkManager 或者 AlarmManager 中。

**2. 如何运行 WorkRequest？**

![](/source/images/2019_11_18_11.png)


假设在 API 23+ 的设置上，有一个约束条件为网络连接的 Work，当有网络后，JobScheduler 就会唤醒你的应用，进行相关的工作，WorkManager 会运行相关的 Work。


如上图所示，JobScheduler、GcmNetWorkManager、AlarmManger 都属于应用外的组件，而 GreedyScheduler 存在你的应用进程中，它们的作用是一样的，会追踪部分约束条件，在符合条件后要求 WorkManager 运行相关的 Work。由于 GreedyScheduler 存在于引用进程中，所以他无法唤醒你的应用，这个组件不依赖于 OS 的其他部分，它会适时并且更快的运行定义的 Work。


### 0x0002 WorkManager -- Your old work will still work!

WorkManager 不仅对应用内产生影响，也会对整个 OS 系统产生影响，从上文 WorkManager 的运行机制也可以看出， WorkManager 的信息会被存到数据库中，加入到相应的队列中，然后通过 OS 的其他组件去调用(在这里可以看出 WorkManager 需要应用外的组件进行配合执行)，即使此刻停用 WorkManager，那么之前已经存在的请求(未执行)也会被执行的，当然这是不符合开发者的要求的，所以如果在应用中弃用  WorkManager 的话，需要手动的取消所有的请求。


### 0x0003 what if i don't initialize WorkManager for an experimental population?

#### 1. with auto initialization, it will throw an uninitialization execption.

![](/source/images/2019_11_19_05.png)

抛出一个异常。

#### 2. your old work will still run for on-demand initalization.
![](/source/images/2019_11_19_04.png)

#### 3. what if i remove WorkManager for an experimental population.


![](/source/images/2019_11_19_06.png)

your old work will get ingored,but still use system resources!

比如应用中仍然存在一个网络连接约束，JobScheduler 会持续追踪、持续等待联网，并且在联网成功之后通知应用开始运行，但是此刻应用中已经没有 WorkManager 了

#### 4. Best Practice

Cancel all your WorkRequests to clean up after youself.

### 0x0003 why is my work not running?


1. 未满足约束条件

use `adb shell dumpsys jobscheduler` to debug on api 23+

2. 处于低电量模式(doze mode)

Job can be delayed in doze mode to preserve battery.

3. Battery saver (省电模式)

Background jobs don't run in battery saver mode.

pixel 电量低于 15% 就会默认这一模式。


4. overall system or app workload

OS 或者 App 工作负荷太大。


* Android only run a certain number of active jobs at a time.
JobScheduler 在同一时间下只允许一定数据的活跃工作
  
* WorkManager is limited by the ThreadPool you give it in its Configuration.

线程池内默认 2~4 个活跃的 Job，如果 Job 过多，就会加入延迟执行的队列中。

5. Failed or incomplete prerequistites(先决条件).


* Are your prerequiste WorkRequests to finished.
* Have they all Successed
  * A Failed prent job will fail all descendents.
  
如果有的添加没有满足，导致母工作失败，将会导致一切子工作失败。


Be carefull of this with unique work and Existing *WorkPolicy .APPEND，官方会马上提供解决此功能的 API





6. Is  your app force-stopped?

* Force-stopped wipes out all jobs and alarms.
* The next time the app is started and workmanager is initialized,it will rescheduler everything for you.

但是由于被强制停止属于具有破坏性的操作，所以无法保证被强制停止的应用被唤醒，那么如果应用不能被唤醒，应用内的 Work 将不会被执行，但是 Work 仍然存在于 应用中，因为这些 Work 存储在数据库中。


### 0x0004 why is my Work running  too ofen(为什么我的 Work 运行的如下次频繁)?


我们常常在应用中看到这样的代码：

```
class ThirdActivity : AppCompatActivity() {

    private val duration: Duration = Duration.ofHours(1)
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        WorkManager.getInstance(this)
                .enqueue(PeriodicWorkRequestBuilder<BackgroundWork>(duration).build())
    }
}
```

但是这是错误的，因为每次在 onCreate 的时候都会将 Work 添加到队列中，然后 PeriodicWorkRequestBuilder 会越积越多，正确的做法应该如下：


```
class ThirdActivity : AppCompatActivity() {

    private val duration: Duration = Duration.ofHours(1)
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        WorkManager.getInstance(this)
                .enqueueUniquePeriodicWork(
                        "test_work",
                        ExistingPeriodicWorkPolicy.KEEP,
                        PeriodicWorkRequestBuilder<BackgroundWork>(duration).build())
    }
}
```


需要把单一的 WorkRequest 加入到队列中，而针对这条周期性 Work 可以指定当同名的 WorkRequest 已经存在时该发生什么事情，在上例中采取的措施为保留旧的 WorkRequest ,如果之前我们已经把它加入到队列，那么就不需要重复创建这个 Work，继续使用它就好，这才是正确的处理方式。


### 0x0005 WorkManager 初始化


* 自动初始化
* 按需初始化


**1. 自动初始化**

让 WorkManager 采用默认的配置自动完成初始化，其具体原理为存在一个 名为 WorkManagerInitializer 的 ContentProvider，它可以把 manifest 导入到应用中，ContentProvider 的工作原理是 ContentProvider 首先进行初始化，然后才轮到 Application 的 onCreate() 方法，这一点利用了 ContentProvider 的初始化时机，在 LeakCanary3.x 中也是使用了这一点完成的自动初始化。

![](/source/images/2019_11_19_01.png)

基于此，在应用中调用 `WorkManager.getInstance(this)` 才会获取到一个非空对象。

**2. 按需初始化**

WorkManager 的自动初始化中，应用启动时除了完成自身的初始化，还要对 WorkManager 进行初始化，无疑这使应用初始化变得重起来，不是很好的操作，所以提供了 WorkManager 的按需初始化操作，这样我们可以在需要时初始化 WorkManager。

延迟初始化 WorkManager、


如何进行按需初始化?
 
![](/source/images/2019_11_19_02.png)


1. 禁用自动初始化
2. 初始化

**3. 如何发现 WorkManager 所需的配置？**


![](/source/images/2019_11_19_03.png)


自定义  Application 实现 Configuration.Provider:

```
class App:Application(), Configuration.Provider {
    override fun getWorkManagerConfiguration(): Configuration {
        TODO("not implemented") //To change body of created functions use File | Settings | File Templates.
    }
}
```

按需配置 WorkManager 的利弊：


利：

1. 避免 Application 初始化过重，提供应用的性能
2. 使用按需初始化而不是自动初始化，会避免在一些设备上的问题。


弊：

1. 未能正确初始化 WorkManager，导致一些问题。
2. 重新启动应用，Work 重排会被延迟执行。



### 0x0006 Test your Workers.


在 2.1 版本中，增强了测试功能。

提供了 TestListenableWorkerBuilder，对 WorkManager 的初始化、配置流程的步骤的简化。

具体查看 Demo。



---

**知识来源：**

[WorkManager 进阶课堂 ](https://www.bilibili.com/video/av74528360)