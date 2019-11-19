---
title: 使用 CPU Profiler 检查 CPU Activity 和函数跟踪
date: 2019-11-18 16:11:35
tags:
---




### 0x0001 使用 Call Chart 标签检查跟踪


Call Chart 标签提供函数跟踪的图形表示形式。

水平方法表示函数执行的时间段和时间，并 **沿垂直轴显示其被调用者**。
图形不同颜色表示不同函数级别：
* 橙色 -- 对系统 API 的调用
* 绿色 -- 对应自有函数的调用
* 蓝色 -- 对第三方 API(包括 Java 的 API) 的调用

<!-- more -->

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
![](/source/images/2019_11_18_01.png)

#### 0x0002 使用 Flame Chart 标签检查跟踪

使用 **Call Chart** 图并不能很容易的发现那些长时间运行的代码，所以需要 Flame Chart。

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

![](/source/images/2019_11_18_02.png)

因此 Flame Chart 被称为 聚合栈调用，与 Call Chart 的转换关系：

![](/source/images/2019_11_18_03.png)


Call chart 会保留时间线信息，会记录函数调用发生的确切时间，但是 Flame Chart 会显示聚合持续的时间信息，方便探索代码中长时间运行的函数，。

 #### 0x0003 使用 Top Down 检查跟踪

Flame Chart 有助于通览全局，但是有时候想要找到准确的时间信息，这时候需要 Top Down。

Top Down 根据语义显示为自上而下，映射到函数调用关系为自上而下产生调用关系。

 Top down 显示  **一个** 函数的调用列表，在该列表中展开函数节点将展示该函数调用的其他函数，即最上面的位置为一系列函数调用的起点()。

 Top Down 为 Flame Chart 图表的另一种表示方法，所以该图标的绘制原则也是汇总相同共享调用堆栈的函数。

 上面 Flame Chart 图表对应的 Top Down 图表如下：


 <img src="https://developer.android.google.cn/studio/images/profile/top_down_tree-2X.png" height="30%" width="50%">

 Top Bottom 标签提示信息可以向我们展示 **每个函数调用上所花费的 CPU 时间**（时间也可以用线程总时间占所选时间范围的持续时间的百分比表示）：


![](/source/images/2019_07_24_01.png)


|Self|Children|Total
|--|--|--
|函数调用在执行自己的代码（而非被调用方的代码）上所花的时间(对应 Call Chart 中函数 D 中相应的时间)|函数调用在执行自己的被调用方（而非自己的代码）上所花的时间(同左)|函数的 Self 和 Children 时间的总和(同左)。

举例查看具体含义

![](/source/images/2019_11_18_04.png)

很明显调用函数的最底端的函数 Self = Total，Chidren = 0。



#### 0x0004 Bottom Up 检查跟踪

Top Down 可以找到准确的时间信息，但是有时候需要花费很长时间寻找，才能寻找到某个特定的方法，这时候可以使用 Bottom Up。


Bottom Up 标签显示一个函数调用列表，在该列表中展开函数节点将显示函数的调用方。

<img src="https://developer.android.google.cn/studio/images/profile/bottom_up_tree-2X.png" height="30%" width="50%">

上面图形向我们展示了  Flame Chart 图形中函数 C 的 Bottom Up 函数调用关系图，即展示了函数 C 的所有的调用者以及延续。

举例查看具体的 Flame Chart 向 Bottom Up 的转换：


![](/source/images/2019_11_18_05.png)

 

Bottom Up 标签用于按照消耗最多（最少）CPU 时间排序函数，通过各个节点可以查看函数的哪个调用方所花费的时间

|节点属性|Self|Children|Total
|----|--|--|--
|最顶层节点|函数在执行自己的代码（非该函数所调用的函数的代码）上所花的时间，同时也是所有调用该函数(可被记录的)的函数的时间和|调用该函数|Self 和 Children 的总和
|子节点|被调用函数的 self time 总和(函数 B 的 self time 为函数 C 每个执行的 self time 总和 )|被调用的函数的总 Children time(函数 B 的 Children time 为函数 C 每个执行的 Children time 总和 )||

![](/source/images/2019_11_18_06.png)


**什么时候使用 Bottom Up**

当想要知道某个方法是如何被调用的，找到它的调用点，寻找感兴趣的节点

![](/source/images/2019_11_18_07.png)


#### 0x0005 如何选择使用哪个图表

四张图表的对比：


|Call Chart| Flame Chart|Top Down|Bottom Up|
|--|--|--|--|
|提供清晰的函数调用路径|很容易的发现那些长时间运行的代码|找到准确的时间信息|快速寻找到某个特定的方法



![](/source/images/2019_11_18_08.png)
 
----

**知识链接：**

[Google 官方文档](https://developer.android.google.cn/studio/profile/cpu-profiler#method_traces)


[Android性能优化之CPU Profiler](https://www.jianshu.com/p/a3d91986b4c7)


[读懂 Android Studio 分析工具数据](https://www.bilibili.com/video/av75349812/)