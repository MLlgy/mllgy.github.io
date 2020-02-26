---
title: 'Java 并发编程-四:Java 并发包下的并发工具类'
tags:
---


并发包也就是 java.util.concurrent 及其子包，Java 并发的工具类主要包含以下几方面：


* 并发工具类

提供了比  synchronized 更加高级的各种同步结构。

利用 CountDownLatch、CyclicBarrier、Semaphore 等，可以实现更加丰富的多线程操作。

* 各种线程安全的容器

ConcurrentHashMap、有序的 ConcurrentSkipListMap，或者通过类似快照机制，实现线程安全的动态数组 CopyOnWriteArrayList 等。

* 各种并发队列实现

各种 BlockingQueue 实现，ArrayBlockingQueue、 SynchronousQueue 或针对特定场景的 PriorityBlockingQueue 等。


* 强大的 Executor 框架

创建不同类型的线程池。




## 并发包下的线程安全的容器


![](/source/images/2020_02_12_01.png)


如果我们的应用侧重于 Map 放入或者获取的速度，而不在乎顺序，大多推荐使用 ConcurrentHashMap，反之则使用 ConcurrentSkipListMap；如果我们需要对大量数据进行非常频繁地修改，ConcurrentSkipListMap 也可能表现出优势。


关于两个 CopyOnWrite 容器，其实 CopyOnWriteArraySet 是通过包装了 CopyOnWriteArrayList 来实现的，所以在学习时，我们可以专注于理解一种。

**CopyOnWrite**


任何修改操作，如 add、set、remove，都会拷贝原数组，修改后替换原来的数组，通过这种 **防御性的方式**，实现另类的线程安全。

比如 CopyOnWriteArrayList 的 add 方法：

```
private transient volatile Object[] array;
public boolean add(E e) {
    //  使用 ReentrantLock 保证线程安全，获得锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 获得原始的 Array： array
        Object[] elements = getArray();
        int len = elements.length;
        // 拷贝原数组，容量增加一
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        // 将新数组赋值给原始数组
        setArray(newElements);
        return true;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```

这种数据结构，相对比较适合读多写少的操作，不然修改的开销还是非常明显的。




ConcurrentHashMap:


内部通过 CAS 来实现线程安全。

## 线程安全队列


### BlockQueue

Queue 额外支持的操作：

* 当队列为非空时，检索元素
* 等待队列中存在可用空间是存储元素


BlockingQueue 的方法有四种形式，处理操作的不同方式无法立即满足，但将来某个时候可能会得到满足：

* 引发异常
* 返回特殊的值，null 或 false
* 无限期的阻止线程，直到操作成功
* 在指定时间内阻塞线程

BlockingQueue 的方法总结如下表：

<table BORDER CELLPADDING=3 CELLSPACING=1>
<caption>Summary of BlockingQueue methods</caption>
 <tr>
   <td></td>
   <td ALIGN=CENTER><em>Throws exception(抛出异常)</em></td>
   <td ALIGN=CENTER><em>Special value(返回特殊值)</em></td>
   <td ALIGN=CENTER><em>Blocks(阻塞)</em></td>
   <td ALIGN=CENTER><em>Times out(超时)</em></td>
 </tr>
 <tr>
   <td><b>Insert</b></td>
   <td>add(e)</td>
   <td> offer(e)</td>
   <td>put(e)}</td>
   <td>offer(Object, long, TimeUnit)</td>
 </tr>
 <tr>
   <td><b>Remove</b></td>
   <td> remove()</td>
   <td> poll()</td>
   <td>take()</td>
   <td>poll(long, TimeUnit)</td>
 </tr>
 <tr>
   <td><b>Examine</b></td>
   <td> element()</td>
   <td>peek()</td>
   <td><em>not applicable</em></td>
   <td><em>not applicable</em></td>
 </tr>


BlockQueue 的特点：

* 不接收 null
* 有限容量
* 被设计用来表示生产者-消费者队列
* 其实现类为线程安全的






-------









