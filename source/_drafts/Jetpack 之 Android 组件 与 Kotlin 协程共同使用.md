---
title: 译:Jetpack 之 Android 组件 与 Kotlin 协程共同使用
tags: [Jetpack, Kotlin 协程]
---


### 0x0001 概述

Kotlin 协程提供了 API 以供开发者写出简单的异步代码。在 Kotlin 协程中，可以定义 CoroutineScope 作用域，它可以帮助开发者何时运行协程，每一个异步操作运行在特定的作用域内。



本主题说明如何协程如何与系统组件一起有效使用。

### 0x0002 添加组件的协程依赖

通过引入对应的 KTX 扩展包，来使用相应组件的内置 Kotlin 协程作用域，相应依赖如下：

```
// ViewModelScope
androidx.lifecycle:lifecycle-viewmodel-ktx:xxx
// LifeCycleScope
use androidx.lifecycle:lifecycle-runtime-ktx:xxx
// LiveData
androidx.lifecycle:lifecycle-livedata-ktx:xxx
```

<!-- more -->
在不同组件中使用 Kotlin 协程需要引入不同的依赖。


### 0x0003 不同组件的内置协程作用域

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


**LifcycleScope**


为每个 Lifecycle 对象定义一个 LifecycleScope，当 Lifecycle 对象销毁时，所有在该作用域构建的协程会自动取消。可以通过 `lifecycle.coroutineScope` 或者 `lifecycleOwner.lifecycleScope` 的方式获得 Lifecycle 对象的协程作用域。

下面的例子展示如何通过 `lifecycleOwner.lifecycleScope` 异步创造 text:

```
class MyFragment: Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        viewLifecycleOwner.lifecycleScope.launch {
            val params = TextViewCompat.getTextMetricsParams(textView)
            val precomputedText = withContext(Dispatchers.Default) {
                PrecomputedTextCompat.create(longTextContent, params)
            }
            TextViewCompat.setPrecomputedText(textView, precomputedText)
        }
    }
}
```

### 0x0004 挂起特定生命周期状态下的协程

尽管 CoroutineScope 提供了一种自动取消长时间的运行操作的方式，但是也存在这样一种需求：在 Lifecycle 处于特定的状态才会执行相应的代码。

存在这样一种场景：为了执行 FragmentTransaction 相应操作，需要 Lifecycle 至少处于 STARTED 的状态。为了解决这样情况，Lifecycle 提供了以下方法：`lifecycle.whenCreated, lifecycle.whenStarted, and lifecycle.whenResumed`，**当 Lifecycle 没有执行到以上指定状态时，处于以上协程内的代码会被挂起**。

```
class MyFragment: Fragment {
    init { // Notice that we can safely launch in the constructor of the Fragment.
        lifecycleScope.launch {
            whenStarted {
                // 此闭包中的代码只有在 STARTED 状态下才会运行，针对当前情况，当 Fragment 启动时，协程闭包中的代码会被执行，并且可以调用其他挂起函数。
                loadingView.visibility = View.VISIBLE
                val canAccess = withContext(Dispatchers.IO) {
                    checkUserAccess()
                }

                // When checkUserAccess returns, the next line is automatically
                // suspended if the Lifecycle is not *at least* STARTED.
                // We could safely run fragment transactions because we know the
                // code won't run unless the lifecycle is at least STARTED.
                // 当checkUserAccess返回时，如果 Fragmeng 的生命周期不是 STARTED或 STARTED 以后的状态，那么下一行自动挂起，。
                loadingView.visibility = View.GONE
                if (canAccess == false) {
                    findNavController().popBackStack()
                } else {
                    showContent()
                }
            }
            // This line runs only after the whenStarted block above has completed(此处的代码只有在 whenStarted 闭包执行完毕后才会执行)。
        }
    }
}
```

当协程是通过 lifecycleScope 的 whenxxx 方法开启的，如果组件生命周期处于销毁(destroyed) 状态，那么这个协程会被自动取消，示例如下：

