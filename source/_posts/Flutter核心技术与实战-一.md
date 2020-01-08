---
title: Flutter核心技术与实战(一)：Dart语言基础
date: 2020-01-06 15:45:58
tags: [Flutter,Dart,极客时间专栏]
---

## 1. Dart的变量与类型

在 Dart 中，可以使用 var 或者具体的类型来声明一个变量，当使用 var 定义变量时，表示类型交由编译器推断决定。Dart 是 **类型安全** 的语言，所以的类型均继承与 Object，因此所有的类型都是对象类型，默认情况下，**未初始化的变量的值为 null**。

### 1.1 num、bool与 String

num 为 Dart 的数据类型，只有两种子类：int 和 double，使用 bool 表示布尔值，有 false 和 true 两种状态，它们均是编译时常量。在 Dart 中字符串可以通过单引号、双引号、嵌入表达式、多行字符串的形式表示。
<!-- more -->

```
int x = 1;
double y = 12.0;
var isZero = false;
var str = 'name';
var stri = "name";
var length = "length is &{str.lenght}";
var string = """ name is 
hahah""";
```

### 1.2 Dart 中的集合类型


在 Dart 中集合类型为 List 和 Map，对应其他编程语言中的数组和字典，具体使用如下：

```
// 以下变量的声明，没有对集合中的元素进行限制，那么集合中可以添加任何类型的值
var array = ["one","two"];
var array2 = List.of([1,2,3]);
var array3 = [1,2,"Name"];// 集合中同时包含 int 和 String 类型的元素
var map = {"name":"tom","age":2}; // Map 中同时存在 String：String、String：int 的键值对
// 对集合元素进行限制，那么添加到集合中的元素必须为限制的类型
var array4 = <int>[1,2,3];
var array5 = <int>[1,2,"name"];// 错误示例，对 array5 添加元素限制
var map2 = new Map<String,String>();// 对 Map 集合添加限制
var map3 = <String,String>{"name": "tom", "age": 1};// 错误示例，对 Map 元素添加限制，那么 第二个键值对不可以添加到 Map 中
```

### 1.3 常量定义

Dart 中定义不可变的变量，可以在变量前添加 final 或者 const 关键字：

* final：在运行期确定值，定义运行时常量
* const：在编译期确定值，定义编译器常量

## 2. Dart 中的函数、类和运算符

### 2.1 函数

在 Dart 中一切类型都是对象类型，包括函数，它的类型为 Function，所以在 Dart 中函数也可以被定义为变量，也可以通过参数传递给另外一个函数。

test.dart
```
bool isMan(String sex) => sex == "man";
// 在文件中声明 Function 对象
Function test = isMan;
// 将 Function 作为参数传入函数
void showSex(String sex,Function checker){
  print('$sex isMan:${checker(sex)}');
}
main(){
  showSex("man", test);
  // 在函数中声明 Function 对象并完成调用
  Function test2 = isMan;
  showSex("man", test2);
}
```


### 2.2 函数重载

在 Dart 中支持 Java 中提供同名函数不同参数列表形式的方法重载，而提供了 **可选命名参数** 和 **可选参数**。

在定义函数时：

* 可选命名参数

给参数增加 {},以 paramName：param 的形式指定参数。

```
//声明可选参数函数
void enable1Flags({bool bold, bool hidden}) => print("$bold , $hidden");
//为可选命名参数时增加默认值 
void enable2Flags({bool bold = true, bool hidden = false}) => print("$bold ,$hidden");
```


* 可选参数

给参数增加 [],意味这些参数是可以忽略的。

```
// 声明可忽略的可选参数函数
void enable3Flags(bool bold, [bool hidden]) => print("$bold ,$hidden");
//为可选参数增加默认值
void enable4Flags(bool bold, [bool hidden = false]) => print("$bold ,$hidden");
```

### 2.3 类的声明


与 Java 的构造函数略有不同，Dart 中可以定义 **命名构造函数**，使类的实例化过程更加清晰，并且在构造函数真正执行前，还有机会给实例变量赋值。

```
class Point { 
    num x, y, z;
    Point(this.x, this.y) : z = 0; 
    // 初始化变量z 
    Point.bottom(num x) : this(x, 0); // 重定向构造函数 
    void printInfo() => print('($x,$y,$z)');
}
```

### 2.4 Dart 中的复用

与 Java 中不同，Dart 中不存在 interface 关键字，所以在 Dart 中可以对同一个父类进行继承（extend）和接口实现（implement），在 Dart 中继承与实现的不同：

* 继承

子类继承父类，子类可以获取父类的成员变量和方法，也可以覆写构造函数以及父类方法。

* 实现

在 Dart 中，接口的实现代表子类仅仅获得接口的成员变量符号和方法符号，需要重新实现成员变量、方法的声明以及初始化。

* 混入（Mixin）
  
在 Dart中还支持通过 **混入（关键字：with）** 来实现复用，混入可以被视为 **具有方法实现的接口**，**Dart 语言不支持多继承**，但是可以通过 混入 实现 Dart 的多继承。


具体编码如下：

```
class Point {
  num x = 0, y = 0;

  void printInfo() => print('x is $x,y is $y');
}

// 继承
class Vector extends Point {
  num z = 0;

  // 覆写父类的方法
  @override
  void printInfo() => print('x is $x,y is$y,z is $z');
}

// 实现，需要重新声明父类的成员变量以及方法
class Coordinate implements Point {
  num x = 0, y = 0;

  void printInfo() => print('x is $x,y is $y');
}

class WithCoordinate with Point {
  @override
  void printInfo() => print("with x is $x,y is $y");
}

main() {
  var vector = Vector();
  vector
    ..x = 0
    ..y = 2
    ..z = 3;//级联运算符，等同于vector.x=0; vector.y=2;vector.z=3;
  vector.printInfo();

  Coordinate coordinate = new Coordinate();
  coordinate
    ..x = 23
    ..y = 90;
  coordinate.printInfo();

  var withCoordinate = WithCoordinate();
  withCoordinate
    ..x = 2
    ..x = 98;
  withCoordinate.printInfo();
}
```

### 2.5 运算符

Dart 中的运算符和其他语言中大致相同，但是提供了几个特殊的运算符，简化处理实例变量为 null 的情况：

* ?. 运算符

`a?.length`: a 不为 null 时，调用 a.length,否则抛出异常。

* ??= 运算符

`a??=value`: a 在为 null时，进行 a=value 的赋值。


* ?? 运算符

`a??b`:a 不为 null 时返回 a 的值，否则返回 b 的值，类比 Java 中的三目运算符。

### 2.6 覆写或者自定义运算符

在 Dart 中，一切均为对象，就连运算符也是对象成员函数的一部分。在 Dart 中可以通过 opeartor 进行自定义或者覆写运算符：

```

class Vector {
  num x, y;
  Vector(this.x, this.y);
  // 自定义相加运算符，实现向量相加
  Vector operator +(Vector v) =>  Vector(x + v.x, y + v.y);
  // 覆写相等运算符，判断向量相等
  bool operator == (dynamic v) => x == v.x && y == v.y;
}

final x = Vector(3, 3);
final y = Vector(2, 2);
final z = Vector(1, 1);
print(x == (y + z)); //  输出true
```