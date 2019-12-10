---
title: 记 Retrofit 下一次网络请求的实现流程
tags:
---


# 1. Retrofit 动态代理的实现


如何查看生成的动态代理类：






# 2.Retrofit 动态代理的实现


## 2.1
```
    public <T> T create(final Class<T> service) {
        Utils.validateServiceInterface(service);
        if (validateEagerly) {
            eagerlyValidateMethods(service);
        }
        return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[]{service},
                new InvocationHandler() {
                // 在 Android 平台上获得为 Andorid 类实例对象
                    private final Platform platform = Platform.get();

                    @Override
                    public Object invoke(Object proxy, Method method, @Nullable Object[] args)
                            throws Throwable {
                        // If the method is a method from Object then defer to normal invocation.
                        // （如果方法是来自对象的方法，则遵从正常调用）
                        if (method.getDeclaringClass() == Object.class) {
                            return method.invoke(this, args);
                        }
                        // 默认为 false，不会进入代码块
                        if (platform.isDefaultMethod(method)) {
                            return platform.invokeDefaultMethod(method, service, proxy, args);
                        }
                        //根据我们的method将其包装成ServiceMethod
                        ServiceMethod<Object, Object> serviceMethod =
                                (ServiceMethod<Object, Object>) loadServiceMethod(method);

//            通过ServiceMethod和方法的参数构造retrofit2.OkHttpCall对象
                        // 见 2.9
                        OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);

//              通过serviceMethod.callAdapter.adapt()方法，将OkHttpCall进行代理包装，
// 最后返回的对象是 {@link ExecutorCallAdapterFactory#ExecutorCallbackCall},
// 得到 Call 对象 或自定义 对象如 Observer
                        return serviceMethod.callAdapter.adapt(okHttpCall);
                    }
                });
    }

```


## 2.2 loadServiceMethod 

```
ServiceMethod<?, ?> loadServiceMethod(Method method) {
    // 第一次进入，没有该方法的缓存
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;
    synchronized (serviceMethodCache) {
        result = serviceMethodCache.get(method);
        if (result == null) {
            result = new ServiceMethod.Builder<>(this, method).build();
            serviceMethodCache.put(method, result);
        }
    }
    return result;
}
```
loadServiceMethod 的主要作用是将 Method 实例对象包装成 ServiceMethod 对象返回。

## 2.3 ServiceMethod$Builder#builder


```
/**
 * 将 相应的方法包装成相应的 ServiceMethod 类对象
 * @return
 */
public ServiceMethod build() {
    // 见 2.4
    callAdapter = createCallAdapter();//获取对应的 calladdpter
    responseType = callAdapter.responseType();//返回的是我们方法的实际类型，例如:Call<User>,则返回User类型
    if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"
                + Utils.getRawType(responseType).getName()
                + "' is not a valid response body type. Did you mean ResponseBody?");
    }
    // 见2.6
    responseConverter = createResponseConverter();// 将 http 数据解析成相应的 Java 对象
    //遍历方法接口注解
    for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
    }
    if (httpMethod == null) {
        throw methodError("HTTP method annotation is required (e.g., @GET, @POST, etc.).");
    }
    if (!hasBody) {
        if (isMultipart) {
            throw methodError(
                    "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
            throw methodError("FormUrlEncoded can only be specified on HTTP methods with "
                    + "request body (e.g., @POST).");
        }
    }
    int parameterCount = parameterAnnotationsArray.length;
    parameterHandlers = new ParameterHandler<?>[parameterCount];
    for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];//参数类型
        if (Utils.hasUnresolvableType(parameterType)) {
            throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
                    parameterType);
        }
        //参数类型注解
        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
            throw parameterError(p, "No Retrofit annotation found.");
        }
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
    }
    if (relativeUrl == null && !gotUrl) {
        throw methodError("Missing either @%s URL or @Url parameter.", httpMethod);
    }
    if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
        throw methodError("Non-body HTTP method cannot contain @Body.");
    }
    if (isFormEncoded && !gotField) {
        throw methodError("Form-encoded method must contain at least one @Field.");
    }
    if (isMultipart && !gotPart) {
        throw methodError("Multipart method must contain at least one @Part.");
    }
    return new ServiceMethod<>(this);
}
```


## 2.4 createCallAdapter

