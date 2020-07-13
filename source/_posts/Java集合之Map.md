---
title: Java集合之Map
date: 2019-05-10 18:17:08
tags: [Java Collection,Map,Java]
---

### 概述 

Map 用于保存具有 **映射关系** 的数据，因此 Map 集合内保存着两组数据，一组保存 Map 的 key，一组保存 Map 的 value，并且 key 和 value 可以是任何类型的数据。

![Map](/../images/2019_05_09_02.jpg)

可以把 Map 里的 Key 放在一起，它们组成一个 **Set 集合**(Key 没有顺序、不可重复)，其实Map 的内部的 keySet() 方法确实是返回的所有的 key 组成的 Set 集合对象。而把 Map 内的 Value 放在一起，它们类似一个 **List 集合**(可重复，此时的索引为 key)。

如果抽象的看 Map 的 key 可以看作为 Set 集合(即无序、不可重复)，而 value 可以看作为 Array 集合(可重复)

### 操作

向 Map 中添加数据中，如果 Map 中有存在 key，那么新添加的 value 会覆盖原来的 value。

<!-- more -->

### HashMap 和 Hashtable

HashMap 和 Hashtable 是 Map 的典型实现类。 Hashtable 在 JDK1.0 起就已经存在。在 Java8 时改进了 HashMap 的实现，使其在 HashMap 存在 key 冲突时具有更好的性能。

Hashtable 与 HashMap 主要区别：

* Hashtable: **线程安全** 的 Map 实现，但是 HashMap 是线程不安全的实现，所以 HashMap 的性能更高一些。多个线程访问同一个 Map 对象时，使用 Hashtable 会更好。
* Hashtable 不允许使用 null 作为 key 和 value，否则会引起异常，但是 HashMap 可以。

为了更好的在 HashMap 和 Hashtable 中存储、获取对象，**用作 key 的对象必须实现 hashCode() 和 equals() 方法**。

HashMap、Hashtable **key 相同的标准** 是：**两个 key 通过 equals() 方法比较返回为 true，那么两个 key 的 hashcode 值必须相等**。

HashMap、Hashtable **value 相等** 的标准：只要两个对象的 equals() 方法比较返回为 true 即可。

与 HashSet 相同，采用自定义类作为 HashMap、Hashtable 的 key，如果程序修改了作为 key 的可变对象，那么也会出现与 HashSet 类似的情形：程序无法准确访问到 Map 中被修改过的 key。

### LinkedHashMap

与 LinkedHashSet 一致，LinkedHashMap 内部也是通过双向链表来维护添加的 key-value 的顺序(以 key 的添加顺序为准)。

同样的，由于维护链表 LinkedHashMap 的性能比 HashMap 差一些，但是也正是这个原因在遍历 Map 中元素时，LinkedHashMap 的效率更高一些。

### SortedMap && TreeMap

SortedMap 为 Map 派生的一个子接口，而 TreeMap 为 SortedMap 的一个实现类。

TreeMap 是一个红黑树结构，每个 key-value 为红黑树的一个节点，而 TreeMap 存储数据时，是以 key 为标准对 key-value 进行排序。

TreeMap 的两种排序方式：

* 自然排序
  
    TreeMap 的所有的 key 必须实现 Comparable 接口，而且所有的 key 必须为同一个类的对象，否则会报 ClassCastException 。

    key 相同的标准为：两个 key 通过 compareTo() 方法返回 0，则认为相等。

* 定制排序

    在 创建 TreeMap 时，传入 Comparator 对象，该对象负责对 key 进行排序，此时 key 不要求实现 Comparator 接口。 

    如果想让排序更加顺利的进行，那么需要重写 key 对应类的 equal() 方法，并且 equal() 方法的返回值与 compareTo() 的返回值表达的结果一致，否则将于 Map 接口规则冲突。


**自然排序**

```
public class Bean implements Comparable {

    private int age;
    private String name;

    ...
    ...

    @Override
    public boolean equals(Object obj) {
        if (this == obj) {
            return true;
        }
        if (obj != null && obj.getClass() == Bean.class) {
            Bean bean = (Bean) obj;
            return age == bean.age;
        }
        return false;
    }

    @Override
    public int compareTo(Object obj) {
        Bean bean = (Bean) obj;
        return Integer.compare(age, bean.age);
    }

    @Override
    public String toString() {
       ....
    }
}
```

