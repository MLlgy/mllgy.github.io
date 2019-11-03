---
title: Kotlin 核心编程(十)
tags:
---


### 0x0001 多态的种类

**1. 子类型多态**

通过类之间的继承来实现，比如说 Java 层里的多态就是子类型多态。


**2. 参数多态**

最常见的示例：泛型：

```
fun <T: Animal> show(t: T){
    ...
}
```

**3. 对第三方类进行扩展**

如果说为常见的 Json 进行扩展函数：

```
fun Json.show(){
    ...
}
```

通过扩展函数，可以避免被扩展的第三方类被污染。

**4. 特设多态和运算符重载**

特设多态提供了量身定制的能力，可以理解为：一个 **多态函数** 是有多个不同的实现，根据 **实参的签名的不同** ，而 **调用不同版本的函数**。

借助 **运算符重载** 可以实现这种需求：
```
data class Area(val value: Double)

operator fun Area.plus(that: Area): Area {
    return Area(this.value + that.value)
}

operator fun Area.plus(value: Double): Area {
    return Area(this.value + value)
}

fun main(args: Array<String>) {
    println(Area(1.0) + Area(2.0))// 实参的类型为 Area，所以调用 Area.plus(that: Area)
    println(Area(2.0) + 3.0)// 实参的类型为 Double，所以调用  Area.plus(value: Double) 
}
```

上面的示例中存在两个关键字，分别是 `operator`和 `plus`:
* operator 它的作用是：将一个函数标记为重载一个操作符或者实现一个约定，
* plus 是 **Kotlin 规定的函数名**，除了重载加法，还可以重载减法(minus)、乘法(times)、除法(div)、取余(rem) 等函数来实现重载操作符。


运算符重载的具体实现，反编译以上代码：
```
public final class OneKt {
   @NotNull
   public static final Area plus(@NotNull Area $this$plus, @NotNull Area that) {
      Intrinsics.checkParameterIsNotNull($this$plus, "$this$plus");
      Intrinsics.checkParameterIsNotNull(that, "that");
      return new Area($this$plus.getValue() + that.getValue());
   }

   @NotNull
   public static final Area plus(@NotNull Area $this$plus, double value) {
      Intrinsics.checkParameterIsNotNull($this$plus, "$this$plus");
      return new Area($this$plus.getValue() + value);
   }

   public static final void main(@NotNull String[] args) {
      Intrinsics.checkParameterIsNotNull(args, "args");
      Area var1 = plus(new Area(1.0D), new Area(2.0D));
      boolean var2 = false;
      System.out.println(var1);
      var1 = plus(new Area(2.0D), 3.0D);
      var2 = false;
      System.out.println(var1);
   }
}
```
可以看到运算符重载的实现还是通过 Java 类中方法的重载来实现的.

### 0x0002 扩展函数及属性

**1. 扩展函数**

声明在 Kotlin 文件中的扩展函数，基本实现方式：

```
fun MutableList<Int>.exchange(fromIndex: Int, toIndex: Int) {
    val temp = this[fromIndex]
    this[fromIndex] = this[toIndex]
    this[toIndex] = temp
}
```

扩展函数的实现，反编译以上代码：

```
public final class OneKt {   
   public static final void exchange(@NotNull List $this$exchange, int fromIndex, int toIndex) {
      Intrinsics.checkParameterIsNotNull($this$exchange, "$this$exchange");
      int temp = ((Number)$this$exchange.get(fromIndex)).intValue();
      $this$exchange.set(fromIndex, $this$exchange.get(toIndex));
      $this$exchange.set(toIndex, temp);
   }
}
```

可以看到扩展方法最终会被编译为静态方法，**所以扩展方法是全局可用的**，并且扩展方法的类型接收者会被传入生成的方法中，如上方法的第一个参数。



**2. 扩展属性**


声明在文件中的扩展属性：

```
val MutableList<Int>.sumIsEven:Boolean
    get() = this.sum() % 2 == 0// 访问器
```
一般的，**扩展属性需要提供访问器**，具体实现见反编译后的代码：

```
public final class OneKt {
   public static final boolean getSumIsEven(@NotNull List $this$sumIsEven) {
      Intrinsics.checkParameterIsNotNull($this$sumIsEven, "$this$sumIsEven");
      return CollectionsKt.sumOfInt((Iterable)$this$sumIsEven) % 2 == 0;
   }
}
```
可以看到扩展属性的实现和扩展函数的实现方式是一样的，通过生成对应的方法。

**扩展函数、属性的作用域**

其实以上的扩展函数是声明在 one.kt 文件中，所以反编译后的代码如上所示，其为 **全局可用的**，但是如果将扩展方法声明在类的内部，那么此时的扩展方法只能在 **该类和该类的子类** 中使用。

一般的，为了管理方便，我们会将所有的扩展函数放在 Kotlin 文件中，但是如果程序需要，也可以声明类中，避免其他类实例调用。


**扩展属性不可以指定默认值**，由于扩展函数的实现其实和扩展函数一样，生成对应的函数，所以扩展的属性并没有真正的插入到具体的类中，幕后字段对扩展属性是无效的，所以扩展属性不能指定默认值。

> 幕后字段：在 Kotlin 中，如果属性存在访问器使用默认实现，那么 Kotin 会自动提供幕后字段 field，以避免递归访问属性字段，而该 field 仅可用于自定义的 getter 和 setter 方法。

**3. 特殊情况**


1. 声明静态的扩展函数

前文说到如果将扩展函数定义在类内部，那么该扩展函数只能被该类以及其子类访问，那么现在如果我想为该类扩展一个静态函数，该怎么操作？

要想为一个类声明一个静态的扩展函数，那么该 **函数必须要定义在该类的伴生对象上**，具体示例如下：

```

class Son{
    companion object{
        var age = 10
    }
}

fun Son.Companion.showInfo(){
    println(this.age)
}

fun main(args: Array<String>) {
    // 进行调用
    Son.showInfo()
}
```

2. 成员方法优先级高于扩展方法，字面意思，好理解。


3. 强制指定接收者类型
```
class Baby {
    fun foo() {
        println("i am baby")
    }
}

object Parent {
    private fun foo() {
        println("i am parent")
    }

    @JvmStatic
    fun main(args: Array<String>) {
        fun Baby.foo2() {
            this.foo()
            this@Parent.foo()
        }
        Baby().foo2()
    }
}
```

在上文中指出扩展函数中的 this 为接受者类型对象，但是可以通过 `this@类名` 的形式指定 this 所代指的对象，打印日志为：

```
i am baby
i am parent
```


### 0x0003 标准库中的扩展函数


**run**

```
public inline fun <T, R> T.run(block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```


具体实现：


```
class Baby {
    val name = "aline"
    fun foo() {
        println("i am baby")
        run {
            println(this.name)
        }
    }
}
```
类型接收者对象 this 为 Baby。


**let**

```
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block(this)
}
```
具体使用：

```
var name:String? = null
name?.let {
    val len = it.length
    len
}
```



**apply**

```
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```

具体使用：
```
var name:String? = null
name?.apply {
    println(it.length)
}
```

apply 和 let 使用类似，通过其扩展函数的声明，其不同的是： apply 返回者为本身，而 let 的返回对象为闭包中的返回值。

**also**

```
public inline fun <T> T.also(block: (T) -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block(this)
    return this
}
```

具体实现：

```

```


在扩展函数的函数体中，this 代表的是接收者类型的对象，至于扩接收者类型是什么，它就是扩展函数名前的接口和类的名称。

为什么用 it ，Lambda 中的语法糖，当其参数只有一个时，可以使用 it 代替。