---
title: Jetpack之Lifecycle 笔记
date: 2019-05-10 10:42:25
tags: [Jetpack,Lifecycle]
---

### 概述

在 A/F 生命周期变化后，生命周期感知组件(Lifecycle—aware component)会响应相应动作，它会帮助你编写有良好组织的、轻量级、易维护的代码。

传统模式下，要想做的生命感知需要做的是实现其他组件的接口，并在 A/F 的生命周期函数中调用其他组件的方法。但是这样并不是好的代码组织方式，并且容易产生错误。而通过 生命周期感知组件(Lifecycle—aware component) 可以把这部分逻辑从 A/F 中移到组件自身中。

` android.arch.lifecycle ` 包下的类和接口允许你创建 生命周期感知组件(lifecycle-aware components),它们可以根据 A/F 的生命周期来调整自己的行为。

### Lifecycle

Lifecycle 持有 A/F 组件有关生命周期的信息，并且允许其他对象监听它的状态。在 Android API 26.0.1以及其后 A/F实现了LifecycleOwner，  可在 A/F 中通过 getLifecycle() 获得 A/F 的 Lifecycle 对象。

<!-- more -->

Lifecycle 使用两个重要的枚举类来跟踪它所关联的组件的生命周期状态：

* Event
  
    从 Android Framework 层面和 Lifecycle 类调度的生命周期事件。这些事件映射到 A/F 中的回调事件。
* State

    Lifecycle 对象跟踪的组件的当前状态。

下图展示 Event 和 State 的关联关系

![Event和Statue](https://developer.android.com/images/topic/libraries/architecture/lifecycle-states.png)

在图中 State 作为一个个结点，将事件作为结点间的边缘。

LifecycleObserver 通过在在其方法上添加注解来监听组件的状态，LifecycleOwner 通过 addObserver() 关联此 Observer 。

### LifecycleObserver

观其行为 生命周期的观察者，它的主要作用是监听 LifecycleOwner 对象的生命周期变化，在自己的方法中进行相关业务处理。


### LifecycleOwner 


通过实现 LifecycleOwner 接口来 **表明该类具有生命周期**，例如：

```
public class ComponentActivity extends Activity
        implements LifecycleOwner, KeyEventDispatcher.Component
```
LifecycleOwner 只有一个方法 getLifecycle().

可见我们最常使用的 AppCompatActivity 已经对 LifecycleOwner 做了兼容。实现 LifecycleObserver 的类作为 观察者 监听实现 实现 LifecycleOwner 接口类的声明周期的变化。

LiveData 的生命周期相关就是通过这种方式实现了。

### 自定义 LifecycleOwner

如果自定义实现 LifecycleOwner ,那么需要使用 LifecycleRegistry ,这个类用来管理多个 LifecycleObserver ,在其内部通过 Map 来存储 LifecycleObserver 对象。

自定义 LifecycleOwner 时，需要在相应的方法中显示的声明事件：

```
class MyActivity : Activity(), LifecycleOwner {

    private lateinit var lifecycleRegistry: LifecycleRegistry

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleRegistry = LifecycleRegistry(this)
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


### Lifecycle-aware 组件最佳实践


1. 尽可能保证 UI 组件简单，比如不要在 UI 中获取数据，而在 ViewModel 中做这些事情， LiveData 对象会在数据变化后更新 UI。
2. 尽可能在 ViewModel 中编写逻辑代码，ViewModel 应该是 UI(A/F) 和其他部分的桥梁。 但是这并不是说 ViewModel 的职责是获取数据，而应该调用其他组件来获取数据并返回个 UI。
3. 使用 DataBinding 来维护 UI 和 View 间关系。这使得可以使用更少的代码来更新 UI 。
4. 如果 UI 十分复杂，可以考虑创建 presenter 来处理 UI 更改工作，虽然这个过程是费精力的，但是这对测试是十分友好的。
5. 避免在 ViewModel 中引用 View 和 Context，避免造成内存泄漏。
6. 使用 Kotlin 的协程来管理长时间运行的任务以及可以异步运行的其他操作。


### 何时使用 lifecycle-awar 组件

1. 在粗粒度和细粒度的两种状态的地图切换展示。当 UI 在前台时使用细粒度的地图，在后台时切换为粗粒度的地图。
2. 暂停和恢复动画。UI 在后台暂停动画，在前台恢复动画。


### 关于 LiveData 

LiveData 同为 生命中期感知组件，其实它这种功能的实现主要是依托了 Lifecycle 的实现。在 LiveData 的 obsever() 方法中将 LiveData的 Observer 包装成 LifecycleObserver 并与 LifecycleOwner 相关联，至此 LiveData 实现了生命周期的感知功能。

---
以上为个人翻译官方文档

以下为优秀博客：

[Android官方架构组件Lifecycle:生命周期组件详解&原理分析](https://www.jianshu.com/p/b1208012b268)

