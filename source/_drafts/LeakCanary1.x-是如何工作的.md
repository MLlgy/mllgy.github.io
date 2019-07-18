---
title: LeakCanary1.x 是如何工作的
tags:
---


### 0x0001 
监听对象

Activity、Activity.view Fragement Fragment.view

refWatch.watch(object)(可以自定义添加监听对象吗)


添加检测对象的逻辑：

以 Activity 为例，
首先在 Activity 被销毁后(执行 destory())，此时 LeakCanary 会此时会触发 LeakCanary 的检测引用可达性，如果一个引用为不可达状态




### 0x0002
步骤 ：

安装
监听
分析
显示


#### 安装

在 LeakCanary1.x 版本中，LeakCanary 的安装位置为自定义的 Application 中：

```
LeakCanary.install(this);
```

#### 设置监听对象 
LeakCanary 本身可以实现的监听对象呀 Activity、Fragment、以及 Fragment 中的 View。

顺着上一步我们查看一下关键代码：

```
public static @NonNull RefWatcher install(@NonNull Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
}
```

这一步是十分关键的，不仅完成了监测者 RefWatch 的构建和初始化，其中我们需要关注的是 DisplayLeakService。

下面是重头戏，就是在这个位置设置监听对象：


```
//AndroidRefWatcherBuilder#buildAndInstall
public @NonNull RefWatcher buildAndInstall() {
    ....
    if (watchActivities) {
        ActivityRefWatcher.install(context, refWatcher);
    }
    if (watchFragments) {
        FragmentRefWatcher.Helper.install(context, refWatcher);
    }
    ......
    return refWatcher;
  }
```
重点关于以上两行代码，以 Activity 为例分析设置监听对象的过程。

```
  public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
    Application application = (Application) context.getApplicationContext();
    ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);

    application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
  }

  /**
   * 在此处开启监听 Activity 的开关(在 Activity 执行 destory() 方法 )
   */
  private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new ActivityLifecycleCallbacksAdapter() {
        @Override public void onActivityDestroyed(Activity activity) {
          refWatcher.watch(activity);
        }
      };
```

很明显以上过程主要使用了 Application#registerActivityLifecycleCallbacks 方法监听 Activity 的生命周期，在 Activity 执行 onDestory()后将该 Activity 实例对象添加到 RefWatch 的检测范围内。

上面的逻辑很简单：当 Activity 在执行 onDestory()后，检测该 Activity 实例是否发生内存泄漏，合情合理。


关于 Fragment 的检测和 Activity 机制相同，不再重现分析过程，但是需要注意的是 RefWatch 除了检测 Fragment 实例对象也会检测 Fragment 的 View 对象。根据 SDK 版本不同 Fragment d检测过程分别由 SupportFragmentRefWatcher(在库中通过反射获得对象) 和 AndroidOFragmentRefWatcher 具体实现。


除了 LeakCanary 库实现的监听的对象外，使用者自己可以通过在 Application 中实例化的 RefWatch 对象自定义添加需要检测的对象：

```
RefWatch refWatch = LeakCanary.install(this);
refWatch.wathc(object);
```



#### 检出内存泄漏

在上一步 设置监听对象 中我们在 Activity 执行 onDestory() 后将 Activity 实例对象添加到 RefWatch 的检测内存泄漏状态的范围内，同时也开启了检测动作。

```
public void watch(Object watchedReference) {
    watch(watchedReference, "");
  }
```

根据调用流程：

```
public void watch(Object watchedReference, String referenceName) {
   ......
   String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    //构建以 key 为唯一标识的弱引用对象，将原来的对象放入弱引用队列
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);
    ensureGoneAsync(watchStartNanoTime, reference);
}
```

其中关键点：使用 WeakReference 引用 Activity 实例对象(本例中的 watchedReference)，并且管理 ReferenceQueue，这样就可以根据 ReferenceQueue 的元素判断 Activity 实例对象是否被回收，具体关于 ReferenceQueue 的信息参见：[grgf]()。同时在后面的操作中 **配合 retainedKeys 来检测最终是否发生内存泄漏**。

继续跟进流程：

```
private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }

  Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

    removeWeaklyReachableReferences();

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks.
      return RETRY;
    }
    if (gone(reference)) {
      return DONE;
    }
    gcTrigger.runGc();
    removeWeaklyReachableReferences();
    // 如果 retainedKeys 包含 reference 则执行如下操作
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      // AndroidHeapDumper 得到堆转储文件
      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);

      HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
          .referenceName(reference.name)
          .watchDurationMs(watchDurationMs)
          .gcDurationMs(gcDurationMs)
          .heapDumpDurationMs(heapDumpDurationMs)
          .build();

      // ServiceHeapDumpListener
      heapdumpListener.analyze(heapDump);
    }
    return DONE;
  }
```

