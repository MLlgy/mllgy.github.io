---
title: Android系统启动SystemServer进程概述
tags:
---


## SystemServer进程概述

Zygote 通过Zygote.forkSystemServer 函数克隆出  SystemServer进程，那么  SystemServer 进程的主要职责是什么呢？fork 成功后，通过调用 handleSystemServerProcess 函数来承担自己的职责，在该函数中进行一系列操作，比如关闭从zygote fork 而来的 socket、设置进程的参数、调用 RuntimeInit.zygoteInit函数。


在 RuntimeInit.zygoteInit 函数中存在以下几个关键点：

1. native 层的初始化——zygoteInitNative

具体实现在 AndroidRuntime.cpp 中，具体工作为启动了一个线程，该线程主要用于 Binder 通信。

zygoteInitNative 在调用后，将与 Binder 通信系统建立联系，这样 SystemServer 进程就可以使用了 Binder 了。

2. 调用 invokeStaticMain 函数，通过反射调用 Java 世界的 SystemServer#main 函数。

而最后之所以能够顺利的执行 SystemServer#main 函数，是因为在代码中建立的异常机制，具体在梳理代码详述。


注意这一系列动作的执行是由 handleSystemServerProcess 触发的，而 handleSystemServerProcess 执行的线程为 fork Zygote 进程产生的新进程，即 system_servert：

```
if(pid = 0){
    handleSystemServerProcess(XXX);
}
```

> 基于 Android9.0 源码虽然方法名和调用关系与以上不同，但是大体流程是一致的。

3. 进入 SystemServer 


在第二步中，最终调用了 SystemServer 的 main 函数，而 **这一切还都是发生在 system_server 进程中**，记住这一点是十分重要的。


在 SystemServer#main 的调用 SystemServer#run 方法，完成如下工作：


* 创建各种 SystemContext，比如：ActivityThread、Instrumentation、ContextImpl、LoadedApk、Application
* 加载 libandroid_server.so 库，通过 SystemServerManager 对象启动各种服务，比如 Installer、AMS、PMS 等服务

具体代码如下：


```9.0
private void run() {
        try {
            // Record the process start information in sys props.
            // 记录进程开启的属性值
            SystemProperties.set(SYSPROP_START_COUNT, String.valueOf(mStartCount));
            SystemProperties.set(SYSPROP_START_ELAPSED, String.valueOf(mRuntimeStartElapsedTime));
            SystemProperties.set(SYSPROP_START_UPTIME, String.valueOf(mRuntimeStartUptime));

            EventLog.writeEvent(EventLogTags.SYSTEM_SERVER_START,
                    mStartCount, mRuntimeStartUptime, mRuntimeStartElapsedTime);
            // 设置默认时区
            String timezoneProperty = SystemProperties.get("persist.sys.timezone");
            if (timezoneProperty == null || timezoneProperty.isEmpty()) {
                Slog.w(TAG, "Timezone not set;  to GMT.");
                SystemProperties.set("persist.sys.timezone", "GMT");
            }
            if (!SystemProperties.get("persist.sys.language").isEmpty()) {
                final String languageTag = Locale.getDefault().toLanguageTag();
                SystemProperties.set("persist.sys.locale", languageTag);
                SystemProperties.set("persist.sys.language", "");
                SystemProperties.set("persist.sys.country", "");
                SystemProperties.set("persist.sys.localevar", "");
            }

            // The system server should never make non-oneway calls
            Binder.setWarnOnBlocking(true);
            // The system server should always load safe labels
            PackageItemInfo.forceSafeLabels();

            // Default to FULL within the system server.
            SQLiteGlobal.sDefaultSyncMode = SQLiteGlobal.SYNC_MODE_FULL;

            // Deactivate SQLiteCompatibilityWalFlags until settings provider is initialized
            SQLiteCompatibilityWalFlags.init(null);

            // Here we go!
            Slog.i(TAG, "Entered the Android system server!");
            int uptimeMillis = (int) SystemClock.elapsedRealtime();
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, uptimeMillis);
            if (!mRuntimeRestart) {
                MetricsLogger.histogram(null, "boot_system_server_init", uptimeMillis);
            }
           
            SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());

            VMRuntime.getRuntime().clearGrowthLimit();

            // 一些设备依赖于运行时就产生的指纹信息，所以在开机前需要设置相关属性
            Build.ensureFingerprintProperty();

            // Within the system server, it is an error to access Environment paths without
            // explicitly specifying a user.
            // 在访问环境变量前，明确指定用户
            Environment.setUserRequired(true);

            // Within the system server, any incoming Bundles should be defused
            // to avoid throwing BadParcelableException.
            BaseBundle.setShouldDefuse(true);

            // Within the system server, when parceling exceptions, include the stack trace
            Parcel.setStackTraceParceling(true);

            // Ensure binder calls into the system always run at foreground priority.
            BinderInternal.disableBackgroundScheduling(true);

            // Increase the number of binder threads in system_server
            BinderInternal.setMaxThreads(sMaxBinderThreads);

            // Prepare the main looper thread (this thread).
            // 确保当前系统进程的 binder 调用，总是运行在前台优先级
            android.os.Process.setThreadPriority(
                    android.os.Process.THREAD_PRIORITY_FOREGROUND);
            android.os.Process.setCanSelfBackground(false);
            // 主线程 looper 就在当前线程运行
            Looper.prepareMainLooper();
            Looper.getMainLooper().setSlowLogThresholdMs(
                    SLOW_DISPATCH_THRESHOLD_MS, SLOW_DELIVERY_THRESHOLD_MS);

            //加载 native 服务，该库源码位置：frameworks/base/services/目录下
            System.loadLibrary("android_servers");

            // Debug builds - allow heap profiling.
            if (Build.IS_DEBUGGABLE) {
                initZygoteChildHeapProfiling();
            }
            // 检查上次关机是否失败，该方法可能没有返回值，
            performPendingShutdown();

            // 初始化系统上下文（system context）
            createSystemContext();

            // 创建系统服务管理器
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            mSystemServiceManager.setStartInfo(mRuntimeRestart,
                    mRuntimeStartElapsedTime, mRuntimeStartUptime);
            // 将 mSystemServiceManager 添加到本地服务的成员 sLocalServiceObjects
            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
            // 为task 准备线程池
            SystemServerInitThreadPool.get();
        } finally {
            traceEnd();  // InitBeforeStartServices
        }

        // 开始启动各种系统服务
        try {
            traceBeginAndSlog("StartServices");
            startBootstrapServices();// 启动引导服务
            startCoreServices();//启动核心服务
            startOtherServices();//启动其他服务
            SystemServerInitThreadPool.shutdown();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        } finally {
            traceEnd();
        }
        StrictMode.initVmDefaults(null);
        // 一直循环下去
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```


**Java 世界核心的 Service 都在这里启动**，至此 system_server 进程的启动工作总算完成了，进入了 Looper.loop 状态，等待其他线程通过 Handler 发送消息到主线程中在进行处理。



---

此处遗留的疑问：


* SystemServer 运行在 system_server 进程中，那么如何能够调用 Loop.loop(),

此时的线程为何物，不是运行在 进程中吗？


导致进程中的主线程加入到 Binder 的通信大潮中，此处的主线程是 system_server 中的主线程吗，还是 Android 系统层面的主线程。
