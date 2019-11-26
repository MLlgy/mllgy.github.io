---
title: 'Java 泛型:深入理解泛型的类型擦除'
date: 2018-11-20 18:39:35
tags: [Java,泛型]
---
### 泛型代码和虚拟机

**在 Java 虚拟机中没有泛型类型对象 -- 所有对象都是属于普通类** ， 所以我们要了解一下 **类型擦除** 的概念。
<!-- more -->
Java 中的的泛型是 **伪泛型**，为什这么说呢？因为 Java 在编译期间，所有的泛型信息都被擦除掉，
称为 **类型擦除(type erasure)**。


#### 1. 类型擦除

无论何时定义一个泛型类型，都会自动提供一个相应的 **原始类型 (raw type)(不存在泛型  <T>)**。`原始类型的名字就是删去类型参数后泛型类的类型名,擦除类型变量`，并替换为 **限定类型（没有限定的变量就用 Object ）**。

程序 1.1
```
public class Pair<T> {

    private T mFirst;
    private T mSecond;

    public T getmFirst() {
        return mFirst;
    }

    public void setmFirst(T mFirst) {
        this.mFirst = mFirst;
    }

    public T getmSecond() {
        return mSecond;
    }

    public void setmSecond(T mSecond) {
        this.mSecond = mSecond;
    }
}
```
那么 Pair<T> 的 `原始类型`如下：

程序 1.2
```
public class Pair {

    private Object mFirst;
    private Object mSecond;

    public Object getFirst() {
        return mFirst;
    }

    public void setFirst(Object mFirst) {
        this.mFirst = mFirst;
    }

    public Object getSecond() {
        return mSecond;
    }

    public void setmSecond(Object mSecond) {
        this.mSecond = mSecond;
    }
}
```

**这里需要注意到 Pair< T > 与 Pair 为不同的数据类型，可以认为 Pair< T > 为 Pair 的子类型,但是 JVM 不会把 Pair< T > 当做新类来处理，会把他们当做同一个类处理。**

* 如果 Pair<T extends Comparable & Serializable>等此类有多个限定符时，就用 **第一个** 限定符的类型来替换(即把程序 1.2 中的 Object 替换为 Comparable  )。

```
class Pair{
    private Comparable mFirst;
    private Comparable mSecond;
    ...
}
```
#### 2. 类擦除后如何保证类型限定符的类型
那么当我们调用 
```
ArrayList< String > list = new ArrayList();
list.add(new Object);// 报错
```
 出现了报错信息，很明显在 Java 中这样是不行的。 Java 是怎样在类型擦除后，保证只能使用泛型限定符的类型呢？

答案就是： **Java 编译器通过先检查代码中泛型的类型，然后再进行类型擦除，再进行编译**。

---
`new ArrayList()` 只是在内存中开辟一个存储空间，可以存储任何的类型对象，但是 **真正涉及类型检的是它的引用**，因为我们是使用引用 `list` 来调它的方法，所以 list 引用完成了泛型类的检查，在这里 list 的类型参数为 String ，所以 list 只能添加 String 对象。

---

**菱形语法**
这里阐述一下菱形语法，在 Java1.7 之前实例化带有类型参数的对象，需要如下书写：
```
ArrayList<String> list = new ArrayList<String>();
```

很明显等号右侧的 `<String> ` 就显得多余了，于是在 Java1.7 开始Java 引入了菱形语法，即等号右侧的类型参数可以不显式声明：

```
ArrayList<String> list = new ArrayList();
```

#### 3. 翻译泛型表达式

当程序调用 **泛型方法** 时，由于 JVM 会对泛型类实现类型擦除，以 Pair 为例，那么当我们调用 Pair 方法的 get 方法时，那么我们获得返回值应该为 Object ，JVM 会进行如何操作，来保证我们得到相应的限定符类型的对象。

**答案就是： 如果擦除返回类型，编译器将会插入强制类型转换操作。**

```
Pair<Employee> buddies = ...
Employee buddy = buddies.getFirst();
```
**类型擦除 getFirst() 返回类型后将返回 Object 类型，编译器将自动强制插入 Employee 的强制类型转换。**

编译器把这个方法翻译为两条虚拟机指令：

1. 对 原始方法 Pair.getFirst 的调用。
2. 将返回的 Object 类型强制转换为 Employee 类型。

那么我们可以将 `Employee buddy = buddies.getFirst();` 理解为以下两步操作：

```
Object object = buddies.getFirst();
Employee buddy = (Employee)object;
```

同理，当 **存、取** 泛型类的 **变量** 时也会插入强制类型转换。

`buddies.setFirst(new Employee)`

个人猜想：
> 可以这是 JVM 所做的一部分工作，就如 **类擦除后如何保证类型限定符的类型** 中表述的一样，真正涉及类型参数的检查为对象引用，因为我们进行操作的实际是调用对象引用的方法，那么在我们对于对象引用的方法时，JVM 就会进行相应的类型检查，包括在存取泛型类时插入的强制类型转换(在字节码中插入强制类型转换)。


#### 4. 翻译泛型方法

类型擦除也会出现在泛型方法中，例如：

`public static <T extends Comparable> T min(T[] a)`

此为一个完整的 ==方法族==，但是经过类型擦除后，就会变成一个方法：

`public static Comparable mim(Comparable[] a)`

类型参数在这里已经被擦除了，只留下了限定符 Comparable 。

但是方法的类型擦除会带来两个问题：

例子：

```
class DateInternal extends Pair<LocalDate>{
    
    public void setSecond(LocalDate second){
        ...
    }
}

```

