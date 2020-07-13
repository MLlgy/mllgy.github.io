---
title: Java 异常机制浅析
date: 2019-01-26 18:23:46
tags: [Java,Exception]
---


###  处理错误

#### 异常分类

所有的异常都是继承于 Throwable

![运行时内存区域](/images/2019_01_26.jpg)

 Error 类层次结构描述了 Java 运行时内部的错误以及资源耗尽错误，应用程序不应该抛出这种类型的对象，而是让程序来处理，一般情况下程序会直接中断，并报出错误以及堆栈信息。


<!--more-->

关于 Exception 派生的两个分支的依据：

* **RuntimeException: 由程序错误导致导致的异常。**

* **其他 Exception： 程序本身没问题，但是由于像 IO 错误这类问题导致的异常属于其他异常。**


&nbsp;&nbsp;&nbsp;&nbsp;**RuntimeException 的几种异常：**

&nbsp;&nbsp;&nbsp;&nbsp;**ClassCastException:** 类变换异常

&nbsp;&nbsp;&nbsp;&nbsp;**IllegalArgumentException:** 传递非法参数异常

&nbsp;&nbsp;&nbsp;&nbsp;**IndexOutOfBoundsException:** 索引越界异常

&nbsp;&nbsp;&nbsp;&nbsp;**NoSuchElementException:**  表明枚举中没有更多的元素

&nbsp;&nbsp;&nbsp;&nbsp;**NullPointerException:** 空指针异常



对于这些异常，我们可以选择进行处理(捕获、抛出)，也可以选择不处理。如果我们不处理的话，那么异常会交给 Java 虚拟机，不断的向上层传递，那么在不同条件下导致的情况是  **当前运行的线程中断或程序中断**。

一般情况下我们对这类情况是不作处理的，如上文所说 **"RuntimeException: 由程序错误导致导致的异常"**，我们在写代码时应该尽力避免这种异常，而不应该通过 try/catch 、抛出等操作来隐藏异常。

**如果出现了 RuntimeException 异常，那么就一定是你的问题。**


**不是派生于 RuntimeException 的异常：**
1. 在文件后面读取数据
2. 打开一个不存在的文件
3. 根据给定的字符串去查找 Class 对象，但是这个对象表示的类不存在

**对于这种异常，Java 编译器强制要求对这类异常进行 try/catch 并处理 或将异常抛出，否则程序就不能编译通过。**



#### 声明受查异常

派生于 Error 或 RuntimeException 类所有的异常称为 **非受查异常**，其他所有的异常称为 **受查异常**。

**需要手动的捕获或抛出受检异常，而在代码层面尽力避免非受检异常。**


**编译器将检查是否为所有的受查异常提供了异常处理器。**



**一个方法不仅需要告诉编译器将要返回什么值，还要告诉编译器有可能发生什错误**,我们需要在声明方法的时候同时声明该方法可能会抛出的异常。
如下例：

```
public FileIputStream(String name) throw FileNotFoundException;
```

自己在编写方法时，不可能将所有可能抛出的异常声明，下面的 4 种情况应该抛出异常：

1. 调用一个抛出异常的方法
2. 程序运行中发现错误，并且利用 throw 语句抛出一个受查异常
3. 程序出现错误，例如一个数据越界的非受查异常
4. Java 虚拟机或运行时库出现内部错误


如果是前两种异常，则必须告诉调用这个方法的程序员有可能抛出的异常，因为一个抛出异常的方法都有可能是死亡陷阱。


对于一个有可能被其他人使用的 Java 方法，要根据 Java 异常规范，在方法的首部声明可能会抛出的异常。

我们不需要声明从 Error 继承的错误，不声明继承于 RuntimeException 的非受查异常。因为这些运行时错误是在我们的控制范围内，我们应该尽力避免这些错误，而不是在可能异常的位置抛出异常。


 对于一个方法必须声明所有可能抛出的受查异常，而非受查异常要么是不可控制的，要么就必须避免发生。如果方法没有声明所有可能发生的受查异常，编译器就会发出一个错误消息。
 
#### 如何抛出异常

**首先要决定抛出什么类型的异常**。

对于一个已经存在的异常类，抛出异常有一下几个步骤：

1. 找到一个合适的类。
2. 创建这个类的一个对象。
3. 将对象抛出。


以下例子：

```
String readDate(Scanner in) throw EOFException{//声明这个方法可能会抛出的异常
    
    ...
    ...
    if(..){
        throw new EOFException();//抛出异常
    }
}
```
**一旦方法抛出了异常，这个方法就不可能返回到调用者中，不必为返回的默认值或错误代码担忧。**

#### 创建异常类

定义一个继承于 Exception 的子类，习惯上，这个类应该包含两个构造器，一个默认构造器，另一个带有详细描述信息。

