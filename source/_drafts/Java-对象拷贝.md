---
title: Java 对象拷贝
tags:
---

### 浅拷贝

浅拷贝是按位进行拷贝，它会创建一个对象，这个对象的属性为原对象一份复制。

在对象属性复制过程中不同的是：
* 如果属性是基本数据类型，那么拷贝的就是基本数据类型的值；
* 如果属性为引用数据类型，那么拷贝的就是引用类型所指向的内存地址。

也就是说原对象和拷贝对象的引用数据类型指向同一块内存地址，如果其中一个对象改变了这个内存地址，那么另外的一个对象也会改变。

![图例](/../images/2019_07_03_01.jpg)


### 浅拷贝实现



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

深拷贝会赋值所有的属性，即当属性为对象引用时会同时拷贝其所指向的内存，即对象和其引用的对象一起拷贝。相较于浅拷贝，深拷贝拷贝速度更慢花销会更大。

![图例](/../images/2019_07_03_04.jpg)

### 深拷贝实现

[GitHub 完整代码](https://github.com/leeGYPlus/JavaCode/blob/master/src/copy/DeepMain.kt)

打印日志发现，对两个对象的任何操作都只对自己的对象有影响。


### 深拷贝







---
[Android Road](https://www.androidos.net.cn/codebook/AndroidRoad)

[Copy](https://www.androidos.net.cn/codebook/AndroidRoad/java/basis/copy.html)