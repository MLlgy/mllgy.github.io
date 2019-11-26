---
title: ' Kotlin 核心编程(七):Kotlin 泛型，让类型更加安全'
tags:
---


### 为什么引入泛型？

* 泛型让类型更安全。

有了泛型后，不仅可以在编译期进行类型检查，在运行时还会自动转型。


**使用泛型时是否可以主动的指定类型？**

在 Java 中这样是可以的：

```
List list = new ArrayList();
```

而 Kotlin 中这样是不可以的：

```
// 这样是不可以的
val arrayList = ArrayList()
```

为什么 Java 中是可以的，因为泛型是 Java1.5 时加入的，Java 为了 **向前兼容**，可以允许声明没有具体类型参数的泛型类，而 Kotlin 是基于 Java1.6 的，不存在兼容老版本的情况，所以是不可以的。

至于 Java 中如何使用泛型，请查看:[]()

### 0x0002 如何在 Kotlin 中使用泛型

**泛型类**

```
class Test<T>{...}
```

**泛型方法**

```
fun <T> show(data:T){..}
```
**扩展函数支持泛型**

```

fun <T> ArrayList<T>.find(t: T): T? {
    val index = this.indexOf(t)
    return if (index >= 0) this.get(index) else null
}

fun main(args: Array<String>) {
    val list = ArrayList<String>()
    list.add("one")
    println(list.find("one"))
    println(list.find("two").isNullOrEmpty())
}

```



### 设定类型上限

上界约束表示方法：

```
// 不可空
class FuritPlant<T : Furit>(val t: T)
// 可空
class FuritPlantNullable<T : Furit?>(val t: T)
// 使用 where 对泛型添加多个约束条件
fun <T> cut(t:T) where T:Furit,T:Ground{
    println("i am on the ground fruit")
}
```

### 泛型背后：类型擦除

见 Java 泛型实现原理



### 由类型擦除引起的问题:如何获取到参数类型?

一般情况下，我们不用在意类型是否被类型擦除了，但是在一些场景下，我们却需要知道运行时泛型参数的类型，比如序列化/反序列化、Gson 解析时，那如何获取呢？

**主动指定泛型参数类型**

Kotlin 和 Java 都是在编译后擦除泛型参数类型，那么我们是否可以 **主动的指定参数类型** 来达到运行时获取泛型参数的类型呢？答案是肯定的。

Java 代码：

```
public class One<T> {
    private Class<T> clazz;

    public One(T t, Class<T> clazz) {
        this.clazz = clazz;
    }

    public void getType(){
        System.out.println(clazz);
    }

    public static void main(String[] args) {
        One<String> one = new One<String>("", String.class);
        one.getType();
    }
}
```

Kotlin 代码：
```
class OneKt<T>(val t:T,val clazz: Class<T>){
    fun getType(){
        println(clazz)
    }
}

fun main(args: Array<String>) {
    val one = OneKt("",String::class.java)
    one.getType()
}
```

Kotlin 和 Java 都可以获得相应泛型参数类型：
```
class java.lang.String
```

但是这种方法只适用于自定义的泛型类，我们无法对三方库中的泛型类做如此操作：

```
// 这样是不被允许的
val clazz = ArrayList<String>::class.java
```

那么是否有其他方法可以获得类型参数？答案是可以的，通过匿名内部类：

**通过匿名内部类获得泛型参数类型**
具体示例：
```
// Java 
ArrayList arrayList = new ArrayList<String>(){};
System.out.println(arrayList.getClass().getGenericSuperclass());

// Kotlin
val clazz = object :ArrayList<String>(){}
println(clazz.javaClass.genericSuperclass)
```
为什么可以通过匿名内部类获得泛型参数的类型呢？这是因为 **泛型的类型擦除并不是完全的将所有信息擦除**，而会 **将类型信息放在所属 class 的常量池中**，这样我们就可以通过相应的方式获得类型信息，而匿名内部类就可以实现这个功能。

