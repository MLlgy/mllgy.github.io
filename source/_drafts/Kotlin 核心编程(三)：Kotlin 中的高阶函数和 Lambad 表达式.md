---
title: Kotlin 核心编程(三)：Kotlin 中的高阶函数和 Lambad 表达式
tags:
---


### 什么是高阶函数？

Kotlin 天然支持函数特性，与 Java 类为一等公民不同，在 Kotlin 中 **函数为一等公民**，函数可以直接定义在 kt 文件中。


在 Java 中限制方法只能接收数据作为参数，而高阶函数除此外，还可以 **接收方法为参数或者返回值为函数**。


### 一个实例说明为什么使用高阶函数


下面通过一个示例，依据需求的演进来一步一步揭开为什么要引入高阶函数。

此例 《Java8 实战》中的一个例子,具体需求从众多 country 中过滤符合要求的国家。


首先构造 Country 类：

```
data class Country(
    val name: String,
    val continient: String,
    val population: Int
)
```

需求一：过滤出 EU 的国家

```
class CountApp {
    fun filterCounties(countries: List<Country>): List<Country> {
        val res = mutableListOf<Country>()
        for (country in countries) {
            if (country.continient == "EU") {
                res.add(country)
            }
        }
        return res
    }
}
```

需求二：过滤出符合指定洲的国家

```
class CountApp {
    fun filterCounties(continient: String,countries: List<Country>): List<Country> {
        val res = mutableListOf<Country>()
        for (country in countries) {
            if (country.continient == continient) {
                res.add(country)
            }
        }
        return res
    }
}
```


需求三：在以上基础上加入过滤条件：人口数


```
class CountApp {
    fun filterCounties(continient: String,population:Int,countries: List<Country>): List<Country> {
        val res = mutableListOf<Country>()
        for (country in countries) {
            if (country.continient == continient && country.population > population) {
                res.add(country)
            }
        }
        return res
    }
}
```

按照这样的设计，当筛选条件类间时，业务逻辑也会高度耦合。解决问题的核心是 **把 filterCounties 方法进行解耦**，常见的思路是传入一个类对象，根据不同需求创建不同的子类。但是在 Kotlin 中支持高阶函数，可以吧筛选条件抽象成一个方法传入。

根据此可以很快的抽象出一个方法：

```
class CountryTest{
    fun isWantCountries(country:Country):Boolean{
        return country.continient == "EU" && country.population > 10000
    }
}
```
但是如何将一个函数作为另一个函数的参数，很明显直接传入方法名 isWantCountries 是不满足要求的，而且函数时类型是什么？

#### 函数作为参数时的声明类型



参数中声明函数的类型：

* 通过 -> 符号组织参数类型和返回值，左边是参数类型，右边是返回值类型。
* 参数必须使用括号包裹。
* 返回值类型为 Unit，也要显式声明。

基于此我们可以举例如下：

```
(Int) -> String
() -> Int
(Int,String) -> Unit
(name: String,age: Int) -> People// 为声明参数指定名字
(Int,String?) -> Int // 参数可为空
(Int) -> ((Int,String) -> Unit)// 返回值为另外一个函数
(Int) -> ((Int) -> Unit) 简化等效 (Int) -> Int -> Unit 
```

这里需要注意的是，以上声明的是 **作为参数** 时的函数的类型，而在定义函数时需要注意写法：

**此处预留一个问题**
```
// 以 (Int) -> ((Int,String) -> Unit) 为例编写方法
```


这是可以重新定义 filterCounties 方法:


```
fun filterCounties(test: (Country) -> Boolean, countries: List<Country>): List<Country> {
    val res = mutableListOf<Country>()
    for (country in countries) {
        if (test(country)) {
            res.add(country)
        }
    }
    return res
}
```


#### 将方法传递给另外一个方法

Kotlin 中存在一种特殊的语法：通过两个冒号(::)实现对某个类的方法进行引用。

```
private fun test105() {
    val countApp = CountApp()
    val countryTest = CountryTest()
    val countries = listOf(
        Country("A", "EU", 100000)
        , Country("B", "AS", 1000)
        , Country("C", "AS", 1000)
    )
    // 双冒号引用，传入函数
    val result = countApp.filterCounties(countryTest::isWantCountries, countries)
    result.forEach {
        println("Country info is: ${it.name}")
    }
}
```


#### 近一步：匿名函数


Kotlin 可以在 缺省方法名时，直接定义一个函数，如下：

```
fun (country:Country):Boolean{
        return country.continient == "EU" && country.population > 10000
    }
```

那么可以直接使用该函数：


```
fun(country: Country): Boolean {
    return country.continient == "EU" && country.population > 10000
}
```

那么我们可以直接将匿名函数传入方法中，这样就省去了声明 CountryTest 变量：


```
private fun test105() {
    val countApp = CountApp()
    val countryTest = CountryTest()
    val countries = listOf(
        Country("A", "EU", 100000)
        , Country("B", "AS", 1000)
        , Country("C", "AS", 1000)
    )
    // 传入匿名函数
    val result = countApp.filterCounties(fun(country: Country): Boolean {
        return country.continient == "EU" && country.population > 10000
    }, countries)
    result.forEach {
        println("Country info is: ${it.name}")
    }
}
```

#### 更进一步： Lambda 表达式

其实可以把 Lambda 理解成 **简化后的匿名函数**，实质是一种语法糖。将上例改进为 Lambda 表示式：

```
private fun test105() {
    val countApp = CountApp()
    val countries = listOf(
        Country("A", "EU", 100000)
        , Country("B", "AS", 1000)
        , Country("C", "AS", 1000)
    )
    // 使用 Lambad 表达式
    val result =
        countApp.filterCounties({ country -> country.continient == "EU" && country.population > 10000 }, countries)
    result.forEach {
        println("Country info is: ${it.name}")
    }
}
```

那么具体看一下 Lambda 表达式的语法：

* 使用 -> 连接参数和返回值。
* 如果 Lambda 表达式返回值不为 Unit，那么默认最后一行为表达式的返回类型，此时 return 可以省略。
* Lambda 表达式必须通过 {} 包裹。
* 如果 Lambda 声明了参数部分，且返回值类型支持类型推导，那么 Lambda 变量的类型可以省略。
* 如果 Lambda 变量声明了函数类型，那么 Lambda 的参数部分可以省略。

```
val sum = {x:Int,y:Int -> x + y}
或
val sum:(Int,Int) -> Int = {x,y -> x + y}
```


至此，代码解耦了过滤方法，可以按照需求传入过滤 Lambda 表达式。


### Lambda 表达式的进阶使用


```
fun test301() {
    listOf(1, 2, 3).forEach {
        foo(it)
    }
}

fun foo(int: Int) {
    println(int)
}
```

可以在 test301 看到关键字 **it**，