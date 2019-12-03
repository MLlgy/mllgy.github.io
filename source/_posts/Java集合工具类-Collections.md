---
title: Java 集合工具类--Collections
date: 2019-05-29 17:17:24
tags: [Java Collection,Collections,Java Basic]
---

**HashSet、TreeSet、ArrayList、ArrayDeque、LinkedList、HashMap 和 TreeMap 都是线程不安全的**，
如果多个线程对用一个集合对象进行存、取、删等操作，不免会产生 **线程同步问题**。

Java 提供了 Collections 工具类，使用 synchronizedxxx()  方法可以将集合类包装成线程安全的集合。

```
Collections collections = Collections.synchronizedCollection(new ArrayList());

List list = Collections.synchronizedList(new ArrayList());

Set set = Collections.synchronizedSet(new HashSet());

Map map = Collections.synchronizedMap(new HashMap());
```
同时 Collections 还提供了排序、查找、替换、设置不可变集合等功能，有空自己看 API。