类型擦除后，得到：

```
class DateInternal extends Pair{
     @Override
    public void setSecond(LocalDate second){
        ...
    }
}
```

奇怪的现象就发生了，如上文见到的是 Pair 在经过类型擦除后，如下：

```
public class Pair {

    private Object mFirst;
    private Object mSecond;
	....
    ....
	....
    public void setmSecond(Object mSecond) {
        this.mSecond = mSecond;
    }
}
```
如代码所示 Pair 类型擦除后 setSecond 具体如下：

`public void setSecond(Object second)`

但是 DateInternal 存在着从 Pair 继承来的 setSecond(LocalDate second) 方法。



显然这是两个方法,因为这两个方法的参数不同，然而，不应该不一样(为什么？？？？)，留下悬念，往下看。




#### 5. 类型擦除多态冲突以及解决办法
下面具体分析一下上面遇到的情况。

有 Pair< T> 如下：
```
public class Pair<T> {
    private T one;

    public T getOne() {
        return one;
    }

    public void setOne(T mOne) {
        this.one = mOne;
    }
}

```
以及它的子类：
```
public class InterPair extends Pair<Date> {

    @Override
    public Date getOne() {
        return super.getOne();
    }

    @Override
    public void setOne(Date mOne) {
        super.setOne(mOne);
    }
}
```
在 InterPair 的继承关系中，如果我们把其父类 Pair 的类型限定符设置 Date，可以看到 InterPair 的方法相关类也为 Date，通过 `@Override` 字符可知，子类 InterPair 重写了父类 Pair 的相关方法。

在类型擦除后，Pair< T> 的原始类型(raw type)如下：
```
public class Pair {
    private Object one;

    public Object getOne() {
        return one;
    }

    public void setOne(Object mOne) {
        this.one = mOne;
    }
}

```
而 InterPair 的原始类为：

```
public class InterPair extends Pair {

    @Override
    public Date getOne() {
        return super.getOne();
    }

    @Override
    public void setOne(Date mOne) {
        super.setOne(mOne);
    }
}
```

此时，在类继关系中，Pair 与 InterPair 的 getOne 和 setOne 方法签分别不同，应该为重载而不是重写，但是如果我们按重载的关系去进行相关调用，会发报错，如下所示：

```
InterPair interPair = new InterPair();
interPair.setOne(new Date());
interPair.setOne(new Object());//报错信息：setOne (java.util.Date) in InterPair cannot be applied to (java.lang.Object)

```
**所以它们之间的关系为：重写**。

但是为什么 **重写** 会变成这样呢？

按照我的思维，在 InterPair 的继承关中，我们为 Pair<T> 设置的类型参数为 Date,那么我们想要得到的是：

```
public class Pair {
    private Date one;

    public Date getOne() {
        return one;
    }

    public void setOne(Date mOne) {
        this.one = mOne;
    }
}
```

这样 InterPair 继承 Pair，并 setOne 、getOne 方法进行重写，实现多态。但是在类型擦除后，Pair 的类型参数 T 变成了 Object，这样的话却只成为重载(方法名相同，参数不同)，由此，**类型擦除和多态之间产生冲突（子类继承父类重写相关方法，实现多态，但是类型擦除后只能变成重载，因为方法的签名不同嘛）**。JVM 虚拟机虽然知道你的本意，但是没有办法直接实现。

那么我们如何重写我们想要的Date类型参数的方法， JVM 采用了一个特殊方法 -- **桥方法**。

我们对 InterPair 类编译在进行反编译操作：
```
javac InterPair.java Pair.java
javap -c InterPair
```
得到反编译字节码如下：
```
Compiled from "InterPair.java"
public class unittwo.InterPair extends unittwo.Pair<java.util.Date> {
  public unittwo.InterPair();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method unittwo/Pair."<init>":()V
       4: return

  public java.util.Date getOne();
    Code:
       0: aload_0
       1: invokespecial #2                  // Method unittwo/Pair.getOne:()Ljava/lang/Object;
       4: checkcast     #3                  // class java/util/Date
       7: areturn

  public void setOne(java.util.Date);
    Code:
       0: aload_0
       1: aload_1
       2: invokespecial #4                  // Method unittwo/Pair.setOne:(Ljava/lang/Object;)V
       5: return

  public void setOne(java.lang.Object);
    Code:
       0: aload_0
       1: aload_1
       2: checkcast     #3                  // class java/util/Date
       5: invokevirtual #5                  // Method setOne:(Ljava/util/Date;)V
       8: return

  public java.lang.Object getOne();
    Code:
       0: aload_0
       1: invokevirtual #6                  // Method getOne:()Ljava/util/Date;
       4: areturn
}
```

JVM 生成的类型参数为 Object 的 **桥方法**，这样子类 InterPair 中我们看不到的桥方法来实现覆盖父类的方法。而桥方法的内部实现，就只是去调用我们自己重写的那两个方法。JVM 使用了巧方法，解决了类型擦除和多态的冲突。

在 `Java 中方法的签名为方法名和方法参数`，但在 `JVM 中使用参数类型和返回值来作为方法的签名`。


需要记住 Java 泛型转换的几个事实：
* 虚拟机中没有泛型，有的只是普通类和方法
* 所有的类型参数都有它们的限定类型替换
* 桥方法被用来保持多态
* 为保持类型安全性，必要时插入强制类型转换



---

#### 知识链接：
[Java 核心技术 卷一](http://product.dangdang.com/24035306.html)
[Java 泛型：类型擦除以及带来的问题](https://www.jianshu.com/p/f5773dec6a99)