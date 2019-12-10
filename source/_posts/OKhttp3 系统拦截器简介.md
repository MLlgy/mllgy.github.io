---
title: OKhttp3 核心拦截器、应用拦截器、网络拦截器
date: 2019-01-17 14:31:26
tags: [Okhttp3]
---

2019-12-10 更新，见文末。

### RetryAndFollowUpInterceptor

功能：实现重试、跟踪

实现原理： 

while(true) 死循环，如果发生重定向则构建 Request，进入下一次循环，从新发起网络请求，但是最大重定向次数为 20 次。



* 检验返回的 Response ，如果没有异常（包括请求失败、重定向等），那么执行 `return Response`, return 会直接结束循环操作，将结果返回到下一个拦截器中进行处理。

* 检验返回的 Response ，如果出现异常情况，那么会根据 Response 新建 Request，并且执行一些必要的检查（是否为同一个 connnetion ，是的话抛出异常，不是的话是否旧的 connection 的资源，并新建一个 connection）
，在构建重定向请求时，从 Response 中取出 `Location` 字段，构建重定向后的 Request。下一次循环时，将进行新一轮的拦截器的处理。


<!--more-->

### BridgeInterceptor


#### 功能

* 将用户构建的 Request 请求转换为能够进行网络访问的请求。

在用户构建的 Request 的基础上 **添加了许多的请求头**，具体内容参看代码。

* 将符合网络请求的 Request 进行网络请求。

在责任链模式的过程中，在此拦截器的到响应 Response。

```
Response networkResponse = chain.proceed(requestBuilder.build());
```

* 将请求回来的响应 Response 转化为用户可用的 Response。

主要是根据响应是否对 Response 进行 gzip 压缩，具体是使用 Okio 的库对 Response 进行压缩，并返回 Response。


