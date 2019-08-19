---
title: 'Java 多线程(二):并发'
date: 2019-08-19 15:12:35
tags: [Java 多线程]
---



### synchronized 同步方法

多个线程同时访问 **同一个对象** 的实例变量，则很有可能发生“非线程安全”问题。

在操作实例变量的方法前添加关键字 synchronize 关键字，则可以使原本“非线程安全” 变为 “线程安全”。

在 method 方法前添加 synchronize 关键字，使得多个线程在执行该方法时，以排队的方法进行处理。当一个线程执行 method 方法前，首先判断该方法有没有被上锁，如果上锁，说明有其他线程在调用 method 方法，必须等其他线程对 method 方法调用结束后才可以继续调用 method 方法。这样实现了排队调用 method 方法的目的，达到了顺序操作实例变量的目的。

<!-- more -->

synchronized 可以在任意对象及方法上加锁，而加锁的这段代码被称为“互斥区” 或 “临界区”。

当一个线程想要执行同步方法中的代码时，线程首先尝试去获得方法上的锁，如果能够拿到这把锁，那么这个线程就可以执行 synchronized 里面的代码。如果拿不到，那么这个线程就会不断的尝试去拿这把锁，直到拿到为止，而且有可能是多个线程同时去争抢这把锁。



### synchronized 同步方法中的锁属于谁？

同步方法中关键字 synchronized 取得的锁都是 **对象锁**，哪一个线程先执行带有 synchronized 关键字的方法，哪一个线程就持有该方法 **所属对象** 的锁 Lock，那么其他对线程只能呈等待状态，前提是 **多个线程访问的是同一个对象**。

如果多个线程访问不同对象，那么 JVM 会创建多个锁。如果多个线程访问多个对象，那么执行结果显示为异步的。通过 [代码](https://github.com/leeGYPlus/JavaCode/blob/master/src/thread/SynchronizedMethod.kt) 中的 one()、two() 的对比可得到此结论，结论：**多个对象多个锁**。

那么什么时候需要对方法进行同步操作呢？
> 只有共享资源的读写访问才需要同步化，如果不是共享资源，那么根本不需要进行同步操作。

既然同步方法的中的锁为对象锁，那么很自然的，**当多个线程访问同一个对象中的多个同步方法时，同样呈现同步效果**，那么当多个线程访问同一个对象中的 **其他非同步方法** 时，呈现异步效果。




### synchronized 锁重入

关键字 synchronize 用于锁重入的功能，也就是当一个线程得到一个对象锁之后，再次请求此对象锁时是可以再次得到该对象的锁的。

具体表现：synchronize 方法/块的内部调用 **本类** 的其他 synchronize 方法/块时，是永远可以获得锁的。

**自己可以再次获取自己的内部锁**。

可重入锁也 **支持在父子继承关系** 的环境中。



### 出现异常，锁自动释放

当一个线程执行代码时出现异常，其所持有的锁会自动释放。

### 同步不支持继承性

子类继承父类中的同步方法，则子类中的该方法不具有可同步性。





----

需要明确的一点：
线程的同步性最终是体现在对象的同步方法上，即执行该方法的同步性，是线程中的一次执行动作的同步性，而不是线程的 run 方法上。比如这样一个例子：

```

class CustomThreadA extends Thread{

    private Task task;
    @Override
    public void run(){
        for(int i = 0;i < 10;i++){
           task.doSomething(); 
        }
    }
}

class CustomThreadB extends Thread{

    private Task task;
    @Override
    public void run(){
        for(int i = 0;i < 10;i++){
           task.doSomething(); 
        }
    }
}
```

这样的两个线程执行 Task 中的方法，并不是当线程 CustomThreadB 抢到 doSomething 的锁就会将 run 方法中的动作执行完毕，而是线程 CustomThreadB 只拿到了一次执行 doSomething 的锁，执行结束后，会释放锁，下一次 CustomThreadB 、CustomThreadA 会同时争抢锁，从而执行自己 run 方法中的下一次循环。

----
**知识链接：**

[Java多线程编程核心技术](http://product.dangdang.com/23711315.html)