---
title: 从 EventBus 3.0 源码理解事件总线机制
tags:[EventBus,源码解析]
date: 2019-11-04 17:44:46
---


### 0x0001 基本介绍

EventBus 可以将 事件在 **线程之间传递**，使用简单。

### 0x0002 项目配置 EventBus 以及基本使用
在 app 模块下进行以来配置：
```
dependencies {
    implementation "org.greenrobot:eventbus:$versions.eventbus_version"
}
```

<!-- more -->

具体使用：

```
// 1. 定义事件
public class EventBusEvent {
    private String msg;
}

// 2. 在相应类中注册 EventBus，定义相应事件的方法，并且在销毁时解绑 EventBus

public class EventBusActivity extends AppCompatActivity {

    private Button button;
    private TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_event_bus);
        button = findViewById(R.id.button);
        textView = findViewById(R.id.text);
        EventBus.getDefault().register(this);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                EventBusEvent event = new EventBusEvent();
                event.setMsg("已接收到事件!");
                // 发布事件
                EventBus.getDefault().post(event);
            }
        });
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onTestEvent(EventBusEvent event) {
        textView.setText(event.getMsg());
    }

    @Override
    protected void onDestroy() {
        EventBus.getDefault().unregister(this);
        super.onDestroy();
    }
}

// 3. 在相应类的操作中向 EventBus 发送事件,就如上面 `EventBus.getDefault().post(event)` 的操作。
```


### 0x0003 EventBus 源码中的关键类
EventBusBuilder：

    为 EventBus 设置初始条件的类。

SubscriberMethodFinder：

    在订阅者的类中寻找订阅方法 ，即标注 @Subscribe 的方法。从 EventBusBuilder 中获取初始对象。


FindState:

    保存了订阅者是谁、订阅者的订阅方法。EventBus 防止大量创建 FindState，使用享元模式复用 FindState。

SubscriberInfo：(子类 AbstractSubscriberInfo、最终子类 SimpleSubscriberInfo))

    见字识意，订阅者的信息，保存了父类信息、订阅方法等信息。

Subscription：

    一个订阅关系。其中包含的信息有：订阅者、订阅方法。

SubscriberMethod：

    订阅者方法描述类。



Map<Object, List<Class<?>>> typesBySubscriber:

 一个订阅者拥有多少个事件类型。

Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType:

事件类型下的订阅关系。


### 0x0004源码解析

#### 1. 在订阅者中定义 EventBus

具体订阅见上面示例中的代码，最终通过以下代码开始注册逻辑：

```
EventBus.getDefault().register(this);

public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    // 获取订阅事件
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            // 产生订阅关系
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

通过以上代码，EventBus 完成了获得订阅事件、产生订阅关系的过程。



#### 2. 获得订阅者中的订阅事件的方式

获得订阅者中的订阅事件(被 @Subscribe 标记的方法)，会从子类到父类依次寻找订阅方法。

根据是否配置 APT 分为两种情况：

1. 没有使用 APT 生成索引

通过反射识别订阅者中被 @Subscribe 标记的方法。

2. 使用 APT 生成索引

通过索引直接获得订阅方法，使用 APT 生成索引不在此处详述。


#### 3. 通过反射的方式获取订阅事件

```
List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
```
继续跟进代码：
```
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    // 首先，检查缓存中是否含义订阅者相应的订阅方法
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
    // 默认为 false
    if (ignoreGeneratedIndex) {
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        // 将新的订阅者添加到缓存中
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```

Eventbus 会通过 findUsingInfo 方法获取订阅者的订阅方法，会在此处根据 findState.subscriberInfo 的具体值产生两个分支：
1. 通过反射获取订阅事件。
2. 通过 APT 方式获取订阅事件。

```
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        // 如果使用索引，那么 findState.subscriberInfo 不为 null
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            findUsingReflectionInSingleClass(findState);
        }
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}
```

使用反射获取订阅事件的具体流程，在 findUsingReflectionInSingleClass 方法中操作如下：


```
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    // 通过反射获取订阅者中的所有方法
    methods = findState.clazz.getMethods();
    ....
    for (Method method : methods) {
        int modifiers = method.getModifiers();
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 1) {
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) {
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                    }
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException("@Subscribe method " + methodName +
                        "must have exactly 1 parameter but has " + parameterTypes.length);
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException(methodName +
                    " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
        }
    }
}
```

通过反射的方式遍历订阅者的所有方法，从而获得所有的订阅方法，如果订阅者没有订阅方法，则会报出相应的异常。


#### 4. 产生订阅关系


在第一步中获得了订阅者的所有订阅方法，下面为第二阶段，重点为：订阅者与如何与订阅方法产生订阅关系(订阅者和一个订阅方法产生一个 Subscription 对象)。
```
subscribe(subscriber, subscriberMethod);
```
subscribe d的关键代码：
```
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    // 订阅者和订阅方法产生的订阅关系
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        // 以事件类型为 key，将订阅关系集合作为 value ，添加到 Map 中
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        ....
    }
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            // 将 newSubscription 添加到集合中
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    // typesBySubscriber 以 订阅者为 key ，以订阅事件类型为 value 的 Map
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);
    ....
}
```

至此，订阅的动作得以完成，说是订阅动作，其实就是将订阅关系、事件类型、订阅者三者之间的对应关系通过集合保存下来，重点关注两个 Map 集合。


#### 5. 两个重要的 Map 对象

**第一个 Map 集合：subscriptionsByEventType**

根据变量名就可知为以 **事件类型为 key**，以 **订阅关系集合为 value** 的 Map

```
    private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
