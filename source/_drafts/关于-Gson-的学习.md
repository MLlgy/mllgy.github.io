---
title: 关于 Gson 的学习
tags:
---
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
