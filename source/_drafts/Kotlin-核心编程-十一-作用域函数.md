---
title: 'Kotlin 核心编程(十一):作用域函数'
tags:
---
这篇文章的起源：
```

data class Student(val age: Int, val name: String, val score: Int)

class Kot {
    val student: Student = Student(9, "aline", 3)
    var age = 2
    fun showTest() {
        
        student?.apply {
            println(name)
        }
        
        student?.also {
            println(it.name)
            println(it.age)
            println(this.age)
        }

        student.let {
            println(this.age)
            println(it.name)
        }
    }
} 
```

在 also、let 的 Lambda 表达式内，it 代表接收者类型对象，而不像 《Kotlin 核心编程》 一书中所说的，this 代表接收者类型对象，而且在 also、let 中同时存在 this，并且通过 this 可以访问到外层变量，自己对此充满疑问，感觉书中对 this 为接收者对象的方法过于武断，查官方文档，十分幸运，正好有一篇关于此内容的文档，不过为英文，自己读了一遍，觉得在翻译此文在自己能力范围内，故做大致翻译，但是不会逐字翻译，得其大概即可。


### 作用域函数的作用


Koltin 标准库中存在这样一类函数，它存在的唯一作用就是在执行传入闭包中的代码。当向这类函数传入 Lambda 表达式时，在该闭包范围内存在作用域对象，可以不通过对象的名字直接访问该对象的成员，这样的函数在 Kotlin 称为作用域函数。在 Kotlin 中有 5 个作用域函数： let、run、with、apply、also。

上面提到的 5 个函数的作用基本都一致：在对象作用域内之下代码块，所以在开发中如何根据需求选择合适的函数就变得不那么容易，需要考虑具体需求意图、一致性等因素。基于此，以下内容主要用于陈述以上 5 个作用域函数的区别以及常规用法。


### 区别

不同点主要体现在以下两点：
1. 闭包内上下文对象的表示方法。
2. 整个表达式的返回结果。


#### 上下文对象的表示方法：this or it


在作用域函数中，上下文对象可以通过短引用而不是实际名称来使用。在作用域函数中可以通过
每个作用域函数上下文对象只能是以下两种方式中的一种方式获得：
1. 作用域函数的接收者类型对象，在闭包中通过 this 进行访问。
2. Lambda 表达式的传入参数对象，在闭包中通过 it 进行访问。

**this**

作用域函数 run、with、apply 中的上下文对象为作用域函数的类型接收者对象，在闭包中通过 this 进行指代，通过以下示例具体展示：

```
data class Student(val age: Int, val name: String, val score: Int)

fun show(){
    val student: Student = Student(9, "aline", 3)
    student.run { 
        this.age
        this.score
        name
    }
}
```
可以看到在 Lambd 中 this 即为 run 函数的类型接收者对象 student，可以在闭包中通过 this 访问 student 的成员变量，同时关键字 this 也可以省略。

**it**

let、also 作用域函数的上下文对象为 Lambda 对象中传入的类型对象。如果在 Lambda 中参数名称没有显式声明，那么该对象可以通过 it 隐式访问。

```
fun show(){
    val student: Student = Student(9, "aline", 3)

    student.also { 
        it.age
        it.score
    }
    // 指定参数的名称，与上面的效果一样
    student.also {  stu -> 
        stu.age
        stu.score
    }
}
```



### 返回值

以上提到的 5 个作用域函数的返回值对象是不同的：

* apply、also 返回值为上下文对象。
* let、run、with 返回值为 Lambda 表达式结果。

**上下文对象**

apply、also 返回的是上下文对象，所以使用它们进行链式操作：


```

fun show() {
    val student: Student = Student(9, "aline", 3)

    //返回值对象为上下文对象
    val returnResult: Student = student.also { stu ->
        stu.age
        stu.score
    }.apply {
        this.score++
        println(score)
    }
    println(returnResult)
}
```

**Lambda 表达式结果**


let、run、with 返回的对象为 Lambda 表达式结果，同时也可以不返回任何对象，如在 Lambda 表达式的最后执行 print 操作。
```
fun show() {
    val student: Student = Student(9, "aline", 3)

    val str = student.let { 
        var getScore = it.score
        getScore++
        getScore// 此为 Lambda 表达式结果，也为作用域函数的返回结果
    }
    println(str)

    val withResult = with(student){
        var getScore = score
        getScore++
        getScore
    }
    println(withResult)
}
```

