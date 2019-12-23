---
title: 自己需要知道的有关 CPP 的基本知识
tags:
---

## 数组


```
int nums[5] = {1,2,4,5,6}
```


## 结构体

### 结构体的声明

结构体中可以存储多种数据类型的数据。

```
struct student{
    int age;
    float price;
    char name[20];
};
```

struc：标识为结构体的关键字
student：定义结构体的名称

### 结构的声明位置

结构体的声明位置：

* 外部声明--在文件前面声明，可以用在文件中的所有函数
* 局部声明-只能在声明的函数内使用

### 结构体的初始化

```
student mike {1,2.0,"mike"};
student nike {};
// 调用
mike.age;
```
**声明处初始化**

```
struct student{
    int age;
    float price;
    char name[20];
} mike;

struct student{
    int age;
    float price;
    char name[20];
} mike = {
    1,
    2.0,
    "mike"
};
```


## 函数


在 CPP 中使用函数，必须完成以下工作：

1. 提供函数原型(一般的，将原型函数定义在头文件中，通过 include 的方式引入到 Cpp 文件中，进行使用)
2. 提供函数定义
3. 调用函数

```
#include <iostream>
//1. 函数原型
void simple();
int main(){
    //3. 调用函数 
    simple();
    return 0;
}
//2.函数定义
void simple(){
    xxx
}
```

### 为什么需要原型函数

因为编译器需要原型函数。


原型描述了函数到编译器的接口，它将函数的返回值类型、 参数类型以及数量告诉编译器。

以下面的原型函数为例：

```
double add(double x,double y);


int main(){
    double sum = add(1.0,2.0);
}

double add(double x,double y){
    return x+y;
}
```

1. 首先，原型函数告诉编译器，add 函数有两个参数，数据类型皆为 double。

    如果程序没有提供这样的参数，那么由于原型函数的存在，编译器会捕获这种错误。

2. 调用 add 函数完成计算，会将返回值存储到指定位置，然后调用函数从该位置取得返回值。

    由于在原型函数中指定 add 的返回值类型为 double，那么此时编译器就知道在内存或者 CPU 寄存器中检索多少个字节（double 为 8 个字节）以及如何解释它们，如果没有原型函数提供这些信息，编译器只能进行猜测，但是这是不允许的。






## 类

### 类的定义


stock.h
```
class Stock{
    private:// 私有成员只能通过公共成员访问，默认为 private
        int name;
        double age;
        double total;
        void showInfo{
            tatoal = total * 2;
        }
    public:// 类的公共成员
        void show();
        void buy();
        int phone;
}
```

### 实现类成员函数


stock.cpp
```
#include stock.h

void Stock::buy(){
    xxx
}
```

* 定义成员函数时，需要使用 **域解析运算符(::)** 来标识该函数所属的类。
* 类方法可以访问类的私有成员。


### 创建对象以及函数调用

**创建对象**


最简单的方式：声明类变量：


```
Stock mike,nike;
mike.buy();
```


使用构造函数初始化对象


