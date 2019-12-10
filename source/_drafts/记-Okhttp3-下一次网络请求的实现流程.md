---
title: 记 Okhttp3 下一次网络请求的实现流程
tags:
---



# 1. 


## 1.1 RealCall#enqueue

```
public void enqueue(Callback responseCallback) {
    synchronized (this) {
        if (executed) throw new IllegalStateException("Already Executed");
        executed = true;
    }
    captureCallStackTrace();
    // Dispatcher 调度器
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```




## 1.2 Dispatcher#enqueue

异步调用：

```
synchronized void  enqueue(AsyncCall call) {
    /**
     * 正在运行的异步请求队列的大小 < 最大并发请求数  同一个host发起的请求数 < 并且每个主机最大请求数
     *
     * 满足该要求，把 Call 添加到运行着的消息的队列
     * 不满足的话，把消息添加到将要运行的信息的队列
     */
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
        runningAsyncCalls.add(call);
        // 线程池执行请求
        executorService().execute(call);
    } else {
        // 加入相应的队列中
        readyAsyncCalls.add(call);
    }
}
```

获得相应的线程池对象：

```
public synchronized ExecutorService executorService() {
    if (executorService == null) {
        executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
}
```


## 1.3 AsyncCall#execute

```
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;
    AsyncCall(Callback responseCallback) {
        super("OkHttp %s", redactedUrl());
        this.responseCallback = responseCallback;
    }
    String host() {
        return originalRequest.url().host();
    }
    Request request() {
        return originalRequest;
    }
    RealCall get() {
        return RealCall.this;
    }
    @Override
    protected void execute() {
        boolean signalledCallback = false;
        try {
            /**
             * 开始真正的发起网络请求
             */
            Response response = getResponseWithInterceptorChain();
            if (retryAndFollowUpInterceptor.isCanceled()) {
                signalledCallback = true;
                responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
            } else {
                signalledCallback = true;
                responseCallback.onResponse(RealCall.this, response);
            }
        } catch (IOException e) {
            if (signalledCallback) {
                // Do not signal the callback twice!
                Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
            } else {
                responseCallback.onFailure(RealCall.this, e);
            }
        } finally {
            client.dispatcher().finished(this);
        }
    }
}
```

## 1.4 RealCall#getResponseWithInterceptorChain


```
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    // 在 CacheInterceptor 中传入 缓存路径、大小等相关的对象
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
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
拦截器集合为 Okhttp 流程的核心之一，通过责任链模式驱动网络请求一步一步向下进行，每个拦截器的具体作用可以查看 [拦截器]()


# 2. 关注的两个点：建立连接、网络传输

针对标题中关注的两点，需要关注的拦截器为 ConnectInterceptor 和 CallServerInterceptor。







---

OkHttp3 的请求方式一共有三种：

* Http1Codec
* Http2Codec
* RealWebSocket


几个关键函数和类：

findHealthyConnection
findConnection
connectTunnel
createTunnel
RealConnection
ConnectInterceptor：建立连接的拦截器
CallServerInterceptor: 在建立连接的基础上进行数据交互
StreamAllocation：协调 Connections  Streams   Calls 直接的关系，流分配器