```
class MyFragment: Fragment {
    init {
        lifecycleScope.launchWhenStarted {
            try {
                // Call some suspend functions.
            } finally {
                // This line might execute after Lifecycle is DESTROYED.
                if (lifecycle.state >= STARTED) {
                    // Here, since we've checked, it is safe to run any
                    // Fragment transactions.
                    // 对状态进行审核，这样更安全
                }
            }
        }
    }
}
```

在这里例子中，一旦生命周期处于 DESTROYED 状态，那么就会执行 finally 闭包中的代码，此处为了确保安全调用，对生命周期的状态进行校验。

> 注意：尽管这些方法在使用 Lifecycle 时提供了便利，但仅在信息（例如，预先计算的文本）在组件生命周期范围内，才能使用它们。需要注意的是:如果 Activity 重新启动，协程将不会重新启动。

### 0x0005 LiveData 组件与 Kotlin 协程组合使用

在使用 LiveData 时，你可以需要异步的计算的操作。比如你想要获取用户的选择项，并把选择呈现到 UI 上，在这个场景下就可以使用 `livedata 构造器方法` 去调用挂起函数，并且将然后的结果包装成 LiveData 对象，代码如下：
```
val user: LiveData<User> = liveData {
    val data = database.loadUser() // loadUser is a suspend function.
    emit(data)
}
```

liveData 构建块在 协程和 LiveData 之间充当 `结构化并发函数` 的作用。以上代码块在 LiveData 变为活跃状态后开始执行，在 LiveData 变为非活跃状态下会自动取消，其取消时间是可配置的。如果在完成之前取消，那么该闭包会在 LiveData 变为活跃状态后重新执行，而如果在取消前已经完成，那么该闭包就不会重新执行。由其他原因(比如抛出异常)导致该闭包取消执行，那么该闭包不会重复执行。

同时可以在该闭包内发送多个值，每次调用 emit() 都会挂起该闭包的执行，直到在主线程上设置 LiveData 值为止，代码如下：

```
val user: LiveData<Result> = liveData {
    emit(Result.loading())//挂起，直到 Result.loading() 执行完毕
    try {
        emit(Result.success(fetchUser()))
    } catch(ioException: Exception) {
        emit(Result.error(ioException))
    }
}
```

同时可以使用 Transformations API 来组合 liveData，示例如下：

```
class MyViewModel: ViewModel() {
    private val userId: LiveData<String> = MutableLiveData()
    val user = userId.switchMap { id ->
        liveData(context = viewModelScope.coroutineContext + Dispatchers.IO) {
            emit(database.loadUserById(id))
        }
    }
}
```

每当要发出新值时，都可以通过调用 embedSource() 函数从 LiveData 中发出多个值。请注意，每次调用emit() 或 emitSource()都会删除先前添加的源,示例如下：

```
class UserDao: Dao {
    @Query("SELECT * FROM User WHERE id = :id")
    fun getUser(id: String): LiveData<User>
}

class MyRepository {
    fun getUser(id: String) = liveData<User> {
        val disposable = emitSource(
            userDao.getUser(id).map {
                Result.loading(it)
            }
        )
        try {
            val user = webservice.fetchUser(id)
            // Stop the previous emission to avoid dispatching the updated user
            // as `loading`.
            // 停止之前发起的 emission，以避免调度更新的用户为 loading 
            disposable.dispose()
            // Update the database.
            userDao.insert(user)
            // Re-establish the emission with success type.
            // 用成功类型重新建立 emission。
            emitSource(
                userDao.getUser(id).map {
                    Result.success(it)
                }
            )
        } catch(exception: IOException) {
            // Any call to `emit` disposes the previous one automatically so we don't
            // need to dispose it here as we didn't get an updated value.
            // 任何对 emit 的调用都会自动处理前一个，因此我们不用调用 dispose 方法，因为我们没有更新的值。
            emitSource(
                userDao.getUser(id).map {
                    Result.error(exception, it)
                }
            )
        }
    }
}
```

----

**知识链接**

[Google 官方文档 : Use Kotlin coroutines with Architecture components](https://developer.android.com/topic/libraries/architecture/coroutines)