```

以订阅的事件类型(eventType)为 Key，将订阅关系集合存储到对应的 Map 中：


```
// Map<事件类型,List<该事件对应的订阅关系(Subscription 对象)>>
 CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
 ...
subscriptionsByEventType.put(eventType, subscriptions);
```

**第二个 Map 集合：typesBySubscriber**


根据变量名可知，typesBySubscriber 为 **以订阅者为 key**，以 **事件类型为 value** 的 Map 集合

 Map<订阅者(比如在CustomActivity 中有订阅方法，那么 key 为 CustomActivity),该 Class 中所以的订阅事件类型的集合>。

```
private final Map<Object, List<Class<?>>> typesBySubscriber;
```
List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);

以订阅者为 Key，对存储该订阅者想所有的订阅事件：

```
List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
if (subscribedEvents == null) {
    subscribedEvents = new ArrayList<>();
    typesBySubscriber.put(subscriber, subscribedEvents);
}
subscribedEvents.add(eventType);
```

通过以上两个 Map 对象，可以获得所有的关系：订阅者 -> 订阅事件 -> 订阅关系。

#### 6. 这两个 Map 的使用


以上两个 Map 对象的一个重要用途：解除订阅关系。

1. 通过 typesBySubscriber 找到该类(订阅者)所有的订阅事件类型
2. 通过事件类型获得所有该类型事件的所有订阅关系集合，
3. 根据订阅者类型去 订阅关系集合 中删除对应的订阅关系。
4. 移除此订阅者

```
// 步骤 1
List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
if (subscribedTypes != null) {
    for (Class<?> eventType : subscribedTypes) {
        unsubscribeByEventType(subscriber, eventType);
    }
    typesBySubscriber.remove(subscriber);
} else {
    logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
}

private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    // 步骤 2
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            // 移除指定订阅者中订阅关系
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}

```
同样在事件发布时两个 map 对象也发挥十分重要的作用：通过事件类型获得所有的所有的订阅关系(订阅关系中含有订阅者和订阅方法的信息)，为每一个订阅关系发布事件。

理解这两个 Map 对象的意义，那么可以大致猜想 EventBus 发布事件时的基本流程：会根据事件类型在 subscriptionsByEventType 获得相应的订阅关系，由于订阅关系中含有订阅者和订阅方法的信息，那么就可以执行订阅者的订阅方法来，下面进一步查看事件发布的流程。

### 0x0005 事件发布：post 事件

post 具体代码：

```
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);
    if (!postingState.isPosting) {
        ....
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        ....
    }
}
```

这里需要注意的是 currentPostingThreadState 这个变量，看一下其初始化：

```
private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
    @Override
    protected PostingThreadState initialValue() {
        return new PostingThreadState();
    }
};
```

需要注意的是 ThreadLocal 这个类，它存储线程相关的变量，ThreadLocal 原理请查看：[ThreadLocal 源码分析](https://leegyplus.github.io/2019/09/19/ThreadLocal(Jdk1.8)%20%E4%BD%BF%E7%94%A8%E5%8F%8A%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)。使用 ThreadLocal 来存储消息队列，其目的是让每个线程维护自己单独的事件队列，避免重复发送事件，如果多个线程共用同一个事件序列，那么会出现以下情况：

```
共用事件队列：Queue

