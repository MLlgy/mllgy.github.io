---
title: 'Java 并发编程(一):synchronized和ReentrantLock有什么区别'
tags:
---





Java 5 以前，synchronized 是仅有的同步手段，ReentrantLock，翻译为再入锁，是 Java 5 提供的锁实现，它的语义和 synchronized 基本相同。


## 线程安全


线程安全是 一个多线程环境下正确性的概念，也就是保证多线程环境下共享的、可修改的状态的正确性，这里的状态反映在程序中其实可以看作是数据。

根据线程安全的定义推断，保证线程安全的两个方法：

* 封装

通过封装将对象内存隐藏起来。

* 不可变


线程安全需要保证以下几个特征：

* 原子性


相关操作不会中途被其他线程干扰，一般通过同步机制实现。

* 可见性


一个线程修改了某个共享变量，其状态能够立即被其他线程知晓，通常被解释为将线程本地状态反映到主内存上，volatile 就是负责保证可见性的。

* 有序性


保证线程内串行语义，避免指令重排等。

## ReentrantLock

再入锁，具体再入的含义为：当一个线程试图获取一个它已经获取的锁时，这个获取动作就自动成功。



ReentrantLock 的使用：

```
// 对再入锁设置公平性，这里是演示创建公平锁，一般情况不需要。
ReentrantLock fairLock = new ReentrantLock(true);
fairLock.lock();
try {
  // do something
} finally {
   fairLock.unlock();
}
```

**公平性** 是指在竞争场景中，当公平性为真时，会倾向于将锁赋予等待时间最久的线程。公平性是减少线程“饥饿”（个别线程长期等待锁，但始终无法获取）情况发生的一个办法。

为保证锁释放，每一个 lock() 动作，我建议都立即对应一个 try-catch-finally。

## Condition：条件变量

关键字synchronized与wait（）和notify（）/notifyAll（）方法相结合可以实现等待/通知模式，类 ReentrantLock 也可以实现同样的功能，但需要借助于Condition对象。Condition 将 wait、notify、notifyAll 等操作转化为相应的对象，将复杂而晦涩的同步操作转变为直观可控的对象行为


ArrayBlockingQueue 中对 Condition 的使用：

1. 通过再入锁获取条件变量


```

/** Condition for waiting takes */
private final Condition notEmpty;

/** Condition for waiting puts */
private final Condition notFull;
 
public ArrayBlockingQueue(int capacity, boolean fair) {
  if (capacity <= 0)
      throw new IllegalArgumentException();
  this.items = new Object[capacity];
  lock = new ReentrantLock(fair);
  notEmpty = lock.newCondition();
  notFull =  lock.newCondition();
}
```
两个条件变量是从同一再入锁创建出来，然后使用在特定操作中，如下面的 take 方法，判断和等待条件满足:

```
public E take() throws InterruptedException {
  final ReentrantLock lock = this.lock;
  lock.lockInterruptibly();
  try {
      while (count == 0)
          notEmpty.await();
      return dequeue();
  } finally {
      lock.unlock();
  }
}
```

当队列为空时，试图 take 的线程的正确行为应该是等待入队发生，而不是直接返回，这是 BlockingQueue 的语义，使用条件 notEmpty 就可以优雅地实现这一逻辑。


保证入队触发后续 take 操作呢？请看 enqueue 实现:


```

private void enqueue(E e) {
  // assert lock.isHeldByCurrentThread();
  // assert lock.getHoldCount() == 1;
  // assert items[putIndex] == null;
  final Object[] items = this.items;
  items[putIndex] = e;
  if (++putIndex == items.length) putIndex = 0;
  count++;
  notEmpty.signal(); // 通知等待的线程，非空条件已经满足
}
```

通过 signal/await 的组合，完成了条件判断和通知等待线程，非常顺畅就完成了状态流转。注意，signal 和 await 成对调用非常重要，不然假设只有 await 动作，线程会一直等待直到被打断（interrupt）.
--- 


极客时间

