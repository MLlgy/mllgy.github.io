---
title: Kotlin 核心编程(四)：表达式、中缀函数、中缀表达式
date: 2019-11-25 18:30:09
tags: [Kotlin 核心编程,读书笔记]
---
### 0x0001 在 Kotlin 中什么是表达式？

> 在 Kotlin 中表达式可以是一个值、变量、常量、操作符、函数，或者它们的组合，编程语言对其进行解释和计算，以 **产生另外一个值**。

其定义的关键点：

<!-- more -->

* 表达式必须会有一个值
* Unit 类型让函数称为表达式，没有返回值的函数此时的返回类型为 Unit，符合表达式产生了新值的定义。


### 0x0002 常见的表达式

`if` 和 `when` 作为表达式时必须要有 else 语句。

枚举类结合 when 表达式可以十分方便的表示多分支情况，同时keyi使用 when 来代替 if-else 表达式。

**范围表达式**

`..` 表示范围，**包括前面不包括后面**，如 1..3 表示 12

`in` 表示成员关系，意为在的含义。

其他关键字，如 until、step、downTo 等


### 0x0003 中缀函数和中缀表达式

until、step 等可以不通过点号进行使用，这是中缀表达式调用方式。

downTo 的定义：
```
public infix fun Int.downTo(to: Int): IntProgression {
    return IntProgression.fromClosedRange(this, to, -1)
}
```

在 Kotlin 中，像 downTo 这样通过关键字 infix 修饰的函数称为中缀函数。

定义一个中缀函数满足以下条件：
* **为某个类型的扩展函数或者成员方法**。
* **只有一个参数**。
* 参数不支持默认值，不为可变参数。

定义中缀函数需要使用 **infix** 修饰，下面为自定义中缀函数：

```
class People{
    // 在类内部定义中缀函数
    infix fun message(name:String){
        println("name is $name")
    }
}

// 通过扩展函数定义中缀函数
infix fun People.showAge(age:Int){
    println("age is $age")
}

fun main(args: Array<String>) {
    val people = People()
    people message "Mike"
    people.message("Mike")// 正常方法调用
}


```

**中缀表达式**：类似 `A 中缀方法 B` 的形式。

像 in、step、until、downTo 这样不通过点号，而是通过中缀表达式调用。




