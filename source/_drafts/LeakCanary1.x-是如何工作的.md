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

很明显以上过程主要使用了 Application#registerActivityLifecycleCallbacks 方法监听 Activity 的生命周期，在 Activity 销毁时将该 Activity 实例对象添加到 RefWatch 的检测范围内。

上面的逻辑很简单：当 Activity 销毁后检测该 Activity 实例是否发生内存泄漏，合情合理。


关于 Fragment 的检测和 Activity 机制相同，不再重现分析过程，但是需要注意的是 RefWatch 除了检测 Fragment 实例对象也会检测 Fragment 的 View 对象。根据 SDK 版本不同 Fragment d检测过程分别由 SupportFragmentRefWatcher(在库中通过反射获得对象) 和 AndroidOFragmentRefWatcher 具体实现。


除了 LeakCanary 库实现的监听的对象外，使用者自己可以通过在 Application 中实例化的 RefWatch 对象自定义添加需要检测的对象：

```
RefWatch refWatch = LeakCanary.install(this);
refWatch.wathc(object);
```



#### 分析内存泄漏
在




### 0x0003
安装









----

[WeakRefenerce 的理解和使用](https://blog.csdn.net/zmx729618/article/details/54093532)

https://mp.weixin.qq.com/s/uwDk5D986OdMzKgtjpHdHg

https://mp.weixin.qq.com/s/idjFaJsLpVLw52RSYHA_Vg




[LeakCanary 1.x 源码原理](https://mp.weixin.qq.com/s/idjFaJsLpVLw52RSYHA_Vg)


[LeakCanary实战](https://www.sunmoonblog.com/2018/02/02/leakcanary-application/)