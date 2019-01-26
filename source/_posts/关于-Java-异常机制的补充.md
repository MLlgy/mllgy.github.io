---
title: 关于 Java 异常机制的补充
date: 2019-01-26 18:44:41
tags: [Exception]
---



### 为什么 Java 中打开物理资源，如磁盘文件、网络连接、数据库连接等，必须需要显式的关闭？

JVM 提供的垃圾回收机制只负责堆内存分配出来的内存，打开的物理资源，GC 是不进行回收的，所以需要手动的关闭。

### 如何正确的关闭资源？
1. 在 finall 中执行资源关闭操作。
2. 保证关闭资源前资源不为 null,因为存在资源初始化前就发生异常情况，所以在 finally 语句中对资源对象进行非空判断。
3. 为每个资源关闭操作执行 try/catch 操作，因为在资源关闭过程中也有可能发生异常，导致程序中断。

<!--more-->

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

### finally 与 return 的关系

阐述两者的关系，我们主要关注 return 关键字的位置，为了表达的更形象，看下例：

```
private static int test(){
        int count = 5;
        try {
            count++;
            return count;
        }finally {
            System.out.println("finally 语句执行");
            System.out.println("finally 语句执行 " + count);
            count++;
            return count;
        }
    }
    

public static void main(String[] args) {
        int  a = test();
        System.out.println(a);
    }
```
打印日志为：

```
finally 语句执行
finally 语句执行 6
7
```

当 Java 程序执行 try/catch 中有 return 语句，return 语句会导致方法立即结束。但是系统执行完 return 语句不会马上结束该方法，而是查看在这个异常处理的流程中是否存在 finally 语句，如果存在 finally 语句，那么需要执行 finally 语句。如果 finally 语句中有 return 语句，那么会更新 try/catch 语句 return 返回的值，但是需要注意的是因为该方法已经结束，此处的 return 不会像无法返回到 try/catch 中执行代码(因为 try/catch 之后才会来到 finally 语句)。具体可以参看上例自行揣摩其中含义。

在举一个例子，我们在 catch 中执行 return 语句:

```
private static int test(){
        int[] counts = new int[]{5,3};
        int count = counts[0];

        try {
            int error = counts[2];
            count++;
            return count;
        }catch (ArrayIndexOutOfBoundsException e){
            System.out.println("catch 语句执行" + count);
            return count;

        }finally {
            count++;
            System.out.println("finally 语句执行" + count);
            return count;
        }
    }
    

public static void main(String[] args) {
        int  a = test();
        System.out.println(a);
    }
```

打印日志为：

```
catch 语句执行5
finally 语句执行6
6
```
针对该例不做阐述。

### 关于异常的捕获

当 Java 运行时环境接收到异常对象时，系统会根据catch(TypeException e) 来决定使用哪一个异常分支来处理程序引发的异常。程序进入负责异常处理的 catch 块时，系统生成的异常对象 ex 将会被传给 cathc(TypeException ex) 的异常形参，从而可以在 catch 块中访问异常信息。

### 关于异常捕获的顺序

捕获父类异常的 catch 块在捕获子类异常的 catch 块后，即先处理小异常，再处理大异常。

### 使用异常机制的注意点

1. 不要使用 try/catch 来控制流向。
2. 精准捕获可能抛出的异常，不要乱抛异常，不要放大异常的范围。
3. 不要在 finally语句中递归调用可能引起异常的方法，因为这将导致该方法的议程不能被正常抛出，甚至StackOverflowError 也不能终止程序，只能强制终止 java 进程才可以终止程序运行。
4. 在子类复写时，子类方法只能抛出父类方法声明抛出的异常的子类。


### 知识链接

[疯狂 Java 系列书籍](http://product.dangdang.com/22581959.html)