```
private CallAdapter<T, R> createCallAdapter() {
    Type returnType = method.getGenericReturnType();//获取方法返回的指的类型
    if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
                "Method return type must not include a type variable or wildcard: %s", returnType);
    }
    if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
    }
    Annotation[] annotations = method.getAnnotations();//获取方法上的所有注解
    try {
        //noinspection unchecked
        return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
    } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create call adapter for %s", returnType);
    }
}
```


## 2.5 Retrofit#nextCallAdapter


callAdapter() 直接调用了 nextCallAdapter() 方法。

需要立即 CallAdapter 为何物，具体查看 [CallAdapter]()
```
public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
                                         Annotation[] annotations) {
    
    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
        // Android 中默认实现为：ExecutorCallAdapterFactory
        // 将返回值信息、注解传入，查看是否有对应的 CallAdapter，比如 returnType 为 Obsever，而 annotations 为 @GET
        CallAdapter<?, ?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
        if (adapter != null) {
            return adapter;
        }
    }
    ...
}
```


## 2.6 ServiceMethod$Build#createResponseConverter

至此，我们获得了 CallAdapter 对象。

现在继续 2.3 中的步骤向下分析。

```
private Converter<ResponseBody, T> createResponseConverter() {
    // 获得注解信息
    Annotation[] annotations = method.getAnnotations();
    return retrofit.responseBodyConverter(responseType, annotations);
}
```

## 2.7 Retrofit#nextResponseBodyConverter

```
public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
        @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
        // Converter 的默认实现为 BuiltInConverters
        Converter<ResponseBody, ?> converter =
                converterFactories.get(i).responseBodyConverter(type, annotations, this);
        if (converter != null) {
            //noinspection unchecked
            return (Converter<ResponseBody, T>) converter;
        }
    }
    ...
}
```

可以看到这一步操作和获得 CallAdapter 对象类似，需要重点理解的类为 Converter，具体可以参见 [Converter]().


## 2.8 ServiceMethod$Build#parseMethodAnnotation

```
private void parseMethodAnnotation(Annotation annotation) {
    if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
    } else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
    } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
    } else if (annotation instanceof PATCH) {
        parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
    } else if (annotation instanceof POST) {
        parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
    } else if (annotation instanceof PUT) {
        parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
    } else if (annotation instanceof OPTIONS) {
        parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
    } else if (annotation instanceof HTTP) {
        HTTP http = (HTTP) annotation;
        parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
    } else if (annotation instanceof retrofit2.http.Headers) {
        String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
        if (headersToParse.length == 0) {
            throw methodError("@Headers annotation is empty.");
        }
        headers = parseHeaders(headersToParse);
    } else if (annotation instanceof Multipart) {
        
        isMultipart = true;
    } else if (annotation instanceof FormUrlEncoded) {
        isFormEncoded = true;
    }
}
```

针对注解进行解析，从而对相应变量进行赋值。

经过一系列操作，最终生成了 ServiceMethod 对象，接着 2.1 继续分析接下去的步骤。

## 2.9 构建 OkHttpCall 对象

```
 OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
```

使用上面一系列步骤生成的 ServiceMethod 结合参数类别 args 生成 OkHttpCall 对象。



## 2.10 callAdapter#adapt

接下来的操作如下，返回指定的 CallAdapter 对应的对象
```
return serviceMethod.callAdapter.adapt(okHttpCall);
```

在这里以默认的 ExecutorCallAdapterFactory 为例进行分析：

```
public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
        return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Object, Call<?>>() {
        @Override
        public Type responseType() {
            return responseType;
        }
        @Override
        public Call<Object> adapt(Call<Object> call) {
            return new ExecutorCallbackCall<>(callbackExecutor, call);
        }
    };
}

static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;
    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
        this.callbackExecutor = callbackExecutor;
        this.delegate = delegate;
    }
    @Override
    public void enqueue(final Callback<T> callback) {
        checkNotNull(callback, "callback == null");
        //delegate 实际为 OkHttpCall
        delegate.enqueue(new Callback<T>() {
            @Override
            public void onResponse(Call<T> call, final Response<T> response) {
                /**
                 * 线程池的执行
                 * 将 Runnable 发送到 UI 线程中
                 */
                callbackExecutor.execute(new Runnable() {
                    @Override
                    public void run() {
                        if (delegate.isCanceled()) {
                            // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                            callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                        } else {
                            callback.onResponse(ExecutorCallbackCall.this, response);
                        }
                    }
                });
            }
            @Override
            public void onFailure(Call<T> call, final Throwable t) {
                callbackExecutor.execute(new Runnable() {
                    @Override
                    public void run() {
                        callback.onFailure(ExecutorCallbackCall.this, t);
                    }
                });
            }
        });
    }
}
```


