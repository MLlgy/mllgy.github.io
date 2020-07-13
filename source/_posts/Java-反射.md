---
title: 简单理解 Java 反射机制
date: 2019-04-30 10:35:03
tags: [Java,Inflection]
---


## 根据一个对象获得一个类

```
String str = "adb";
Class class = str.getClass();
```

## 根据一个字符串获得一个类

字符串需要包括完整的包名和类名。

```
Class class = Class.forName("java.lang.String");
Class class2 = Class.forName("android.widget.Button");

// 获得对象的父类型
Class class3 = class2.getSuperClass();
```
<!-- more -->

## 获取类的构造函数

### 实例类
```
public class TestClass {
    private String name = "default";
    private int age;
    private static String tag;

    public TestClass() {
    }

    public TestClass(String name, int age) {
        this.name = name;
        this.age = age;
    }

    private String showName(String string) {
        return name + " && " + string;
    }

    private static void testStaticsMethod() {
        System.out.println("testStaticsMethod");
    }

    public static void printStatics(){
        System.out.println(tag);
    }

    @Override
    public String toString() {
        return "TestClass{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

### 获得所有构造函数

```
TestClass testClass = new TestClass();
Class clazz = testClass.getClass();
String className = clazz.getName();
// 获得所有的构造函数
Constructor[] classList = clazz.getDeclaredConstructors();

// 获得所有 public 构造函数
Constructor[] classList2 = clazz.getConstructors();
```

### 获得无参构造函数

```
Constructor constructor = clazz.getDeclaredConstructor();
// 获得 public 无参构造器
Constructor constructor = clazz.getConstructor();
```

### 获得有参构造器

```
Class params = {String.class,int.class};
Constructor constructor = clazz.getDeclaredConstructor(params);

// 获得指定的 public 有参构造器
Constructor constructor = clazz.getConstructor(params);
```


## 调用类的构造器


### 调用无参构造器

```
Constructor constructor = clazz.getDeclaredConstructor();
Object object = constructor.newInstance();
```
或
```
Object obj = clazz.newInstance();
```

### 调用有参构造器

```
Class params = {String.class,int.class};
Constructor constructor = clazz.getDeclaredConstructor(params);
Object obj = constructor.newInstance("Mike",23);
```

## 调用方法

### 调用私有实例方法

```
//通过反射获得实例对象
Class clazz = Class.forName("reflect.TestClass");
Class[] classes = {String.class, int.class};
Constructor constructor = clazz.getConstructor(classes);
Object object = constructor.newInstance("test", 1);
TestClass testClass = (TestClass) object;

//获得指定的 private 方法
Class[] params = {String.class};
Method method = clazz.getDeclaredMethod("showName", params);
method.setAccessible(true);

// 调用指定对象的的方法
Object[] argList = {"call private method"};
Object returnParam = method.invoke(testClass, argList);
System.out.println(returnParam);
```

### 调用静态方法

```
// 获取私有静态方法
Class clazz = Class.forName("reflect.TestClass");
Method method = clazz.getDeclaredMethod("testStaticsMethod");
method.setAccessible(true);

//调用
method.invoke(null);
```

## 获得类的实例变量并修改


### 非静态变量

```
// 通过反射获得类实例
Class clazz = Class.forName("reflect.TestClass");
Class[] classes = {String.class, int.class};
Constructor constructor = clazz.getConstructor(classes);
Object object = constructor.newInstance("Mike", 3);

//获得实例变量 getField 获取一个类的public 成员变量包括基类
Field field = clazz.getDeclaredField("name");
field.setAccessible(true);

//非静态实例变量传入object ,获得 object 的实例变量的值，并把它包装成类
Object fieldObject = field.get(object);

//修改 object 对应属于的值,注意只会修改 object 这个对象的字段值
field.set(object, "test Field");
System.out.println(fieldObject);

Object fieldObject1 = field.get(object);
System.out.println(fieldObject1);
```

### 静态变量

```
Class clazz = Class.forName("reflect.TestClass");

// 获取类的 name 静态字段
Field field = clazz.getDeclaredField("tag");
field.setAccessible(true);

// 为 static 变量时传入 null,获得静态变量，并包装
Object fieldObject = field.get(null);

//修改值
field.set(fieldObject, "ABCD");

//查看值，静态变量一次修改，一生受用
TestClass.printStatics();
```

## 对泛型进行反射


通过指定类对应的 Class 对象，可以获得该类包含的所有成员变量，如获取指定成员变量的类型：

```
Class<?> clazz = f.getType();
```

但是这种方式只对普通的变量生效，如果该变量有泛型类型，如 Map<String,String>，则不能准确的获得该成员的泛型类型。为了获得指定变量的泛型类型，可以使用如下方法获得该变量的泛型类想：

```
Type type = f.getGenericType();// Field 的方法
```
然后将 Type 类型的对象强制转换为 ParameterizedType 类型对象，ParameterizedType 代表被参数化的类型，也就是增加泛型限制的类型，ParameterizedType 有以下两个方法：
```
Type getRawType();// 返回没有泛型信息的原始类型
Type[] getActualTypeArguments();// 返回泛型参数的类型
```

以下为示例：

```
public class GenericsClass {

    private Map<String,Integer> map;

    public static void main(String[] args) throws Exception {
        Class<GenericsClass> clazz = GenericsClass.class;
        Field field = clazz.getDeclaredField("map");
        // getType 只对普通成员变量有效
        Class<?> fieldType = field.getType();
        System.out.println("map 的数据类型是：" + fieldType);

        Type type = field.getGenericType();
        if(type instanceof ParameterizedType){
            ParameterizedType parameterizedType = (ParameterizedType) type;
            Type rawType = parameterizedType.getRawType();
            System.out.println("map 的原始类型是：" + rawType);
            // 取得泛型的的泛型参数
            System.out.println("取得泛型的的泛型参数:");
            Type[] types = parameterizedType.getActualTypeArguments();
            for(int i = 0;i < types.length;i++){
                System.out.println("map 的第 " + i + " 个泛型类型是：" + types[i]);
            }
        }else {
            System.out.println("取得泛型类型错误");
        }
    }
}
```

打印日志：

```
map 的数据类型是：interface java.util.Map
map 的原始类型是：interface java.util.Map
取得泛型的的泛型参数:
map 的第 0 个泛型类型是：class java.lang.String
map 的第 1 个泛型类型是：class java.lang.Integer
```

使用以上 API 可以获得泛型成员变量的类型参数。