具体代码可以查看官方 Github 库，也可以查看自己的源码学习库：[BridgeInterceptor](https://github.com/leeGYPlus/SourceCodeLearn/blob/master/app/src/main/java/okhttp3/internal/http/BridgeInterceptor.java)

### CacheInterceptor

功能： 实现缓存功能的拦截器


#### 设置启用缓存功能

在新建 OkhttpClient.Builder 的时候进行设置：

```
File sdcache = getExternalCacheDir();
int cacheSize = 10 * 1024 * 1024;
OkHttpClient.Builder builder = new OkHttpClient.Builder()
    .cache(new Cache(sdcache.getAbsoluteFile(), cacheSize));
mOkHttpClient = builder.build();
```

其底层实现还是 大神 的 开源库 DiskLruCache，如下可以看到：

```
Cache(File directory, long maxSize, FileSystem fileSystem) {
    this.cache = DiskLruCache.create(fileSystem, directory, VERSION, ENTRY_COUNT, maxSize);
}
```

#### 缓存策略的基本流程

##### 1. 获取缓存响应
```
Response cacheCandidate = cache != null
                ? cache.get(chain.request())
                : null;//本地缓存
```
##### 2. 根据 **request** 和 **缓存响应 cacheCandidate** 获取缓存策略

```
CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
```
##### 3. 获取响应缓存策略下的 request 和 response

```
//缓存策略中的请求
Request networkRequest = strategy.networkRequest;
//缓存策略中的响应
Response cacheResponse = strategy.cacheResponse;
```

##### 4. 根据响应缓存策略下的 request 和 response 分情况判断几种具体情况。


###### 1. 缓存响应不为空但是策略的响应为空，关闭缓存响应流
    
```
if (cacheCandidate != null && cacheResponse == null) {
    closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
}
```
    
###### 2. networkRequest 和 cacheResponse 皆为空，构建 504 的响应，==直接返回==。
    
```
if (networkRequest == null && cacheResponse == null) {
    return new Response.Builder()
            .request(chain.request())
            .protocol(Protocol.HTTP_1_1)
            .code(504)
            .message("Unsatisfiable Request (only-if-cached)")
            .body(Util.EMPTY_RESPONSE)
            .sentRequestAtMillis(-1L)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build();
    }
```
    
###### 3. networkRequest 为空，直接使用缓存，==返回缓存响应==。
```
if (networkRequest == null) {
    return cacheResponse.newBuilder()
            .cacheResponse(stripBody(cacheResponse))
            .build();
}
```
    
###### 4. 获取网络请求的响应后，进行操作，此时也要分情况讨论。
```
Response networkResponse = null;
networkResponse = chain.proceed(networkRequest);
```
1. networkResponse 的响应码为 304，说明请求的资源未过期，构建 Response 对象，==直接反正该对象==。

```
if (cacheResponse != null) {
    // 304 304 的标准解释是：Not Modified 客户端有缓冲的文档并发出了一个条件性的请求（
    // 一般是提供If-Modified-Since头表示客户只想比指定日期更新的文档）。
    // 服务器告诉客户，原来缓冲的文档还可以继续使用。
    if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
    networkResponse.body().close();

     // Update the cache after combining headers but before stripping the
    // Content-Encoding header (as performed by initContentStream()).
    cache.trackConditionalCacheHit();
    cache.update(cacheResponse, response);
    return response;
    } else {
        closeQuietly(cacheResponse.body());
    }
}
```

2. 根据 构建 Response，==并直接返回该对象==。

```
Response response = networkResponse.newBuilder()
    .cacheResponse(stripBody(cacheResponse))
    .networkResponse(stripBody(networkResponse))
    .build();
    
....

return response;
```

##### 5. 将 Response 写入缓存

```
if (cache != null) {
    if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
    // Offer this request to the cache.
    CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
    }

    if (HttpMethod.invalidatesCache(networkRequest.method())) {
    try {
        cache.remove(networkRequest);
        } catch (IOException ignored) {
        // The cache cannot be written.
        }
    }
}
```


> 以上被标注为 ==== 的字样，说明执行 `return Response` 操作,直接返回响应，进入下一个拦截器的相关处理。
      
    
### ConnectInterceptor

功能： Opens a connection to the target server and proceeds to the next interceptor。

打开一个面向指定服务器的连接，并且执行下一个拦截器。


#### HttpCodec 

在这个拦截器中 HttpCodec 的作用是编码 Http 请求和解码 Http 响应。根据 HTTP版本不同分为 

* Http1Codec(HTTP/1.1) 
* Http2Codec(HTTP/2)


打开连接的关键代码为：

```
HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
```

以下为具体代码调用链：

```
StreamAllocation#newStream() 
--> this#findHealthyConnection(..) 
-->this#findHealthyConnection(..)//获得连接的顺序：存在的链接 、 连接池、新建一个连接
-->this#findConnection(...)
-->RealConnection#connect(...)// 连接并握手
-->RealConnection#connectTunnel(...)或
   RealConnection#connectSocket(..)(最终都会调用connectSocket(...))
-->Platform.get()#connectSocket(...)
-->socket.connect(address, connectTimeout);//最终可以获得建立连接后的 Socket
```

```
-->RealConnection#newCodec(..)// 返回 HttpCode
```

在 findHealthyConnection() 中有以下代码进行连接：


```
result.connect(connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled);
```

至此连接指定服务器的 connection 已经建立。



### CallServerInterceptor

这是 Okhttp 库中拦截器链的最后一个拦截器，也是这个拦截器区具体发起请求和获取响应。

大致分为以下几个步骤：

1. 写入请求头

```
httpCodec.writeRequestHeaders(request);
```
2. 根据具体情况判断是否读取

3. 根据具体情况判断是否写入相应请求头

```
if (responseBuilder == null) {
                // Write the request body if the "Expect: 100-continue" expectation was met.
                Sink requestBodyOut = httpCodec.createRequestBody(request, request.body().contentLength());
                BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
                request.body().writeTo(bufferedRequestBody);
                bufferedRequestBody.close();
            } else if (!connection.isMultiplexed()) {
                // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection from
                // being reused. Otherwise we're still obligated to transmit the request body to leave the
                // connection in a consistent state.
                streamAllocation.noNewStreams();
            }
```

4. 构建 Response 
```
 Response response = responseBuilder
                .request(request)
                .handshake(streamAllocation.connection().handshake())
                .sentRequestAtMillis(sentRequestMillis)
                .receivedResponseAtMillis(System.currentTimeMillis())
                .build();

```

5. 写入 Response 的 body


```
if (forWebSocket && code == 101) {
            // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
            response = response.newBuilder()
                    .body(Util.EMPTY_RESPONSE)
                    .build();
        } else {
            response = response.newBuilder()
                    .body(httpCodec.openResponseBody(response))
                    .build();
        }
```

至此，网络请求经过拦截器链获得 Response ，那么再按照拦截器链逆向返回 Response，在此过程中对 Response 进行相应的处理。


### 如何驱动拦截器链

具体源码如下：

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
核心代码如下：

```
Interceptor.Chain chain = new RealInterceptorChain(
                interceptors, null, null, null, 0, originalRequest);
chain.proceed(originalRequest)
```
具体实现细节十分简单，不过多详述，请查看源码。

### 总结
在对开源库的研读中，我们首先需要做的是对大致流程有个清晰的认识，但是不能深陷细节、具体实现上在后期对相关功能的具体使用时在进行相关研究。而自己在此过程中，就深陷入细节，针对具体的实现真是绞尽脑汁，最后还是 "一败涂地"。此处再次告诫自己和后来人：对开源库的研读不要纠结于细节、不要纠结于细节、不要纠结于细节。



---

2019-12-10 更新：

Okhttp 的责任链驱动网络请求和响应的进行，和 RxJava 的事件驱动，原本想依照 RxJava 流程图画一下 Okhttp 拦截器的执行流程，但是发现在网络上有针对该流程的，于是引用。

![](/source/images/2019-12-10-02.jpg)

该系列博文也再一次让自己对 Okhttp 有了更深的认识,

[OkHttp 源码解析系列](http://ww4.sinaimg.cn/large/006tNc79gy1g5hvb8euorj30tk0rwn0o.jpg)

