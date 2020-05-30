---
title: Java Tips
tags:
---



* 关于transient关键字

1.原理：
一个对象只要实现了Serilizable接口，这个对象就可以被序列化，java的这种序列化模式为开发者提供了很多便利，可以不必关系具体序列化的过程，只要这个类实现了Serilizable接口，这个的所有属性和方法都会自动序列化。

但是有种情况是有些属性是不需要序列化的，所以就用到transient这个关键字。只需要实现Serilizable接口，将不需要序列化的属性前添加关键字transient，序列化对象的时候，这个属性就不会序列化到指定的目的地中。

2.添加关键字transient，就是为了不被序列化。为什么要不被序列化呢？
(1)为了数据安全，避免序列化和反序列化。在实际开发过程中，我们常常会遇到这样的问题，这个类的有些属性需要序列化，而其他属性不需要被序列化，打个比方，如果一个用户有一些敏感信息（如密码，银行卡号等），为了安全起见，不希望在网络操作（主要涉及到序列化操作，本地序列化缓存也适用）中被传输，这些信息对应的变量就可以加上transient关键字。换句话说，这个字段的生命周期仅存于调用者的内存中而不会写到磁盘里持久化。

(2)节省存储空间。类中的字段值可以根据其它字段推导出来，如一个长方形类有三个属性：长度、宽度、面积。那么在序列化的时候，面积这个属性就没必要被序列化了，因为没有多大意义。

摘要处： [Java 序列化以及transient关键字使用](https://blog.csdn.net/zz13995900221/article/details/79751595?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.nonecase)