```
class CustomException extends IOException{
    public CustomException(){}
    public CustomException(String description){
        super(description);
    }
}

```

### 捕获异常

上面的抛出异常中，我们只需将异常抛出不用理睬了，同时有些异常是需要我们捕获的。

#### 捕获异常

在异常发生的位置如果没有进行捕获操作，那么程序就会终止执行，并且在控制台打印出异常信息和堆栈内容。


捕获异常的方法：
1. try/catch 语句
2. 方法首部声明异常，抛给方法调用处理

* try/catch

我们使用 try/catch 语句，具体语法如下：


```
try{
  
  ...
  
  ...
  
}catch(ExceptionType e){
  ...
  ...
}

```

try 语句中出现 catch 中出现的异常，那么程序将执行 catch 子句中的代码。


* 抛出异常给调用者

除了自己通过 try/catch 来处理异常，我们有没有更好的处理方式？答案是抛给调用者，很明显嘛，谁使用谁负责，让该方法的调用者去处理。

```
public void read(String filename) throw IOException{
    InputStream in = new InputStream(filename);
    int b;
    while((in.read()) != -1){
        ...
    }
}
```

编译器严格执行 throw 说明符，如果调用了一个受查异常的方法，就必须对它进行处理，或者继续传递。

* 如果选择 try/catch 处理还是继续传递呢？

**通常，应该捕获那些知道如何处理的异常，而将那些不知道怎么处理的异常继续传递。**

为具体说明，看一个例子：

**抛出异常**

如果传递一个异常，那么应该在方法的首部使用 throw 声明抛出的异常，告诉方法的调用者这个方法可能会抛出的一个异常。


```
public class ReadFile {
    public void read(String filename) throws FileNotFoundException {
        InputStream inputStream = new FileInputStream(filename);
    }
}
```
当我们在定义 read() 方法时，针对方法体中的 FileNotFoundException 异常我们不知道该怎么处理，怎么办？秉持着谁调用谁处理的原则 ，将该异常抛出，即如代码所示在方法头中 throws FileNotFoundException。在调用该方法时对异常进行处理，则有：
```
private static void showTest() {
        ReadFile readFile = new ReadFile();
        try {
            readFile.read("");
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
    }
```
同样，如果我们还是不知道该如何处理该异常，也可以将异常继续抛出：
```
private static void showTest() throws FileNotFoundException {
        ReadFile readFile = new ReadFile();
        readFile.read("");
    }
```
**处理异常**

如果我们在定义时方法体知道如何处理该异常，那么我们可以在定义方法处对异常进行处理：
```
public class ReadFile {
    public void read(String filename)  {
        try {
            InputStream inputStream = new FileInputStream(filename);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```
那么我们正常调用该方法就可以了， 不再需要对该方法中的异常进行任何处理。

```
private static void showTest() {
        ReadFile readFile = new ReadFile();
        readFile.read("");
    }
```



在类继承关系中，如果超类中的方法没有抛出异常，而子类重写了这个方法，那么这个方法必须捕获方法代码出现的每一个受查异常。在子类中不允许出现 throw 说明符中出现超过超类方法声明的异常范围。


#### 再次抛出异常和异常链

**可以在 catch 字句中抛出一个异常，这样的目的是改变异常的类型。** 原来抛出的异常为 catch(异常) 中的异常，现在在 catch 语句中抛出了一个新的异常，那么最终异常类型为新异常。

场景描述：

如果开发一个供其他程序员使用的子系统，那么用于表示子系统故障的异常类型可能有多种。 ServletException 就是这样一个异常的例子，执行servlet 的代码可能不想知道发生错误的细节原因，但是希望知道 servlet 是否有问题，这是可以通过再次抛出异常。下面是一个捕获异常将它再次抛出的基本方法：

```
try{
    acess the database
}catch(SQLException e){
    // 这样的话通过异常信息，可以知道更多出错的细节。
    throw new ServletException("database error: " + e.getMessage());
}
```

不过还是强烈推荐通过包装技术，**将原始异常设置为新异常的原因**，代码如下：

```
try{
    acess the database
}catch(SQLException e){
    Throwable se = new ServletException("database error");
    se.initCause(e);
    throw se;
}
```
这样，当我们捕获异常时，可以重新获取原始异常，不会丢弃原始异常的细节：
```
Throwable e = se.getCause();
```

一个完整的例子，帮助理解：

```
## Main.java
public static void main(String[] args) {
    showTest();
}

private static void showTest() {
    ReadFile readFile = new ReadFile();
    try {
        readFile.read("");
    } catch (Throwable throwable) {
        System.out.println(throwable.getMessage());
    }
}

## ReadFile.java    
public void read(String filename) throws Throwable {
    try {
        InputStream inputStream = new FileInputStream(filename);
    } catch (FileNotFoundException e) {
        Throwable throwable = new CustomFileExpection("在 catch中再次抛出了异常");
        throwable.initCause(e);
        throw throwable;
    }
}
```

