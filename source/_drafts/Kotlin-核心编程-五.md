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


### Koltlin 中的接口


在 Kotlin 中，接口可以拥有属性和默认方法：

```
interface IShow{
    val value:Int
    fun show()
    fun test(){
        println("test")
    }
}
```

反编译后的代码为：

```
public interface IShow {
   int getValue();
   void show();
   void test();
   public static final class DefaultImpls {
      public static void test(IShow $this) {
         String var1 = "test";
         boolean var2 = false;
         System.out.println(var1);
      }
   }
}
```

由于 Kotlin 是基于 Java6 的，所以是不支持在接口中对属性进行赋值以及默认方法的实现，而 Kotlin 接口中的属性其实是通过方法来实现的，所以在 Kotlin 接口中不可以为属性提供默认值，同时通过以上代码看到 Kotlin 接口中的默认方法也是通过接口内部类实现的，赤裸裸的黑科技啊。


### 类的构造方法

**主从构造方法**


在类外部定义的构造方法称为主构造方法，在类内部通过 constructor 定义的方法称为从构造方法。如果主构造方法存在注解或者可见修饰符，那么需要添加 constructor 关键字。

从构造器由两部分组成：
1. 对其他构造方法的委托。
2. {} 内部包裹的代码块。

$如果一个类存在主构造方法，那么所有的从构造方法需要直接或间接的委托给它。$

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
**为构造器参数指定默认值**

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

**为构造器参数添加修饰符**

使用 val 或者 var 声明构造方法的参数，有两个作用：
* 代表参数的引用可变性，var 可变，val 不可变。
* 简化构造类中语法

至于怎样简化，示例如下：

```
class Bird(val color:String = "white", val age : Int = 1){
        ...
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


**当构造方法的参数没有 val 或者 var 时，构造方法的参数可以在 init 语句中使用，除了 init 代码块，此时构造方法中的参数不可以在其他位置使用**。


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

一个类可以拥有多个 init 代码块，执行顺序：自上而下，多个 init 代码块可以将初始化的操作进行智能分离，在复杂业务时使逻辑更加清晰。



### 延迟初始化

延迟初始化的属性，可以在对象初始化时不必有值。


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


lateinit 主要用于声明 var 变量，**不能用于基本数据类型**，如 Int 等，如果需要的话，使用 Interger 等包装类替代。


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



Kotlin 中除了使用 final 来限制类的继承外，还可以使用密闭类来限制一个类的继承。

Kotlin 通过 sealed 关键字来修饰一个类为密闭类，若要继承，**则需要在子类定义在同一个文件中**，其他文件无法继承它。

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



### Kotlin 中的多继承问题


Java 中的类时不支持多继承的，但是可以通过接口实现多继承，但是此方式存在一个缺陷：当多个类中存在同一个方法时，那么无法完成实现的目的：

```
public interface Animal {
    void eat();
    default void kind(){
        System.out.println("i am animal");
    }
}

public interface Flyer {
    void fly();
    default void kind(){
        System.out.println("i am flyer");
    }
}

// 由于Flyer,Animal存在相同的默认方法，所以 Bird 无法定义
public class Bird implements Flyer,Animal {
    @Override
    public void eat() {
    }
    @Override
    public void fly() {
    }
}
```

但是在 Kotlin 中支持这样的写法：
```
interface Flyer {
    fun fly()
    fun kind() {
        println("i am flyer")
    }
}

interface Animal {
    fun eat()
    fun kind() {
        println("i can animal")
    }
}

class Bird : Flyer, Animal {
    override fun fly() {
    }

    override fun eat() {
    }

