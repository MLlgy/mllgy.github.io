---
title: Jetpack之ViewModle 笔记
date: 2019-05-06 11:52:23
tags: [Jetpack,ViewModle]
---

### 0x0000 概述

ViewModel 是一个设计通过用来存储、管理 UI 相关的数据类。

ViewModel 类旨在通过生命周期感知的方式存储、管理与 UI 相关的数据。ViewModel 类可以在屏幕旋转情况下保持数据处于 **存活** 的状态。

Android 通过 Framework 层管理 UI(Activity/Fragment) 的生命周期。响应用户的动作系统可能会新建或重建 UI，但是这并不在用户的控制范围内。

<!-- more -->

**面临问题：**

1. 如果系统销毁或者新建 UI，则存储在其中的任何与 UI 相关短暂的数据都将丢失。对于简单的数据， Activity 可以在  onSaveInstanceState() 中保存下来，在新建的 Activity 的 onCreate() 方法中重新获得这些数据，但是这方法只适用于数据量较小、可以序列化和反序列化的数据，不使用大量的数据，例如对象列表、Bitmap 列表。

2. 另外一个问题是：UI 中频繁的异步任务会占用一些时间返回数据。UI 控制器需要管理这些请求并且保证系统能够在 UI 销毁后回收这些请求，避免潜在的内存泄漏，而管理上述情况会占用大量的资源。

UI 的主要职责是用来展示数据、响应用户的动作或者组件间交流，例如权限请求。让 UI 承担从网络或者数据库获取数据等职责，使得类变的十分的臃肿，同时使 UI 变得难以测试。我们不应该为 UI 分配过多的职责，不能让一个类去处理所有的工作，而不是将工作委托给其他类。这其实是 MVC、MVP、MVVM 这些架构演进的重要原因。

将视图数据所有权与 UI 控制器逻辑分离，使逻辑更加简单、更易于维护。

### 0x0001 ViewModel 使用

ViewModel 的主要职责是为 UI 提供数据，Activity 重建后会复用第一次建立 Activity 的 ViewModel 对象。与之前的写法不同，现在我们需要将获取到的数据保存到 ViewModel，这样就保证数据在当前 Activity 重建后可以复用数据。

官方文档示例：

```
class MyViewModel : ViewModel() {
    private val users: MutableLiveData<List<User>> by lazy {
        MutableLiveData().also {
            loadUsers()
        }
    }

    fun getUsers(): LiveData<List<User>> {
        return users
    }

    private fun loadUsers() {
        // Do an asynchronous operation to fetch users.
    }
}
```


```
class MyActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        // Create a ViewModel the first time the system calls an activity's onCreate() method.
        // Re-created activities receive the same MyViewModel instance created by the first activity.

        val model = ViewModelProviders.of(this).get(MyViewModel::class.java)
        model.getUsers().observe(this, Observer<List<User>>{ users ->
            // update UI
        })
    }
}
```
If the activity is re-created, it receives the same MyViewModel instance that was created by the first activity。
当 ViewModel 的宿主 Activity 销毁后，系统会调用 ViewModel 对象的 onCleared() 方法，用来清除资源。


**ViewModel 禁止引用 View、Lifecycle 以及其他任何引用 Activity 环境变量的对象。**


ViewModel 旨在创建超过 View 或 LifecycleOwners 对象生命周期的对象。这使得你可以更加容易的测试 ViewModel ，因为不需要知道 View 或 Lifecycle 对象。

ViewModel 可以持有 LifecycleObserver ，例如 LiveData 对象。但是 ViewModel 对象不能观察到生命周期感知的可观察对象（例如LiveData对象）的更改。

如果 ViewModel 需要 Application 上下文，例如查找系统服务，可以使用 ViewModel 的子类 AndroidViewModel 类，并在构造函数中传入 Application 对象。


### 0x0002 ViewModel 的生命周期

![ViewModel 的生命周期](https://developer.android.google.cn/images/topic/libraries/architecture/viewmodel-lifecycle.png)

You usually request a ViewModel the first time the system calls an activity object's onCreate() method. The system may call onCreate() several times throughout the life of an activity, such as when a device screen is rotated. The ViewModel exists from when you first request a ViewModel until the activity is finished and destroyed.

ViewModel 会在 Activity 第一次创建存在，直到该 Activity 被彻底销毁不再重建。


获取 ViewModel 对象时，ViewModel 对象的范围限定为传递给ViewModelProvider 的对象的生命周期。

----

有问题欢迎点击：[官方文档](https://developer.android.com/topic/libraries/architecture/viewmodel)