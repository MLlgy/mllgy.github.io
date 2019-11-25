---
title: 'Rxjava 源码学习(四)-自定义操作符:lift、compose'
date: 2019-11-23 15:46:18
tags: [Rxjava 源码分析]
---


### 0x0001 lift 和 compose 的区别

看一下在源码中的是如何对两者进行描述的：

> This compose operates on the ObservableSource itself whereas lift operates on the ObservableSource's Observers.
>  If the operator you are creating is designed to act on the individual items emitted by a source ObservableSource, use lift. If your operator is designed to transform the source ObservableSource as a whole(for instance, by applying a particular set of existing RxJava operators to it) use {@code compose}

<!-- more -->

> 如果自定义的操作符的操作对象为数据源发出的每条数据，那么使用 lift 操作符；如果自定义的操作符的操作对象为整个可观察源(比如对它使用的 Rxjava 操作符)，那么使用 compose。

两个操作符的作用对象不同：
* compose 操作符作用于 Observable 事件源。
* lift 操作符作用于 ObservableSource 的 Observers。

官方文档中建议，不要滥用 lift 操作符，因为它的使用存在诸多限制，也会对性能有轻微的影响，具体请查看 [Writing-operators-for-2.0](https://github.com/ReactiveX/RxJava/wiki/Writing-operators-for-2.0)。

### 0x0002 lift、compose 实现原理

lift、compose 操作符的实现其实和 Rxjava 中其他内置的操作符实现是一样的，同样是通过 `onSubscribe` 和 `subscribeActual` 方法作为关键点，链路到整个事件流中，两者的具体实现实现通过源码就可以很清晰的看到，因为分析源码的套路和之前的文章一样，在此不详述。

关于两者对整个事件流的影响，个人认为不过是在原来的链式添加了其他的 Observable 和 Obsever，和其他操作符没有什么区别。不过有了这两个操作符确实能够很方便的实现一些需求。

### 0x0003lift、compose 的使用

合理使用量这，可以大幅度的减少重复代码的书写，以下列举 lift、compose 常见的使用场景：

* compose:相同的 operator 组合重复出现

比如在 Rxjava 中实现线程切换的操作，使用 compose 可以避免对每个 Rxjava 事件流书写重复代码，具体实现可以查看:[RxJava Transformer](https://leegyplus.github.io/2019/01/17/RxJava%20%E4%BD%BF%E7%94%A8%20Transformer%20%E8%BF%9B%E8%A1%8C%E5%8F%98%E6%8D%A2/)

* lift:对事件发射器发出的事件执行相同的逻辑操作
  
比如下例：

```
Observable.fromIterable(list)
    .flatMap(integer -> {
        switch (integer) {
            case 1: return Observable.just("First Odd Number : " + integer);
            case 2: return Observable.just("First Even Number : " + integer);
            default: return Observable.empty();
        }
    }).subscribe(System.out::println);
```

类似对数据源的操作还有可能出现在其他 Rxjava 的事件流中，那么这时我们就可以把该部分逻辑操作封装在定义类中，使用 lift 操作符链接到原原来的事件流中，同样也避免了重复代码。


----

**知识来源：**

[Rxjava 操作符分类 中文翻译文档](https://mcxiaoke.gitbooks.io/rxdocs/content/Operators.html)

[Writing-operators-for-2.0](https://github.com/ReactiveX/RxJava/wiki/Writing-operators-for-2.0)

[lift() 和 compose()](https://ansgarlin.github.io/zh-tw/rxjava/compose_and_lift.html):文章的质量很高，让我对 lift 和 compose 的朦胧的理解具象化，真心赞。


