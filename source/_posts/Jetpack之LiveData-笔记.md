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

 LiveData#observe() 方法中完成了对 observer 的包装(ObserverWrapper)，并将其添加到 LifecycleOwner 对象观察者列表中，完成了 observer 关联 LifecycleOwner(A/F) 组件的生命周期的操作。

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
    3. 组件被销毁后， Observer 对象会被移除。

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

使用 LiveData 的基本步骤：

1. 创建持有特定类型的 LiveData 对象，这部分工作一般在 ViewModel 中完成。
2. 创建 Observer 对象，当 LiveData 持有的数据变化时，在 Observer 对象的 onChanged() 中进行相应的逻辑处理。一般在 Activity/Fragment 中创建 Observer 对象。
3. 通过 LiveData 的 observe() 关联 Observer 对象，并且传入 LifecycleOwner 对象，所以在 LiveData 数据更改时通知 Observer 对象。


在 LiveData 对象中的数据变化时，如果相应的 LifecycleOwner 处于活动状态，它就会触发所有注册的观察者。

当 LiveData 对象保存的数据更改时，UI 会自动更新数据。




**创建 LiveData 对象**


LiveData 是泛型包装类，可以持有任意类型的数据，一般在 ViewModel 中初始化 LiveData：

```
class NameViewModel : ViewModel() {

    // Create a LiveData with a String
    val currentName: MutableLiveData<String> by lazy {
        MutableLiveData<String>()
    }

    // Rest of the ViewModel...
}
```


出于以下原因，请确保将更新 UI 的 LiveData 对象存储在ViewModel对象中，而不是 Activity/Fragment 中：

* 避免臃肿的 Activity和 Fragment。 UI 控制器 只负责显示数据，而不负责持有数据状态。
* 使 LiveData 实例与 Activity/Fragment 脱离，允许 LiveData 对象在配置变改后不被销毁。

**创建 Observer 对象**

一般在 onCreate 方法进行初始化 Observer 对象，并且关联 LiveData 对象，原因有以下几条：

* 确保不会在 onResume（）方法中进行多余的调用。
* 确保 Activity/Fragment 在进入 Active 状态后可以立即显示的数据。只有在 Observer 对象创建完毕后，当应用 A/F 处于 Active 状态，才可以就可以从关联的 LiveData 对象中接收最新值，所以应该尽早的创建 Observer 对象。

```
// androidx 包下创建方式有所变更
class NameActivity : AppCompatActivity() {

    private lateinit var model: NameViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Other code to setup the activity...

        // Get the ViewModel.
        model = ViewModelProviders.of(this).get(NameViewModel::class.java)


        // Create the observer which updates the UI.
        val nameObserver = Observer<String> { newName ->
            // Update the UI, in this case, a TextView.
            nameTextView.text = newName
        }

        // Observe the LiveData, passing in this activity as the LifecycleOwner and the observer.
        model.currentName.observe(this, nameObserver)
    }
}
```

需要注意的是将 LifecycleOwner 的实例 this 传入 observer 方法中，这就意味着 Observer 对象的生命周期与其宿主 LifecycleOwner 的生命周期绑定，下文所说的 Observer 的 Active 状态和 InActive 状态与 LifecycleOwner 的状态是一致的。


**更新 LiveData 持有的数据**


LiveData 类没有公开的方法来更新数据，但是  MutableLiveData 暴露了两个方法：setValue(T) 和 postValue(T)，如果想要更改 LiveData 中持有的数据可以使用这两个方法。通常在 ViewModel 中使用 MutableLiveData，而 ViewModel 仅向观察者公开不可变的 LiveData 对象，保证了数据的安全性。

```
button.setOnClickListener {
    val anotherName = "John Doe"
    model.currentName.setValue(anotherName)
}
```

调用 setValue 方法会触发 Observer 对象的 onChanged() 方法，并且会携带变更后的数据。


> 能够触发 App 组件更新的唯一情况是 LiveData 的数据源发生变化，即

    1. 调用了 LiveData 的 setValue(T)// 必须在主线程调用 setValue 变更数据
    2. 调用了 LiveData 的 postValue(T)// 必须在子线程执行操作



**和其他组件组合使用 LiveData**

Jetpack 的 Room 组件、与 Kotlin 协程组合使用。



### 扩展 LiveData

如果观察者的生命周期处于 STARTED 或 RESUMED 状态，则 LiveData 认为观察者处于 Active 状态。下面的示例代码说明了如何扩展 LiveData 类：

这是一个更新股价的示例：

```
class StockLiveData(symbol: String) : LiveData<BigDecimal>() {
    private val stockManager = StockManager(symbol)

    private val listener = { price: BigDecimal ->
        value = price
    }

    override fun onActive() {
        stockManager.requestPriceUpdates(listener)
    }

    override fun onInactive() {
        stockManager.removeUpdates(listener)
    }
}
```

在实现 LiveData 的方法中，需要注意以下几个方法：

* onActive: 当 LiveData 对象具有活动的观察者时，将调用onActive（）方法。这意味着您需要从这种方法开始观察股价更新。
* 当 LiveData 对象没有任何活动的观察者时，将调用 onInactive（）方法。由于没有观察者在监听，不需要与 StockManager 建立连接。
* setValue（T）方法将更新 LiveData 实例的值，并将有关更改通知所有处于 Active 状态的观察者。


