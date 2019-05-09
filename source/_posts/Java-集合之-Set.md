---
title: 'Java 集合之 Set 集合 '
date: 2019-05-09 15:49:35
tags: [Java 集合,Set]
---

### Set 集合

Set 集合的特点无序、不可重复。

Set 集合不能记住元素的添加顺序。Set 集合不能包含相同的元素，把两个相同的元素添加到同一个 Set 集合中，则添加失败，add() 方法返回 false，且新元素不会被添加。

### HashSet 类

HashSet 集合按 `Hash 算法` 来存储集合的元素，**因此具有很好的存取和查找功能**。

HashSet 具有以下特点：
1. 不能保证元素的排列顺序，这也是 Set 集合元素不能通过索引只能通过元素本身访问的原因。
2. HashSet 不是线程安全的。
3. 元素可以为 null。


### 如何存储元素

当向 HashSet 集合存入一个元素时， HashSet 会调用该对象的 `hashcode()` 方法来得到对象的 `hashcode` 值，**然后根据该 hashcode 值决定对象在 HashSet 中的存储位置**。如果两个元素通过 equals() 方法比较返回 true，但是它们的 hashcode() 方法返回值不相等，那么 HashSet 会把它们存储在不同的位置，可以添加成功。

### HashSet 如果定义元素相等

HashSet 集合判断两个元素相等的标准是：

1. 两个对象通过 equals() 方法比较相等。
2. 两个对象的 hashcode() 方法的返回值也相等。


**两者缺一不可**。


所以当把一个对象放入 HashSet 中时，如果需要重写对象的 equals() 方法，那么也应该重写 hashcode() 方法。**重写两方法的规则是：如果两个对象的 equals() 方法返回 true，这两个对象的 hashcode 也应该相同。**

两个对象的 equals() 方法返回 true，hashcode 返回不同的 hashcode 值，HashSet 会把两个对象保存在 Hash 表的不同位置，两个元素都会添加成功，但是这与 Set 集合的规则冲(元素不可重复)突了。如果是 equals() 方法返回 false，但是 hashcode 值相同，也会导致一些问题：hashcode 相同，HashSet 会尝试将多个元素保存在同一个位置，但又不行(这样的结果就是只剩下一个对象)，所以这样的结果就是在这个位置上通过链式结构来保存多个对象。HashSet 的特点就是通过元素的 hashcode 值来快速定位，而现在 HashSet 对象中有两个以上的元素具有相同的hashcode 值，导致性能下降。


### hash 算法

哈希算法的功能是，他能快速的查找被检索的对象，**hash 算法的价值在于速度**。当查询集合中的某个元素时，hash 算法可以直接根据该元素的 hashcode 值计算出该元素的存储位置，从而快速定位该元素。实现快速定位的原因是 HashSet 通过 hashcode 的值进行存储元素。

#### hashCode() 方法

由上面可知 hashCode() 方法对于 HashSet 的重要性，那么针对 hashCode() 方法的重写有如下规则：
1. 同一个对象多次调用 hashCode 方法应该返回相同的值。
2. 两个对象通过 equals() 方法比较返回 true 时，这两个对象的 hashCode() 方法应返回相同的值。
3. 对象中用于比较 equals() 方法比较标准的实例变量都应该用于计算 hashCode 值。
4. 程序把可变对象添加到 HashSet 中之后，尽量不要去修改集合元素中参与计算 hashCode()、equals() 的实例变量，否则会导致 HashSet 无法正确的操作这些集合元素。


### LinkedHashSet 类

LinkedHashSet 为 HashSet 的一个子类，LinkedHashSet 也是通过hashcode 的值来决定元素的存储位置，**但同时使用链表维护元素的次序**，使得元素看起来是以插入顺序保存的。因为需要维护元素的插入顺序，所以性能略低于 HashSet。但是在迭代访问 Set 集合中的全部元素时有很好的性能，因为它以链表维护内部顺序。

遍历 LinkedHashSet 集合元素时，元素的顺序与添加顺序一致。

### TreeSet 类

TreeSet 为 SortSet 接口的实现类，其可以确保集合元素处于 **排序状态**。TreeSet 不是根据添加顺序进行排序而是 **根据元素的实际值进行排序**。

与 HashSet 根据 hash 算法来决定元素的存储位置不同，TreeSet 采用 **红黑树** 的数据结构来存储集合元素。TreeSet 支持两种排序算法：自然排序(默认)和定制排序。

### TreeSet 自然排序

如果将一个对象添加到 TreeSet 中，那么 **这个对象必须实现 Comparable 接口**，依次来实现排序功能，否则程序会报出异常。

TreeSet 会调用集合元素的 compareTo(Object obj) 方法来比较元素之间的大小关系，然后按升序排列，这就是 TreeSet 的自然排序。


**元素如何存储到集合中**

当把一个元素添加到 TreeSet 集合中， TreeSet 会调用这个对象的 compareTo(Object obj) 方法与集合中的其他元素比较大小，**然后根据红黑树结构找到它的存储位置**。如果两个对象通过 compareTo(Object obj) 方法比较相等，那么新的对象将不会添加到集合中。

如果想要 TreeSet 能够正常运行，那么 TreeSet 只能添加同一种类型的对象。

**判断集合元素相同**

**TreeSet 集合元素相等的唯一标准为：两个对象的 compareTo(Object obj) 方法是否返回 0，返回 0 则认为两个对象相等，反之不相等**。基于以上标准，在重写 equals() 方法时，其返回值需要与 compareTo(Object obj) 的返回结果一致，否则 equals() 方法的返回值没有实际意义。

