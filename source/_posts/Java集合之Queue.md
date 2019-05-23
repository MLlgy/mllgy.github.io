---
title: Java集合之Queue
date: 2019-05-10 17:57:03
tags: [Java 集合,Queue]
---
### 概述

Queue 用于模拟 "队列" 这种数据结构，队列为 先进先出(First In First Out,FIFO) 的容器。队列可以将元素插入尾部，可以访问位于队列头部的元素，所以队列不能随机访问队列的元素。

常用的 API：
1. void add(Object obj):将指定元素加入队列的尾部。
2. boolean offer(Object obj):将指定元素加入队列的尾部。
3. Object element(): 获取队列头部的元素，但是不删除该元素。
4. Object peek():获取队列头部的元素，不删除该元素。
5. Object poll():获取队列头部的元素，并删除该元素。
6. Object remove():获取队列头部的元素，并删除该元素。
<!-- more -->

Queue 有 PriorityQueue 实现类，同时 Queue 还有一个 Deque 接口，Deque 代表一个 “双端队列”，双端队列可以在两端进行添加、删除元素，Deque 的实现类既可以当做队列来使用也可以当做栈来使用，其有两个实现类： ArrayDeque、LinkedList。



### PriorityQueue 类

PriorityQueue 保存队列元素的顺序不是按加入队列的顺序，而是按队列元素的大小进行重新排序。而其排序方式有：自然排序和定制排序。两种排序规则的实现与 TreeSet 相同，不赘述。

### Deque 接口与 ArrayDeque 实现类

Deque 为 Queue 的子接口，代表了一个 **双端队列**，可以在双端添加、删除数据的，具体操作可以查看 Deque 的 Api。

与 Queue 方法不同为以上方法可以拆分为 xxxFirst()、xxxLast() 方法，代表对队列的两端进行的处理。

我们可以把 Deque 当做 **队列** 使用，也可以当做 **栈** 来使用。


与 ArrayList 相同，它们底层都是采用 **一个动态的、可重新分配的 Objectp[] 数组** 来存储集合元素。

**把 ArrayDeque 当做 “栈”(Fitst In Last Out,FILO) 来使用**
```
    private static void test2() {
        ArrayDeque arrayDeque = new ArrayDeque();
        //将 3 个元素 push 入栈
        arrayDeque.push("one");
        arrayDeque.push("two");
        arrayDeque.push("three");
        System.out.println(arrayDeque);
        // 访问第一个元素，但不出栈
        System.out.println(arrayDeque.peek());
        System.out.println(arrayDeque);
        // 第一个元素出栈
        System.out.println(arrayDeque.pop());// 实现栈的关键
        System.out.println(arrayDeque);
    }
```

打印日志：

```
[three, two, one]
three
[three, two, one]
three
[two, one]
```

ArrayDeque 很好的实现了 “栈” -- **先入后出** 这种数据结构。在程序中使用栈时推荐使用 ArrayDeque ，避免使用 Stack(性能较差)。

**把 ArrayDeque 当做 “队列” 来使用**

当然 ArrayDeque 也可以当做队列使用，使用 **先进先出** 的方式操作集合元素。

```
    private static void test3() {
        ArrayDeque arrayDeque = new ArrayDeque();
        // 将 3 个元素加入队列
        arrayDeque.offer("one");
        arrayDeque.offer("two");
        arrayDeque.offer("three");
        System.out.println(arrayDeque);
        System.out.println(arrayDeque.peek());
        System.out.println(arrayDeque);
        System.out.println(arrayDeque.poll());//实现队列的关键
        System.out.println(arrayDeque);
    }
```
打印日志：
```
[one, two, three]
one
[one, two, three]
one
[two, three]
```

### LinkedList

LinkedList 是 List 的实现类，同时它也实现了 Deque 接口，所以 LinkedList 可以 根据 **索引** 来随机访问集合的元素，也可以被当做 **双端队列**
 来使用，由此可见 LinkedList **既可以当做队列也可以当做栈来使用**。

LinkedList 内部实现机制与 ArrayList 、 ArrayDeque 不同，后两者内部维护动态、可扩容的 Object[] 数组，**因此访问随机集合元素的性能较高**；LinkedList 内部以 **链表** 的形式来保存集合中的元素，因此随机访问集合元素的性能较差，但是在 **插入、删除集合元素性能较高(只需改变指针所指的地址)**。


### 各种线性表的性能表现

Java 中 List 是一个线性表接口，最具代表性的实现类为：ArrayList(基于数组的线性表)、LinkedList(基于链的线性表)。Queue 代表了队列，Deque 代表了双端队列。

由于数组以一块连续内存来保存数组元素，所以数组的随机访问的性能较好，那么以数组为底层实现的集合在随机访问时性能较好；而内部以链表为顶层实现的集合在添加、删除集合元素时有较好的性能。总体上 ArrayList 的性能比 LinkedList 的性能较好，因此大部分时候选用 ArrayList。

关于使用 List 几点建议：
1. 遍历 List 集合。对于 ArrayList 、ArrayDeque 等底层实现为数组集合使用随机访问来遍历元素，这样性能更好；而对于 LinkedList 则应该使用 Itertor 迭代器来遍历集合元素。
2. 经常添加、删除元素操作的集合，可考虑 LinkedList。使用 ArrayList、Vector 集合可能需要经常重新分配内部数组的大小，性能较差。
3. 多个线程访问 List 集合，可以使用 Collections 工具类对集合进行包装来实现线程安全。

