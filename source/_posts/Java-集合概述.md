---
title: Java 集合概述
date: 2019-05-09 11:06:58
tags: [Java Basic,Java Collection]
---
Java 集合在日常的开发中的使用频率是很高的，我们可以通过 Java
 集合来实现常见的数据结构，如堆、栈等。
 
 
### Java 集合的分类
 
 Java 集合分为 List、Set、Map、Queue 四种体系，以下为各个体系的特点：
 
 * List：有序、可重复的集合。
 * Set：无序、不可重复的集合。
 * Map：具有映射关系的集合。
 * Queue：Java5 后增加了 Queue 体系，代表队列的一种实现。
 
谈到集合就不得不提在 Java5 增加的泛型，但是不在这里进行阐述。


Java 集合可用来 **存储对象**，因此被称为 **容器类**。与 数组既可以存储基本数据类型也可以存储对象不同，集合只能存储对象。

<!-- more -->

### Java 集合继承树

Java 集合主要是 Collection 和 Map 的派生类，集合的继承关系如下:

![List、Set、Queue](/../images/2019_05_09_01.jpg)

![Map](/../images/2019_05_09_02.jpg)



### Collection 和 Iterator 接口

List、Set、Queue 都是继承了 Collection 接口，那么该接口定义的方法都可以操作以上集合。

集合本就是容器，那么相应的操作：增加、删除、替换、清空等，也就对应了相应的方法。

Iterator 与 Collection 、Map 不同，Collection 、Map 主要的功能是为了盛放其他对象，而 **Iterator 的主要用于遍历 Collection 集合中的元素**，所以 Iterator 对象也被称为 **迭代器**。

Itertor 仅用于遍历元素，其本身不能盛放对象的能力，Itertor 对象的创建必须依托于 Collection，没有 Collection 的 Itertor 没有任何价值。

为了更好的理解 Itertor 看以下代码：

```
public static void main(String[] args) {
    List<String> list = new ArrayList();
    list.add("a");
    list.add("b");
    list.add("c");
    System.out.println(list);
    Iterator iterator = list.iterator();
    while (iterator.hasNext()) {
        String str = (String) iterator.next();
        System.out.println(str);
        if (str.equals("c")){
            iterator.remove();//2
            // 使用 Itertor 遍历集合时，不可修改集合元素，以下代码引发异常
            //list.remove(str);//3
        }
        str = "Chage";//1
    }
    System.out.println(list);
}
```

其打印日志如下：

```
[a, b, c]
a
b
c
[a, b]
```

在这里我们看到了一个很有意思的现象，在 `代码1` 处对迭代变量 str 赋了新值，但是通过打印日志我们看到 list 的元素并没有被改变。

**这是因为 Itertor 在对集合进行遍历时，Itertor 并不是把集合元素本身传给了迭代变量 str，仅仅是把集合元素的值传给 迭代变量 str，所以修改迭代变量 str 的值不会对集合元素有任何影响。**

当通过 `Itertor` 迭代遍历 `Collection` 中的集合元素时，Collection 集合中的元素不能被改变，**只有通过 Itertor 的 remove() 方法删除上一次 next() 方法返回的值，否则就会引起 java.util.ConcurrentModificationException 异常**，将 `代码2` 处替换为 `代码3` 就会报出异常。


### 使用 foreach 遍历集合元素

同 Itertor 一样，通过 foreach 遍历元素过程中也不能修改集合元素，系统同样也是把集合的元素的值赋值个迭代变量：


```
for (String str : list) {
    String value = str;
    System.out.println(value);
    if (str.equals("c")) {
        // 下面的代码同样也会引起 ConcurrentModificationException 异常
        list.remove(value);
    }
}
```