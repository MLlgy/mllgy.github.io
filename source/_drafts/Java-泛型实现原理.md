---
title: Java 泛型实现原理
tags:
---

### 泛型背后：类型擦除

所以说 Java 为 **伪泛型**。

### 为什么 Java 中不能声明泛型数组？**




Apple[] 可以赋值给 Furit[]，而 List<Apple> 却不能赋值给 List<Furit>

**数组是协变的，而 List 不是协变的(是不变的)**，看下例：

```
Student[] students = new Student[2];
List list = new ArrayList<Student>();
System.out.println(students.getClass().getName());
System.out.println(list.getClass().getName());
```
打印日志为：

```
[LStudent;
java.util.ArrayList
```
通过日志可以看到，**数组在运行时是可以获取自身类型的**，而 List<Student> 只知道自己是一个 List，但是 **无法获取到泛型参数类型**，所以说 **数组是协变的**，就是说 A 是 B 的父类，那么 A[] 也是 B[] 的父类，**如果让数组也支持泛型，由于 Java 中类型擦除的存在，那么在运行时数组将不知道自己的数组类型，将无法满足数组是协变的原则**，所以数组不支持泛型。

因为 Java中 的泛型是 `类型擦除` 的，所以 Java 中的泛型被称为 **伪泛型**，简单来说，就是 **无法在程序运行时获取到一个对象的具体类型**，具体见以上示例。

Kotlin 与 Java 一样，Kotlin 的数组也是无法获取列表的具体类型:

```
val appleList = listOf(Apple(1.0), Apple(2.0))
val appleArrayList = ArrayList<Apple>()
println(appleArrayList.javaClass)
println(appleList.javaClass)
```

日志：

```
class java.util.ArrayList
class java.util.Arrays$ArrayList
```

但是不同的是 **Kotlin 的数组是支持泛型的**，但是 **Kotlin 的数组不是协变的**:

```
val arrayList = arrayOfNulls<Apple>(3)
val anyArray:Array<Any?> = arrayList
```

### 为什么 Kotlin 和 Java 中的泛型要通过类型擦除实现？

向后兼容的锅！！！！

在这里插一句， 正式由于类型擦除，在实现类泛型的另外一个特性：将类型检查从运行期提前到编译期，原本在运行期才可以知道真正的类型，现在由于泛型机制，在编译期时通过类型擦除，获得类相应的数据类型。

Java 的一大特性：**向后兼容**，即：**老版本 Java 文件编译后可以运行在新版的 JVM 上**。

在 Java1.5 之前是没有泛型存在的，那么在此之前，出现大量以下代码：

```
ArrayList arrayList = new ArrayList();// 没有泛型
```


Java 引入泛型的方式是：在老的集合上进行改造，添加一些新特性，在兼容老版本的情况下支持泛型。

为什么这种方式，历史原因： Java Vector 到 ArrayList、HashTable 到 HashMap 的完全重构，引起了大量使用者的不满。

### 为什么使用类型擦除可以解决 Java 新老版本的兼容问题？


首先看以下示例：

```
public class One {
    ArrayList list  = new ArrayList();
    ArrayList<String> stringList = new ArrayList<>();
}
```
反编译 Java 文件对应的字节码：
```
public class show.One {
  java.util.ArrayList list;

  java.util.ArrayList<java.lang.String> stringList;

  public show.One();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: new           #2                  // class java/util/ArrayList
       8: dup
       9: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
      12: putfield      #4                  // Field list:Ljava/util/ArrayList;
      15: aload_0
      16: new           #2                  // class java/util/ArrayList
      19: dup
      20: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
      23: putfield      #5                  // Field stringList:Ljava/util/ArrayList;
      26: return
}
```

可以看到两种方式声明的 ArrayList 对应的字节码完全一样，这也说明了低版本编译的 class 文件在高版本的 JVM 上运行是没有问题的。

那么类型擦除后如何 Java 泛型的特性如何生效呢，比如类型检查、类型自动转换，其中类型检查是编译器在编译前就会检查，所以类型擦除对此特性没有影响，下面重点说一下自动类型转换。

平时开发中泛型的自动类型转换的例子就是： ArrayList 集合的 get 方法获得 List 泛型参数的类型，我们自己实现一个泛型类，如下：

```
public class Test<T> {

    private T obj;

    public void set(T obj) {
        this.obj = obj;
    }

    public T get() {
        return this.obj;
    }

    public static void main(String[] args) {
        Test<String> test = new Test<>();
        test.set("I am test word.");
        String str = test.get();
        System.out.println(str);
    }
}
```

反编译字节码如下：

```
Compiled from "Test.java"
public class show.Test<T> {
  public show.Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void set(T);
    Code:
       0: aload_0
       1: aload_1
       2: putfield      #2                  // Field obj:Ljava/lang/Object;
       5: return

  public T get();
    Code:
       0: aload_0
       1: getfield      #2                  // Field obj:Ljava/lang/Object;
       4: areturn

  public static void main(java.lang.String[]);
    Code:
       0: new           #3                  // class show/Test
       3: dup
       4: invokespecial #4                  // Method "<init>":()V
       7: astore_1
       8: aload_1
       9: ldc           #5                  // String I am test word.
      11: invokevirtual #6                  // Method set:(Ljava/lang/Object;)V
      14: aload_1
      15: invokevirtual #7                  // Method get:()Ljava/lang/Object;
      18: checkcast     #8                  // class java/lang/String
      21: astore_2
      22: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
      25: aload_2
      26: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      29: return
}
```

可以看到 main 方法中的 `#8` 所在行：
```
18: checkcast     #8                  // class java/lang/String
```

对 get 方法获得对象进行转型，这就是实现自动类型转换的秘密了。同时可以看到编译后的 Test 字节码的 get 方法和 set 方法，返回的数据类型和参数都进行了类型擦除，即为 Object。


---

**知识链接：**

[Kotlin 核心编程]()