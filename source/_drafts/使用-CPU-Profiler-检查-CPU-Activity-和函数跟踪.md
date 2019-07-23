---
title: 使用 CPU Profiler 检查 CPU Activity 和函数跟踪
tags:
---



#### 使用 Call Chart 标签检查跟踪


Call Chart 标签提供函数跟踪的图形表示形式。

水平方法表示函数执行的时间段和时间，并 **沿垂直轴显示其被调用者**。
图形不同颜色表示不同函数级别：
* 橙色 -- 对系统 API 的调用
* 绿色 -- 对应自有函数的调用
* 蓝色 -- 对第三方 API(包括 Java 的 API) 的调用


直接拿官方文档图片来展示 Call Chart 具体含义。


![Call Chart(方法调用图)](https://developer.android.google.cn/studio/images/profile/call_chart_1-2X.png)


根据图片我们可以得知各方法的调用关系：

```
fun A(){
    B();
    D();
}

fun B(){
    C();
}

fun D(){
    C();
    B();
}
```

从图片上我们可以得知方法 D 执行时间以及其调用函数的执行时间：
```
D(self time) + D(children time) = D(total time)

其中 D(chidren time) = C(total time) + B(total time)
B(total time) = B(self time) + B(children time)
B(children time) = C(toal time)
```


#### 使用 Flame Chart 标签检查跟踪


收集 **共享相同调用方顺序的完全相同** 的函数，并在火焰图中用一个较长的横条表示它们（而不是将它们显示为多个较短的横条，如调用图表中所示）。

水平轴不再像调用图一样代表时间线，它表示 **每个函数相对的执行时间**，这样更方便查看 **哪些函数消耗最多时间**。
 
  

 下面展示从 Call Chart 如何转换为 Flame Chart。


 ![Call Chart(方法调用图)](https://developer.android.google.cn/studio/images/profile/call_chart_2-2X.png)

 其中 D 多次调用了 B(B1、B2、B3)，那么B1、B2、B3 共享相同的调用顺序(A->D-B)。B 多次调用了 C(C1、C3)，那么 C1、C3 共享相同的调用顺序(A->D->B->C)。

 那么汇总相同共享调用堆栈的函数，如下:

 ![汇总相同共享调用堆栈的函数](https://developer.android.google.cn/studio/images/profile/flame_chart_aggregation-2X.png)

 汇总的函数调用用于创建 Flame Chart 图形，如下：

 ![Flame Chart](https://developer.android.google.cn/studio/images/profile/flame_chart-2X.png)

 在 Flame Chart 火焰图中 CPU 时间的被调用方首先显示。


 #### 使用 Top Down 和 Bottom Up 检查跟踪