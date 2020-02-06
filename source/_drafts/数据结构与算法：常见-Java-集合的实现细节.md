---
title: 常见 Java 集合的实现细节
tags:
---


之前对 Java 常见集合的进行相关知识的学习：

[Java 集合概述](https://leegyplus.github.io/2019/05/09/Java%E9%9B%86%E5%90%88%E6%A6%82%E8%BF%B0/)<br>
[Java 集合之 List](https://leegyplus.github.io/2019/05/10/Java%E9%9B%86%E5%90%88%E4%B9%8BList/)<br>
[Java 集合之 Set](https://leegyplus.github.io/2019/05/09/Java-%E9%9B%86%E5%90%88%E4%B9%8B-Set/)<br>
[Java 集合之 Map](https://leegyplus.github.io/2019/05/10/Java%E9%9B%86%E5%90%88%E4%B9%8BMap/)<br>
[Java 集合之 Queue](https://leegyplus.github.io/2019/05/10/Java%E9%9B%86%E5%90%88%E4%B9%8BQueue/)<br>
[Java 集合工具类--Collections](https://leegyplus.github.io/2019/05/29/Java-%E9%9B%86%E5%90%88%E5%B7%A5%E5%85%B7%E7%B1%BB-Collections/)

此处针对 Java 集合的实现细节以及它们之间的关系进行学习，并做记录。

### Set 与 Map 的关联之处


Set 为元素无序、不可重复的集合，而 Map 代表有多个 key-value 组成的集合，它们之间有着莫大的联系，可以说 **Map 是 Set 的扩展**。

Map 如果只考虑集合中的 Key，可以发现 Key 的特征：不可重复、无序，如果将 Map 中 Key 集中起来，那么这些 Key 将组成一个 Set 集合，而 Map 也确实提供了这样一个方法来返回所有 Key 组成的 Set 集合：
```
Set<K> keySet();
```

Map 集合如果把 key-value 看做一体，可以将 Map 看做一个数组，而事实上其底层也确实是基于数组实现的。

![](/source/images/2020_02_05_01.png)

**Set 的底层实现为 Map。**



如果将 value 作为 key 的依附，那么此时 Map 集合就可以看做为 Set 集合，而事实上 Map 中 Map.Entry 其实就是一个 key-value 对，而 Map 的存储时，仅仅依据 key 来计算每个 Entry 在集合中的存储位置。


#### HashSet 和 HashMap

可以看到 HashMap 的内部实现：Key 和 Value 的对应关系通过  Map.Node 来反映，而 HashMap 的内部实现为元素为 Node 的数组，由于 Key 存在唯一的特性，和 Set 集合十分相似。

Set 的底层实现为 Map，就是借助了以上 Map 的以上特性：Key 在集合中的唯一性，而 Map 的该特性正好符合了 Set 集合元素的特性，所以 Set 可以用 Map 来实现，在 Set 中元素作为 Map key-value 中的 Key 值，完成 Set 集合的设计。

HashSet 的底层实现依赖于 HashMap，根据 HashSet 的构造函数：

```
public HashSet() {
    map = new HashMap<>();
}

// HashSet 添加元素的实现也是通过 HashMap 来实现
// 将 Set 的元素作为 Map 的 Key 进行存储
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

#### TreeMap 和 TreeSet

TreeSet 中通过 NavigableMap 的对象来保存元素，因此底层也是使用 TreeMap 来包含 TreeSet 中的所有元素。


TreeMap 集合，通过 红黑树 的排序二叉树来保存 Map  中的每个 Entry（红黑树中的节点）。


由于 TreeMap 的底层实现为红黑树，所以 TreeMap 添加元素、取出元素的性能都比 HashMap 低，因为在查找元素时，需要遍历获取插入位置、指定元素，而 HashMap 的底层实现为数组 + 红黑树，通过计算 HashCode 值获得数据的索引，从而可以直接获取元素。


但是 TreeMap 和 TreeSet 相较于 HashMap、HashSet 的优势：有序性。


#### Map 和 List


Map 集合是一个关联数组，它包含两个值：

* 一个数组是所有 key 组成的集合
  
    因为 Map 集合中的 key 不允许重复，而且 Map 不会保存 key 加入的顺序，因此这些 key 可以组成一个 Set 集合。

* 一组是所有 value 组成的集合

    Map 集合的 value 没有任何限制，可以根据 key 获取相应的 value，所以这个 value 可以组成一个 List 集合。


TreeMap 和 HashMap 可以通过 values 获取 value 集合，但是返回值并不是 List 集合，而是由其内部实现的类（Values extends AbstractCollection） ，可以认为这个集合为  List（Map 的 vaule 允许重复）。

