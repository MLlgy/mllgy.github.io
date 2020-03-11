---
title: 'Java 并发编程(二):synchronized 的底层实现'
tags:
---


## 基本了解


synchronized 是 Java 内建同步机制，它提供了互斥的语义和可见性，**当一个线程已经获得当前锁时，其他视图获取锁的线程只能等待或者阻塞在那里**。

* synchronized 基本实现
synchronized 代码块是由一对儿 monitorenter/monitorexit 指令实现的，Monitor 对象是同步的基本实现单元。


在 JDK 6 之前， Monitor 完全有操作系统内部的互斥锁，因为需要进行用户态到内核态的切换，所以同步操作是一个无差别的重量级操作。

后 JDK 进行重构，提供了三种不同的 Monitor 的实现，就是常说的三种不同的锁：偏斜锁（Biased Locking）、轻量级锁和重量级锁，大大改进了其性能。


* 锁的升级、降级

锁的升级、降级，就是 JVM 优化 synchronized 运行的机制，当 JVM 检测到不同的竞争状况时，会自动切换到适合的锁实现，这种切换就是锁的升级、降级。


当没有竞争出现时，默认会使用偏斜锁，JVM 会利用 CAS 操作（compare and swap），在对象头上的 Mark Word 部分设置线程 ID。

如果有另外的线程试图锁定某个已经被偏斜过的对象，JVM 就需要撤销（revoke）偏斜锁，并切换到轻量级锁实现。

轻量级锁依赖 CAS 操作 Mark Word 来试图获取锁，如果重试成功，就使用普通的轻量级锁；否则，进一步升级为重量级锁。



## synchronized 的底层实现

Java 并发编程艺术 -- 2.2.2 锁的升级与对比


## Java 核心类库中其他一些特别的锁类型

* Reentrant#ReadWriteLock(读写锁)

ReentrantLock 和 synchronized 简单实用，但是行为上有一定局限性，通俗点说就是“太霸道”，要么不占，要么独占。实际应用场景中，有的时候不需要大量竞争的写操作，而是以并发读取为主。


Java 并发包提供的 **读写锁** 等扩展了锁的能力，它所基于的原理是多个读操作是不需要互斥的，因为读操作并不会更改数据，所以不存在互相干扰。而写操作则会导致并发一致性的问题，所以写线程之间、读写线程之间，需要精心设计的互斥逻辑。

基于读写锁实现的数据结构，当数据量较大，并发读多、并发写少的时候，能够比纯同步版本凸显出优势:
```

public class RWSample {
  private final Map<String, String> m = new TreeMap<>();
  private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
  // 读
  private final Lock r = rwl.readLock();
  private final Lock w = rwl.writeLock();
  public String get(String key) {
      r.lock();
      System.out.println("读锁锁定！");
      try {
          return m.get(key);
      } finally {
          r.unlock();
      }
  }

  public String put(String key, String entry) {
      w.lock();
  System.out.println("写锁锁定！");
        try {
            return m.put(key, entry);
        } finally {
            w.unlock();
        }
    }
  // …
  }

```

在运行过程中，如果读锁试图锁定时，写锁是被某个线程持有，读锁将无法获得，而只好等待对方操作结束，这样就可以自动保证不会读取到有争议的数据。

读写锁的粒度较于 synchronized 更细，但是其将相对开销还是较大，因此引入了 StampedLock。


* StampedLock

提供类似读写锁的同时，还支持优化读模式。


优化读基于假设，大多数情况下读操作并不会和写操作冲突，其逻辑是先试着读，然后通过 validate 方法确认是否进入了写模式，如果没有进入，就成功避免了开销；如果进入，则尝试获取读锁。


```
public class StampedSample {
  private final StampedLock sl = new StampedLock();

  void mutate() {
      // 进入写模式
      long stamp = sl.writeLock();
      try {
          write();
      } finally {
          // 退出写模式
          sl.unlockWrite(stamp);
      }
  }

  Data access() {
      // 进入读模式
      long stamp = sl.tryOptimisticRead();
      Data data = read();
      if (!sl.validate(stamp)) {
          stamp = sl.readLock();
          try {
              data = read();
          } finally {
              // 退出读模式
              sl.unlockRead(stamp);
          }
      }
      return data;
  }
}
这里的 writeLock 和 unLockWrite 一定要保证成对调用。

```

----


Java 并发编程艺术 -- 2.2.2 锁的升级与对比