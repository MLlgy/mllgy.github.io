---
title: EventBus 3.0 源码
tags:
---

EventBusBuilder：

    为 EventBus 设置初始条件的类。

SubscriberMethodFinder：

    在订阅者的类中寻找订阅方法 ，即标注 @Subscribe 的方法。从 EventBusBuilder 中获取初始对象。


FindState:

    保存了订阅者是谁、订阅者的订阅方法。EventBus 防止大量创建 FindState，使用享元某事复用 FindState。

SubscriberInfo：(子类 AbstractSubscriberInfo、最终子类 SimpleSubscriberInfo))

    见字识意，订阅者的信息，保存了父类信息、订阅方法等信息。

Subscription：

    一个订阅关系。其中包含的信息有：订阅者、订阅方法。

SubscriberMethod：
    订阅者方法描述类。



Map<Object, List<Class<?>>> typesBySubscriber: 一个订阅者拥有多少个事件类型。

通过反射获得订阅者中的方法，遍历方法，获得其中被 @Subscribe 标记的方法，




### 第一步： 获得订阅者中的订阅事件

获得订阅者中的订阅事件(被 @Subscribe 标记的方法)，会从子类到父类依次寻找订阅方法。

分为两种情况：

1. 没有使用 APT 生成索引

通过反射识别订阅者中被 @Subscribe 标记的方法。

2. 使用 APT 生成索引

通过索引直接获得订阅方法。

#### 第一种方式

```
List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
```

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

Eventbus 会通过 findUsingInfo 方法获取订阅者的订阅方法：

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

在该方法中，根据有无使用 索引 则分为两种情况：



* 没有使用索引

如果没有使用 索引，那么直接通过反射去获得订阅者中的订阅方法：

findUsingReflectionInSingleClass(findState);
```
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
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

* 使用 Apt 生成索引




### 第二步：产生订阅关系

在第一步中获得了订阅者的所有订阅方法，那么在此处使订阅者与订阅方法产生订阅关系(订阅者和一个订阅方法产生一个 Subscription 对象)。
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
        // 以事件类型为 key，以订阅关系集合为 value ，添加到 Map 中
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



#### 第一个 Map 集合：subscriptionsByEventType

根据变量名就可知为以 **事件类型为 key**，以 **订阅关系集合为 value** 的 Map

```
    private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
```

以订阅的事件类型(eventType)为 Key，将订阅关系集合存储到对应的 Map 中：


```
// Map<事件类型,List<该事件对应的订阅关系(Subscription 对象)>>
 CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
subscriptionsByEventType.put(eventType, subscriptions);
```

#### 第二个 Map 集合：typesBySubscriber


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

#### 这两个 Map 的使用


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

到此的思考：订阅者的作用是作为订阅关系的 key ，更是在发布事件时(post)，通过反射调用该类的订阅方法，当然，会根据事件类型的不同来过滤调用的方法。

到此，可以猜想：在发布事件时，会根据事件类型在 subscriptionsByEventType 获得相应的订阅关系，根据订阅信息中的订阅者，可以调用订阅者参数为该事件类型的方法。


### post 事件


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

需要注意的是 ThreadLocal 这个类，它存储线程相关的变量，即在 线程 A 存储的变量值 a，只能在 线程 A 中获得，在其他线程无法获取。

ThreadLocal 原理请查看：[ThreadLocal 源码分析]()。


这是使用 ThreadLocal 来存储消息队列，其目的是让每个线程维护自己单独的消息队列，避免重复发送事件。加入多个线程共用一个队列，那么会出现以下情况：


```
共用消息队列：Queue

线程 A 向 Queue 添加消息 a, 线程 B 向 Queue 添加消息 b，两个线程成功向  Queue 中添加消息成功。此时由于某些原因，代码短暂没有执行下去，而 Queue 有两个元素：a、b。一段时间后，代码得以恢复执行，那么此时线程 A、B 均可以访问到 Queue 中的两个数据，并且分别执行，很明显接下来的动作会被 A、B 分别执行一次，重复执行。
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

根据 ThreadMode 的不同分别执行：
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

主线程

直接通过反射执行对应方法。
subscription.subscriberMethod.method.invoke(subscription.subscriber, event);

子线程

```
public class HandlerPoster extends Handler implements Poster

class AsyncPoster implements Runnable, Poster

final class BackgroundPoster implements Runnable, Poster
```




isMainThread()：
    检查当前线程(事件发布的线程)运行在主线程。, 如果没有主线程支持(如非Android),总是返回“true”。在这 MAIN 情况下，订阅者会被发布时所处的线程调用; BACKGROUND 下的订阅者会被 BackgroundPoster 调用

订阅者线程模式：

POSTING：在事件发布所在线程执行响应，通过反射实现。

MAIN：

* 发布者线程为主线程，在主线程执行响应，通过反射实现。
* 发布者为非主线程，在子线程中通过 Handler 将事件发布到主线程，在主线程通过反射实现。

MAIN_ORDERED：

判断 mainThreadPoster 是否为 null ，进行执行，最终还是通过反射实现。

BACKGROUND：

* 发布者为主线程，那么子线程中通过反射实现方法调用。
* 发布者为非主线程，直接反射调用方法。

ASYNC：在 AsyncPoster 中通过反射实现方法调用。


### 解绑 EventBus

List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);

以订阅者为 Key，对所有的订阅事件分类：

    Map<订阅者(比如在CustomActivity 中有订阅方法，那么 key 为 CustomActivity),该 Class 中所以的订阅事件类型的集合>。

解除订阅关系需要。找到该类(订阅者)所有的订阅事件类型，通过事件类型获得所有该类型事件的所有订阅关系集合，根据订阅者类型去 订阅关系集合 中删除对应的订阅关系。




### 总结

EventBus 作为事件总线框架，将应用中所有的订阅者与订阅函数统一维护，在 post 事件后，触发所有该事件类型的订阅函数，进行相关逻辑的执行。



----

https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650242621&idx=1&sn=cc9c31aba5ff33b20fb9fecc3b0404b2&chksm=88638f52bf14064453fc09e6e53798ac5a1299d84df490b15764f9a85292732879eb4a1c0665&scene=38#wechat_redirect

