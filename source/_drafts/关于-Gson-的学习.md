---
title: 关于 Gson 的学习
tags:
---
为什么可以通过匿名内部类可以在运行期获得泛型参数的类型呢？这是因为 **泛型的类型擦除并不是完全的将所有信息擦除**，而会 **将类型信息放在所属 class 的常量池中**，这样我们就可以通过相应的方式获得类型信息，而匿名内部类就可以实现这个功能。

[Java 将泛型信息存储在何处](https://stackoverflow.com/questions/937933/where-are-generic-types-stored-in-java-class-files/937999#937999)：类信息的签名中。

**匿名内部类在初始化的时候就会绑定父类或者父接口的信息，这样就能通过获取父类或父接口的泛型类型信息，来实现我们的需求**，可以通过利用此来设计一个获得所有类型信息的泛型类：
```
open class GenericsToken<T>{
    var type : Type = Any::class.java
    init {
        val superClass = this.javaClass.genericSuperclass
        type = (superClass as ParameterizedType).actualTypeArguments[0]
    }
}

fun main(args: Array<String>) {
    // 创建一个匿名内部类
    val oneKt = object:GenericsToken<Map<String,String>>(){}
    println(oneKt.type)
}
```

打印日志：
```
java.util.Map<java.lang.String, ? extends java.lang.String>
```

至于如果获得参数化类型，可参见此博客：[ParameterizedType应用，java反射，获取参数化类型的class实例](https://blog.csdn.net/datouniao1/article/details/53788018)


## 2. Gson 解析



其实正是因为类型擦除的原因，在使用 Gson 反序列化对象的时候除了制定泛型参数，还需要传入一个 class：
```
UserBean user = new Gson().fromJson(jsonStr, UserBean.class);
```
## 2.1 Gson#fromJson
```
public <T> T fromJson(String json, Class<T> classOfT) throws JsonSyntaxException {
  Object object = fromJson(json, (Type) classOfT);
  return Primitives.wrap(classOfT).cast(object);
}
```
在 Java 中所有的 Class 都是 Type 的子类，在 Java 中 Type 是所有类型的父接口，包括原始类型、参数化类型、数组类型、类型变量和基本数据类型。


## 2.2 Gson#fromJson





```
// typeOfT: xxx.User
public <T> T fromJson(String json, Type typeOfT) throws JsonSyntaxException {
  if (json == null) {
    return null;
  }
  StringReader reader = new StringReader(json);
  T target = (T) fromJson(reader, typeOfT);
  return target;
}
```

## 2.3 Gson#fromJson


```
public <T> T fromJson(Reader json, Type typeOfT) throws JsonIOException, JsonSyntaxException {
    //见 2.4 
  JsonReader jsonReader = newJsonReader(json);
  // 见 2.5
  T object = (T) fromJson(jsonReader, typeOfT);
  assertFullConsumption(object, jsonReader);
  return object;
}
```

## 2.4 Gson#newJsonReader

```
public JsonReader newJsonReader(Reader reader) {
  JsonReader jsonReader = new JsonReader(reader);
  jsonReader.setLenient(lenient);
  return jsonReader;
}
```


JsonReader 是解析 Json 的关键类。

## 2.5 Gson#fromJson(JsonReader reader, Type typeOfT)

```
public <T> T fromJson(JsonReader reader, Type typeOfT) throws JsonIOException, JsonSyntaxException {
  boolean isEmpty = true;
  boolean oldLenient = reader.isLenient();
  reader.setLenient(true);
  try {
    // 见 2.
    reader.peek();
    isEmpty = false;
    // 获取 TypeToken，见 2.6 ，获得的 TypeToken 为 xxx.User，typeOfT 同样为 xxx.User
    TypeToken<T> typeToken = (TypeToken<T>) TypeToken.get(typeOfT);
    // 获取指定 Type 的 TypeAdapter，见 2.7 
    TypeAdapter<T> typeAdapter = getAdapter(typeToken);
    // 见 2.12 
    T object = typeAdapter.read(reader);
    return object;
  } catch (EOFException e) {
    /*
     * For compatibility with JSON 1.5 and earlier, we return null for empty
     * documents instead of throwing.
     */
    if (isEmpty) {
      return null;
    }
    throw new JsonSyntaxException(e);
  } catch (IllegalStateException e) {
    throw new JsonSyntaxException(e);
  } catch (IOException e) {
    // TODO(inder): Figure out whether it is indeed right to rethrow this as JsonSyntaxException
    throw new JsonSyntaxException(e);
  } finally {
    reader.setLenient(oldLenient);
  }
}
```

## 2.6 TypeToken#get

```
public static TypeToken<?> get(Type type) {
  return new TypeToken<Object>(type);
}
```

```
TypeToken(Type type) {
  // 获取 type，见 2.7 
  this.type = $Gson$Types.canonicalize($Gson$Preconditions.checkNotNull(type));
  // 获取 rawType，见 2.8
  this.rawType = (Class<? super T>) $Gson$Types.getRawType(this.type);
  this.hashCode = this.type.hashCode();
}
```
## GsonTypes#canonicalize

```
public static Type canonicalize(Type type) {
  if (type instanceof Class) {
    Class<?> c = (Class<?>) type;
    return c.isArray() ? new GenericArrayTypeImpl(canonicalize(c.getComponentType())) : c;
  } else if (type instanceof ParameterizedType) {
    ParameterizedType p = (ParameterizedType) type;
    return new ParameterizedTypeImpl(p.getOwnerType(),
        p.getRawType(), p.getActualTypeArguments());
  } else if (type instanceof GenericArrayType) {
    GenericArrayType g = (GenericArrayType) type;
    return new GenericArrayTypeImpl(g.getGenericComponentType());
  } else if (type instanceof WildcardType) {
    WildcardType w = (WildcardType) type;
    return new WildcardTypeImpl(w.getUpperBounds(), w.getLowerBounds());
  } else {
    // type is either serializable as-is or unsupported
    return type;
  }
}
```
## 2.8 $Gson$Types.getRawType



## 2.9 Gson#getAdapter


```
public <T> TypeAdapter<T> getAdapter(TypeToken<T> type) {
  // 如何内存中存在(ConcurrentHashMap)，则取出
  TypeAdapter<?> cached = typeTokenCache.get(type == null ? NULL_KEY_SURROGATE : type);
  if (cached != null) {
    return (TypeAdapter<T>) cached;
  }
  // threadCalls(ThreadLocal) 线程本地变量，如果存在则返回
  Map<TypeToken<?>, FutureTypeAdapter<?>> threadCalls = calls.get();
  boolean requiresThreadLocalCleanup = false;
  if (threadCalls == null) {
    threadCalls = new HashMap<TypeToken<?>, FutureTypeAdapter<?>>();
    calls.set(threadCalls);
    requiresThreadLocalCleanup = true;
  }
  // the key and value type parameters always agree
  FutureTypeAdapter<T> ongoingCall = (FutureTypeAdapter<T>) threadCalls.get(type);
  if (ongoingCall != null) {
    return ongoingCall;
  }
  try {
    FutureTypeAdapter<T> call = new FutureTypeAdapter<T>();
    threadCalls.put(type, call);
    // 遍历 factories ，获得 TypeAdapter，factories 的初始化见 2.10
    for (TypeAdapterFactory factory : factories) {
      //   
      TypeAdapter<T> candidate = factory.create(this, type);
      if (candidate != null) {
        call.setDelegate(candidate);
        typeTokenCache.put(type, candidate);
        return candidate;
      }
    }
    throw new IllegalArgumentException("GSON cannot handle " + type);
  } finally {
    threadCalls.remove(type);
    if (requiresThreadLocalCleanup) {
      calls.remove();
    }
  }
}
```

## 2.10 new Gson()

```
Gson(...){
    // Gson 内部的 TypeAdapter
    factories.add(TypeAdapters.JSON_ELEMENT_FACTORY);
    factories.add(ObjectTypeAdapter.FACTORY);
    // 添加排除器
    factories.add(excluder);
    // 用户自定义的 TypeAdapter
    factories.addAll(typeAdapterFactories);
    // Java 平台数据类型对应的 TypeAdapter
    factories.add(TypeAdapters.STRING_FACTORY);
    factories.add(TypeAdapters.INTEGER_FACTORY);
    factories.add(TypeAdapters.BOOLEAN_FACTORY);
    factories.add(TypeAdapters.BYTE_FACTORY);
    factories.add(TypeAdapters.SHORT_FACTORY);
    ....
}
```


## 2.11 TypeAdapters#newFactory

在初始化时，将 Gson 支持的所有 TypeAdapter 添加到集合中，查看一个具体的 TypeAdapter：

```
public static final TypeAdapterFactory STRING_FACTORY = newFactory(String.class, STRING);

```
对应 STRING 的实例变量，具体如下：
```
public static final TypeAdapter<String> STRING = new TypeAdapter<String>() {
  @Override
  public String read(JsonReader in) throws IOException {
    JsonToken peek = in.peek();
    if (peek == JsonToken.NULL) {
      in.nextNull();
      return null;
    }
    if (peek == JsonToken.BOOLEAN) {
      return Boolean.toString(in.nextBoolean());
    }
    return in.nextString();
  }
  @Override
  public void write(JsonWriter out, String value) throws IOException {
    out.value(value);
  }
};
```

```
// typey: String.class,typeAdapter：STRING
public static <TT> TypeAdapterFactory newFactory(
    final Class<TT> type, final TypeAdapter<TT> typeAdapter) {
  return new TypeAdapterFactory() {

    // 在此处用于判断要处理的类型是不是 String，如果是的话就返回 STRING
    @SuppressWarnings("unchecked") 
    @Override public <T> TypeAdapter<T> create(Gson gson, TypeToken<T> typeToken) {
      return typeToken.getRawType() == type ? (TypeAdapter<T>) typeAdapter : null;
    }
    @Override public String toString() {
      return "Factory[type=" + type.getName() + ",adapter=" + typeAdapter + "]";
    }
  };
}

```

## 2.12 TypeAdapter.read


如果按照 2.11 展示的 STRING ，那么调起的动作应该是：

```
String name = new Gson().fromJson("Mike",String.class);
```

那么会执行 STRING.read 方法，最终返回对应的值-- Mike，但是在反序列化时，我们更常用的是反序列类，具体如下：

```
new Gson().fromJson("{\"name\":\"Mike\",\"age\": 2}",User.class);
```




----

因为 Gson 没有办法根据 T 直接去反序列化，所以 Gson 也是使用了相同的设计，通过匿名内部类获得相应的类型参数，然后传到 fromJson 中进行反序列化。

看一下在 Kotlin 中我们使用 Gson 来进行泛型类的反序列化：

```
val  json = "...."
val rType = object: TypeToken<List<String>>(){}.type// 获得反序列化的数据类型
val stringList = Gson().fromJson<List<String>>(json,rType)
```

当然可以直接传输数据类型：

```
// 存在局限，比如不能传入 List<String> 的数据类型
val stringList = Gson().fromJson<String::class.java>(json,rType)
```

在 Kotlin 中除了使用匿名内部类获得泛型参数外，还可以使用内联函数来获取。



----

**知识链接：**

[我眼中的Java-Type体系(2)](https://www.jianshu.com/p/e8eeff12c306)


[一个容错的 Gson 新世界](https://mp.weixin.qq.com/s?__biz=MzIxNzU1Nzk3OQ==&mid=2247485850&idx=1&sn=436cbfc7b3eb2549fe664063b693748f&chksm=97f6b72ea0813e38b6e7a7b5f7736465c252f7ba739fd1510ec52b8fb78adbe8f25c701a1396&scene=38#wechat_redirect)

[Gson全解析（中）-TypeAdapter的使用
](https://www.jianshu.com/p/8cc857583ff4)


UnsafeAllocator#create()#


UnsafeAllocator : 可以不通过构造函数分配对象

  Unsafe：allocateInstance 方法