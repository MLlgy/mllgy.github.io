---
title: EventBus 线程模式
tags: [EventBus]
date: 2019-09-12 11:37:55
---

### ThreadMode:POSTING (default)
订阅者将在同一个线程中发布事件,这是默认情况。订阅者将在事件发布者的线程中响应事件，这是默认情况。事件传送将`同步`完成，一旦发布完成，所有订阅者就会被调用。这种 ThreadMode 避免了线程间的切换，因此所需的开销最小。因此，当任务简单、所需时间短、不需要占用主线程时，这种 ThreadMode 是推荐使用的。但由于事件分发可能发生在主线程，所以使用此模式的事件处理程序应该快速返回，以避免阻塞发布线程。处理函数中禁止更新UI操作。

<!-- more -->

````
// Called in the same thread (default)
// ThreadMode is optional here
@Subscribe(threadMode = ThreadMode.POSTING)
public void onMessage(MessageEvent event) {
    log(event.message);
}
````

### ThreadMode: MAIN
订阅者将在主线程中响应事件。如果发布线程为主线程，那么订阅者事件将会直接响应（像 ThreadMode.POSTING 线程模式进行事件同步）。订阅者响应事件必须快速处理完自己方法内的业务以避免阻塞主线程。处理函数中禁止更新UI操作。

````
// Called in Android UI's main thread
@Subscribe(threadMode = ThreadMode.MAIN)
public void onMessage(MessageEvent event) {
textField.setText(event.message);
}
````



### ThreadMode: MAIN_ORDERED

 在这种模式下，订阅者会在主线程中执行。该模式下，事件总是排队等待以后发送给订阅者，因此对 post 的调用将立即返回。这使事件处理更加严格且更加一致（因此名称为MAIN_ORDERED）。例如，如果您在具有 MAIN 线程模式的事件处理程序中发布另一个事件，则第二个事件处理程序将在第一个事件处理程序之前完成（因为它被同步调用 - 将其与方法调用进行比较）。使用 MAIN_ORDERED，第一个事件处理程序将完成，然后第二个事件处理程序将在稍后的时间点调用（一旦主线程具有容量）。

由于在主线程中执行，则应避免耗时操作。
```
// Called in Android UI's main thread
@Subscribe(threadMode = ThreadMode.MAIN_ORDERED)
public void onMessage(MessageEvent event) {
    textField.setText(event.message);
}
```

### ThreadMode: BACKGROUD

在这种模式下，订阅者将会在非UI线程中被调用。如果事件发布所在非UI线程，那么订阅者将会在该线程中被直接调用。反之，EventBus使用单个后台线程，它将顺序地处理响应事件。在这种模式下，订阅者快速处理完自己方法内的业务以避免阻塞线程。处理函数中禁止更新UI操作。

````
// Called in the background thread
@Subscribe(threadMode = ThreadMode.BACKGROUND)
public void onMessage(MessageEvent event){
    saveToDisk(event.message);
}
````

### ThradMode: ASYNC
事件处理方法会在单独的线程中被调用，这个线程与主线程、事件发布线程相互独立。在这种模式下，事件发布不需要等待事件处理方法。当事件处理方法需要进行耗时操作，比如：网络请求等，这是需要使用这种模式。为了限制并发线程的数量，应避免同时触发大量长时间运行的异步处理操作。EventBus通过线程池通过重用，使用线程池从完成的异步事件处理程序通知中有效地重用线程。
EventBus 通过重用线程池中已经完成异步事件的线程来达到线程的高效复用。处理函数中禁止更新UI操作。

```
// Called in a separate thread
@Subscribe(threadMode = ThreadMode.ASYNC)
public void onMessage(MessageEvent event){
    backend.send(event.message);
}
```





---

**知识链接**

[官方地址](http://greenrobot.org/eventbus/documentation/delivery-threads-threadmode/)