#### finally 字句

应用场景：
在 finally 语句中释放资源。
```
InputStream in = new InputStream(...);
try{
    // 1
    // code that migth throw exception
    //2
}catch(IOException e){
    //3
    show error message
    //4
}finally{
    //5
    in.close();
}
//6
```

1. 代码没有抛出异常。代码会执行 1、2、5、6
2. 抛出一个在 catch 子句中捕获的异常。try 语句中，程序发生异常，跳过剩余代码，执行 catch 子句中代码。
   
    1. 如果 catch 中子句没有抛出异常，那么执行 1、3、4、5、6。
    2. 如果 catch 中子句抛出一个异常，异常将被抛回给这个方法的调用者，执行 1、3、5.
   
3. 代码抛出了一个不是 catch 捕获的异常(代码 1 处)，这种情况下，程序执行 try 语句中所有的语句，直到有异常被抛出为止，代码执行 1、5。

在日常代码中，**强烈建议解耦 try/catch 和 try/finally**，这样可以提高代码的清晰度，上面的代码可以这样书写：

```
InputStream in ...;
try{
    try{
        code that might throw exception
    }finally{
        in.close();
    }
}catch(IOException e){
    show error message
}
```

内层 try 代码的职责是关闭输入流，外层的 try 语句的职责就是报告出现的错误。

面临的问题：
**finally 语句也可能抛出异常，这时会覆盖原来的异常。**



#### 带资源的 try 语句

```
open a resource
try{
    work with the resource
}finally{
    close the resource
}
```

假如资源属于一个实现了 AutoCloseable/Closeable 的类，Java 7 提供了一个有用的快捷方式，AutoCloseable 有一个接口方法：

`void close() throw Exception`

Closeable 接口的 close() 方法是一个 抛出 IOException 的方法。

带资源的 try 语句的最简形式为：

```
try(Resource res = ...;){
    work with resource
}
```

try 块退出时或者存在一个异常时，**会自动调用 res.close()**，就好像使用了 finally 块一样。 

可以指定多个资源：
```
try(Scanner in = new Scanner(new InputStream("/usr/share/dict/words"),"UTF-8");
PrintWriter out = new PrintWrite("out.txt");){
    while(in.hasNext()){
        out.println(in.next().toUpperCase());
    }
}
```

不论这个块怎么退出， in 和 out 都会关闭。

这种处理异常的方式，就避免了上文所说的在 finally 语句中抛出的异常会覆盖 try 语句抛出异常的情况。

如果此时在 finally 中 close() 也会抛出异常，那么原来在 try 子句中的异常会被重新抛出，而 close 方法抛出的异常会被抑制，这些异常会被抑制，并由 addSuppressed() 方法增加到原来的异常，可以通过 getSuppressed() 获取这个被抑制的异常。

**在我们查看一些开源库代码时，这种实现 Closeable 接口的方案随处可见，其目的就是在相应的代码执行完毕后，关闭资源。**

#### 分析堆栈轨迹元素 

堆栈轨迹 (stack trace) 是一个方法调用过程的列表，它包含了程序执行过程中方法调用的特定位置。当 Java 程序正常终止，而没有捕获异常时，这个列表就会显示出来。

1. 可以调用 Throwable 类的 printStackTrace 方法访问堆栈轨迹的文本描述。
```
Throwable t = new Throwable();
StringWriter out = new StringWriter();
t.printStackTrace(new PrintWriter(out));
String des = out.toString();
```
2. 一种更为灵活的方法是使用 getStackTrace 方法，他会得到 StackTraceElement 对象的一个数组，可以在你的程序中分析这个对象数组
```
Throwable t = new Throwable();
StackTraceElement[] frames = t.getStaceTrace();
for(StackTraceElement frame: frames){
    ...
}
```

**StackTraceElement 类含有能够获得文件名和当前执行的代码行号的方法，同时还含有获得类名和方法名的方法**，这是一切能够实现的基础。

静态的Thread.getAllStackTrace() 方法，它可以产生所有线程的堆栈轨迹，下面给出这个方法的具体方式。

```
Map<Thread,StackTraceElement[]> map = Thread.getAllStackTrace();
for(Thread t : map.keySet()){
    StackTraceElement[] frames = map.get(t);
    ...
}

```

使用以上方法我们可以自定义 Android 的 LogCat 的打印信息，具体代码如下:

