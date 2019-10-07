---
title: Rxjava 学习
tags:
---


### Rxjava 操作符


* 创建 Observable

Create, Defer, Empty/Never/Throw, From, Interval, Just, Range, Repeat, Start, Timer

* 变换 Observable 

Buffer, FlatMap, GroupBy, Map, Scan, Window

* 过滤 Observable

Debounce, Distinct, ElementAt, Filter, First, IgnoreElements, Last, Sample, Skip, SkipLast, Take, TakeLast

* 合并 Observable

And/Then/When, CombineLatest, Join, Merge, StartWith, Switch, Zip

* 错误处理操作符

Catch, Retry

* 多用途操作符

Delay, Do, Materialize/Dematerialize, ObserveOn, Serialize, Subscribe, SubscribeOn, TimeInterval, Timeout, Timestamp, Using

* 条件和布尔运算符

All, Amb, Contains, DefaultIfEmpty, SequenceEqual, SkipUntil, SkipWhile, TakeUntil, TakeWhile

* 数学和聚合运算符

Average, Concat, Count, Max, Min, Reduce, Sum

* 转换操作符

To

* 连接操作符

Connect, Publish, RefCount, Replay


* 背压操作符

各种运算符，它强制实施特定的流控制策略。


具体操作符的使用可以查看  Rxjava 官方文档：[Rxjava Opeartors](http://reactivex.io/documentation/operators.html)




### Rxjava 中的调度


subscribeOn：指定 Observable 运行的线程。
ObserveOn：指定 Observer 运行的线程。

无论 subscribeOn 在何处调用，subscribeOn 指定的都是 Observable 在哪个线程上运行，而 ObserveOn 只可以影响其位置以下出现的 Observer 运行的线程，所以可以通过多次调用 ObserveOn 来指定运行的线程。