可以按如下方式使用 StockLiveData :
```
override fun onActivityCreated(savedInstanceState: Bundle?) {
    super.onActivityCreated(savedInstanceState)
    val myPriceListener: LiveData<BigDecimal> = ...
    myPriceListener.observe(this, Observer<BigDecimal> { price: BigDecimal? ->
        // Update the UI.
    })
}
```


因为 LiveData 是生命周期感知的组件，所以可以在多个 Activity、Fragment、Service 中共享它，可以将 StockLiveData 变为单例：


```
class StockLiveData(symbol: String) : LiveData<BigDecimal>() {
    private val stockManager: StockManager = StockManager(symbol)

    private val listener = { price: BigDecimal ->
        value = price
    }

    override fun onActive() {
        stockManager.requestPriceUpdates(listener)
    }

    override fun onInactive() {
        stockManager.removeUpdates(listener)
    }

    companion object {
        private lateinit var sInstance: StockLiveData

        @MainThread
        fun get(symbol: String): StockLiveData {
            sInstance = if (::sInstance.isInitialized) sInstance else StockLiveData(symbol)
            return sInstance
        }
    }
}
```
可以在 Fragment 中按照如下方式使用：

```
class MyFragment : Fragment() {

    override fun onActivityCreated(savedInstanceState: Bundle?) {
        StockLiveData.get(symbol).observe(this, Observer<BigDecimal> { price: BigDecimal? ->
            // Update the UI.
        })

    }
}
```

多个Activity 和 Fragment 可以观察 MyPriceListener 实例。LiveData 只有当 **一个或多个** Activity 和 Fragment 处于可见状态并且处于活动状态时，才会连接到系统服务。


### 0x0004  LiveData 操作符

### 1. 转换 LiveData

和 Rxjava 相似，通过使用 `Transformations` 的操作符转换 LiveData 对象。

#### 2. map()

对 LiveData 对象中存储的值进行相应函数的操作，并将结果传播到下游。 


Transformations.map 变换 LiveData 持有的数据类型，即 Lambda 表达式完成数据类型 X 到 Y的转换。

```
val userLiveData: LiveData<User> = UserLiveData()
val userName: LiveData<String> = Transformations.map(userLiveData) {
    user -> "${user.name} ${user.lastName}"
}
```
本例中通过 map 实现了 LiveData<User> 向 LiveData<String> 的转换。

#### 3. switchMap()


Transformations.switchMap 同样也是变换 LiveData 持有的数据类型，但是不同的是，传入的 Lambda 表达式完成数据类型 X 到 LiveData<Y> 的转换。


```
private fun getUser(id: String): LiveData<User> {
  ...
}
val userId: LiveData<String> = ...
val user = Transformations.switchMap(userId) { id -> getUser(id) }
```


switchMap 和 map 都实现了 LiveData 所持有数据类型的转换，但是与 map() 不同的 switchMap 的第二个参数 Function 的返回值类型必须为 LiveData。


在组件的生命周期内，可以使用 操作符方法进行传递信息，

可以使用转换方法在 Observer 的整个生命周期中传递信息。只有在 Observer 对象在监听 LiveData 对象时，转换操作符才会执行。因为转换操作是延时操作的，因此与生命周期相关的行为会隐式传递，而不需要显式调用或依赖。

如果你在 ViewModel 中需要 Lifecycle 对象，那么转换操作符可能是一个更好的解决方案。假设一个 UI 组件接收一个地址后返回该地址的邮编，该业务的实现可以用 ViewModel 实现，代码如下：

```
class MyViewModel(private val repository: PostalCodeRepository) : ViewModel() {

    private fun getPostalCode(address: String): LiveData<String> {
        // DON'T DO THIS
        return repository.getPostCode(address)
    }
}
```

但是这是不被推荐的，UI 组件每次调用 getPostalCode（）时都需要从先前的LiveDat a对象中注销并注册到新实例。此外，如果重新创建了 UI 组件，则会触发对 repository.getPostCode（）方法的调用，而不是使用先前调用的结果。


可以将地址到邮编的转换用 操作符实现，代码如下：


```
class MyViewModel(private val repository: PostalCodeRepository) : ViewModel() {
    private val addressInput = MutableLiveData<String>()
    val postalCode: LiveData<String> = Transformations.switchMap(addressInput) {
            address -> repository.getPostCode(address) }


    private fun setInput(address: String) {
        addressInput.value = address
    }
}
```

在这种情况下，postalCode 为 addressInput 转换后得到的。只有在应用程序具有与 postalCode 关联的处于 Active 的观察者的情况下，当 addressInput 发生更改，便会重新执行转换符相应的操作，获得新的 postalCode 值 。

此机制允许较低级别的应用程序创建按需延迟计算的 LiveData 对象。ViewModel 对象可以轻松获取对 LiveData 对象的引用，执行相应的转换操作。



### 合并不同的 LiveData 数据源

MediatorLiveData 是 LiveData 的子类，可以实现合并多个 LiveData 的数据源，当其中的一个 LiveData 数据源发生改变时，所以与该 MediatorLiveData 对象关联的 Observer 对象会被触发。

