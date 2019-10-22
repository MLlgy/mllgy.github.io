---
title: Kotlin 泛型简单理解
date: 2019-10-07 15:50:47
tags: [Kotlin 泛型]
---

### 1. Java 中的泛型


协变(covariance)：子类的泛型类型也属于泛型类型的子类。

由于 Java 中类型擦除的存在，所以 Java 不支持协变，Kotlin 继承了这种限制。在 Java 中可以通过使用通配符(?) 来解除这种限制。

这样的话有：

```
// 使用了通配符，就可以把子类的泛型类型对象赋值给父类的泛型类型声明了
List<? extends TextView> textViews = new ArrayList<Button>();
```

虽然这种写法解除了赋值的限制，但是却增加了另外一个限制：在使用这个引用的时候，不能调用它的参数包含类型参数的方法，也不能给它的包含类型参数的字段赋值(除了空值)。

限制：父类的泛型类型声明的实际值不能是子类的泛型类型对象

<!-- more -->

```
private ArrayList<? extends TextView> mList = new ArrayList<Button>();
//mList.add(textView); 此调用是错误的 
```

看下例：

```
public void showTexts(List<TextView> list){
    for(TextView tv: list){
        sout(tv.getText());
    }
}

List<Button> buttons = ...;
....
showTexts(buttons);// 如此调用，编译期会报错
```

做如下修改：


```
public void showTexts(List<? extends TextView> list){
    for(TextView tv: list){
        sout(tv.getText());// getText() 的参数中不包含类型参数
    }
}

List<Button> buttons = ...;
....
showTexts(buttons);// showTexts 方法做以上更改，无错
```


当遇到只想使用不想修改的情况时，可以使用 ？ extends ，使不具有协变的 Java 支持协变。

同时，还存在 ？ super 可以使原本不具有 逆变 的 Java 具有 逆变的性质。

```
public void addTextView(List<TextView> list){
    TextView textView = ..';
    list.add(textview);
}
List<View> views = new ArrayList<View>();
addTextView(views);// 如此调用，编译期错误
```

做如下更改：

```
public void addTextView(List<? super TextView> list){
    TextView textView = ..';
    list.add(textview);
}
List<View> views = new ArrayList<View>();
addTextView(views);// 如此更改，无错

```
限制：只能修改，不能使用


？extents 、？super 的使用场景：

PECS 法则：Producer extends，Conssumer super。


### 2. Kotlin 中的泛型

? extends --> out:只能输出，不能输入，只能读不能写。

? super --> in：只能输入不能输出，只能写不能读。


Kotlin 中的 in 和 out 不仅可以直接使用在变量和参数的声明里，还可以使用在泛型类型声明的类型参数上。

```
interface Producer<out T>{
    fun producer():T
}

interface Consumer<in T>{
    fun consumer(t:T)
}
```
表示这个类型只能用来输入或输出。




### Kotlin 中的星号(*) 


Kotlin 中 * 相当于 Java 中的 ？

```

//Java
List<?> list;
等效于 
List<? extends Object> list;


// Kotlin

var list:List<*>
等效于
var list:List<out Any>
```
如果类型声明已经有了 in 或者 out，那么这个限制在变量声明是也依然存在，不会被 * 去掉，如下例：

```
interface Conter<out T : Number>{
    fun count(): T
}
// 虽然变量在声明时类型参数为 *，但是其效果依旧是 out Numbuer
var counter:Counter<*> =
```



### 4. Java 声明中的上界和下界


Java 在类型声明时可以指定类型声明的上界,和 ？ extends 不同

```
class Bird<T extends Animal & Food>{
    ...
}


class Bird<T:Animal>{
    ...
}

class Bird<T> where T:Animal, T:Food{
    ...
}
```


### 5. Kotin 与 Java 的不同


Kotlin中有一个 Java 中没有的关键字： refield

Java 中的类型参数(比如说 T)，它不是一个真正的类型， 而仅仅是一个代号， 不能把它作为一个普通的类型来用：

```
<T> void test(Object item){
    if(item instanceof T){// 这样是不可以的
        sout(item);
    }
}
```
同样的在 Kotlin 中这样也是不可以的：
```
fun <T> test(item: Any){
    if(item is T){// 这样不可以
        sout(item)
    }
}
```
但是在 Kotlin 中只有添加一个 refield 关键字就可以解除以上限制：

```
inline fun <refield T> test(item: Any){
    if(item is T){// 这样不可以
        sout(item)
    }
}
```

但是 refield 自身有限制，只能用在 inline 函数上


  
---

[Kotlin 泛型](https://www.bilibili.com/video/av66340216)