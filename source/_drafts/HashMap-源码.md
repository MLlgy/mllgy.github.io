---
title: HashMap 源码
tags:
---



### JDK7 中 HashMap 的实现原理
HashMap 在 jdk7底层实现：数组 + 链表。


以下示例说明：
```
HashMap<String, String> hashMap = new HashMap<String, String>();
hashMap.put("你", "你");
hashMap.put("我", "我");
hashMap.put("他", "他");
hashMap.put("它", "它");

```

### put 的实现原理

具体源代码：

```
createEntry()
```

根据源码我们可以理解其实现原理：

通过 Key 的 hash 中计算出对应 vulue 在数组中的位置，标记为 index，如果存在不同 Key 计算出的 index 存在相同的情况，此时就需要链表来实现存储的功能了。遍历对应链表，如果链表中已经存在该 Key 存储的元素，那么更新该元素，否则将新的 value 存储在对应链的头节点，并且将数组该索引处的元素指向链表的头节点。至于最后一步为什么要移动链表，这是因为该链表为单链表，无法实现从表尾到表头的遍历，这就是 HashMap 的 put 实现原理。

我们可以大致模拟出 HashMap 的实现原理：

```
for (String str :hashMap.keySet()) {
    int hashcode = str.hashCode();
    int index = hashcode % 3;
    System.out.println(String.format("key is %s,index is %d", str, index));
}
```
通过 Key 的 hash 中计算出对应 vulue 在数组中的位置，但是计算出的 index 存在重复情况，以上示例的打印结果：
```
key is 你,index is 1
key is 我,index is 1
key is 他,index is 1
key is 它,index is 0
```
可以看到不同的 key 计算出了相同的 index，此时就需要链表来实现存储的功能了。HashMap 在 jdk7 的实现中，重复的 index 对应的 value 会被添加到对应链表的头节点处，此时示意图如下：

![](/source/images/2019_11_24_01.png)

移动链表：

![](/source/images/2019_11_24_02.png)

如果接下来的操作计算出的 index 仍旧为 1，那么则会出现以下操作：

将新元素添加到链表中：

![](/source/images/2019_11_24_03.png)

移动链表：

![](/source/images/2019_11_24_04.png)


### get 


明白了 put 的实现原理，那么 get 的实现就可以很简单的理解了，在执行 HashMap 的get 函数时，会根据 Key 的值计算出 index，获取数组中指定索引的元素，如果该元素存在 next 节点，说明在该索引处存储的链表，遍历链表，根据 Key 的 hash 值获取指定的 value 值，不再赘述，查看源代码即可。



**其他点：**
扩容
modcount

### 关于 HashMap 中针对 hash 的算法的几点


为什么要对 Key 的 hash 值进行一系列的算法操作？

hash 值重要的一个属性：散列性，一个好的 hash 算法算出的值是均匀分布的，如果一个 hash 算法算出的值均为相同的值，那么我们说这是一个不好的算法。


而对 Key 的 hash 值进行一系列后续的操作的原因，就是要保证其散列性。


**hash & (length -1)**

假设数组的长度为 16，对两者进行与操作：

```
hash            0101 0110
length-1(15)    0000 1111
```
可以看到 `hash & (length -1)` 取决于 hash 值的后四位，得出结果的范围为 0~15，可以完整的覆盖数组的索引。

这也解释了在计算数组的长度时，为什么将 length 取为 2 的 n 次方。


**为什么对 hash 值进行移位**

```
hash            0101 0110
hash            1101 0110
hash            0111 0110
```

根据以上分析，hash 值的高四位其实是不生效的，这时 3 者得到的计算结果完全相同，这样的散列性是很差的，而对 hash 进行移位的原因是使 hash 值的高位可以参与运算，那么相对的其散列性将会大幅度改善。


**关于扩容**

两个条件：

1. 数组中的元素数量大于阈值
2. 指定索引处的元素不为 null

基于第二个条件，数组扩容时一定会产生链表结点 。


### 在扩容时死锁问题

https://www.cnblogs.com/ldbangel/p/9883518.html

基于此，可以在初始化 HashMap 时指定其容量，这样就不会进行扩容。

### JDK8 中 HashMap 的实现原理


**jdk8 与 jdk7 中的区别：**
* jdk8 中将链表转变为红黑树
* 新节点插入链表的顺序不同，jdk7 中插入头节点，而 jdk8 为插入尾节点。
* hash 算法的优化
* resize 的修改(jdk7 中会出现死锁现象，而 jdk8 中不会)。


### 为什么使用红黑树

红黑树与链表：红黑树插入效率比链表低，查找效率比链表高。
红黑树与完全平衡二叉树：红黑树插入效率比完全平衡二叉树高，查找效率比完全平衡二叉树低。


HashMap 基于插入和查询操作的综合考虑，使用红黑树。

基于 jdk7 中的链表，当链表比较长时，其查询效率比较低，所以使用红黑树。

**何时更换为红黑树**？

在执行 put 操作时，当链表的结点数达到为 `TREEIFY_THRESHOLD(8)` 时，将链表更换为 红黑树,以提高查询效率。

在执行 remove 操作时，当链表的结点数减少到 `UNTREEIFY_THRESHOLD(6)`，会将红黑树更换为链表，以提高插入效率。

### put


1. 数组为 null 时，进行初始化数组

```
if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
```
2. 数组元素为空时

将 value 添加到数组指定位置。

3. 指定索引处的元素不为空

此时分为三种情况：

a. 当 key 相同时，更新值

b. 当数组中对应的元素类型为 TreeNode，则将 vaule 添加的红黑树中，至于如何添加的红黑树中的，不做过多阐述，有时间自己再探究。

c. 当数组中对应的元素类型为 Node，则将 vaule 添加的链表中，此处有一个需要注意的点：将数据添加到链表的尾部，这与 jdk7 添加到头部的做法是不同的，至于为什么要这样做，因为此处需要对链表进行遍历，那么直接将新结点添加到尾部是十分方便的。 若链表的结点数大于 8 ，则将链表更改为红黑树。

### 扩容

jdk7 中扩容时，会将链表的顺序倒置，而 jdk8 中扩容后的链表顺序和原链表的顺序一致。

jdk7 在扩容时会出现死锁问题

### 关于 HashMap 的 Key 为对象的问题

如果 HashMap 的 Key 为对象，那么这个对应所属的类必须重新 hashCode() 方法，因为在 HashMap 的实现过程中，需要使用对象的 hashCode() 判断 key 是否相等。

### 关于 JDK7 中 HashMap 的中问题

在链表中传入数据的方式是 jdk7 死锁的产生的原因，在 jdk7 中将新结点插入链表插入头节点，在扩容时会将链表的顺序导致，这就导致多线程执行扩容操作时会产生死循环。而 jdk8 中扩展是不会将链表导致，所以不会产生死循环的问题。



----

**知识链接：**

[老生常谈，HashMap的死循环](https://www.jianshu.com/p/1e9cf0ac07f4)

[HashMap扩容死循环问题](https://blog.csdn.net/Leon_cx/article/details/81911223)











 