通过重写 compareTo(Object obj) 方法，使其不返回 0，此时可以实现将一个对象多次添加到 TreeSet集合中，但是此时集合多个元素均指向一个对象引用，如果对象发生变化，那么集合中的多个元素会同时发生相同变化。

基于以上描述，在重写对象类的 equals() 方法时，应保证该方法与 compareTo(Object obj) 方法有一致的结果，如果equals() 方法返回 true ，那么这两个对象通过 compareTo(Object obj) 方法应该返回 0。

改变Treeset 集合中的可变元素的实例变量后，这会导致它与其他对象的大小顺序发生改变，但是 TreeSet 不会再次调整它们的顺序。当尝试删除该对象时，TreeSet 也会删除失败。所以基于此原因 **TreeSet 可以删除没有被修改实例变量、且不与其他被修改的实例变量的对象重复的对象。**

```
TreeSet treeSet = new TreeSet();
Dog dog1 = new Dog("mike",1);
Dog dog2 = new Dog("mike",2);
Dog dog3 = new Dog("mike",3);
Dog dog4 = new Dog("mike",4);
treeSet.add(dog1);
treeSet.add(dog2);
treeSet.add(dog3);
treeSet.add(dog4);
System.out.println("1 " + treeSet);
treeSet.remove(dog1);
System.out.println("2 " +treeSet);
Dog showDog = (Dog) treeSet.first();
showDog.setAge(6);
System.out.println("3 " +treeSet);
System.out.println("4 " +treeSet.remove(new Dog("Mike",6)));
System.out.println("5 " +treeSet.remove(new Dog("Mike",2)));
System.out.println("6 " +treeSet.first());
System.out.println("7 " +treeSet.remove(treeSet.first()));
System.out.println("8 " +treeSet);
```

打印日志：

```
1 [Dog{name='mike', age=1}, Dog{name='mike', age=2}, Dog{name='mike', age=3}, Dog{name='mike', age=4}]
2 [Dog{name='mike', age=2}, Dog{name='mike', age=3}, Dog{name='mike', age=4}]
3 [Dog{name='mike', age=6}, Dog{name='mike', age=3}, Dog{name='mike', age=4}]
4 false
5 false
6 Dog{name='mike', age=6}
7 false
8 [Dog{name='mike', age=6}, Dog{name='mike', age=3}, Dog{name='mike', age=4}]

```

可以看到 日志3 TreeSet 对象并没有重新排序，通过日志 4、5、7 可以对象更改后 TreeSet 无法对其进行删除。

### TreeSet 定制排序

实现定制排序，则需要在创建 TreeSet 集合对象时结合 Comparator 对象，**并由 Comparator 负责集合元素的排序逻辑**。

如例：

```
private static void test2() {
        TreeSet treeSet = new TreeSet((o1, o2) -> {
            ClassD classD1 = (ClassD) o1;
            ClassD classD = (ClassD) o1;
            return classD1.getNum() > classD2.getNum() ? -1 : 1;
        });
        treeSet.add(new ClassD(1));
        treeSet.add(new ClassD(-1));
        treeSet.add(new ClassD(0));
        System.out.println(treeSet);
    }
    
// 打印日志：[ClassD{num=1}, ClassD{num=-1}, ClassD{num=0}]
```

对于自然排序和定制排序而言，添加到 TreeSet 集合对象中的元素应该为同一种类型的对象，否则会引发 ClassCastException 异常。

### 3. EnumSet 类

EnumSet 是一个专门为枚举类设计的集合类， EnumSet 中的所有的元素都必须是枚举类型的枚举值，该枚举类型在创建 EnumSet 时显式或隐式的指定。EnumSet 的集合元素也是有顺序的，**EnumSet 以枚举值在枚举类中的定义顺序来决定集合元素在顺序**。


```
public enum  EnumA {
    ONE,TWO,THREE
}

public class Main2 {

    public static void main(String[] args) {
        test4();
    }
    
    private static void test4() {
        EnumSet enumSet = EnumSet.noneOf(EnumA.class);
        enumSet.add(EnumA.ONE);
        enumSet.add(EnumA.THREE);
        enumSet.add(EnumA.TWO);
        System.out.println(enumSet);
        EnumSet enumSet2 = EnumSet.allOf(EnumA.class);
        System.out.println(enumSet2);
    }
}    
// 打印日志：
[ONE, TWO, THREE]
[ONE, TWO, THREE]
```

EnumSet在内部以位向量的形式保存，这种存储方式十分的高效，所以 EnumSet 占用的内存小、运行效率高。

**EnumSet 不可添加 null 元素。**

### 各 Set 集合的性能

HashSet 和 TreeSet 为 Set 集合的两个典型实现，两者如何选择呢？

由于 TreeSet 内部需要额外的红黑二叉树维护元素的添加顺序，所以 HashSet 的效率高于 TreeSet。**当需要保持排序的 Set 时才使用 TreeSet，否则使用 HashSet。**

LinkedHashSet 为 HashSet 的子类，对于删除、插入操作 LinkedHashSet 比 HashSet 稍慢一些，这是因为 LinkedHashSet 内部需要维护链表造成的，但正是这个原因，遍历操作时 LinkedHashSet 会更快一些。

EnumSet 是所有 Set 集合中效率最好的，但是它只能保存他一个枚举类的枚举值作为集合元素。

Set 的三个实现类 HashSet 、TreeSet、 EnumSet 都是 **线程不安全的**，多个线程操作 Set 集合时需保证线程同步。通常使用 Collections  工具类的 synchronizedSortSet 方法来包装该 Set 集合，此操作最好在创建时进行，以防止对 Set 集合意外的非同步访问。

```
SortedSet treeSet = Collections.synchronizedSortedSet(new TreeSet((o1,o2)->{
    return 0;
}))
```
