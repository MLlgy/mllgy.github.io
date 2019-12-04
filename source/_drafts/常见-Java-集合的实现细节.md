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


Set 的底层实现为 Map。