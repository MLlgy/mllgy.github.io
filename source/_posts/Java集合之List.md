---
title: Java集合之List
date: 2019-05-10 16:59:47
tags: [Java Collection,List,Java]
---

### 概述

![List、Set、Queue](/../images/2019_05_09_01.jpg)


List 集合为元素有序、可重复的集合。

List 集合判断元素相等的标准：两个对象的 equals() 方法比较返回值为 true 。

实现排序：List 的 sort() 方法需要一个 Comparator 对象来控制元素的排序。

```
public static void main(String[] args) {

        List list = new ArrayList();
        list.add(new ClassE(2));
        list.add(new ClassE(1));
        list.add(new ClassE(3));
        System.out.println(list);
        list.sort((o1, o2) ->
            ((ClassE)o1).getNum() -((ClassE)o2).getNum()
        );
        System.out.println(list);
    }
```
打印日志：

```
[ClassD{num=2}, ClassD{num=1}, ClassD{num=3}]
[ClassD{num=1}, ClassD{num=2}, ClassD{num=3}]
```
<!-- more -->

迭代遍历：

与 Set 集合只提供了 iterator() 方法不同，List 还提供了 listiterator() 方法，返回一个 ListIterator 对象， ListIterator 接口继承了 Itertor 接口，提供了专门操作 List 的方法。

ListIterator 在遍历过程中可以通过 add() 方法向上一个迭代元素后面添加一个新的元素,如果 list 对象为排过序的集合，以上操作不会触发再次排序，而 Itertor 只能实现删除元素的功能。


```
ListIterator listIterator = list.listIterator();
int a = 6;
while (listIterator.hasNext()) {
    ClassE classE = (ClassE) listIterator.next();
    if (classE.getNum() == 2) {
        listIterator.add(new ClassE(6));
    }
    System.out.println("listIterator " + classE.toString());
}
System.out.println(list);
```

打印日志为:
```
listIterator ClassE{num=1}
listIterator ClassE{num=2}
listIterator ClassE{num=3}
[ClassE{num=1}, ClassE{num=2}, ClassE{num=3}, ClassE{num=6}]
```

### ArrayList 和 Vector 类

ArrayList 和 Vector 在用法上几乎一致， Vector 在 JDK 1.0 就有了，那时候还没有 Java 集合框架的概念。在 JDK 1.2 将 Vector改为实现 List 接口，而 Vector 作为 List 的实现之一。

ArrayList 和 Vector 作为 List 的典型实现，支持 List 接口的全部功能。

ArrayList 和 Vector 都是基于数组实现的 List 类，在ArrayList 和 Vector **内部封装了一个动态的、允许再分配的 Object[] 数组**，使用 initiaCapacity 来初始化数组的长度，当数组元素超过数组的长度时，数组长度自动增加 initiaCapacity ,即长度增加 **0.5** 倍。

线程安全性：
* ArrayList 为线程不安全的，可以使用 Collections 工具类使 ArrayList 变成线程安全的。
*  Vector 时线程安全的，所以 Vector 性能比 ArrayList 低， 但是不推荐使用 Vector。


### Vector 的子类 -- Stack

Vector 有一个子类 Stack，用于模拟 “栈”  这种数据结构：

1. 出栈 
    1. peek()：返回栈中的第一个元素，但是不将该元素移除。
    2. pop():返回栈中的第一个元素，会将该元素移除
2. 进栈
    1. push(Object obj): 将元素压入栈，该元素位于集合的顶部。

**但是也应该尽量的避免使用 Stack，如果需要“栈”这种数据结构，可以使用 ArrayDeque**。

ArrayDeque 是实现了 List 接口，也实现了 Deque 接口，因此可以作为栈使用，其也是基于数组的实现。
 


### 固定长度的 List


Java 中有一个操作数组的工具类：`Arrays`，该工具类的 asList(Object ...obj) 方法 **可以把一个数组或指定个数的对象转换成一个 List 集合**，这个 List 不是 ArrayList 或 Vector 的实现类，**而是 Arrays 的内部类 ArrayList 的实例**。 Arrays$ArrayList 为一个固定大小的 List  集合，不可删除或增加集合里的元素。
```
List arrayList = Arrays.asList(new Object());
```