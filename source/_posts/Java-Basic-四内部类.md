---
title: Java Basic (四) -- 内部类
date: 2019-04-04 14:47:26
tags: [内部类]
---
### 1. 提问

1. 内部类为什么会出现
2. 内部类的调用
3. 内部类与外部类访问成员变量的不同
4. 静态内部类和非静态内部类


#### 为什么使用内部类

1. 内部类提供了 **更好的封装**，可以把内部类的细节隐藏在外部类中，对同一个包中的其他类不可见。
2. **内部类可以访问外部类的私有数据**。因为内部类为外部类的成员，成员之间可以互相访问,外部类不可以访问内部类的私有数据。
3. **可以使用匿名内部类创建访问一次的类**，十分方便。


### 2. 非静态内部类

例子 ：
```
public class OutClass {

    private int num = 1;
    private int num2 = 3;

    private class InnerClass {

        private int num = 2;

        private void method() {
            System.out.println("innerclass method");
            System.out.println("innerclass method inner num " + num);
            System.out.println("innerclass method inner num " + this.num);
            System.out.println("innerclass method out num " + OutClass.this.num);
            System.out.println("innerclass method out num2 " + num2);
        }

    }

    public static void main(String[] args) {
        OutClass outclass = new OutClass();
        System.out.println("main "+ outclass.num);
        outclass.test();
    }

    private void test() {
        InnerClass inner = new InnerClass();
        inner.method();
        System.out.println("outclass test");
    }
}
```
<!-- more -->
```
// 打印日志：
main 1
innerclass method
innerclass method inner num 2
innerclass method inner num 2
innerclass method out num 1
innerclass method out num2 3
outclass test
```

#### 2.1 内部类可以访问外部类的成员变量(包括私有变量)

原因：

在非静态内部类对象里，保存了外部类对象的引用。

内存模型：
![内存模型图](/images/2019_04_04_1.jpg)

但是当外部类与内部类有相同名字的变量时，引用外部类变量需要指定外部类对象 --  `Outclass.this` 

#### 2.2 编译后的 Class 文件

在 JVM 中没有内部类这个概念，所有的类都是普通类（POJO），内部类也会被编译成带有前缀的类。

编译后，得到两个 class 文件:
1. `OutClass.java` 
2. `OutClass$InnerClass.class`

#### 2.3 内部类方法中的变量的访问顺序

**内部类方法内部中 --> 内部类中的成员变量 --> 外部类中的成员变量 --> 不存在，编译异常**

#### 2.4 内、外部类变量名相同
访问使用如下格式：

`this.field` : 内部类变量

`OutClass.this.field` : 外部类变量


#### 2.5 外部类不可直接访问内部类成员

非静态内部类成员只在非静态内部类范围内是可知的，外部类不能直接访问。

通过 **内部类实例对象来访问**。
```
new InnerClass().method();
new InnerClass().num++;
```

#### 2.6 外部类中静态成员中不允许直接使用非静态内部类

静态成员为类成员，在编译期对其初始化，如果直接使用非静态成员(如非静态内部类)，则会发生错误。

#### 2.7 内、外部类关系

非静态内部类对象寄生在外部类对象里，有非静态内部类对象一定存在外部类对象，反之不成立。

#### 2.8 非静态内部类中的静态成员(重点理解)

**如果内部了声明静态成员变量，那么必须使用 `final` 修饰**。

因为一个静态变量只有一个实例，而对于每一个外部对象，分别有一个单独的内部类实例，如果这个变量不是 final 的，那么他就不是唯一的，这与 static 的含义相互冲突。

**非静态内部类中不能含有静态方法**。

```
public class OutterClass {
    private int age;
    private void outMethod(){}
     class InnerClass {
        private final static String name = "";
        private /*static*/ void innerMethod() {
            age++;
            outMethod();
        }
    }
}
```


### 3. 静态内部类

```
class OutClass {

    private int num = 1;
    private static int age = 2;

    private void doSomething(){
        StaticInnerClass.staticNum++;
        StaticInnerClass staticInnerClass = new StaticInnerClass();
        staticInnerClass.method();
        staticInnerClass.num++;
    }

    protected static class StaticInnerClass {
        private static int staticNum = 10;
        private int num = 100;

        private void method() {
            num++;
            staticNum++;
            age++;
            // 'four.OutClass.this' cannot be referenced from a static context
            // -- four.OutClass.this 不能再 static 环境下被引用
            // OutClass.this.num++;

        }
    }
}

```

#### 3.1 静态内部类不可以访问外部类实例成员，可以访问类成员

静态内部类为外部类的类成员，只能访问外部类的静态变量，不可以访问非静态变量。

#### 3.2  编译后的 Class 文件

与非静态内部类相同，编译后得到两个 class 文件 -- OutClass.class 、StaticInnerClass.class