### 常用用法

一般情况下，以上几个作用域函数都是可以互相替换的，但是此处说明一下每个函数的适合的场景。


**let**


上下文对象为 Lambda 传入的参数，使用 it 表示，整个表达式的返回值为 Lambda 的结果。

1. let 可以用于在调用链的结果上调用一个或多个函数。

```
val numbers = mutableListOf("one", "two", "three", "four", "five")
numbers.map { it.length }.filter { it > 3 }.let { 
    println(it)
    // and more function calls if needed
} 
```
如果let 闭包中的只有一个函数，并且 it 为该函数的参数，那么可以使用使用(::xxx) 的形式代替 lambda 闭包：

```
val student: Student = Student(9, "aline", 3)
val str1 = student.let{
    show(it)
}
val str2 = student.let(::show)
fun show(stu:Student){
}
```
2. let 常用于对象不为空时执行相关操作

```
val str: String? = "Hello"
val length = str?.let{
    sout(it)
    it.lenght
}
```

3. let 常用于为闭包引入局部变量，以提高代码的可读性：

```
val numbers = listOf("one", "two", "three", "four")
val modifiedFirstItem = numbers.first().let { firstItem ->
        println("The first item of the list is '$firstItem'")
    if (firstItem.length >= 5) firstItem else "!" + firstItem + "!"
}.toUpperCase()
println("First item after modifications: '$modifiedFirstItem'")
```


**with**


上下文对象 with 传入的参数，整个表达式的返回值为 Lambda 的结果。


1. with 常用于不返回 Lambda 结果的情况，给人直观的感受：通过这个对象，你可以进行如下操作：


```
val numbers = mutableListOf("one", "two", "three")
with(numbers) {
    println("'with' is called with argument $this")
    println("It contains $size elements")
}
```

2. with 的另一个用例是引入一个辅助对象，其属性或函数将用于计算值。

```
val numbers = mutableListOf("one", "two", "three")
// firstAndLast 引入的辅助对象
val firstAndLast = with(numbers) {
    "The first element is ${first()}," +
        " the last element is ${last()}"
}
println(firstAndLast)
```


**run**

上下文对象通过 this 表示，整个表达式的返回值为 Lambda 的结果。


run 的功能和 with 相同，不同的是通过扩展函数进行执行相关操作。
 


1. 当 lambda 同时包含对象初始化和返回值的计算时，t推荐使用 run：


```
val service = MultiportService("https://example.kotlinlang.org", 80)

val result = service.run {
    port = 8080
    query(prepareRequest() + " to port $port")
}

// the same code written with let() function:
val letResult = service.let {
    it.port = 8080
    it.query(it.prepareRequest() + " to port ${it.port}")
}
```


2. run 同时可以作为非扩展函数使用，可以需要表达式的地方同时执行多行代码：

```
val hexNumberRegex = run {
    val digits = "0-9"
    val hexDigits = "A-Fa-f"
    val sign = "+-"

    Regex("[$sign]?[$digits$hexDigits]+")
}

for (match in hexNumberRegex.findAll("+1234 -FFFF not-a-number")) {
    println(match.value)
}
```

**apply**

上下文对象通过 this 表示，整个表达式的返回值为 上下文对象。

使用 apply 不会

1. 对象配置，不返回任何值，获取上下的成员。

```
val adam = Person("Adam").apply {
    age = 32
    city = "London"        
}
```
将上下文对象作为返回对象，可以十分方便的引用到调用链上。


**also**

上下文对象使用 it 表示，整个表达式的返回值为 上下文对象。


1. 常用于不改变对象的其他操作，例如记录或打印调试信息，这样即使从调用链移除 also 也不会对逻辑造成影响。

```
val numbers = mutableListOf("one", "two", "three")
numbers
    .also { println("The list elements before adding new one: $it") }
    .add("four")
```


**简单总结：**

* 在非空对象上执行lambda：let
* 将表达式引入为局部作用域中的变量：let
* 对象配置：apply
* 对象配置和计算结果：run
* 需要表达式的运行语句：非扩展函数 run
* 附加效果：also
* 分组对对象的函数调用：with