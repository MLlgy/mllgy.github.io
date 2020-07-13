---
title: 浅析 Java 对象拷贝
date: 2019-08-08 18:02:13
tags: [对象拷贝,Java]
---

### 浅拷贝

浅拷贝是按位进行拷贝，它会创建一个对象，这个对象的属性为原对象一份复制。

在对象属性复制过程中不同的是：
* 如果属性是基本数据类型，那么拷贝的就是基本数据类型的值；
* 如果属性为引用数据类型，那么拷贝的就是引用类型所指向的内存地址。

也就是说原对象和拷贝对象的引用数据类型指向同一块内存地址，如果其中一个对象改变了这个内存地址，那么另外的一个对象也会改变。

<!-- more -->

![图例](/../images/2019_07_03_01.jpg)


### 浅拷贝实现


实现 Cloneable 接口：[GitHub 完整代码](https://github.com/leeGYPlus/JavaCode/blob/master/src/copy/Student.java)

[]()
```
public class Main {
    public static void main(String[] args) {
        Student student = new Student(1, "杰克");
        Student copyStudent = (Student) student.clone();
        printMessage(student, copyStudent);
        student.setName("再见杰克");
        student.setAge(2);
        student.getSubject().setName("Subject Rename Origin Change");
        printMessage(student, copyStudent);
        copyStudent.setAge(3);
        copyStudent.setName("幻觉");
        copyStudent.getSubject().setName("Subject Rename Copy Change");
        printMessage(student, copyStudent);
    }

    private static void printMessage(Student originStudent, Student copiedStudent) {
        System.out.println("student Name is:" + originStudent.getName() + " ,age is " + originStudent.getAge() + " , subject Name is " + originStudent.getSubject().getName());
        System.out.println("copyStudent Name is:" + copiedStudent.getName() + " ,age is " + copiedStudent.getAge() + " , subject Name is " + copiedStudent.getSubject().getName());
        System.out.println("================");
    }
}
```

具体打印日志：

```
student Name is:杰克 ,age is 1 , subject Name is Subject
copyStudent Name is:杰克 ,age is 1 , subject Name is Subject
================
student Name is:再见杰克 ,age is 2 , subject Name is Subject Rename Origin Change
copyStudent Name is:杰克 ,age is 1 , subject Name is Subject Rename Origin Change
================
student Name is:再见杰克 ,age is 2 , subject Name is Subject Rename Copy Change
copyStudent Name is:幻觉 ,age is 3 , subject Name is Subject Rename Copy Change
================
```

可以很明显的看出，如果对象的属性为对象引用时，原对象和拷贝对象指向的为同一块内存地址，当任何一个对象改变该属性时，另外一个对象的相应属性也会发生改变。




[GitHub 完整代码](https://github.com/leeGYPlus/JavaCode/tree/master/src/copy/Main.java)



### 深拷贝

深拷贝会赋值所有的属性，属性为基本数据类型时，直接拷贝属性值；当属性为对象引用时会同时拷贝其所指向的内存。相较于浅拷贝，深拷贝拷贝速度更慢花销会更大。


深拷贝只是借助源对象在堆中产生一份源对象的复制，自此两者互不干涉。

![图例](/../images/2019_07_03_04.jpg)

### 深拷贝实现

[GitHub 完整代码](https://github.com/leeGYPlus/JavaCode/blob/master/src/copy/DeepMain.kt)

打印日志发现，对两个对象的任何操作都只对自己的对象有影响。


### 通过序列化实现深拷贝


序列化是干什么的?它将整个对象图写入到一个持久化存储文件中并且当需要的时候把它读取回来, 这意味着当你需要把它读取回来时你需要整个对象图的一个拷贝。这就是当你深拷贝一个对象时真正需要的东西。请注意，当你通过序列化进行深拷贝时，必须确保对象图中所有类都是可序列化的。

需要类实现序列化相关接口。详细内容可参见序列化相关内容。

### 如何选择拷贝类型

如果对象的属性全是基本类型的，那么可以使用浅拷贝，但是如果对象有引用属性，那就要基于具体的需求来选择浅拷贝还是深拷贝。我的意思是如果对象引用任何时候都不会被改变，那么没必要使用深拷贝，只需要使用浅拷贝就行了。如果对象引用经常改变，那么就要使用深拷贝。

---
[Android Road](https://www.androidos.net.cn/codebook/AndroidRoad)

[Java 对象的浅拷贝和深拷贝](https://www.androidos.net.cn/codebook/AndroidRoad/java/basis/copy.html)