至此获得了 Call 对象，使用 Retrofit 应用会 Call 类十分熟悉，使用默认的 CallAdapter 时，我们动态代理接口中的方法的返回值必须为 Call,比如如下定义：

```
public interface Server {

    @GET("/sherpa-web-api/newcoupon")
    Call<CouponResp> checkCoupon(@Query("customId") String customId, @Query("couponNumber") String code, @Query("totolValue") int totalValue);
}
```

在 Activity 中使用 Retrofit 进行网络请求：

```

RetrofitFactory.getInstance().checkCoupon("2018040202512043", "133", 315)
        
        .enqueue(new Callback<CouponResp>() {
            @Override
            public void onResponse(Call<CouponResp> call, Response<CouponResp> response) {
                CouponResp mCouponResp = response.body();
                LogUtil.d(mCouponResp.toString());
            }
            @Override
            public void onFailure(Call<CouponResp> call, Throwable t) {
            }
        });
```

至此，动态代理在 Retrofit 中作用一目了然了，最终结果就是获得 Call 实例对象，对于以上代码对应的过程为：
```
RetrofitFactory.getInstance().checkCoupon("2018040202512043", "133", 315)
```


# 3. 第二阶段：发起网络请求

```
public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");
    //delegate 实际为 OkHttpCall，见 3.1
    delegate.enqueue(new Callback<T>() {
        @Override
        public void onResponse(Call<T> call, final Response<T> response) {
            /**
             * 线程池的执行
             * 将 Runnable 发送到 UI 线程中
             */
            callbackExecutor.execute(new Runnable() {
                @Override
                public void run() {
                    if (delegate.isCanceled()) {
                        // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                        callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                    } else {
                        callback.onResponse(ExecutorCallbackCall.this, response);
                    }
                }
            });
        }
        @Override
        public void onFailure(Call<T> call, final Throwable t) {
            callbackExecutor.execute(new Runnable() {
                @Override
                public void run() {
                    callback.onFailure(ExecutorCallbackCall.this, t);
                }
            });
        }
    });
}
```

获取 Call 实例对象，进行接下来的操作。


## 3.1 Okhttp#enqueue


```
public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");
    okhttp3.Call call;
    Throwable failure;
    synchronized (this) {
        executed = true;
        call = rawCall;
        failure = creationFailure;
        if (call == null && failure == null) {
            try {
                // 见 3.2
                call = rawCall = createRawCall();
            } catch (Throwable t) {
                throwIfFatal(t);
                failure = creationFailure = t;
            }
        }
    }
    if (failure != null) {
        callback.onFailure(this, failure);
        return;
    }
    if (canceled) {
        call.cancel();
    }
    call.enqueue(new okhttp3.Callback() {
        @Override
        public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
                throws IOException {
            Response<T> response;
            try {
                response = parseResponse(rawResponse);
            } catch (Throwable e) {
                callFailure(e);
                return;
            }
            callSuccess(response);
        }
        @Override
        public void onFailure(okhttp3.Call call, IOException e) {
            callFailure(e);
        }
        private void callFailure(Throwable e) {
            callback.onFailure(OkHttpCall.this, e);
            }
        }
        private void callSuccess(Response<T> response) {
            callback.onResponse(OkHttpCall.this, response);
            }
        }
    });
}
```



## 3.2 Okhttp#createRawCall

```
private okhttp3.Call createRawCall() throws IOException {
    //完成对 Request 的构建
    Request request = serviceMethod.toRequest(args);
    // serviceMethod.callFactory 即为 OkhttpClient，调用返回的为 RealCall 对象
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
        throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
}
```


Retrofit 网络请求是基于 Okhttp 实现的，至此完成了 Retrofit 向 Okhttp d的转换，接下来的操作即为 Okhttp 的操作流程，具体 Okhttp 中网络请求的过程参见 [Okhttp]()

# 4. 总结


从以上分析可以看出，Retrofit 本质上并不是网络请求库，充其量可以算是一个框架、脚手架，正如 Retrofit 的译意 - "改造"一样，Retrofit 为基于动态代理、注解、反射等技术，对 Okhttp 进行封装，使用 Retrofit 框架完成 Okhttp Call 的自动化构建。


----

**知识链接：**

https://juejin.im/post/5dbebfb2e51d45581a2d782f#heading-15