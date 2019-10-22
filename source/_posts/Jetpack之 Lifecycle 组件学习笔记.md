---
title: Jetpack 之 Lifecycle 组件学习笔记
date: 2019-05-10 10:42:25
tags: [Jetpack,Lifecycle]
---


> 2019.10.21 更新

### 0x0000 概述

在 A/F 生命周期变化后，生命周期感知组件(Lifecycle—aware component)会响应相应动作，它会帮助你编写有良好组织的、轻量级、易维护的代码。

传统模式下，要想做的生命感知需要做的是实现其他组件的接口，并在 A/F 的生命周期函数中调用其他组件的方法。但是这样并不是好的代码组织方式，并且容易产生错误。而通过 生命周期感知组件(Lifecycle—aware component) 可以把这部分逻辑从 A/F 中移到组件自身中。

` android.arch.lifecycle (androidx 下为 androidx.lifecycle )` 包下的类和接口允许你创建 生命周期感知组件(lifecycle-aware components),它们可以根据 A/F 的生命周期来调整自己的行为。

像 Activity、Service 等组件的生命周期均是由 Android Framework 管理，同样的，Lifecycle 也由运行的系统或者 Framework 中的进程管理，在编写 Andorid 程序时需要遵守相应的规则，不然会产生内存泄漏，甚至会导致应用奔溃。


### 0x0001 Lifecycle

**Lifecycle 持有 A/F 组件有关生命周期的信息，并且允许其他对象监听它的状态**。在 Android API 26.0.1以及其后 A/F实现了LifecycleOwner，可在 A/F 中通过 getLifecycle() 获得 A/F 的 Lifecycle 对象。

<!-- more -->

Lifecycle 使用两个重要的枚举类来跟踪它所关联的组件的生命周期状态：

* Event
  
    从 Android Framework 层面和 Lifecycle 类调度的生命周期事件,这些事件映射到 A/F 中的回调事件。
* State

    Lifecycle 对象跟踪的组件的当前状态。

下图展示 Event 和 State 的关联关系

![Event和Statue](/source/images/2019_10_21_01.png)

在图中 State 作为一个个结点，作为事件间的边缘。

LifecycleObserver 通过在在其方法上添加注解来监听组件的状态，LifecycleOwner 通过 addObserver() 关联此 Observer 。

### 0x0002 LifecycleOwner 

LifecycleOwner 是只有一个方法的接口：

```
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}
```

LifecycleOwner 接口抽象出单个类的生命周期所有权，比如 A/F，并且允许编写其他组件和它配合。owner 可以提供生命周期的变化，而 observer 可以注册到 owner 并且监听其生命周期的变化。



通过实现 LifecycleOwner 接口来 **表明该类具有生命周期**，例如：

```
public class ComponentActivity extends Activity
        implements LifecycleOwner, KeyEventDispatcher.Component
```
LifecycleOwner 只有一个方法 getLifecycle()。

可见我们最常使用的 AppCompatActivity 已经实现了 LifecycleOwner 。实现 LifecycleObserver 的类作为 `观察者` 监听实现 LifecycleOwner 接口的类，监听 LifecycleOwner(也就是 Activity) 生命周期的变化。

**LiveData 的生命周期相关就是通过这种方式实现的。**

### 0x0003 自定义 LifecycleOwner

平时通过实现 AppCompatActivity 创建的 Activity 已经实现了 LifecycleOwner 接口，如果想要自定义 LifecycleOwner，同样也需要实现 LifecycleOwner 接口，并且需要通过 LifecycleRegistry 对象来管理多个 LifecycleObserver，LifecycleRegistry 内部通过 Map 来存储 LifecycleObserver 对象。


自定义 LifecycleOwner 时，需要在相应的方法中显示的声明状态，其中生命周期事件 Event 与状态的对应关系可以参见上图。

```
class MyActivity : Activity(), LifecycleOwner {

    private lateinit var lifecycleRegistry: LifecycleRegistry

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleRegistry = LifecycleRegistry(this)
        // 在该方法中声明对应声明周期的状态
        lifecycleRegistry.markState(Lifecycle.State.CREATED)
    }

    public override fun onStart() {
        super.onStart()
        lifecycleRegistry.markState(Lifecycle.State.STARTED)
    }

    override fun getLifecycle(): Lifecycle {
        return lifecycleRegistry
    }
}
```

从以上代码可以看到，自定义实现 LifecycleOwner 需要定义 LifecycleRegistry 对象，该对象可以在 LifeOwner 的每个 Event 中标记该 Event 的 State，这样 LifeObserver 才可以在不同的 state 下执行相应的操作。


### 0x0004 LifecycleObserver

生命周期的观察者，它的主要作用是监听 LifecycleOwner 对象的生命周期变化，通过注解的方式,将 LifecycleOwner 生命周期的变化映射到自己相应的方法上，并在自己的方法中进行相关业务处理。

```
class CustomObserver : LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    fun initObserver(){
        Log.e("initObserver","initObserver")
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun back(){
        Log.e("back","back")
    }
}
```

```
class TestLifecycleActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_test_lifecycle)
        lifecycle.addObserver(CustomObserver())

    }
}
```


### 0x0005 Lifecycle-aware 组件最佳实践


1. 尽可能保证 UI 组件简单，比如不要在 UI 中获取数据，而在 ViewModel 中做这些事情， LiveData 对象会在数据变化后更新 UI。
2. 尽可能在 ViewModel 中编写逻辑代码，ViewModel 应该是 UI(A/F) 和其他部分的桥梁。 但是这并不是说 ViewModel 的职责是获取数据，而应该调用其他组件来获取数据并返回 UI。
3. 使用 DataBinding 来维护 UI 和 View 间关系。这使得可以使用更少的代码来更新 UI 。
4. 如果 UI 十分复杂，可以考虑创建 presenter 来处理 UI 更改工作，虽然这个过程是费精力的，但是这对测试是十分友好的。
5. 避免在 ViewModel 中引用 View 和 Context，避免造成内存泄漏。
6. 使用 Kotlin 的协程来管理长时间运行的任务以及可以异步运行的其他操作。


### 0x0006 何时使用 lifecycle-awar 组件

1. 在粗粒度和细粒度的两种状态的地图切换展示。当 UI 在前台时使用细粒度的地图，在后台时切换为粗粒度的地图。
2. 暂停和恢复动画。UI 在后台暂停动画，在前台恢复动画。


### 0x0007 关于 LiveData 的关于感知生命周期的实现

LiveData 同为 生命中期感知组件，其实它这种功能的实现主要是依托了 Lifecycle 的实现。在 LiveData 的 obsever() 方法中将 LiveData 的 Observer 包装成 LifecycleObserver 并与 LifecycleOwner 相关联，至此 LiveData 实现了生命周期的感知功能。

LiveData#observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) 源码：
```
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    // LifecycleOwnner 添加 LifecycleObserver
    owner.getLifecycle().addObserver(wrapper);
}
```

---
以上为个人翻译官方文档

以下为优秀博客：

[Android官方架构组件Lifecycle:生命周期组件详解&原理分析](https://www.jianshu.com/p/b1208012b268)

