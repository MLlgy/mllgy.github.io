---
title: Kotlin 核心编程(五)：面向对象
tags:
---



### Kotlin 中的类

Kotlin 中类与 Java 中的几点不同:

* 在 Kotlin 中除非显示声明延时初始化，那么属性需要显示指定默认值。
* val 为不可变属性
* 修饰符的访问权限不同，Kotlin 默认全局可见， Java 默认包可见。
* 在接口中，不可为属性初始化值( Kotlin 接口中抽象属性，背后通过方法实现)


### 类的构造方法

在 Kotlin 中可以给构造函数的参数指定默认值:

```
class Bird(val color:String = "white", val age : Int = 1){
    ...
}
```

在类初始化时可以指定其参数名，或者按照参数的顺序赋值:

```
val  bird = Bird(age = 3,color =  "red")
或者
val  bird = Bird( "red",3)
```


使用 val 或者 var 声明构造方法的参数，有两个作用：
* 代表参数的引用可变性，var 可变，val 不可变。
* 简化构造类中语法

至于怎样简化，示例如下：

```
class Bird(val color:String = "white", val age : Int = 1){

}
// 与下面的写法等效，很明显上面写法简洁
class Bird( color:String = "white",  age : Int = 1){

    private val color: String
    private val age: Int
    init {
        this.age = age
        this.color = color
    }
}
```


**init 代码块**

类中可以存在多个 init 代码块，执行顺序按照类中的顺序自上而下，在复杂的业务场景下，可以按照职能对代码进行分离，使代码条理清晰、逻辑分明。


当构造方法的参数没有 val 或者 var 时，构造方法的参数可以在 init 语句中使用，除了 init 代码块，此时构造方法中的参数不可以在其他位置使用。


Kotlin 中规定类中的所有非抽象属性成员必须在对象创建时被初始化,val 声明的属性不可二次赋值，但是通过在 init 代码块中进行初始化：

```
class Bird( color:String = "white",  age : Int = 1){

    private val color: String
    private val age: Int
    init {
        this.age = age
        this.color = color
    }
}
```


**主从构造方法**


在类外部定义的构造方法称为主构造方法，在类内部通过 constructor 定义的方法称为从构造方法。如果主构造方法存在注解或者可见修饰符，那么需要添加 constructor 关键字。

从构造器由两部分组成：
1. 对其他构造方法的委托。
2. {} 内部包裹的代码块。

如果一个类存在主构造方法，那么所有的从构造方法需要直接或间接的委托给它。

执行顺序是为先执行委托的方法，然后执行自身代码块的逻辑。

```
class Dog(val country: String) {
    var telNum: Int? = 0
    constructor(tel: Int, country: String) : this(country) {
        telNum = tel
    }
}

fun main() {
    val dog = Dog(123,"China")
    println("dog`s tel num is ${dog.telNum}, country is ${dog.country}")
}
```



### 延迟初始化


延迟初始化可以不用在对象初始化时必须有值。


**by lazy**


可以使用 by lazy 修饰 val 声明的变量：

```
class Dog{
    val name by lazy { 
        "Ni"
    }
}
```

在首次调用时，才会进行赋值操作，一旦赋值，后续将不能更改。

```
public actual fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)


public inline operator fun <T> Lazy<T>.getValue(thisRef: Any?, property: KProperty<*>): T = value

```

lazy 的背后是接受一个 Lambda 表达式，并且返回一个 Lazy<T> 实例的函数，第一次调用该属性时，会执行 lazy 对应 Lambda 表达式并记录，后续访问该属性会返回记录的结果。


默认系统会为 lazy 属性加上同步锁 - LazyThreadSafetyMode.SYNCHRONIZED，同一时刻只有一个线程可以对 lazy 的属性进行初始化，为线程安全的。如果能够确保可以并行执行，可以给 lazy 传递其他线程参数。


**lateinit**


lateinit 主要用于声明 var 变量，不能修饰基本数据类型，如 Int 等，如果需要的话，使用 Interger 等包装类替代。


```
class Dog {
    lateinit var color: String

    fun printColor(newColor: String) {
        color = newColor
    }
}
```


### 类的访问修饰符权限


* Kotlin 中类和方法默认修饰符是 final，默认是不被继承或重写的，如果有这样的需求，必须添加 open 修饰符。

* Kotlin 的默认修饰符为 public，Java 为 default。
* Kotlin 独特的修饰符：internal。
* Kotlin 和 Java 的 protected 的访问权限不同，Java 中是包、子类可访问，而 Kotlin 中只允许子类访问。


|修饰符|含义|与Java 比较|
--|--|--
public|Kotlin 的默认修饰符，全局可见|与Java 中public 效果相同
protected|受保护的修饰符，类及子类可见|含义一致，但是访问权限为类、子类、包
private|私有修饰符，类内修饰只有本类可见，类外修饰本文件可见| 私有修饰符，本类可见
internal|模块可见|无

#### 密闭类



Kotlin 中除了使用 fina 来限制类的继承外，还可以使用密闭类来限制一个类的继承。

Kotlin 通过 sealed 关键字来修饰一个类为密闭类，若要继承，则需要在子类定义在同一个文件中，其他文件无法继承它。

```
sealed class Animal{
    abstract fun eat()

    fun breath(){
        println("I can breath.")
    }
    class Cat:Animal() {
        override fun eat() {
            println("I eat mouse")
        }
    }
}

class Fish: Animal(){
    override fun eat() {
        println("I drink water")
    }
}
```

密闭类可以看成一个功能强大的枚举类。