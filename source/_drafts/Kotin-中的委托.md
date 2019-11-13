---
title: Kotin 中的委托
tags:
---


### 类委托

类委托实现如下:

```
// 创建接口
interface Base {   
    fun print()
}

// 实现此接口的被委托的类
class BaseImpl(val x: Int) : Base {
    override fun print() { print(x) }
}

// 通过关键字 by 建立委托类
class Derived(b: Base) : Base by b

fun main(args: Array<String>) {
    val b = BaseImpl(10)
    Derived(b).print() // 输出 10
}
```

通过 by 实现了委托关系，则委托类拥有了和被委托类一样的状态和行为。


如果不使用 by 实现委托，而通过 Java 中的静态委托实现如下：


```
// 创建接口
interface Base {
    fun print()
}

// 实现此接口的被委托的类
class BaseImpl(val x: Int) : Base {
    override fun print() { print(x) }
}

// 通过方法调用
class Derived(private val b:Base){
    fun printSomthing(){
        b.print()
    }
}

fun main(args: Array<String>) {
    val b = BaseImpl(10)
    Derived(b).printSomthing()
}

```

可以看到在 Derived 中我们需要实现单独的方法，并在方法中调用被委托类的方法，以此来实现委托，而通过 by 实现委托则更为简洁。通过反编译代码，看到 by 的底层其实也是通过静态委托来实现的。