---
title: ThreadLocal(Jdk1.8) 使用及源码分析
date: 2019-09-19 14:30:16
tags: [Java Basic,Java 多线程]
---


### 0x0001 ThreadLocal 简介

使用 ThreadLocal 对象存储 **线程相关** 的变量，只能有存储动作执行所在线程内才能够获取相应变量。


### 0x0002 ThreadLocal 中相关类简介

#### 1.ThreadLocal#ThreadLocalMap

ThreadLocalMap 是 ThreadLocal 中一个自定义的哈希映射，仅适用于维护线程本地值。不会在 ThreadLocal 类之外导出任何操作。该类是包私有的，允许在 Thread 类中声明字段。为了帮助处理非常大且长期使用的用法，哈希表条目使用WeakReferences 作为键。但是，由于未使用引用队列，因此只有在表开始空间不足时才能保证删除过时条目。
<!-- more -->

虽然从 ThreadLocalMap 类名来看它是一个 Map 类型的数据，但是它并不是一个 Map，它内部维护的是一个初始长度为 16 的数组，而该数组的元素 Entry 更像是一个维护 key-value 的实体类，可以理解为一个 Map。


#### 2. ThreadLocalMap 中的 Entry

```
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
在类中以下是对 Entry 的介绍：

> The entries in this hash map extend WeakReference, using
its main ref field as the key (which is always a
ThreadLocal object).  Note that null keys (i.e. entry.get()
== null) mean that the key is no longer referenced, so the
entry can be expunged from table.  Such entries are referred to
as "stale entries" in the code that follows.

> Entry 继承了 WeakReference，使用 Entry 对象的引用作为 ThreadLocalMap 存储元素的 key。当通过 entry.get() 获得为 null 时，说明不再引用该 key，因此可以从表中删除该条目。


### 0x0003 ThreadLocal 实现的关键

这也是 ThreadLocal 可以存储线程相关的变量的关键，这是 **因为 ThreadLocalMap 的对象是在 Thread 中维护的**。


在通过 set 存储变量时：
1. 首先会获得所在线程对象
2. 接着可以获取线程的属性 ThreadLocalMap 对象
3. 从而实现以 ThreadLocal 为 key 的变量存储在 ThreadLocalMap 对象中，这一步就实现了存储线程相关的变量。

在通过 get 获取变量时：
1. 首先获得所在线程对象，
2. 接着那么就自然可以获得其属性值 -- ThreadLocalMap 对象，
3. 以当前 ThreadLocal 对象为 key 自然可以获得其中存储的变量。

当然这只是大致步骤，其中还是有许多细节的。

### 0x0004 ThreadLocal#set 方法

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
ThreadLocalMap#set
```
private void set(ThreadLocal<?> key, Object value) {
    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // 更新 key 对应的 value
        if (k == key) {
            e.value = value;
            return;
        }
        if (k == null) {
            // 经过一系列算法操作，添加 value
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
将 ThreadLocal 对象 key 经过一系列的算法，获得该对象的对应的 value 在数组中的索引值，然后对数组中该索引值下元素进行操作。


数组的操作：

* 更新：如果指定索引的位置存在元素，那么就对该位置元素进行更新。
* 添加：如果指定索引的位置不存在元素，那么就将 value 添加到该位置。 

**总结一下：**

现在我们可以看到的关系是：一个 Thread 对应一个 ThreadLocalMap， 在 ThreadLocalMap 内部维护 Entry 数组，这个数组的索引由 ThreadLocal 对象经过一系列计算得到：

```
int i = key.threadLocalHashCode & (len-1);
```
在通过 ThreadLocal 对象 set 值时，其实是通过一系列的算法，用来初始化、添加或者更新数组中指定索引的元素。


### 0x0005 ThreadLocal#get 方法

ThreadLocal#get
```
// 返回当前线程相关的 threadlocal 中变量，如果变量为 null，则返回 setInitialValue() 线程相关的初始值(null)。
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {  
        // 很明显，通过 key 获取 Map 中的 value
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            // 返回相应的 value
            return result;
        }
    }
    // 线程相关的变量，返回初始化值
    return setInitialValue();
}
```
* 如果获取到的 ThreadLocalMap 对象 map 为 null ，会进行如下操作：

ThreadLocal#setInitialValue
```
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

```
protected T initialValue() {
    return null;
}
```
* 如果获取到的 ThreadLocalMap 对象 map 为 null ，会进行如下步骤，获取对应的 ThreadLocalMap.Entry 对象：

ThreadLocalMap#getEntry
```
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    // 
    if (e != null && e.get() == key)
        return e;
    else
        // 通过散列的 hash 索引获取不到值的话，那么就需要变量 ThreadLocalMap 中的数组进行遍历。
        return getEntryAfterMiss(key, i, e);
}
```
ThreadLocalMap#getEntryAfterMiss

```
// 通过遍历 table 中元素，寻找对对应的 Entry 对象
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i,  Entrye) {
    Entry[] tab = table;
    int len = tab.length;
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

通过 Entry 对象就可以获得具体的变量值。

### 0x0006 多线程与同一个 ThreadLocal 对象

对 ThreadLocal 的原理有一定的理解，那么以下两个场景理解就十分容易了。


* 情景一：同一个 ThreadLocal 对象，维护不同线程的变量

明白了 ThreadLocal 原理，这个问题就不难理解了，因为每一个线程都维护者自己的 ThreadLocalMap 对象，不同所存储的变量在各自 ThreadLocalMap 对象中，所以即使同一个 ThreadLocal 对象，在不同线程中会多次存储，所以可以实现在各自线程获取属于各自存储的变量。


在存储元素时，ThreadLocalMap 内部维护一个数组，以 ThreadLocal 对象的哈希值(一系列操作的 hash)经过一系列算法后得出的 index 索引，将 value 存储在数组中的索引处，所以在同一个线程中对同一个 ThreadLocal 对象进行多次 set 的调用，那么会对值进行覆盖。


* 情景二：同一线程下，使用多个 ThreadLocal 对象进行变量存储


同样根据对象计算的索引值是唯一的，所以多个 ThreadLocal 对象获取的变量一定是自己存储的。

### 0x0007 使用场景

#### 1. 线程内单例

我们平时使用到的单例为进程为单例，而通过 ThreadLocal 可实现线程内同步，具体代码如下：


```
public class User {
    private static ThreadLocal<User> threadLocal = new ThreadLocal<>();

    private User() {
    }

    private static User getInstance() {
        User instance = threadLocal.get();
        if (instance == null) {
            instance = new User();
            threadLocal.set(instance);
        }
        return instance;
    }
}
```


#### 2. 变量的作用域为线程

当一些数据是以线程为作用域，并且不同线程拥有数据的不同副本的时候，就可以考虑使用 ThreadLocal。



在子线程中初始化 Handler 需要手动的创建 Looper，因为 Looper 是线程相关的，那么 Looper 是怎样实现线程相关的呢？本质就是使用了 ThreadLocal。

```
Handler mHandler;
new Thread(new Runnable() {
    @Override
    public void run() {
        Looper.prepare();//Looper初始化
        //Handler初始化 需要注意, Handler初始化传入Looper对象是子线程中缓存的Looper对象
        mHandler = new Handler(Looper.myLooper());
        Looper.loop();//死循环
    }
}).start();
```
具体看一下源码：

```
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
public static void prepare() {
    prepare(true);
}
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

可以看到如果不执行 `Looper.prepare()` ，则 `Looper.myLooper()` 就无法获取到线程相关的 Looper 实例对象。

当然 Android 给了更为简单的实现方式： HandlerThread，但是本质还是 ThreadLocal。

```
HandlerThread handlerThread = new HandlerThread("HandlerThread");
handlerThread.start();
Handler mHandler = new Handler(handlerThread.getLooper()){
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        Log.d("Log","current thread is "+Thread.currentThread().getName());
    }
};
mHandler.sendEmptyMessage(1);
```

#### 3. 复杂逻辑下的对象传递


有时候一个线程的逻辑过于复杂，导致函数的调用栈比较深，而这时候我们需要监视器能够贯穿整个线程的执行过程，这是就可以使用 ThreadLocal 。使用 ThreadLocal 存储监视器，这样就可以在线程中获得监视器对象。

其他能够想到的两种方式：
1. 将监视器对象通过参数的方式传递
    
   这种方式当调用栈过深时，会让整个逻辑更加复杂、难懂。

2. 将监视器作为静态变量供线程访问 

    这种方式是可以接受的，但是这种方式是不具有扩充性，如果有两个线程在执行，那么就需要提供两个静态的监听对象。如果是更多的线程呢？这无疑是代码中的”坏味道“。

而使用 ThreadLocal 则完全不会遇到上面问题。




---
**知识链接**


[Android 开发艺术探索](https://item.jd.com/11760209.html)

[带你了解源码中的 ThreadLocal](https://www.jianshu.com/p/4167d7ff5ec1)

[ThreadLocal类及应用技巧](https://www.bilibili.com/video/av7592261?from=search&seid=7861472405873308618) : 视频建议 2 倍速看完，没什么营养，但可以向你展示如何使用 ThreadLocal 