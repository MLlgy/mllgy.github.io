---
title: Jetpack之LiveData 笔记
date: 2019-05-08 11:21:37
tags: [Jetpack,LiveData]
---

### 0x000 概述


在官方文档中首先对 LiveData 做了一个概述 : `LiveData is an observable data holder class`, LiveData 是一个 **可观察的** **数据持有者类**。它是可以感知 Activity/Fragment/Service 的生命周期的，这使得 LiveData 只会在以上组件处于 **活跃状态下** 更新组件。

LiveData 认为上述中的 **活跃状态** 为对应的 Observer 处于 **STARTED** 或 **RESUMED** 状态。LiveData 数据更改不会触发非活跃组件的更新。

LiveData 与 观察者(实现 LifecycleOwner 的类) 建立的连接会在组件处于 DESTORY 状态后被移除。

### 0x0001 个人理解

官方文档看了几遍，大致明白了 LiveData 的作用， LiveData 可持有数据，并且它有一个重要的方法:

`public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer)` 

<!-- more -->

第一个参数为 LifecycleOwner 对象，一般为 Activity/Fragment/Servic 对象，第二参数为 Observer 对象，它的重要工作是一个回调-- `onChanged(T t)`，用于更新 UI 等工作。

 LiveData#observe() 方法中完成了对 observer 的包装，并将其添加到 LifecycleOwner 对象观察者列表中，完成了 observer 关联 LifecycleOwner 生命周期的操作。

```
 @MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    ....
    owner.getLifecycle().addObserver(wrapper);
}
```

至此 LiveData 就实现了如下功能；
    1. 数据更改时触发处于 ACTIVE 的组件的功能。
    2. UI 生命周期变更时，LiveData 会通知 Observer。

#### 0x0002 LiveData 优点


1. 保证 UI 与 实时数据完美匹配，这种模式下：
    
    1. UI 生命周期变更时，LiveData 会通知 Observer。
    2. LiveData 在数据发生变化时，通知 Observer 对象更新 UI，这就保证了数据在每次变更时 **自动更新 UI**, 而不需要手动的更新 UI。


2. 不会内存泄漏

    Lifecycle 对象销毁后(Activity/Framgent/Service)，Observer 与 Lifecycle 对象 的绑定关系会被移除，这样就不会因为两者的互相引用而导致无法回收对象。

3. 不会因为 Activity 的停止而导致崩溃

    Observer 处于 InActive 状态时, 它不会接收到 LiveData 的 Event。根据 LiveData 的相关定义只有在 LifecycleOwner 的状态为 START 或 RESUME 时 Observer 才处于 Active 状态。

4. 不再需要手动生命周期处理

    UI组件只是观察相关数据，不会停止或恢复观察。LiveData自动管理生命周期，因为它可以观察到组件的生命周期状态变化。

5. 时刻保持最新的数据

    当 LifecycleOwner 的生命周期从 incative 转换到 active 时，会更新最新的数据。

6. 旋转屏幕等配置发生变化时，可以收到最新数据
7. 共享数据资源
   LiveData 对象一致，那么那所持有的数据或资源就可以被多个页面共享。


### 0x0003 使用 LiveData


1. 创建 LiveData 对象。
2. 观察 LiveData 对象，传入 LifecycleOwner 和 Observer ,以 LiveData 为桥梁，创建两者的监听关系。
3. 变更 LiveData 所持有的数据。

能够触发 App 组件更新的唯一情况是 LiveData 的数据源发生变化，即

    1. 调用了 LiveData 的 setValue(T)// 在主线程中变更数据
    2. 调用了 LiveData 的 postValue(T)// 在 Worker 线程执行操作


### 0x0004 自定义 LiveData

### 1. 转换 LiveData

和 Rxjava 相似，通过使用 `Transformations` 的操作符转换 LiveData 对象。

#### 2. map()

该操作符做转换的工作如下：

    LiveData<T> -> LiveData<R>

```
val userLiveData: LiveData<User> = UserLiveData()
val userName: LiveData<String> = Transformations.map(userLiveData) {
    user -> "${user.name} ${user.lastName}"
}
```
本例中通过 map 实现了 LiveData<User> 向 LiveData<String> 的转换。

#### 3. switchMap()

该操作符做转换的工作如下：

    LiveData<T> -> LiveData<R>


```
private fun getUser(id: String): LiveData<User> {
  ...
}
val userId: LiveData<String> = ...
val user = Transformations.switchMap(userId) { id -> getUser(id) }
```

本例中通过 map 实现了 LiveData<String> 向 LiveData<User> 的转换。

switchMap 和 map 都实现了 LiveData 所持有数据类型的转换，但是与 map() 不同的 switchMap 的第二个参数 Function 的返回值类型必须为 LiveData。
