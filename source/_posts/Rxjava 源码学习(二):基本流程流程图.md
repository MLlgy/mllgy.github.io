---
title: 'Rxjava 源码学习(二):基本流程流程图'
date: 2019-10-18 15:32:54
tags: [Rxjava 源码分析]
---



在 [Rxjava 源码学习(一):基本流程分析](https://leegyplus.github.io/2019/10/18/Rxjava%20%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0(%E4%B8%80):%E5%9F%BA%E6%9C%AC%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/#more) 分析了基本流程，并且通过 Map 操作符一窥 Rxjava 操作符的特色，而本篇主体只有一张 Rxjava 流程图，但是这张流程图基本上可以概括 Rxjava 的框架，整个流程可以看做是一个 “横向 S” 模型。


该图共涉及 Rxjava 事件流的以下几个方面：
1. Observable 的创建
2. Observer 的创建、产生订阅关系
3. 订阅关系的传递
4. 取消订阅的流程


<!-- more -->
具体看图上的标记:

![](/../images/2019_10_18_01.png)

**关于源码中的 upsteam 和 downstream**

从图上可以看到，在最终订阅 Observer 之前，执行每一个操作符并不会同时生成相应的 Observable 和 Observer，以调用 subscribe 为分界线，将整个事件流分成两部分：
1. 调用 subscribe 之前，生成相应操作符的 Observable。
2. 调用 subscribe 之后，生成相应操作符的 Observer，并产生订阅关系。

需要注意的一点是在查看源码会看到 upstream、downstream，具体的 up 和 down 不是有相应对象的生成顺序决定的，而是有 Rxjava 相应操作符的调用先后决定。


**关于自定义 Observer 的 onSubscribe 方法的执行线程问题**


Rxjava 中的 observerOn 和 subscribeOn 可以指定相应的 Observer 和 Observable 的运行线程，但是通过打印日志我们可以看到 onSubscribe 运行的线程并不是两个操作符指定的线程，而是代码执行的线程。


```
public void onSubscribe(Disposable d) {
    Log.e("TAG", "onSubscribe(): 所在线程为 " + Thread.currentThread().getName());
}
```
打印日志为：
onSubscribe(): 所在线程为 main


至于为什么会这样，可以看一下最上游的 Observable 的 subscribeActual 方法：

```
@Override
protected void subscribeActual(Observer<? super T> observer) {
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    observer.onSubscribe(parent);
    try {
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}

@Override
public final void subscribe(Observer<? super T> observer) {
    subscribeActual(observer);
}
```
通过 [Rxjava 源码学习(一):基本流程分析](https://leegyplus.github.io/2019/10/18/Rxjava%20%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0(%E4%B8%80):%E5%9F%BA%E6%9C%AC%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/#more)  的分析知道，Rxjava 整个事件流的核心方法为 subscribeActual、subscribe，而两个方法均在代码调用的线程执行(这里是 main)，基于此 onSubscribe 方法就是在这里开始调用下游对象的 onSubscribe 方法，所以 onSubscribe 的执行线程也不会发生改变(这里是 main)。



**关于取消订阅关系**

在日常开发中，我们会遇到类似这样的需求：当退出 Activity 时，需要取消正在执行的实现，此功能的实现就是通过取消订阅关系来实现。


情况一：

```
   Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("one");
                emitter.onNext("two");
                emitter.onNext("three");
            }
        }).subscribe(new Observer<String>() {
            Disposable mDisposable;
            @Override
            public void onSubscribe(Disposable d) {
                mDisposable = d;
            }

            @Override
            public void onNext(String s) {

                if(s == "two"){
                    mDisposable.dispose();
                }
            }
        });
```


情况二：

```
CompositeDisposable compositeDisposable = new CompositeDisposable();

private void test(){
   Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("one");
                emitter.onNext("two");
                emitter.onNext("three");
            }
        }).subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                // 将订阅关系添加到 compositeDisposable
                compositeDisposable.add(d);
            }
        });
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 解除所有的订阅关系
        compositeDisposable.clear();
    }
```