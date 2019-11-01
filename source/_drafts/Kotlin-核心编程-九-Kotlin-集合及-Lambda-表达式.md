---
title: 'Kotlin 核心编程(九):Kotlin 集合及 Lambda 表达式'
tags:
---

虽然 Lambda 表达式使用起来十分的优雅、简洁，但是在 Kotlin 中使用 Lambda 表达式会有一些额外的开销，而这个问题可以使用内联函数解决。

Kotlin 集合中的 API 大量使用了 Lambda。

### Kotlin 调用 Java 的函数式接口

Kotlin 调用 Java 的函数式接口在之前的相关篇章以及提及，如下所示：

Java 的函数式接口
```
interface IClick{
    void onClick()
}

view.setIClickListener(new IClick(){
    @Override
    public void onClik(){
        ....
    }
})
```


以上例子在 Kotlin 中做如下调用：

```
view.setIClickListener(object:IClick{

    override public void onClik(){
        ....
    }
})
```
使用 Lambda 语法进行简化：

```
view.setIClickListener({
    ...
})
```
由 Kotlin 语法糖的存在：

```
view.setIClickListener{
    ...
}
```

### 带接收者的 Lambda  这个是什么？？ 存疑


通过之前的内容我们知道了函数的类型，比如以下：

```
(Int) -> String// 参数类型为 Int ，返回值类型为 String 的函数
(Int) -> (Int,String) -> String 
```

同时在 Kotlin 中还允许定义带有参数的函数类型：

```
Int.(Int) -> Int
```

具体应用：

```
val sum:Int.(Int) -> Int = {other -> plus(other)}
```


**类型安全构造器：**

```
class HTML{
    fun body(){
        println("body")
    }

    fun header(){
        println("head")
    }
    fun content(num:Int):Int{
        println("content $num")
        return 2
    }

    fun showText(a:Int):String ="showText"
}

fun html(init:HTML.() -> Unit):HTML{
    val html = HTML()//创建接受者
    html.init()// 把接受者对象传给 Lambda
    return html
}

fun html2(init:HTML.() -> String):HTML{
    val html = HTML()
    html.init()
    return html
}

fun <R> html4(init:HTML.() -> R):R{
    val html = HTML()
    val result = html.init()
    return result
}

fun main(args: Array<String>) {
    html { body() }// 调用接受者对象的 body 方法，调用 1
    html { header() }// 调用 2，往下依次类推
    html { content(2) }
    html { showText(2) }

//    html2 { body() }
//    html2 { header() }
//    html2 { content(2) }
    html2 { showText(2) }

//    html3 { body() }
//    html3 { header() }
    html3 { content(2) }
//    html3 { showText(3) }   

    val result = html4 { showText(3)
    "3435"}
    println(result)
}
```
这个概念虽然看示例挺容易看懂，但是概念确实是难以理解，经过验证有以下结论：

`init:HTML.() -> Unit`: 其中 ` HTML` 代表 Lambda 表达式 init 为 HTML 类中的函数，而 `Unit` 代表 Lambda 表达式的返回值，这里需要注意的是，当此处声明返回值类型为 Unit 时，可以调用 HTML 中返回值类型为各个类型的函数，所以 调用 1~4 均是可以的，但是当声明返回值类型为非 Unit 时，只能调用返回值类型相同的函数，所以调用 5、6、7、9、10、12 是非法调用。需要注意调用 13 是可以有返回值的。


其实我们常见的 with、apply 就是这样的函数，正式因为这种机制，我们可以在相应的闭包中使用 this 代指调用接收者,并且 this 可以省略，但是不同的是 with 可以返回自由类型，如上例中的 html4，with、apply 的使用示例：
```
val html = HTML()
with(html){
    body()
    header()
    content(2)
    showText(2)
    "over"
}
html.apply {
    body()
    header()
    content(2)
    showText(3)
}
```

以下是 with 和 apply 的源码，通过源码更深理解两者的使用：

```
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}

public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```

### 使用内联函数优化 Kotlin 中 Lambda 表达式额外的开销

Kotlin 中 Lambda 的额外开销：
在 Kotliln 声明的每一个 Lambda 会在字节码中产生一个匿名类，每次调用都会创建一个新的对象，所以存在额外开销。

**内联函数：**
使用 inline 关键字来修饰函数，这些函数就成为了内联函数。内联函数的函数体在编译期会被嵌入每一个调用的地方，以免减少额外生产的匿名类数量，同时减少函数执行的时间开销。

示例代码：

```
fun main(args: Array<String>) {
    foo { println("block") }
}

fun foo(block:()->Unit){
    println("before block")
    block()
    println("after block")
}
```
代码反编译后的相关代码：

```
public final class TwoKt {
   public static final void main(@NotNull String[] args) {
      Intrinsics.checkParameterIsNotNull(args, "args");
      foo((Function0)null.INSTANCE);
   }

   public static final void foo(@NotNull Function0 block) {
      Intrinsics.checkParameterIsNotNull(block, "block");
      String var1 = "before block";
      System.out.println(var1);
      block.invoke();
      var1 = "after block";
      System.out.println(var1);
   }
}
```

