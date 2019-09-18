---
title: ThreadLocal(Jdk1.8) 使用及源码分析
tags:
---


### ThreadLocal

使用 ThreadLocal 对象存储的变量，只能有存储动作执行所在线程内获取。


### ThreadLocalMap

ThreadLocalMap 是一个自定义的哈希映射，仅适用于维护线程本地值。不会在 ThreadLocal 类之外导出任何操作。该类是包私有的，允许在 Thread 类中声明字段。为了帮助处理非常大且长期使用的用法，哈希表条目使用WeakReferences 作为键。但是，由于未使用引用队列，因此只有在表开始空间不足时才能保证删除过时条目。


这也是 ThreadLocal 可以存储线程相关的变量的关键，其实 ThreadLocalMap 的对象是在 Thread 中维护的。


在通过 set 存储变量时，会获得所在线程对象，接着可以获取线程的属性 ThreadLocalMap 对象，从而实现把变量存储在 ThreadLocalMap 对象中，这一步就实现了存储线程相关的变量。

在通过 get 获取变量时，会获得所在线程对象，那么就自然可以获得其属性值 -- ThreadLocalMap 对象，自然可以获得其中存储的元素。

当然这只是大致步骤，其中还是有许多细节的。


### set 方法

ThreadLocal#set
```
    public void set(T value) {
        // 步骤一：获取当前线程
        Thread t = Thread.currentThread();
        // 步骤二：获取当前线程的属性值 -- ThreadLocalMap 对象
        ThreadLocalMap map = getMap(t);
        // 步骤三：存储变量值，其中 key 为当前 ThreadLocal 对象
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

以下为步骤二的具体代码，可以看到线程维护了 ThreadLocalMap 对象，为 ThreadLocal 能够存储线程相关的变量提供可能。
```
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```



### get 方法



通过散列的 hash 索引索取不到值的话，那么就需要变量 ThreadLocalMap 中的数组进行遍历。




### 情景


* 情景一：同一个 ThreadLocal 对象，维护不同线程的变量

明白了 ThreadLocal 原理，这个问题就不难理解了，因为每一个线程都维护者自己的 ThreadLocalMap 对象，不同所存储的变量在各自 ThreadLocalMap 对象中，所以即使同一个 ThreadLocal 对象，也是可以实现在各自线程获取属于各自存储的变量。


由于存储元素时，ThreadLocalMap 是以 ThreadLocal 对象的哈希值(一系列操作的 hash)为key，所以在同一个线程中对同一个 ThreadLocal 对象进行多次 set 的调用，那么会对值进行覆盖。


* 情景二：同一线程下，使用多个 ThreadLocal 对象进行变量存储


由上面的分析值，在同一个线程下只维护 ThreadLocalMap 对象，而存储是以 ThreadLocal 对象的哈希值为 key 进行存储的，所以多个 ThreadLocal 对象获取的变量一定是自己存储的。

### 使用场景

#### 线程内单例




---
**知识链接**


[Android 开发艺术探索]()

[带你了解源码中的 ThreadLocal](https://www.jianshu.com/p/4167d7ff5ec1)