---
title: Java集合之Map
date: 2019-05-10 18:17:08
tags: [Java 集合,Map]
---

### 概述 

Map 用于保存具有 **映射关系** 的数据，因此 Map 集合内保存着两组数据，一组保存 Map 的 key，一组保存 Map 的 value，并且 key 和 value 可以是任何类型的数据。


可以把 Map 里的 Key 放在一起，它们组成一个 **Set 集合**(Key 没有顺序、不可重复)，其实Map 的内部的 keySet() 方法确实是返回的所有的 key 组成的 Set 集合对象。而把 Map 内的 Value 放在一起，它们类似一个 **List 集合**(可重复，此时的索引为 key)。

### 操作

向 Map 中添加数据中，如果 Map 中有存在 key，那么新添加的 value 会覆盖原来的 value。

<!-- more -->

### HashMap 和 Hashtable

HashMap 和 Hashtable 是 Map 的典型实现类， Hashtable 在 JDK1.0 起就已经存在。在 Java8 时改进了 HashMap 的实现，使其在 HashMap 存在 key 冲突时具有更好的性能。

Hashtable 与 HashMap 主要区别：

* Hashtable: **线程安全** 的 Map 实现，但是 HashMap 是线程不安全的实现，所以 HashMap 的性能更高一些。多个线程访问同一个 Map 对象时，使用 Hashtable 会更好。
* Hashtable 不允许使用 null 作为 key 和 value，否则会引起异常，但是 HashMap 可以。

为了更好的在 HashMap 和 Hashtable 中存储、获取对象，**用作 key 的对象必须实现 hashCode() 和 equals() 方法**。HashMap、Hashtable 判断 key 相同的标准是：**两个 key 通过 equals() 方法比较返回为 true，那么两个 key 的 hashcode 值必须相等**。而判断 value 相等的标准只要两个对象的 equals() 方法比较返回为 true 即可。

与 HashSet 相同，采用自定义类作为 HashMap、Hashtable 的 key，如果程序修改了作为 key 的可变对象，那么也会出现与 HashSet 类似的情形：程序无法准确访问到 Map 中被修改过的 key。