通过前面的内容，我们知道 Lambda 表达式会生成相应的匿名内部类，在这里调用 foo 会产生一个 Function0 类型的 block 类，通过调用其 invoke 方法来执行，这就是所说的增加的额外的生成类和调用开销。

使用 inline 关键子修饰 foo 函数，如下：
```
inline fun foo(block:()->Unit){
    println("before block")
    block()
    println("after block")
}
```

反编译后的代码：

```
public final class TwioKt {
   public static final void main(@NotNull String[] args) {
      Intrinsics.checkParameterIsNotNull(args, "args");
      String var1 = "before block";
      System.out.println(var1);
      // block 函数体从这里开始粘贴
      String var2 = "block";
      System.out.println(var2);
      // block 函数体从这里结束粘贴
      var1 = "after block";
      System.out.println(var1);
   }

   public static final void foo(@NotNull Function0 block) {
      Intrinsics.checkParameterIsNotNull(block, "block");
      String var2 = "before block";
      System.out.println(var2);
      block.invoke();
      var2 = "after block";
      System.out.println(var2);
   }
}
```
可以看到 foo 函数体代码以及被调用的 Lambda 代码都粘贴到了相应的调用位置，从而减少匿名类的生产和调用开销。但是内联函数存在的一个问题是会增加空间复杂度，通过空间换取时间上的优势。

使用内联函数需要注意：
* 普通函数不需要使用 inline 关键字修饰。
* 避免对具有大量函数体的函数进行内联，会导致过多的字节码数量。
* 函数被定义为内联函数，则不能访问闭包类中的私有成员，除非声明为 internal。

### 使用 online 避免参数被内联

通过上节可是看到内联函数的整个函数会被粘贴到调用函数中，但是

存在这样一种情况：函数接收多个参数，我们只想对部分 Lambda 参数进行内联，而其他不内联，应该如何操作。


针对以上情况，我们可以使用关键字 noline 来修饰不想内联的参数，那么该参数就不会有内联效果。我们对以上的示例进行修改：

```
fun main(args: Array<String>) {
    foo ({ println("block1")},{ println("block2")}, "tree") 
}

inline fun foo(block:()->Unit,noinline block2: () -> Unit,mess:String){
    println("before block")
    block()
    block2()
    println(mess)
    println("after block")
}
```

反编译后的代码：

```
public final class TwioKt {
   public static final void main(@NotNull String[] args) {
      Intrinsics.checkParameterIsNotNull(args, "args");
      Function0 block2$iv = (Function0)null.INSTANCE;
      String mess$iv = "tree";
      String var3 = "before block";
      System.out.println(var3);
      String var4 = "block1";
      System.out.println(var4);
      block2$iv.invoke();
      System.out.println(mess$iv);
      var3 = "after block";
      System.out.println(var3);
   }

   public static final void foo(@NotNull Function0 block, @NotNull Function0 block2, @NotNull String mess) {
      Intrinsics.checkParameterIsNotNull(block, "block");
      Intrinsics.checkParameterIsNotNull(block2, "block2");
      Intrinsics.checkParameterIsNotNull(mess, "mess");
      String var4 = "before block";
      System.out.println(var4);
      block.invoke();
      block2.invoke();
      System.out.println(mess);
      var4 = "after block";
      System.out.println(var4);
   }
}
```

使用 online 修饰的 block2 Lambda 表达式， 在调用时并没有将其函数体复制到调用处。



### 非局部返回和具体化参数类型



**使用 inline 实现非局部返回**

```
fun main(args: Array<String>) {
    // foo { return} 促成为非法调用，Lambda 中不允许 return 关键字出现
}


fun foo(block:() -> Unit){
    println("before block")
    block()
    println("after block")
}
```
此时通过 inline 修饰 foo 函数：

```
fun main(args: Array<String>) {
    foo { return} 
}


inline fun foo(block:() -> Unit){
    println("before block")
    block()
    println("after block")
}
```

但是打印日志如下：
```
before block
```

只执行 block 上面的操作，原因很容易理解，使用 inline 将代码进行替换，那么 return 在编译期会出现在 main 函数中，当然会针对全局生效。

**使用 inline 实现具体化参数类型**

其实这部分内容在 Kotlin 泛型提及过，在此处探究器原因。

和 Java 一样，由于运行时存心类型擦除，所以不能直接获取一个参数的类型，而使用内联函数会直接在字节码中生成相应的函数体，这种情况下可以获得参数类型，可以通过关键字 refied 实现这一效果。

```
fun main(args: Array<String>) {
    getType<String>()
}


inline fun <reified T> getType(){
    println(T::class)
}
```

打印日志：

```
class java.lang.String
```

反编译后的代码：

```
public final class TwioKt {
   public static final void main(@NotNull String[] args) {
      Intrinsics.checkParameterIsNotNull(args, "args");
      KClass var1 = Reflection.getOrCreateKotlinClass(String.class);
      System.out.println(var1);
   }

   private static final void getType() {
      Intrinsics.reifiedOperationMarker(4, "T");
      KClass var1 = Reflection.getOrCreateKotlinClass(Object.class);
      System.out.println(var1);
   }
}
```