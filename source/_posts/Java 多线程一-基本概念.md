---
title: Java 多线程(一):基本概念
date: 2019-08-19 15:12:30
tags: [Java 多线程]
---

### 线程

线程可以理解为在进程中独立运行的子任务，使用多线程后可以最大限度的利用 CPU 的空闲时间来处理其他任务。

多线程的执行是异步的。

### Thread 的常用 API

currentThread()(静态方法)

> currentThread() 方法返回的 **代码段正在被哪个线程调用**。

getName()：获得线程的 Name

注意 `currentThread().getName()` 与 `getName()` 的不同。

<!-- more -->
isAlive()：判断当前线程是否处于活动状态。

sleep():在指定的毫秒内让当前“正在执行的线程” 休眠，这个 “正在执行的线程” 的是指 `currentThread()` 返回的线程。

### 实现多线程

* 继承 Thread
* 实现 Runnable


继承 Thread 最大的局限就是不支持多继承，为了实现多继承可以通过实现 Runnable 的方式，但是两者本质上没有区别。

### 线程调用

**随机性**

使用多线程技术时，**代码的运行的结果与代码的执行顺序或者调用顺序无关的**，即 Thread 对象调用 start 的顺序并不代表 run 方法的执行顺序。线程是一个子任务， CPU 以不确定的方式，或者以随机的时间调用线程中的 run 方法。



**线程的调用**

Thread 类中的 start() 方法通知 "线程规划器" 此线程已经准备就绪，等待调用线程对象的 run 方法。



线程的执行其实是让系统安排一个时间来调用 run 方法，但是系统何时执行是随机的，也就印证了线程调用的随机性。


### 实例变量与线程安全

**当多个线程可以同时访问一个变量时，容易发生数据线程安全问题。**

```
public class MyThread extends Thread {
    private int count = 5;
    public MyThread() {
        System.out.println("MyThread 当前的 Thread 的 name: " + this.getName());
        System.out.println("MyThread 代码执行的 Thread：" + Thread.currentThread().getName());
    }
    @Override
    public void run() {
        super.run();
        System.out.println("代码调用的 Thread" + currentThread().getName());

        System.out.println("当前的 Thread 的 Name  " + this.getName());
        count--;
        System.out.println("Thread is：" + currentThread().getName()+",计算结果是：" + "count is " + count);
    }
}
```

在调用过程如下(过滤无用信息)：

```
 MyThread mMyThread = new MyThread();
Thread a = new Thread(mMyThread, "A");
Thread b = new Thread(mMyThread, "B");
Thread c = new Thread(mMyThread, "C");
Thread d = new Thread(mMyThread, "D");
Thread e = new Thread(mMyThread, "E");
a.start();
b.start();
c.start();
d.start();
e.start();
```

打印日志的一种情况：

```
Thread is：A,计算结果是：count is 3
Thread is：B,计算结果是：count is 3
Thread is：C,计算结果是：count is 2
Thread is：D,计算结果是：count is 1
Thread is：E,计算结果是：count is 0
```

很明显，线程 A 和 B 同时对 count 进行处理，获得了相同的打印 count，这是不想得到的结果，这就是所谓的 “非线程安全” 问题。

在本例中，产生线程不安全的原因为: `count--`:

在虚拟机中 count-- 分为以下 3 步：

    1. 取得 count。
    2. 计算 count - 1。
    3. 对 count 赋值。

以上 3 个步骤中，如果有多个线程同时访问，容易产生线程安全问题。


### 非线程安全

什么是非线程安全？

> 非线程安全是指 **多个线程** 对 **同一个对象** 中的 **同一个实例变量** 进行操作时会出现值被更改、值不同步的情况，影响程序的正常执行。

上面这种一个线程在操作中读取实例变量时，此值已经被其他线程更改过的情况被称为 **脏读**。


----
**知识链接：**

[Java多线程编程核心技术](http://product.dangdang.com/23711315.html)