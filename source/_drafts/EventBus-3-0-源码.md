---
title: EventBus 3.0 源码
tags:
---


EventBusBuilder：

    为 EventBus 设置初始条件的类。

SubscriberMethodFinder：

    为指定的类寻找订阅者，即标注 @Subscribe 的方法。从 EventBusBuilder 中获取初始对象。


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

```
List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
```

分为两种情况：

* 没有使用 APT 生成索引

通过反射识别订阅者中被 @Subscribe 标记的方法。

* 使用 APT 生成索引

通过索引直接获得订阅方法。


### 第二步：产生订阅关系

订阅者与订阅方法产生订阅关系(订阅者和一个订阅方法产生一个 Subscription 对象)。
```
subscribe(subscriber, subscriberMethod);
```

在订阅关系的产生的，重点关注两个 Map 集合。

#### 第一个 Map 集合：subscriptionsByEventType

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
            // 移除订阅关系
            Subscription subscription = subscriptions.get(i);
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

到此的思考：订阅者的作用是作为订阅关系的 key ，更是在发布事件时(post)，通过反射调用该类的订阅方法，当然类，会根据事件类型的不同来过滤调用的方法。


### post 事件


将事件添加到对应的 事件序列 中，循环发送队列中的事件。


```


```


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

主线程

直接通过反射执行对应方法。
subscription.subscriberMethod.method.invoke(subscription.subscriber, event);

子线程



HandlerPoster：
    public class HandlerPoster extends Handler implements Poster
AsyncPoster：
    class AsyncPoster implements Runnable, Poster

BackgroundPoster：

    final class BackgroundPoster implements Runnable, Poster




isMainThread()：
    检查当前线程运行在主线程。, 如果没有主线程支持(如非Android),总是返回“true”。在这 MAIN 情况下，订阅者会被发布时所处的线程调用; BACKGROUND 下的订阅者会被 BackgroundPoster 调用





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


PendingPostQueue：等待发送的事件的队列
PendingPost:等待发布事件



### 解绑 EventBus

List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);

以订阅者为 Key，对所有的订阅事件分类：

    Map<订阅者(比如在CustomActivity 中有订阅方法，那么 key 为 CustomActivity),该 Class 中所以的订阅事件类型的集合>。

    解除订阅关系需要。找到该类(订阅者)所有的订阅事件类型，通过事件类型获得所有该类型事件的所有订阅关系集合，根据订阅者类型去 订阅关系集合 中删除对应的订阅关系。







-----


使用 APt 创建索引

把获得订阅方法的步骤，通过 apt 生成代码实现。

https://www.jianshu.com/p/67b495c7b56a


在一次加载时，寻找所有的订阅方法(SubscriberMethodFinder：见字识意--寻找订阅者的方法):

```
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        // 第一次加载，subscriberMethods 为 null
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        // ignoreGeneratedIndex 为 false，默认也为 false
        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            // 将获得所有的订阅方法进行缓存，下次直接获取，提高性能
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```
```

    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            // 进一步查看 getSubscriberInfo 方法
            findState.subscriberInfo = getSubscriberInfo(findState);

            //重要： 如果没有设置索引，那么此时 findState.subscriberInfo 为 null，那么会执行 else 中语句，通过反射来获得订阅者中所有的订阅方法 。
            // 同时通过 Apt 生成索引可以提高性能的原因就是：在此处避免通过反射的方式获得所有订阅方法，因为反射性能较低，虽然已经大幅改善，但是有更好的方案当然要使用到了
            if (findState.subscriberInfo != null) {
                // 通过订阅关系获得所有的订阅方法
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

    private SubscriberInfo getSubscriberInfo(FindState findState) {
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }

        if (subscriberInfoIndexes != null) {
            // 遍历通过 Apt 生成的 所有索引
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {
                // 通过 getSubscriberInfo 方法可以获得 订阅者的订阅信息 SubscriberInfo(SimpleSubscriberInfo)
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }
```



### 总结

EventBus 作为事件总线框架，将应用中所有的订阅者与订阅函数统一维护，在 post 事件后，触发所有该事件类型的订阅函数，进行相关逻辑的执行。



----

https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650242621&idx=1&sn=cc9c31aba5ff33b20fb9fecc3b0404b2&chksm=88638f52bf14064453fc09e6e53798ac5a1299d84df490b15764f9a85292732879eb4a1c0665&scene=38#wechat_redirect