```
 private static String generateTag() {
        StackTraceElement caller = new Throwable().getStackTrace()[2];
        String tag = "%s.%s(L:%d)";
        String callerClazzName = caller.getClassName();
        callerClazzName = callerClazzName.substring(callerClazzName.lastIndexOf(".") + 1);
        tag = String.format(Locale.CHINA, tag, callerClazzName, caller.getMethodName(),
                caller.getLineNumber());
        String customTagPrefix = "h_log";
        tag = TextUtils.isEmpty(customTagPrefix) ? tag : customTagPrefix + ":" + tag;
        return tag;
    }

    public static void d(Object content) {
        if (!isDebug||content==null) {
            return;
        }
        String tag = generateTag();
        Log.d(tag, content.toString());
    }

```

### 使用异常机制的技巧

* 异常处理不能代替简单的测试

捕获异常的时间较代码此时花的时间较长，只有在异常情况下使用异常机制。

* 不过过分细化异常

* 利用异常层次结构

根据具体代码寻找更适合的子类或创建自己的异常子类。

* 不要压制异常



### 使用断言
在执行以下代码时：

```
int = a/b;
```
那么我们需要确认的是，a、b 为数值，并且 b 的值不为 0，我们可以做如下操作：
```
if(b=0) throw new IllegalAraumentException("b=0");
```
但是以上代码会一直保存在代码中，测试工作完毕后也不会自动删除，如果代码中含有大量的这种检查，程序运行起来就会变慢。

**断言机制允许在测试期间向代码中插入一些检查语句，当代码发布时，这些插入的检测语句会自动的移走。**

Java 引入了关键字 assert，有以下两种形式：

```
assert 条件：
```
和
```
assert 条件：表达式;
```

这两种形式都会对条件进行检测，如果结果为 false，则抛出一个 AssertionError 异常。在第二种形式中，表达式将会被传入 AssertionError 的构造器，并转换成一个消息字符串。


#### 启用和禁用断言
启动或禁用断言是类加载器的功能，当断言被禁用时，类加载器将跳过断言代码。

* 启用断言

`java -enablessertions(-ea) MyApp`

在某个类或整个包下使用断言：

`java -ea:MyClass -ea:com.xx.xxx MyApp`

* 禁用断言

`java -disablessertions(-da) MyApp`

有些类不是由类加载器加载，而是有直接虚拟机加载。对于不是由类加载器的系统类可是使用 -enablesystemssertions/-esa 启用断言。


#### 使用断言完成代码检查

Java 中有 3 种处理系统错误的机制：

1. 抛出一个异常
2. 日志
3. 使用断言


选择断言记住以下几点：
* 断言失败是致命的、不可恢复的错误
* 断言只用于开发和测试阶段

所以不应该使用断言向程序的其他部分通告发生了可恢复性的错误，断言只应该用于在测试阶段确定程序内部的错误位置。

**前置条件**

在声明一些方法时，往往针对该方法的使用有一定的说明，有些方法往往定义一些 **前置条件** 来进一步指导方法的使用，如下例子：
```
/**
* @param a array to be sorted (must not be null)
*/
static void sort(int[] a, int fromIndex, int toIndex)

```
那么 对数组的限制就是定义了一个前置条件，在使用这个方法时就不允许用 null 数组调用这个方法，并在这个方法的开头使用以下断言：
`assert a != null;`


### 记录日志

我们在代码添加 `System.out.println()` 方法来调用程序员观察具体的运行过程和结果，记录日志 API 就是为了这种情况下而设计的。记录日志 API 的有点：
1.可以轻易的取消全部日志记录。
2. 可以很简单的禁止日志的输出。
3. 可进行条件过滤。
4. 日志记录可以被定向到不同的处理器，用于控制台输出，用于存储在文件中等。
5. 


#### 关于异常的补充


* 为什么 Java 中打开物理资源，如磁盘文件、网络连接、数据库连接等，必须需要显式的关闭？

**JVM 提供的垃圾回收机制只负责堆内存分配出来的内存，打开的物理资源，GC 是不进行回收的，所以需要手动的关闭。**

* 如何正确的关闭资源？
1. 在 finall 中执行资源关闭操作。
2. 保证关闭资源前资源不为 null,因为存在资源初始化前就发生异常情况，所以在 finally 语句中对资源对象进行非空判断。
3. 为每个资源关闭操作执行 try/catch 操作，因为在资源关闭过程中也有可能发生异常，导致程序中断。


```
public void test() {

        FileInputStream inputStream = null;
        FileOutputStream outputStream = null;

        try {
            inputStream = new FileInputStream("srcPath");
            outputStream = new FileOutputStream("destPath");
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }finally {//操作1
            if(inputStream != null){//操作2
                try {//操作3
                    inputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

            if(outputStream != null){
                try {
                    outputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
```

### 知识链接

[Java 核心知识 卷1](http://product.dangdang.com/24035306.html)