---
title: Kotlin 单例模式的实现
date: 2019-05-24 18:35:39
tags: [Kotlin,Singleton]
---




#### 最简单的实现方式


```
object Singleton {
  fun method() {}
}
```

调用单例

```
//在Kotlin里调用
Singleton.method()

//在Java中调用
Singleton.INSTANCE.method();
```

其实这种方式的实现为 Java 中的饿汉式,在 AS 的 Kotlin 相关的工具对 object 类进行编译和反编译，可以看到起对应的 Java 代码如下：

<!-- more -->

```
public final class Singleton {
   public static final Singleton INSTANCE;

   public final void method() {
   }

   private Singleton() {
   }

   static {
      Singleton var0 = new Singleton();
      INSTANCE = var0;
   }
}
```

和 Java 中的 饿汉式获得单例方式相同。

同样也与 Java 中饿汉方式获得单例拥有相同的问题：

* 如果没有使用该类的对象，那么已经实例化的对象，造成了内存浪费。
* 当构造器中执行复杂操作，造成实例化过程缓慢，可能造成性能问题。

#### 懒汉式 一

在 Kotlin 中单例的实现和 Java 中一样，必须有：
1. 私有化的构造函数
2. 获得对象的方法


那么在 Kotlin 中有：

* 显式声明构造方法为 private
* companion object 用来在 class 内部声明一个对象
```
class Singleton private constructor(){
    companion object {

        @Volatile
        private var instance: Singleton? = null

        fun getInstance() =
                instance ?: synchronized(this) {
                    instance ?: Singleton().also { instance =it }
                }
    }
}
```

此写法 [Java 单例模式的几种写法](https://leegyplus.github.io/2019/03/13/%E5%8D%95%E4%BE%8B%E7%9A%84%E5%87%A0%E7%A7%8D%E5%86%99%E6%B3%95/) 中懒汉模式的 **方式三** 相同。


调用方式：

```
//Kotlin 调用
val singleton = Singleton.getInstance()

//Java 调用
Singleton singleton = Singleton.Companion.getInstance()
```

#### 懒汉式 二


```
class Singleton private constructor(){
    companion object {
        val instance: Singleton by lazy { Singleton() }
    }
}
```


* `Singleton` 通过 `lazy`来实现懒汉式加载。
* `lazy` 默认情况下是 **线程安全** 的，这就可以避免多个线程同时访问生成多个实例的问题。

所以就不用象方式一中那样，对 Singleton 对象进行 **加同步锁** 和 **双重校验** 操作。