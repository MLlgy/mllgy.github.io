---
title: Kotlin 核心编程(三)：Kotlin 中的高阶函数和 Lambad 表达式
tags:
---


### 什么是高阶函数？

Kotlin 天然支持函数特性，与 Java 类为一等公民不同，在 Kotlin 中 **函数为一等公民**，函数可以直接定义在 kt 文件中。


在 Java 中限制方法只能接收数据作为参数，而高阶函数除此外，**方法的参数可以是函数**，**并且函数的返回值也可以为函数**。


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

按照这样的设计，当筛选条件增加时，业务逻辑也会高度耦合，业务维护成长较大。解决问题的核心是 **把 filterCounties 方法进行解耦**，常见的思路是传入一个类对象，根据不同需求创建不同的子类。但是在 Kotlin 中支持高阶函数，可以把筛选条件抽象成一个方法传入。

根据此可以很快的抽象出一个方法：

```
class CountryTest{
    fun isWantCountries(country:Country):Boolean{
        return country.continient == "EU" && country.population > 10000
    }
}
```
但是如何将一个函数作为另一个函数的参数？很明显如果直接传入方法名 isWantCountries， 函数名不是一个表达式，不具有类型信息，而且在 Kotlin 中万物都有类型，那么函数时类型是什么？


基于上面的描述，我们可以终结出，使用高阶函数的一个重要目标是：**解耦**。
#### 函数作为参数时的声明类型

参数中声明函数的类型：

* 通过 `->` 符号组织参数类型和返回值，左边是参数类型，右边是返回值类型。
* **参数必须使用括号包裹**。
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

```
// 以 (Int) -> ((Int,String) -> Unit) 为例编写方法
fun showTest() {
    show { a: Int ->
        { age: Int, name: String ->
            println("$a $age $name")
        }
    }
}

fun show(block: (Int) -> ((Int, String) -> Unit)) {
    //此处的调用为下文中描述的 柯里化风格
    block(1)(1,"")
}

// 声明函数的返回值为泛型
fun showAnother():(Int) ->String = {a:Int -> ""}
```
这时可以重新定义 filterCounties 方法:

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


### Lambda 表达式是如何实现的


```
fun test301() {
    listOf(1, 2, 3).forEach {
        foo(it)
    }
}

// 注意这里是 Lambda，不是普通的函数
fun foo(int: Int) = {
    println(int)
}
```

可以在 test301 看到关键字 **it**，这是 Kotlin 简化 Lambda 表达的一种语法糖，叫做 **单个参数的隐式名称**，代表了 **这个 Lambda 所接收的单个参数**。

带有 it 的写法其实和以下等效：

```
listOf(1, 2, 4).forEach { item ->
    foo(item)
}
```

但是以上方法却不会有任何的打印效果，因为 Kotlin 的 Lambda 在编译以后会被编译成匿名内部类，而 foo 编译后会被编译成如下：

```
    @NotNull
   public static final Function0 foo(final int var0) {
      return (Function0)(new Function0() {
         // $FF: synthetic method
         // $FF: bridge method
         public Object invoke() {
            this.invoke();
            return Unit.INSTANCE;
         }

         public final void invoke() {
            int var1 = var0;
            boolean var2 = false;
            System.out.println(var1);
         }
      });
   }
```

所以在调用 foo 时其实只是返回了一个 Function0 对象，要想指定其方法需要执行其 invoke 方法。

```
listOf(1, 2, 4).forEach { item ->
    foo(item).invoke()
}
```



同时如果觉得调用 invoke 显得比较丑陋，那么可以使用括号来代替,invoke 和 括号 的作用是一样的：

```
listOf(1, 2, 4).forEach { item ->
    foo(item)()
}
```


为什么在 Kotlin 要如此的设计？

这是因为需要兼容 Java 的 Lambda 表达式，而在 Java 中实现 Lambda 的前提是该接口为函数接口，而 Kotlin 这么设计就是为了能够在 Kotlin 中调用 Java 的 Lambda，例如：


```
tvSelectedCh.setOnClickListener {
    // doSomethings
}
```


Lambda 最大参数为22，如果想要添加更多的参数，需要自定义接口，具体代码可参见：[23]()。






### 函数、Lambda 和闭包

* fun 在没有等号、只有花括号的情况下，为我们常见的函数，函数返回值类型为 Unit时，必须声明。

```
// 函数
fun test(){...}
```
* fun 带有等号，是 **单表达式函数体**。
* 不管是 val 还是 fun，如果是等号加花括号的语法，那么构建的就是 Lambda 表达式。如果左侧是 fun，那么就是 Lambda 表达式函数体，必须通过 invoke 或者 () 来调用 Lambda 表达式。

```
//单表达式函数体、Lambda 表达式
fun test(x:Int,y：Int):Int = x+y
// 调用
test.invoke(1,2)
test(1,2)
```

* 在 Kotlin 中，由花括号包裹的代码块如果 `访问了外部的环境变量` 则被称为 **闭包**，闭包可以被当做参数传递或者直接使用，Lambda 是 Kotin 中最常见的闭包。

Kotlin 中的闭包与 Java 中不同，Kotlin 中的闭包不仅可以访问外部变量还可以修改外部变量。

```
public void test(){
    int a = 1;
    oneMain.setIClick(() -> {
        System.out.println(a);
        //a++; 报错，不可以修改外部变量
    });
}
```

Kotlin

```
fun test(){
    var a = 1
    oneMain.setIClick {
        println(a)
        a++
    }
}
```


### 柯里化风格

柯里化语法是 **将函数作为返回值** 的一种典型的应用

简单来说，柯里化是指把接收到的多个参数的函数变换成一系列仅接受单一参数函数的过程，在返回最终结果前，前面的函数可以依次接收单个参数，然后返回下一个新的函数。

概念有点晦涩，直接看代码:

正常编写的代码：

```
fun method(a: Int, b: Int): Int = a + b
// 进行调用
method(1,2)
```

// 按照柯里化的思想，重新编写

```
fun method(a: Int) = { b: Int -> a + b }
// 进行调用
method(1)(2)
```