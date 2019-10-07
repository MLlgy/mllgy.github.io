---
title: 'Rxjava 源码学习(二):线程切换'
tags:
---


subscribeOn：指定 Observable 运行的线程。
ObserveOn：指定 Observer 运行的线程。


SubscribeOn运算符指定Observable将在哪个线程上开始操作，无论该运算符在运算符链中的哪个点被调用。 ObserveOn，而另一方面，影响线程可观察到的将使用下面看起来算哪里。因此，您可以在Observable运算符链中的各个点多次调用 ObserveOn，以更改某些运算符在哪些线程上进行操作。


---