#### 3.3 外部类不可直接访问内部类成员

静态内部类成员、实例成员只在静态内部类范围内是可知的，外部类不能直接访问。




* 通过 内部类名 来访问静态内部类类成员。
```
StaticInnerClass.staticNum++;
```

* 通过内部类实例对象访问静态内部类实例成员。
```
StaticInnerClass staticInnerClass = new StaticInnerClass();
staticInnerClass.method();
staticInnerClass.num++;
```

### 4. 内部类使用



#### 4.1 外部类使用内部类

* 基本的使用
* 外部类使用内部类的子类

外部类中的静态代码块、静态方法中不可使用非静态内部类，因为静态成员不能使用非静态成员。

#### 4.2 外部类以外使用非静态内部类

根据内部类的访问权限修饰符，内部类对其他类的可见性不同。

外部类以外建立非静态内部类实例必须外部类实例和 new 来调用非静态内部类的构造器。

* 非静态内部类

```
OutClass{
    private int num = 1;
    protected InnerClass{
    
        private int num = 2;
        
        private void method(){
            
        }
        
    }
}

```
调用：
```
OutClass.InnerClass innerclass = new OutClass().new InnerClass();
innerclass.num++;
innerclass.method();
```
* 非静态内部类子类
* 

#### 4.2 外部类以外使用非静态内部类
```
OutClass{
    private int num = 1;
    protected static InnerClass{
    
        private int num = 2;
        
        private void method(){
            
        }
        
    }
}

```
调用：

```
 OutClass.InnerClass staticInnerClass = new OutClass.InnerClass();
 staticInnerClass.

```


### 5. 局部内部类

```
public class Test {
    private static void main(String[] args){

        class InnerBase{
            int a ;
        }

        class SubInnerClass extends InnerBase{
            int b;
        }

        SubInnerClass subInnerClass = new SubInnerClass();
        subInnerClass.a = 1;
        subInnerClass.b = 2;
    }
}
```

局部内部类，顾名思义内部类定义在 方法内部，其有效范围也在方法内部，方法外部无法访问,即：对外部世界完全的隐藏起来。

通过 `javac Test.java` 对该类进行编译，生成的 class 有 3 个，分别为： `Test.class`、`Test$1InnerBase.class`、`Test$1SubInnerClass.class`。

局部内部类遵循如下命名格式：
**OutClassName$NInnerClassName 其中 N 表示第 N 个内部类。**


### 6. 匿名内部类(Anonymous Inner Class)

#### 6.1 定义

匿名内部类适合创建只需要一次使用的类。

定义内部类的格式：
```
new 实现接口 或 父类构造器(参数列表)
{
    类的主体
}
```
由其格式可知，**匿名内部类必须且只能继承一个父类，或实现且最多一个接口**。

由于构造器必须与类名相同，而匿名内部类不能有类名，所以匿名类不能有构造器。

例子：

```
public interface IClick {
    void change();
}

public class Main {

    public  void main(String[] args) {

        final String  name = "name";
        show(new IClick() {
            @Override
            public void change() {
                System.out.println(name);
            }
        });
    }

    private  void show(IClick click){
        click.change();
    }
}
```

在这个例子中，main 方法中的调用 show 的方法中定义了一个匿名内部类，这个匿名内部类实现了 IClick 接口。

**不仅在 匿名内部类中，就是在普通的内部类中访问内部类外部的变量也需要添加 final 修饰符。**

原因：

对于普通局部变量而言，它的作用域停留在方法内，当方法执行完毕，该局部变量也随之消失。但内部类则可能产生隐式的 "闭包"，闭包使得局部变量脱离它所在的方法继续存在。


个人理解，匿名内部类是一个实现了接口或继承类的
#### 使用限制

* 由于系统在创建匿名内部类时，会创建匿名内部类的对象，所以匿名内部类不能定义为抽象类。
* 由于匿名内部类没有类名，所以匿名内部类无法定义构造器，但可以使用初始化块进行初始化。


方法的返回值的生成和表示这个返回值的类的定义结合在一起。

----

为什么需要内部类？

答:   如果你想实现一个接口，但是这个接口中的一个方法和你构想的这个类中的一个方法的名称，参数相同，你应该怎么办？这时候，你可以建一个内部类实现这个接口。由于内部类对外部类的所有内容都是可访问的，所以这样做可以完成所有你直接实现这个接口的功能。

真正的原因，java中的内部类和接口加在一起，可以很好的实现多继承的效果




内部类是否有用、必要和安全

内部类是一种编译器现象，与虚拟机无关。编译器会把内部了翻译成用 $ 分割外部类名与内部类名的常规文件，而虚拟机对此一无所知。

----
[java为什么匿名内部类的参数引用时final？](https://www.zhihu.com/question/21395848)