[Java 将泛型信息存储在何处](https://stackoverflow.com/questions/937933/where-are-generic-types-stored-in-java-class-files/937999#937999)：类信息的签名中。

**匿名内部类在初始化的时候就会绑定父类或者父接口的信息，这样就能通过获取父类或父接口的泛型类型信息，来实现我们的需求**，可以通过利用此来设计一个获得所有类型信息的泛型类：
```
open class GenericsToken<T>{
    var type : Type = Any::class.java
    init {
        val superClass = this.javaClass.genericSuperclass
        type = (superClass as ParameterizedType).actualTypeArguments[0]
    }
}

fun main(args: Array<String>) {
    val oneKt = object:GenericsToken<Map<String,String>>(){}
    println(oneKt.type)
}
```

打印日志：
```
java.util.Map<java.lang.String, ? extends java.lang.String>
```
其实正是因为类型擦除的原因，在使用 Gson 反序列化对象的时候除了制定泛型参数，还需要传入一个 class ：

```
 public <T> T fromJson(String json, Class<T> classOfT) throws JsonSyntaxException { 
    ... 
 }
```
因为 Gson 没有办法根据 T 直接去反序列化，所以 Gson 也是使用了相同的设计，通过匿名内部类获得相应的类型参数，然后传到 fromJson 中进行反序列化。

看一下在 Kotlin 中我们使用 Gson 来进行泛型类的反序列化：

```
val  json = "...."
val rType = object: TypeToken<List<String>>(){}.type
val stringList = Gson().fromJson<List<String>>(json,rType)
```

在 Kotlin 中除了使用匿名内部类获得泛型参数外，还可以使用内联函数来获取。

**使用内联函数获取泛型的参数类型**

使用内联函数获取泛型的参数类型十分的简单，只要加上 reified 关键字，意思是：在编译时会将具体的类型插入相应的字节码中，那么在运行时就可以获得对应的参数类型了。

reified 为 Kotlin 中的一个关键字，还有一个叫做 inline，后者可以将函数定义为内联函数，前者可以将内联函数的泛型参数当做真实类型使用，
可以借此来对 Gson 进行扩展：

```
inline fun <reified T> Gson.fromJson(json: String): T{ 
     return fromJson(json, T::class.java) 
 } 
```

有了此扩展方法，就无须在 Kotlin 当中显式的传入一个 class 对象就可以直接反序列化 json 了：

```
 class Person(var id: Int, var name: String) 
 　 
 fun test(){ 
     val person: Person = Gson().fromJson("""{"id": 0, "name": "Jack" }""") 
 }
```

由于是内联的方法调用，T 的类型在编译时就可以确定了，反编译之后的代码：

```
 public static final void test() { 
   Gson $receiver$iv = new Gson(); 
   String json$iv = "{\"id\": 0, \"name\": \"Jack\" }"; 
   Person person = (Person)$receiver$iv.fromJson(json$iv, Person.class); 
 }
```
这就是 Kotin 的泛型被称为真泛型的原因。

### Kotlin 型变


#### Java 中的型变


? extends E 其实就是使用点协变，允许传入的参数可以是泛型参数类型为 Number 子类的任意类型。

? super E 使用点逆变，这表示元素类型为 E 及其父类，这个通常也叫作逆变。

型变包括 协变、逆变、不变 三种。

**为什么 Java 中 List 是不变的**

List<String> 是不能赋值给 List<Object> 的，这是因为 List 是不变的，即两者没有任何关系，我们可以使用反证法来说明：

假设 List<String> 可以赋值给 List<Object>,那么将会出现以下情况：

```
List<String> stringList = new ArrayList<String>();
List<Object> objList = stringList;// 假设这样是可以的
objList.add(Integer(1));
String str = stringList.get(0);// 将会出错，所以假设错误
```

如果假设成立的话，那么 List 将不再保证类型安全，而 Java 设计师明确保证泛型的最基本原则就是保证类型安全，所以不支持这种行为。


**为什么 Kotlin 中的 List 支持协变？**

在 Kotlin 中是允许 List<String> 赋值给 List<Any> 的，这是因为 Java 和 Kotlin 中的 List 不是同一个：

```
// 关键字 out 
public interface List<out E> : Collection<E> {}
```

在 Kotlin 中，如果定义的泛型类和泛型方法的泛型参数前面加上 out 关键字，那么就说明这个泛型和泛型方法是协变的，说明这是一个只读列表。在 Java 中也可以声明泛型协变，用通配符及泛型上界来实现协变：< ? extend Object> 一样。因为这个 List 是协变的，所以它将无法添加元素，只能从里面获取元素,要验证此结论，同样我们可以使用反证法：

```
val stringList: List<String> = ArrayList<String>()
val anyList: List<Any> = stringList
anyList.add(1)
val str:String = anyList.get(0)// 无法转换成 String，同样违背了泛型类型安全的原则
```

**同时，假如一个泛型类 Generic<out T> 支持协变，那么它里面方法的参数不能使用 T 类型，因为一个方法的参数不能使用传入参数父类型的对象，**因此，使用该关键字的泛型类，可以叫做一个 **可读、可写功能受限的类型**。


**Kotlin 中的逆变**

在 Kotlin 中存在 in 关键字，它使泛型有了另一个特性：逆变。如果类型 A 是类型 B 的子类型，那么 Generics<B> 反过来是 Generics<A> 的子类型。与 out 相反，使用该关键字的泛型类，可以看做一个 **可写、可读功能受限的类型**。

表示支持逆变的泛型类：Generic<in T>


**型变的使用**


假设有这么一个场景：将 Double 的数组拷贝到另一个 Double 的数组上，具体试一下：

```
fun copy(dest:Array<Double>,src:Array<Double>){...}
```

但是这个函数只能满足拷贝 Double 的数据类型，如果是 String 的话，又要重新写一个方法，学过了泛型就利用起来：

```
fun <T> copy(dest:Array<T>,src:Array<T>){...}
```

但是如果我们要把 Array<Double> 拷贝到 Array<Number> 呢？上面的方法不可以了，学习了型变以后，用起来：

```
//in 版本
fun <T> copy(dest:Array<in T>,src:Array<T>){...}

// out 版本
fun <T> copy(dest:Array<T>,src:Array<out T>){...}
```

Kotlin 和 Java 中的型变对比

||协变|逆变|不变|
--|--|--|--
Kotlin|实现方式： <out T>，只能作为消费者，只能读取不能添加|实现方式：<in T>，只能作为生产者，只能添加，不能读取|实现方式:<T>,既可以添加，也可以读取
Java|实现方式：<? extends T>，只能作为消费者，只能读取不能添加|实现方式：<? super T>,只能作为生产者，只能添加，不能读取|同上


### Kotlin 中的通配符

与 Java 中的通配符表示方法为问号(?)不同，Kotlin 中表示方法为星号(*)。


示例：

```
val list:MutableList<*> = ...
list.add(2)// 出错
```

以上代码出错，根据通配符的含义 list 不是应该可以添加任意元素吗？其实不是这样的，MutableList<*> 与 MutableList<Any?> 不同，后者可以添加任意元素，而前者只能匹配某一种类型，但是编译器不知道是一种什么样的类型，所以它不允许添加元素，和泛型协变类似，其实通配符是一种语法糖，背后也是通过协变来实现的，所以
MutableList<*> 本质上是 MutableList<out Any?> 。