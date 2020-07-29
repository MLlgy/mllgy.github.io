---
title: ' Kotlin 核心编程(七):Kotlin 泛型，让类型更加安全'
date: 2019-12-03 11:32:19
tags: [Kotlin 核心编程,读书笔记,,Kotlin]
---



### 0x0001 为什么引入泛型？

* 泛型让类型更安全。

有了泛型后，不仅可以在编译期进行类型检查，在运行时还会自动转型。


**使用泛型时是否可以主动的指定类型？**

在 Java 中这样是可以的：
<!-- more -->
```
List list = new ArrayList();
```

而 Kotlin 中这样是不可以的：

```
// 这样是不可以的
val arrayList = ArrayList()
```

为什么 Java 中是可以的，因为泛型是 Java1.5 时加入的，Java 为了 **向前兼容**，可以允许声明没有具体类型参数的泛型类，而 Kotlin 是基于 Java1.6 的，不存在兼容老版本的情况，所以是不可以的。

至于 Java 中如何使用泛型，请查看:[Java泛型初探一 之 泛型类 、泛型接口 、泛型方法](https://www.jianshu.com/p/ff6618f7c840).

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



### 0x0003 设定类型上限

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

### 0x0003 泛型背后：类型擦除


具体内容参见：[Java 泛型:深入理解泛型的类型擦除](https://leegyplus.github.io/2018/11/20/Java-%E6%B3%9B%E5%9E%8B-%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%E6%B3%9B%E5%9E%8B%E7%B1%BB%E5%9E%8B%E6%93%A6%E9%99%A4/)


### 0x0004 由类型擦除引起的问题:如何获取到参数类型?

一般情况下，我们不用在意类型是否被类型擦除了，但是在一些场景下，**我们却需要知道运行时泛型参数的类型**，比如序列化/反序列化、Gson 解析时，那如何获取呢？

**主动指定泛型参数类型**

Kotlin 和 Java 都是在编译后擦除泛型参数类型，那么我们是否可以 **主动的指定参数类型** 来达到运行时获取泛型参数的类型呢？答案是肯定的。


1. **自定义类获得泛型参数类型**
  
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

2. **通过匿名内部类获得泛型参数类型**
   

具体示例：
```
// Java 
ArrayList arrayList = new ArrayList<String>(){};
System.out.println(arrayList.getClass().getGenericSuperclass());

// Kotlin
val clazz = object :ArrayList<String>(){}// Kotlin 的匿名内部类
println(clazz.javaClass.genericSuperclass)
```

为什么可以通过匿名内部类可以在运行期获得泛型参数的类型呢？这是因为 **泛型的类型擦除并不是完全的将所有信息擦除**，而会 **将类型信息放在所属 class 的常量池中**，这样我们就可以通过相应的方式获得类型信息，而匿名内部类就可以实现这个功能。

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
    // 创建一个匿名内部类
    val oneKt = object:GenericsToken<Map<String,String>>(){}
    println(oneKt.type)
}
```

打印日志：
```
java.util.Map<java.lang.String, ? extends java.lang.String>
```

至于如果获得参数化类型，可参见此博客：[ParameterizedType应用，java反射，获取参数化类型的class实例](https://blog.csdn.net/datouniao1/article/details/53788018)

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
val rType = object: TypeToken<List<String>>(){}.type// 获得反序列化的数据类型
val stringList = Gson().fromJson<List<String>>(json,rType)
```

当然可以直接传输数据类型：

```
// 存在局限，比如不能传入 List<String> 的数据类型
val stringList = Gson().fromJson<String::class.java>(json,rType)
```

在 Kotlin 中除了使用匿名内部类获得泛型参数外，还可以使用内联函数来获取。

3. **使用内联函数获取泛型的参数类型**

内联函数的特征：

内联函数(`inline`)在编译时会将具体的函数字节码插入调用的地方，类型插入相应的字节码中，这就意味着泛型参数类型也会被插入到字节码中，那么就可以实现在运行时就可以获得对应的参数类型了。

使用内联函数获取泛型的参数类型十分的简单，只要加上 **`reified`** 关键字，意思是：在编译时会将 **具体的类型** 插入到相应的字节码中，那么就可以获得对应参数的类型，与 Java 中的泛型在编译器进行类型擦除不同，Kotlin 中使用 **reified**  修饰泛型，**该泛型类型信息不会被抹去**，所以 Kotiln 中的该泛型为 **真泛型**。

> **reified** 为 Kotlin 中的一个关键字，还有一个叫做 **inline**，后者可以将函数定义为内联函数，前者可以将内联函数的泛型参数当做真实类型使用.


可以借此来为 Gson 定义一个扩展函数：

```
inline fun <reified T
: Any> Gson.fromJson(json: String): T{ 
     return fromJson(json, T::class.java) 
 } 
```

有了此扩展方法，就无须在 Kotlin 当中显式的传入一个 class 对象就可以直接反序列化 json 了：

```
 class Person(var id: Int, var name: String) 
 　 
 fun test(){ 
     val person: Person = Gson().fromJson<User>("""{"id": 0, "name": "Jack" }""") 
 }
```

由于 `Gson.fromJson` 是内联函数，方法调用时插入调用位置，T 的类型在编译时就可以确定了，反编译之后的代码：

```
 public static final void test() { 
   Gson $receiver$iv = new Gson(); 
   String json$iv = "{\"id\": 0, \"name\": \"Jack\" }"; 
   Person person = (Person)$receiver$iv.fromJson(json$iv, Person.class); 
 }
```
这就是 Kotin 的泛型被称为 **真泛型** 的原因。

但是 refied 存在一个问题：`reified` 只能修饰方法，而当定义一个泛型类时，reified 是无法通过类似以上的方式获得泛型参数的，但是仍然可以通过其他方式获得泛型类中的泛型参数类型，具体如下：

```
class View<T>(val clazz:Class<T>){
    val presenter by lazy { clazz.newInstance() }
    companion object{
        // 在构造函数执行之前，执行了此处，真泛型的重载函数。
        inline operator fun <reified T> invoke() = View(T::class.java)
    }
}

class Presenter

fun main(args: Array<String>) {
    // 两者等效，具体实现如下
    val p = View<Presenter>().presenter
    val a  = View.Companion.invoke<Presenter>().presenter
}
```

这种写法特别适合在 android 中的 MVP，不用再在 Activity 中显式的显示 Presenter 的类名。

**实现一个 Android MVP 框架**

Model 层：

```
data class User(var id: Int, var name: String)
```

Presenter 层：
```
interface IPresenter {
    fun doLogin(): User
}
```

View 层：
```
interface IView {
    fun getLayoutID(): Int
}
```



第一种方式：


```
// 第一种方式，通过动态代理 by 
class MainActivity : AppCompatActivity(),
        IView by MVPView(), IPresenter by EmptyPresenter() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(getLayoutID())

        findViewById<Button>(R.id.button).setOnClickListener {
            getPresenter<EmptyPresenter>().doLogin()
                // 第一种方法，通过动态代理，可以直接调用 EmptyPresenter 的方法
               doLogin() 
        }
    }
}

class EmptyPresenter : IPresenter {
    override fun doLogin(): User {
        //执行各种逻辑
        return User(1, "zhangtao")
    }
}

class MVPView : IView {
    override fun getLayoutID() = R.layout.activity_main
}
```

第二种：


```
open class BaseActivity<T>(val clazz: Class<T>) : AppCompatActivity() {
    val presenter by lazy { clazz.newInstance() }

    companion object {
        inline operator fun <reified T> invoke() = BaseActivity(T::class.java)
    }
}

class MainActivity : BaseActivity<EmptyPresenter>(EmptyPresenter::class.java),
    IView by MVPView() {
    
    // 删除  getPresenter 的逻辑

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(getLayoutID())

        findViewById<Button>(R.id.button).setOnClickListener {
            // 直接获得 present 对象
            presenter.doLogin()
        }
    }
}

class EmptyPresenter : IPresenter {
    override fun doLogin(): User {
        //执行各种逻辑
        return User(1, "zhangtao")
    }
}
```


第三种方式：

如果不想继承 BaseActivity，则可以按照下面的逻辑：

```

class MainActivity : AppCompatActivity(),
    IView by MVPView() {

    inline fun <reified T : IPresenter> getPresenter(): T {
        return T::class.java.newInstance()
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(getLayoutID())

        findViewById<Button>(R.id.button).setOnClickListener {
            getPresenter<EmptyPresenter>().doLogin()
        }
    }
}
```


### 0x0005 Kotlin 型变


#### Java 中的型变


`? extends E `其实就是使用 **点协变**，允许传入的参数可以是 **E及其子类** 的任意类型。

`? super E `使用 **点逆变**，这表示元素类型 **为 E 及其父类**，这个通常也叫作逆变。

型变包括 协变、逆变、不变 三种。

**为什么 Java 中 List 是不变的 ?**

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

在 Kotlin 中是允许 List<String> 赋值给 List<Any> 的，这是因为 Java 和 Kotlin 中的 List 不是同一个，在 Kotlin 中重新定义了 List 接口，具体如下：

```
// 关键字 out 
public interface List<out E> : Collection<E> {}
```

在 Kotlin 中，如果定义的泛型类和泛型方法的泛型参数前面加上 **out 关键字**，那么就说明这个泛型和泛型方法是协变的，说明这是一个 **只读列表**。

在 Java 中也可以声明泛型协变，用通配符及泛型上界来实现协变：< ? extend Object> 一样。因为这个 List 是协变的，所以它将 **无法添加元素，只能从里面获取元素** ,要验证此结论，同样我们可以使用反证法：

```
val stringList: List<String> = ArrayList<String>()
val anyList: List<Any> = stringList
anyList.add(1)
val str:String = anyList.get(0)// 无法转换成 String，同样违背了泛型类型安全的原则
```

**同时，假如一个泛型类 Generic<out T> 支持协变，那么它里面方法的参数不能使用 T 类型，因为一个方法的参数不能使用传入参数父类型的对象**，因此，使用该关键字的泛型类，可以叫做一个 **可读、可写功能受限的类型**。


**Kotlin 中的逆变**

在 Kotlin 中存在 `in` 关键字，它使泛型有了另一个特性：**逆变**。何为逆变？逆变就是：类型 A 是类型 B 的子类型，那么 `Generics<B>` 反过来是 `Generics<A>` 的子类型。与 out 相反，使用该关键字的泛型类，可以看做一个 **可写、可读功能受限的类型**。

表示支持逆变的泛型类：

`Generic<in T>`


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

注意 in 和 out 的位置，修饰不同的类型参数。

Kotlin 和 Java 中的型变对比

||协变|逆变|不变|
--|--|--|--
Kotlin|实现方式：`<out T>`，只能作为消费者，只能读取不能添加|实现方式：`<in T>`，只能作为生产者，只能添加，不能读取|实现方式: `<T>` ,既可以添加，也可以读取
Java|实现方式：<? extends T>，只能作为消费者，只能读取不能添加|实现方式：<? super T>,只能作为生产者，只能添加，不能读取|同上


### 0x0006 Kotlin 中的通配符

与 Java 中的通配符表示方法为 **问号(?)** 不同，Kotlin 中表示方法 **星号(*)**。


示例：

```
val list:MutableList<*> = mutableListOf(1,"test")
list.add(2)// 出错
```

以上代码出错，根据通配符的含义 list 不是应该可以添加任意元素吗？

其实不是这样的，`MutableList<*>` 与 `MutableList <Any?>` 不同，后者可以添加任意元素，**而前者只能匹配某一种类型**，但是编译器不知道是一种什么样的类型，所以它不允许添加元素，和泛型协变类似，其实通配符是一种语法糖，背后也是通过协变来实现的，所以MutableList<*> 本质上是 `MutableList <out Any?>` 。

