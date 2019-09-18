---
title: EventBus 粘性事件
tags:
---



### 普通事件和粘性事件


通过源码我们看一下两者的差别：


```
    public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        post(event);
    }

    public void post(Object event) {
        ....
        while (!eventQueue.isEmpty()) {
            // 每次发布一个事件，会将该事件从事件序列中移除，也就是说，事件发布后，不管是否有订阅者，
            postSingleEvent(eventQueue.remove(0), postingState);
                }
        } 
        ....
    }
```

可以看到相较于普通事件将事件添加到事件队列中，粘性事件还会被添加到 stickyEvents 中，而 stickyEvents 会在订阅者注册时进行发布。


```
   private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        ....
        ....

        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```
可以看到在初始化订阅者时，如果订阅方法的 @Subscribe 中的 sticky 属性为 ture， 那么会从 stickyEvents  获得最新的一个事件，并执行 checkPostStickyEventToSubscription 方法直接发布该事件。


### 为什么使用 Sticky Event


官方文档对 Sticky Event 有以下描述；

> Some events carry information that is of interest after the event is posted. For example, an event signals that some initialization is complete. Or if you have some sensor or location data and you want to hold on the most recent values. Instead of implementing your own caching, you can use sticky events. So EventBus keeps the last sticky event of a certain type in memory. Then the sticky event can be delivered to subscribers or queried explicitly. Thus, you don’t need any special logic to consider already available data。

> 某些事件携带事件发布后有用的信息，比如收到事件表示一些初始化动作完成，或者需要保留有关传感器的最新信息，这时可以使用 Sticky Event，而不用自己去实现缓存。EventBus 会将最后一个 Sticky Event 保留在内存中(实际上是获取 Sticky Event 集合中最新元素))。在注册订阅关系过程中，EventBus 会将该 Sticky Event 发布。


我们可以来看一个 Sticky Event 的使用场景：

> 在一个 A/F 中的成功执行结果，比如说定位信息， 项目中定位信息在很多页面都需要，此时如果使用 post 发布事件，其中一些没有初始化成功的页面就不会收到该事件，无关获得定位信息，而如果使用 postSticky 来发布事件，那么其他页面在初始化时也是可以获得该事件，也可以获得包含其中的定位信息。

### 手动获取和删除


手动获得粘性事件：

```
EventBusEvent stickyEvent = EventBus.getDefault().getStickyEvent(EventBusEvent.class);
```

手动删除粘性事件：

```
// 方法一
EventBusEvent stickyEvent = EventBus.getDefault().getStickyEvent(EventBusEvent.class);
if(stickyEvent != null){
    EventBus.getDefault().removeStickyEvent(stickyEvent);
}

// 方法二 使用此方法：当你传入类时，它将返回先前持有的粘性事件
EventBusEvent stickyEvent = EventBus.getDefault().removeStickyEvent(EventBusEvent.class);
if(stickyEvent != null) {
    //  do something
}
```