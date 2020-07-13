---
title: 'Java 多线程(四):静态同步synchronize方法与synchronize(class)代码块'
date: 2019-08-19 15:12:45
tags: [Java 多线程,Java]
---
### 静态同步 synchronize 方法 

关键字 synchronize 可以添加到静态方法上，这样的写法是对所属的 Class 进行加锁，从而可以实现同步效果。

虽然静态同步 synchronize 方法 和 非同步 synchronize 方法 的同步效果是一样的，但是其本质是不同的：

    静态同步 synchronize 方法为添加在 static 方法的上，是给 Class 类上锁。

    非静态同步 synchronize 方法是给对象加锁。


Class 锁可以对类的所有对象实例起作用，多线程中其所有该类的实例对象调用 **静态同步 synchronize 方法** 都是同步的，但是与非静态同步 synchronize 方法间是异步的。

<!-- more -->


[StaticSynchronziedMethodMain](https://github.com/leeGYPlus/JavaCode/blob/master/src/thread/StaticSynchronziedMethodMain.kt) 的验证方法：

    one:验证同步方法与同步静态方法间的异步性
    two:同一个对象的同步静态方法的同步性
    three:多对象间的静态同步方法的同步性

### synchronize(class)代码块


synchronize(class)代码块与静态同步 synchronize 方法的作用是一样的。

[StaticSynchronziedMethodMain](https://github.com/leeGYPlus/JavaCode/blob/master/src/thread/StaticSynchronziedMethodMain.kt) 的验证方法：

    four:验证同一个对象 synchronize(class)代码块 间的同步性
    five:验证多个对象 synchronize(class)代码块 间的同步性



----
**知识链接：**

[Java多线程编程核心技术](http://product.dangdang.com/23711315.html)