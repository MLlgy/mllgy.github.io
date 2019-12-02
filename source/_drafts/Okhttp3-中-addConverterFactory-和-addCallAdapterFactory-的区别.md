---
title: Okhttp3 中 addConverterFactory 和 addCallAdapterFactory 的区别
tags:
---


### CallAdapter

将一个返回值类型为 R 的 Call 转变成返回值类型为 T 的 Call。
```
public interface CallAdapter<R, T> {
    // 通过适配器将 Http 相应转换为 Java 对象，此函数用于返回该适配器使用到的数据类型。对于 Call<Repo> 来说 responseType 的返回类型为 Repo，
    // 这个类型即为传入 adapter 的参数 Call<R> 中 R 所代表的数据类型。
    // 注意：通常，此处的类型与 Factory 中的 returnType 不同。
    Type responseType();

    // 将 call 转变为 T 
    T adapt(Call<R> call);

    // 基于 Retrofit#create(Class) 中传入的接口中的方法的返回值类型，创建 CallAdapter 实例。
    abstract class Factory {

        // 提取泛型参数的类型上限， 如对于  Map<String, ? extends Runnable> 来说， index 为 1 返回值为 Runnable 
        protected static Type getParameterUpperBound(int index, ParameterizedType type) {
            return Utils.getParameterUpperBound(index, type);
        }

        // 提取传入参数的的原始类型
        protected static Class<?> getRawType(Type type) {
            return Utils.getRawType(type);
        }

        // 在这个方法中判断是否是我们支持的类型，returnType 即Call<Requestbody>和`Observable<Requestbody>`
        // 在 RxJavaCallAdapterFactory 就是判断 returnType是不是Observable<?> 类型
        // 针对接口中返回值类型为 returnType 的方法，返回 call 的适配器，在这个工厂类中不能处理，就返回 null。
        public abstract @Nullable
        CallAdapter<?, ?> get(Type returnType, Annotation[] annotations,
                              Retrofit retrofit);
    }
}
```


没有添加自定义 CallAdapter 之前，我们可以在 Retrofit 接口中定义如下接口：
```
interface MyApi {
   Call<Person> loadA();
   Call<Animal> loadB();
}
```
现在自定义 CallAdapter 如下：

```
public class CustomCallAdapter<R> implements CallAdapter<R, CustomCall<?>> {

    private final Type responseType;

    public CustomCallAdapter(Type mResponseType) {
        responseType = mResponseType;
    }

    @Override
    public Type responseType() {
        return responseType;
    }

    @Override
    public CustomCall<?> adapt(Call<R> call) {
        return new CustomCall<>(call);
    }
}


public class CustomCall<R> {
    public final Call<R> mRCall;


    public CustomCall(Call<R> mRCall) {
        this.mRCall = mRCall;
    }

    public R get() throws IOException {
        return mRCall.execute().body();
    }
}

public class CustomCallAdapterFactory extends CallAdapter.Factory {
    @Nullable
    @Override
    public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {

        Class<?> rawTypr = getRawType(returnType);
        if (rawTypr == CustomCall.class && returnType instanceof ParameterizedType) {
            Type callReturnType = getParameterUpperBound(0, (ParameterizedType) returnType);
            return new CustomCallAdapter(callReturnType);

        }
        return null;
    }

    public static CustomCallAdapterFactory create(){
        return new CustomCallAdapterFactory();
    }
}
```
添加到 Retrofit 的配置中：

```
.addCallAdapterFactory(CustomCallAdapterFactory.create())
```

那么此时可以使用适配器扩展接口中的方法：

```
interface MyApi {
   Call<Person> loadOne();
   Call<Animal> loadTwo();
   CustomCall<Person> loadThree();
}
```

对 loadThree 的调用如下：

```
Person person = myApi.loadThree.get();
```

RxJava2CallAdapterFactory 的实现原理与以上相同，可以参见源码。


### Converter
```
//将 HTTP 返回的数据解析成 Java 对象
public interface Converter<F, T> {
    T convert(F value) throws IOException;

    // Factory 的作用是基于数据类型和目标类型生成 Converter
    abstract class Factory {
        // 和 CallAdater 中相同
        protected static Type getParameterUpperBound(int index, ParameterizedType type) {
            return Utils.getParameterUpperBound(index, type);
        }

        // 和 CallAdater 中相同
        protected static Class<?> getRawType(Type type) {
            return Utils.getRawType(type);
        }
       
        // 将 ResponseBody 转换为指定类型(Type)的数据类 的Converter。
        // 根据接口中的方法的返回值，比如 Call<SimpleResponse>，将 Http response 转换为 SimpleResponse
        public @Nullable
        Converter<ResponseBody, ?> responseBodyConverter(Type type,
                                                         Annotation[] annotations, Retrofit retrofit) {
            return null;
        }

        // 将标注如下注解的数据类型(Type)转换为 RequestBody 的 Converter ， 比如 Body @Body、Part @Part、PartMap @PartMap
        public @Nullable
        Converter<?, RequestBody> requestBodyConverter(Type type,
                                                       Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
            return null;
        }
        
        // 为指定注解的类型转换为 String 的 Converter，比如 Field @Field、 FieldMap @FieldMap、 Header @Header、HeaderMap @HeaderMap、Path @Path、 Query @Query、QueryMap @QueryMap
        public @Nullable
        Converter<?, String> stringConverter(Type type, Annotation[] annotations,
                                             Retrofit retrofit) {
            return null;
        }
    }
}
```
注意 requestBodyConverter 和 responseBodyConverter 两个方法，分针对 Type 到 RequestBody 的转换 和 ResponseBody 到 Type 的转换，注意转换方法的不同。

何时完成 RequestBody 到 Type 的转换，此时的前提是已经完成了网络请求，并且获得了 RequestBody 对象，依次为前提，可以很轻易的导致该动作在何处发生：

1. Okhttp 的同步执行：
    
    OkhttpCall#execute -> parseResponse -> `T body = serviceMethod.toResponse(catchingBody);` -> ServiceMethod#toResponse(responseConverter = createResponseConverter()) -> Retrofit#responseBodyConverter(在该方法中完成 responseBodyConverter 的调用，从而返回转换后的实例)


2. Okhttp 的异步执行：
   
    OkhttpCall#enqueue -> parseResponse -> `T body = serviceMethod.toResponse(catchingBody);` -> ServiceMethod#toResponse(responseConverter = createResponseConverter()) -> Retrofit#responseBodyConverter(在该方法中完成 responseBodyConverter 的调用，从而返回转换后的实例)

对应的 requestBodyConverter 调用时机如下：

```
Request request = serviceMethod.toRequest(args);
```

---

https://blog.csdn.net/chunqiuwei/article/details/82596381