在第一个方法中的关于 `watchExecutor` 的运作机制现在不详述，先着重分析整个库的流程，后面会再回头简述。

在上面代码中可以看到该过程在触发 GC 前后分别执`removeWeaklyReachableReferences()`， 结合 ReferenceQueue(引用队列),确保回收应用内存中监测的对象中可以被正常回收的对象，最终根据 retainedKeys 中的元素是判定被监测的对象是否发生内存泄漏。

```
private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
}
```
在执行 gone(reference) 后，将可以判断指定的实例对象是否发生内存泄漏，如果发生内存泄漏就会生成堆转储文件，并进行相应的分析。

如果发生内存泄漏会通过以下代码获得堆转储文件：
```
File heapDumpFile = heapDumper.dumpHeap();
```

其中过程涉及一系列文件的新建、删除等步骤，但是关键获取堆转储文件代码为：
```
// 将 hprof 生成到指定文件
Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
return heapDumpFile;
```
指定将 hprof 生成到指定文件上，并返回，其中关于 Hprof 文件请参见:[Hprof 拾遗](https://leegyplus.github.io/2019/07/09/Hprof-%E6%8B%BE%E9%81%97/)。

这样兜兜转转终于拿到了有关内存泄漏的堆转储文件，下面就应当是分析文件定位内存泄漏点了，但是在这之前需要就拿到的文件封装进 HeapDump(持有堆转储信息的数据类)，进而进行分析内存泄漏。

```
HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
    .referenceName(reference.name)
    .watchDurationMs(watchDurationMs)
    .gcDurationMs(gcDurationMs)
    .heapDumpDurationMs(heapDumpDurationMs)
    .build();
// ServiceHeapDumpListener
heapdumpListener.analyze(heapDump);
```

#### 内存泄漏分析

由上步骤的 `heapdumpListener.analyze(heapDump);` 正式开启开启内存分析之路。

按照程序流程，执行 ServiceHeapDumpListener#analyze()
```
@Override public void analyze(@NonNull HeapDump heapDump) {
    checkNotNull(heapDump, "heapDump");
    //DisplayLeakService
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
}
```
HeapAnalyzerService#runAnalysis
```

  public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    setEnabledBlocking(context, HeapAnalyzerService.class, true);
    setEnabledBlocking(context, listenerServiceClass, true);
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    // listenerServiceClass -- DisplayLeakService
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    ContextCompat.startForegroundService(context, intent);
  }
```

各种函数回调、系统回调后来到了这里 HeapAnalyzerService#onHandleIntentInForeground：

```
@Override protected void onHandleIntentInForeground(@Nullable Intent intent) {
    if (intent == null) {
      CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
      return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

    HeapAnalyzer heapAnalyzer =
        new HeapAnalyzer(heapDump.excludedRefs, this, heapDump.reachabilityInspectorClasses);

    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey,
        heapDump.computeRetainedHeapSize);
    // 开始显示分析结果
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
  }
```
以上函数中调用了 `heapAnalyzer.checkForLeak` 中使用 haha 库对堆转储文件进行分析，并将分析结果返回。

具体怎么分析的，整个流程比较清晰，但是具体上具体代码上确实比较难以理解，本文着重分析流程，故暂且不表。

#### 显示分析结果

可以看到将分析结果继续向下传，完成显示功能。
```
AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
```
经历了AbstractAnalysisResultService#sendResultToListener、经历了AbstractAnalysisResultService#onHandleIntentInForeground 最终流程执行如下：

```
@Override
  protected final void onHeapAnalyzed(@NonNull AnalyzedHeap analyzedHeap) {
    HeapDump heapDump = analyzedHeap.heapDump;
    AnalysisResult result = analyzedHeap.result;

    String leakInfo = leakInfo(this, heapDump, result, true);
    CanaryLog.d("%s", leakInfo);

    boolean resultSaved = false;
    boolean shouldSaveResult = result.leakFound || result.failure != null;
    if (shouldSaveResult) {
      heapDump = renameHeapdump(heapDump);
      resultSaved = saveResult(heapDump, result);
    }

      PendingIntent pendingIntent =
          DisplayLeakActivity.createPendingIntent(this, heapDump.referenceKey);

      .....
}
```









----

[WeakRefenerce 的理解和使用](https://blog.csdn.net/zmx729618/article/details/54093532)

https://mp.weixin.qq.com/s/uwDk5D986OdMzKgtjpHdHg

https://mp.weixin.qq.com/s/idjFaJsLpVLw52RSYHA_Vg




[LeakCanary 1.x 源码原理](https://mp.weixin.qq.com/s/idjFaJsLpVLw52RSYHA_Vg)


[LeakCanary实战](https://www.sunmoonblog.com/2018/02/02/leakcanary-application/)


[UML时序图(Sequence Diagram)学习笔记](https://blog.csdn.net/fly_zxy/article/details/80911942)