```code
TreeMap treeMap = new TreeMap();
Bean bean = new Bean(1, "mike");
Bean bean2 = new Bean(4, "mike2");
Bean bean3 = new Bean(3, "mike3");
Bean bean4 = new Bean(1, "mike4");
treeMap.put(bean, "one");
treeMap.put(bean4, "two");
treeMap.put(bean3, "three");
treeMap.put(bean2, "four");
treeMap.put(bean2, "five");
System.out.println(treeMap);
```

打印日志：

```
{Bean{age=1, name='mike'}=two, Bean{age=3, name='mike3'}=three, Bean{age=4, name='mike2'}=five}
```

TreeMap 按照 Bean 重写的 compareTo() 规则进行排序，注意重写的 equal() 需要满足以上逻辑。

**定制排序**

```
TreeMap treeMap = new TreeMap((o1, o2) -> {
    Bean2 bean = (Bean2) o1;
    Bean2 bean2 = (Bean2) o2;
    return Integer.compare(bean.getAge(), bean2.getAge());
});
Bean2 bean = new Bean2(1, "mike");
Bean2 bean2 = new Bean2(4, "mike2");
Bean2 bean3 = new Bean2(3, "mike3");
Bean2 bean4 = new Bean2(1, "mike4");
treeMap.put(bean, "one");
treeMap.put(bean4, "two");
treeMap.put(bean3, "three");
treeMap.put(bean2, "four");
treeMap.put(bean2, "five");
System.out.println(treeMap);
```

打印日志：

```
{Bean{age=1, name='mike'}=two, Bean{age=3, name='mike3'}=three, Bean{age=4, name='mike2'}=five}
```

### WeakHashMap

观字识意，WeakHashMap 与 HashMap 的区别是 WeakHashMap 中存在弱引用。

WeakHashMap 中的 key 只保留了对实际对象的弱引用，如果 WeakHashMap 对象的 key 引用的对象没有被其它强引用的变量引用，那么 key 所引用的对象在垃圾回收时很有可能被销毁，同时 WeakHashMap 也会自动移除 key 对应的 key-value 键值对。


### EnumMap

EnumMap 中的所有的 key 必须是 单个枚举类中的枚举值。创建 EnumMap 对象时需要指定其对应的枚举类。

EnumMap 的特征：

* EnumMap 内部以数组的形式保存，所以高效。
* EnumMap 根据 key 的自然排序(枚举类中枚举值的定义顺序)维护 key-value 的顺序。
* EnumMap key 不可为 null。


```
Enum Num{
    ONE,TWO,THREE
}

EnumMap enumMap = new EnumMap(Num.class);
enumMap.put(Num.TWO,"two");
enumMap.put(Num.ONE,"one");
enumMap.put(Num.THREE,"three");
sout(enumMap);
```
打印日志：

```
{ONE=oen,TWO=two,THREE=three}
```

### 性能对比

Hashtable 和 HashMap 的内部实现机制几乎一样，但是由于 Hashtable 为线性安全的集合，所以 HashMap 要比 Hashtable 要快。

TreeMap 的内部通过红黑树来维护添加顺序，所以比 Hashtable、HashMap 要慢(特别是在添加元素、删除元素时，此时需要重新整理红黑树)。同时也是因为内部红黑树 TreeMap 在变量元素时比较高效。

```
System.out.println(Arrays.binarySearch( treeMap.keySet().toArray(),new Bean(3, "mike3")));
```

一般场景下使用 HashMap，因为 HashMap 为快速查询而设计(底层使用数组存储 key-value)。如果需要对元素进行排序，那么使用 TreeMap。

由于需要维护内部链表，LinkedHashMap 比 HashMap慢一些，与 HashMap 实现基本一致，但是 LinkedHashMap 使用 == 而不是 equal() 来判断 key 相等。

EnumMap 性能最好，但是它的 key 只能是枚举值。