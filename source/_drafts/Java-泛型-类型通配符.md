---
title: 'Java 泛型:类型通配符'
tags:
---


#### 1. 通配符概念
 
因为 List 是泛型类，为了 **表示各种泛型 List 的父类**，可以使用类型通配符，类型通配符使用问号(?)表示，将一个问号当做类型元素传递个 List，可以表示为 List<?>,意思是 **元素类型未知的 List**，不同于 List<T> 其元素类型为 T。这个问号被称为通配符，它的元素类型可以匹配任何类型。

一般的，统配符不会出现在泛型类的声明上，而多用于使用泛型类或泛型方法。


```
public class GenericTest {
      
public static void main(String[] args) {
    List<String> name = new ArrayList<String>();
    List<Integer> age = new ArrayList<Integer>();
    List<Number> number = new ArrayList<Number>();
    name.add("icon");
    age.add(18);
    number.add(314);
    getData(name);
    getData(age);
    getData(number);   
}
// 在此处使用通配符，则可以传入各种类型的 List 泛型，
public static void getData(List<?> data) {
    System.out.println("Test date :" + data.get(0));
}
}
```

打印日志为

```
Test data :icon
Test data :18
Test data :314
```


 **通配符的出现，允许类型参数变化。**
 
 #### 2. 上界通配符（子类型通配符）
 
> <? extends ClassType> 该通配符为 ClassType 的所有子类型。
 
 表示任何泛型 ClassType 类型，它的类型参数是 ClassType 的子类，但不是 Pair< String>。
 
  **上界通配符可以使用返回值，但是不可以为方法提供参数。**


 
 
 ##### 继承关系：
 
 ![上界通配符](https://img-blog.csdnimg.cn/20190228151529777.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1N0cmFuZ2VfTW9ua2V5,size_16,color_FFFFFF,t_70)
 
 
 可进行如此赋值操作：
 ```
 Pair<Manager> manager = new Pair<>();
 Pair<? extends Employee> wildCardBuddies = manager;
 ```
 
 我们看一下类型 Pair<? extends Employee>,其方法是这样的：
 
 ```
 ? extends Employee getFirst();
 void setFirst(? extends Employee);
 ```
 
 这样的代码不可能调用 setFirst 方法，编译器只知道需要某个 Employee 的子类型，但是不知道具体的类型，它拒绝传递任何特定的类型，**毕竟  ？不能用来匹配**。
 
 使用 getFirst 就不存在这个问题：将 getFirst 的返回值赋值给 Employee 的引用完全合法。
所以有了上文中的 -- **上界通配符可以使用返回值，但是不可以为方法提供参数。**

 
进一步，用自己的语言理解：
 
 **`? extends Employee` 表示为 Employee 或 Employee 的子类型,可进行 get ，返回值可用 Employee 来接收，而在调用 set 方法时无法确定参数的具体类型(拥有多个子类型，不知道具体哪一个子类型)。**
 
==由上面通配符可以，任何对象调用 getter 和 setter 方法，其返回值或参数必须有对应的类，不能对应一系列类族。==

**使用上限通配符意味着我们可以进行读取，但是不能写入**。
 
 #### 3. 超类型限定符(上界通配符)
 
 > <? super ClassType> 该通配符为 ClassType 的所有超类型。
 
  表示任何泛型 ClassType 类型，它的类型参数是 ClassType 的超类，但是不是 Pair<String>。
 
 **与下界通配符恰好相反，可以为方法提供参数，但是不能使用返回值。**
 
 ##### 继承关系：

![下界通配符](https://img-blog.csdnimg.cn/20190228151628826.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1N0cmFuZ2VfTW9ua2V5,size_16,color_FFFFFF,t_70)

可进行如此赋值操作：
 ```
 Pair<Employee> manager = new Pair<>();
 Pair<? extends Manager> wildCardBuddies = manager;
 ```
 我们看一下类型 Pair<? super Manager>,其方法是这样的：
 
 * 知识链接
 ```
 void setFirst(? super Manager)
 ? super Manager getFirst()
 ```
 
 这不是真正的 Java 语法，但是可以看出编译器知道什么。 编译器无法知道 setFirst 方法的具体类型，因此调用这个方法时不能接受类型为 Employee 或 Object 的参数。只能传递 Manager 或其子类型的对象。另外，如果调用 getFirst() 不能保证返回对象的类型，只能把它赋给 Object。
 
 
 进一步，用自己的语言理解：
 
**`? super Manager` 表示为 Manager 或 Manager 的超类型，可进行 set，对应的 setter 方法的参数可以传进 Manager 或 Manager 的子类型,在调用 getter 方法时返回值为 Manager 或 Manager 的超类，这样是不合法的,但是可以将返回值赋值给 Object 对象。**
    
 **使用上限通配符意味着我们可以写入，不可以读取。**
 
 #### 4. 无限定通配符
 
 > <?> : 该通配符可以匹配任何类型。
 
 Pair<?> 有以下方法：
 
 ```
 ? getData();
 void setData(?);
 ```
 
 **getData() 的返回值只能赋值给 Object，++因为不知道 ？具体代表什么类型++，所以 setData(?) 方法不能被调用。**
 
 Pair<?> 与 Pair 的不同：可以用任意 Object 对象调用原始 Pair 类的 setData() 方法。 --- ==存在十分大的疑问==
 
 为什么使用这样脆弱的类型？它对于许多简单的操作十分有用：例如下面的例子：判断一个 Pair 对象是否是一个 null 引用，它不需要具体的类型参数：
 
```
public static boolean hasNull(Pair<?> pair){
    return p.getData() == null;
}
```
其实通过使用泛型，可以避免使用通配符：

`public static <T> boolean hasNull(Pair<T> pair)`

但是带有通配符的版本更具可读性。