同一订阅者中开启两个线程：线程 A 和 线程 B，线程 A 向 Queue 成功添加事件 EventOne (事件类型为 Test), 线程 B 向 Queue 成功添加事件 EventTwo (事件类型相同，为 Test)。但是在事件处理过程中，事件 EventOne 由于某种原因长时间阻塞，而事件 EventTwo 顺利完成，并从事件序列 Queue 中移除，但是由于事件 a 此时还在事件序列中，所以此时线程 B 会去执行从线程 A 发送的事件，这样就造成了事件的重复处理现象。
```



将事件添加到对应的 事件序列 中，循环发送队列中的事件。

```
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            // 获取到以该事件为 key 的所有订阅关系。
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    // 发布订阅信息
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }
```


postToSubscription 源码
```
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
```

就是在此处实现了 EventBus 实现线程间传递事件的魔法，根据 ThreadMode 的不同，分别在相应的线程执行事件，具体操作如下：


* POSTING

在事件发布的线程执行事件操作，最终通过反射实现。

* MAIN

标记事件在 主线程 中执行，根据事件发布的线程分为两种情况：

  1. 发布者线程为主线程，在主线程执行响应，通过反射实现。
  2. 发布者为非主线程，在子线程中通过 Handler 将事件发布到主线程，在主线程通过反射实现(HandlerPoster)。

如果事件没有执行完毕，那么后续的 post 操作会被阻塞，直到事件执行完毕，所以订阅者的订阅方法不能执行长时间的操作。

* MAIN_ORDERED

标记订阅者在主线程中执行订阅方方法，与 Main 不同的是事件会存储在相应的事件序列中，不会阻塞 post 操作，通过序列依次执行相关事件(HandlerPoster)。

根据 mainThreadPoster 是否为空，分为两种情况：
  1. mainThreadPoster 不为空，加入主线程事件序列，进行顺序执行。
  2. mainThreadPoster 为空，通过回调执行


* BACKGROUND

  1. 事件发布者所在线程为子线程，那么通过线程池分配线程执行(BackgroundPoster)。
  2. 事件发布者所在线程为主线程，通过反射执行。

* ASYNC

无论事件发布者所在的线程是否为主线程，都会在异步线程中执行(AsyncPoster)。


通过源码可以看到，子线程为线程池中分配的线程:

```
eventBus.getExecutorService().execute(this);
```


### 0x0006 解绑 EventBus

List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);

以订阅者为 Key，对所有的订阅事件分类：

    Map<订阅者(比如在CustomActivity 中有订阅方法，那么 key 为 CustomActivity),该 Class 中所以的订阅事件类型的集合>。

解除订阅关系需要。找到该类(订阅者)所有的订阅事件类型，通过事件类型获得所有该类型事件的所有订阅关系集合，根据订阅者类型去 订阅关系集合 中删除对应的订阅关系。


### 0x0007 总结

EventBus 作为事件总线框架，将应用中所有的订阅者与订阅函数统一维护，在 post 事件后，触发所有该事件类型的订阅函数，进行相关逻辑的执行，实现了线程间组件的事件传递。

----

**你还可以阅读其他知识来源：**


[EventBus 源码](https://github.com/greenrobot/EventBus)

[从源码入手来学习EventBus 3事件总线机制](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650242621&idx=1&sn=cc9c31aba5ff33b20fb9fecc3b0404b2&chksm=88638f52bf14064453fc09e6e53798ac5a1299d84df490b15764f9a85292732879eb4a1c0665&scene=38#wechat_redirect)

