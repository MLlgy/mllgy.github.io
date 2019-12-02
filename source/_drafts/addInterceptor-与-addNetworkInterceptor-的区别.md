---
title: Okhttp3 中addInterceptor 与 addNetworkInterceptor 的区别
tags: 
---

![](https://images2018.cnblogs.com/blog/1105320/201804/1105320-20180403182754939-2128769288.png)

通过 addInterceptor 添加的拦截器称为 **应用拦截器**，通过 addNetworkInterceptor 添加的拦截器称为 **网络拦截器**。
由官方给出的示意图可知，两者都能够对请求的 requests 和 responses 进行拦截，并进行相关的操作，但是 **两者不同的时在整个网络请求链中位置不同**：

具体可以看一下 OkHttp 的源码：

```
    Response getResponseWithInterceptorChain() throws IOException {
        // Build a full stack of interceptors.
        List<Interceptor> interceptors = new ArrayList<>();
        // 首先添加 应用拦截器
        interceptors.addAll(client.interceptors());
        // 然后添加 OkhttpCore 的核心拦截器，这是 Okhttp 能够完成网络请求的核心代码
        interceptors.add(retryAndFollowUpInterceptor);
        interceptors.add(new BridgeInterceptor(client.cookieJar()));
        interceptors.add(new CacheInterceptor(client.internalCache()));// 在 CacheInterceptor 中传入 缓存路径、大小等相关的对象
        interceptors.add(new ConnectInterceptor(client));
        // 接着添加 网络拦截器
        if (!forWebSocket) {
            interceptors.addAll(client.networkInterceptors());
        }
        interceptors.add(new CallServerInterceptor(forWebSocket));
        //originalRequest 最原始的 request
        Interceptor.Chain chain = new RealInterceptorChain(
                interceptors, null, null, null, 0, originalRequest);
        return chain.proceed(originalRequest);
    }
```

* 应用拦截器在 Application 与 OkhttpCore 之间。
* 网络拦截器在 OkhttpCore 与 NetWork 服务器之间。

由于其位置的不同，那么 **应用拦截器** 能够拦截的是应用发起的请求的requests、responses以及其他操作，而 **网络拦截器** 可以拦截的是 OkhttpCore 与 NetWork 服务器间网络请求的 requests、responses。

由于存在重定向等操作，由应用发起的一次网络请求可能会多次请求网络服务器，这时 应用拦截器 与 网络拦截器 的不同就显示出来了：

* 应用拦截器 只能拦截到一次网络请求相关的 requests、responses 以及其他信息。
* 由于存在重定向，网络拦截器 可以拦截到多次 OkhttpCore 与 NetWork 服务器间的requests、responses 以及其他信息。
