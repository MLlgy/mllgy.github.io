---
title: Rxjava 源码学习(五)
tags:
---


Do 操作符：

在相应动作执行时注册一个回调：



以下每个类的具体含义：

Flowable - Publisher - FlowableSubscriber/Subscriber
Observable - ObservableSource - Observer
Single - SingleSource - SingleObserver
Completable - CompletableSource - CompletableObserver
Maybe - MaybeSource - MaybeObserver



---
基本流程

线程切换
订阅 
背压
为什么有人反对使用 Rxjava？

subject