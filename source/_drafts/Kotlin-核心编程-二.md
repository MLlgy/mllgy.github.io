---
title: Kotlin 核心编程(二)
tags:
---

### 概述

Kotlin 的基本使用

### var 和 val 的使用规则

在 Kotlin 中声明变量使用 var 和 val 关键字，var 可以理解为 variable，代表变量，在 val 可以理解为 final variable，代表 **引用不可变**，这里需要引用不可变的含义，不再详述。


在 Kotlin 中优先使用 val 避免产生 **副作用**。
> 副作用：修改了某处的一些东西：修改外部变量的值、IO 操作)

通过下面我们可以更直观的理解副作用：

```
var a = 1
private fun test104(x: Int) {
    a += 1
    println(x + a)
}

test104(1)
test104(1)
```

则打印日志为：
```
3
4
```
现在你看到的调用两次 test104(1)，但是打印结果不同，这就是很明显的副作用，而如果使用 val 声明变量 a，那么就不会存在这种情况，因为对 val 变量赋值是错误的。

基于此，在 Kotlin 中 **尽量使用 val 声明本身不可变的变量**。


var 的适用场景：

var 占用内存少，在某些业务中如果需要存储大量的数据，使用 var 则更适合。


### 增强的类型推导

编译器可以在不显示声明类型的情况下，自动推导出它需要的类型。

```
fun main(args: Array<String>) {
    val num = 12
    val numLong = 12L
    println(num.javaClass.name)
    println(numLong.javaClass.name)
}
```

通过日志，Kotlin 会自动推导变量的类型：

```
int
long
```



### 增强的函数


在 Kotlin 中增强了函数特性，可以吧 '{}' 取消，用等号直接定义函数:

```
// 增强的函数
private fun test102(x: Int, y: Int) = x + y 
```

**表达式函数**：在 Kotlin 中支持使用单行表达式与等号的语法来定义函数。

在 Kotlin 中 **表达式** 有非常重要的地位，那么什么是表达式呢？
> 在 Kotlin 中表达式可以是一个值、变量、常量、操作符、函数，或者它们的组合，编程语言对其进行解释和计算，以 **产生另外一个值**。

在 Kotlin 中 if 是一个语句，同时也是表达式，如果把 if 语句赋值给变量，那么此时 if 语句为表达式，则必须存在 else 语句，否则会报错

```
// if 作为语句
if(条件判断){
    // 执行操作
}else{
    // 执行操作
}
// if 作为表达式
val name = if (isTrue) {
    println("true")
} else {
    println("false")
}
```

### 字符串判等

在 Kotlin 中，相等有结构相等和引用相等：

* 结构相等：通过操作符 == 判断两个对象的 **内容** 是否相等。
* 引用相等：通过操作符 === 两个对象的 **引用** 是否相等。

```
val a = "Kotlin"
val b = "Kotlin"
val c = "Kot"
val d = "lin"
val e = c + d
println("a==b? :${a == b}")
println("a===b? :${a === b}")
println("a==e? :${a == e}")
println("a===e? :${a === e}")
```
打印日志：
```
a==b? :true
a===b? :true
a==e? :true
a===e? :false
```

### 高阶函数和 Lambda 

Kotlin 天然支持函数特性，与 Java 类为一等公民不同，在 Kotlin 中 **函数为一等公民**，函数可以直接定义在 kt 文件中。


在 Java 中限制方法只能接收数据作为参数，而高阶函数除此外，还可以定义 **接收方法为参数或者返回值为函数** 的方法。


----

[Kotlin 编译流程简介](https://blog.csdn.net/u010218288/article/details/86062858)