    // 可以通过 super 关键字指定继承哪个父类的方法
    override fun kind() = super<Flyer>.kind()
//同时也可以主动重写
//    override fun kind() {
//        println("i am bird")
//    }
}
```


**内部类解决多继承的问题**


嵌套类和内部类：

嵌套类包括静态内部类和非静态内部类，而非静态内部类被称为内部类，内部类持有外部类的引用。

在 Kotlin 中使用 inner 标记一个类为 内部类：

```
class Outer {
    private val bar: Int = 1
    class Nested {
        fun foo() = 2
    }
}
class Outer2 {
    private val bar: Int = 1
    inner class Inner {
        fun foo() = bar
    }
}
```
反编译代码：

```

public final class Outer {
   private final int bar = 1;
   // 没有使用　inner 修饰的类，会被编译成静态内部类
   public static final class Nested {
      public final int foo() {
         return 2;
      }
   }
}

public final class Outer2 {
   private final int bar = 1;
   // 使用　inner 修饰的类，会被编译成内部类
   public final class Inner {
      public final int foo() {
         return Outer2.this.bar;
      }
   }
}
```

同时可以使用　private　修饰内部类，这样其他类就不能访问该内部类，具有很好的封装性。


使用内部类实现多继承：
```
open class Horse{
    fun runFast(){
        println("run fast")
    }

    fun eat(){
        println("horse eat")
    }
}

open class Donkey{
    fun doLongTimeThing(){
        println("do longtime thing")
    }

    fun eat(){
        println("donkey eat")
    }
}

class Mule{
    fun runFast(){
        HorseC().runFast()
    }de

    fun doLongTimeThing(){
        DonkeyC().doLongTimeThing()
    }

    private inner class HorseC:Horse()
    private inner class DonkeyC:Donkey()
}
```
在　Mule　中定义两个内部类分别继承　Horse 和 Donkey，那么 Mule 可以通过内部类访问 Horse 和 Donkey 的特性，在一定程度上能够达到继承的效果。


**使用委托代替多继承**

什么是委托？

委托是一种特殊的类型，用于方法事件de委托，比如你调用 A 类的 methodA 方法，其实背后是 B 类的 methodB 方法，在 Java 中有静态委托和动态委托。


在 Kotlin 中可以使用关键字 by 来实现委托的效果，比如 by lazy 为通过委托实现延时初始化。

使用 Kotlin 中的类委托实现多继承的需求：

```
interface CanFly{
    fun fly()
}
interface CanEat{
    fun eat()
}

open class Flyer:CanFly{
    override fun fly() {
        println("can fly")
    }
}

open class Animal:CanEat{
    override fun eat() {
        println("can eat")
    }
}

class Bird(flyer: Flyer,animal: Animal):CanEat by animal,CanFly by flyer{
    fun main(args:Array<String>){
        val flyer = Flyer()
        val animal = Animal()
        val bird = Bird(flyer,animal)
        bird.fly()
        bird.eat()
    }
}
```

通过类委托，则委托类(Bird)拥有了和被委托类(Flyer、Animal)一样的状态和属性，在一定程度上实现了多继承的效果。


### 数据类

通过 data 关键字实现数据类：

```
data class Student(val name:String,var age:Int)
```
反编译 

```
public final class Student {
   @NotNull
   private final String name;
   private int age;

   @NotNull
   public final String getName() {
      return this.name;
   }

   public final int getAge() {
      return this.age;
   }

   public final void setAge(int var1) {
      this.age = var1;
   }

   public Student(@NotNull String name, int age) {
      Intrinsics.checkParameterIsNotNull(name, "name");
      super();
      this.name = name;
      this.age = age;
   }

   @NotNull
   public final String component1() {
      return this.name;
   }

   public final int component2() {
      return this.age;
   }

   @NotNull
   public final Student copy(@NotNull String name, int age) {
      Intrinsics.checkParameterIsNotNull(name, "name");
      return new Student(name, age);
   }
    ...
}
```

可以发现最终还是像 JavaBean 中一样实现 getter/setter 方法，重写 hashCode、equal 等方法，但是其中的存在的 copy、componentN 是我们从来没见过的。