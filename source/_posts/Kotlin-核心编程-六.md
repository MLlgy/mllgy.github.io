---
title: 'Kotlin 核心编程(六):空安全'
date: 2019-12-02 18:53:55
tags: [Kotlin 核心编程,读书笔记]
---

### 0x0001 Java 中存在歧义的 null 

null 含义：
* 该值未初始化
* 该值不合法
* 该值不需要
* 该值不存在

### 0x0002 难以控制的 NPE

在 Java 中任何对象引用都可以为 null，而当调用为 null 对象的方法时，将产生 NPE，最终导致的是：编译顺畅通过，但是运行代码时却产生了 NPE。
<!-- mores -->
比如此种情况：

```
String name = null;
System.out.println(name.length());
```
而我们只能通过一些判空代码进行避免，但是往往将 null 和空字符串混为一谈，违背了业务初衷，示例如下：

```
String name = null;
if(name != null || ("").equal.(name)){
    System.out.println(name.length());
}
```


### 0x0003 可空类型


Kotlin 在类型层面提供了空类型，以此来解决由 null 引发的问题。

**Java8 中的 Optional**


对于不确定是否存在的属性，可以使用 Optional 来封装。

但是 Optional 的耗时大约是普通判空的十倍，原因是 Optional<T> 是一个包含类型 T 引用的泛型类，在使用时多个创建一个对象，在数据量比较大的时候，频繁实例化造成很大的性能损失。

**Kotlin 中的可空类型**

而 Kotlin 中处理 NPE 问题十分容易。

Java 中存在 Optional 包装类，而 Kotin 中可空类型使用 `类型？` 表示：

```
var name:String? = null
```


**Kotlin 中的安全调用 ?.**

在 Java 这么写：

```
if(name != null){
    sout(name.length());
}
```

在 Kotlin 中这么写：
```
sout("${name?.length}")
```

`?.` 称为 **安全调用**，只有当 name 存在时才会调用 name.length。


**Kotlin 中的 Elvis 操作符 ?:**


`?:` 左侧为 null 时，则执行其右侧值。
```
val num = name?.length?:-1
println(num)
```


`?:` 左侧不仅可以是语句还可以是表达式或者函数。

```
val type = getType() ?: getDefaultType()
println(type)
val type2 = getType() ?: "Default: ${type?.length}"
println(type2)
```

Kotlin 如今实现类型的空安全：

```

fun test301(name: String? ) {
    println(name?.length)
}
// 编译后的 Java 代码
public static final void test301(@Nullable String name) {
    Integer var1 = name != null ? name.length() : null;
    boolean var2 = false;
    System.out.println(var1);
}
```
其实Kotlin 最终的空安全还是靠 if-else 进行判断。


**非空断言 !!.**

类比 Java 中 Assert(断言)，对象为空时，抛出 NPE 异常。
```
val len = name!!.length
```