---
title: Jetpack 之 ViewModle 与 Kotlin 协程共同使用
tags:
---


### 0x0001 概述


### 0x0002 如何使用

每个组件的内置的协程作用域包括在 KTX 扩展包中，可以根据需要添加具体的 KTX 依赖：


```
// ViewModelScope

// LifeCycleScope

// LiveData


```


### 0x0003 生命周期可感知的协程作用域

Android 框架定义了以下几种内置的作用域，可以在应该开发中使用它们。

**ViewModelScope**

每一个 ViewModel 都定义了 ViewModelScope，当 ViewModel 被销毁时，在它的作用域内构建的所有协程都会被 cancel，因为只有当 ViewModel 处于 Active 状态下，协程中所执行的动作才有意义。假如你正在执行一些计算动作，那么你就应该把这部分工作放在 ViewModel 中，当 ViewModel 被销毁时，计算工作会自动终止，这样避免了消耗不必要的资源。


Kotlin 协程在 ViewModel 中的协程作用作用域对象为 viewModelScope，具体获取如下：

```
class MyViewModel: ViewModel() {
    init {
        viewModelScope.launch {
            // Coroutine that will be canceled when the ViewModel is cleared.
        }
    }
}
```

\

**LifcycleScope**


为每个 Lifecycle 对象定义一个 LifecycleScope，当 Lifecycle 对象销毁时，所有在该作用域构建的协程会自动取消。可以通过 `lifecycle.coroutineScope` 或者 `lifecycleOwner.lifecycleScope` 的方式获得 Lifecycle 对象的协程作用域。

下面的例子展示如何通过 `lifecycleOwner.lifecycleScope` 异步创造 text:

```

```

### 0x0004 挂起生命周期感知的协程

尽管 CoroutineScope 提供了一种自动取消长时间的运行操作的方式，但是也存在这样一种需求：在 Lifecycle 处于特定的状态才会执行相应的代码。

存在这样一种场景：为了执行 FragmentTransaction 相应操作，需要 Lifecycle 至少处于 STARTED 的状态。为了解决这样情况，Lifecycle 提供了以下方法：`lifecycle.whenCreated, lifecycle.whenStarted, and lifecycle.whenResumed`，当 Lifecycle 没有执行到指定状态时，处于以上协程内的代码会被挂起。

