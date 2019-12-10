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


ConnectInterceptor 中的代码如下：

```
public final class ConnectInterceptor implements Interceptor {
    public final OkHttpClient client;

    public ConnectInterceptor(OkHttpClient client) {
        this.client = client;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        RealInterceptorChain realChain = (RealInterceptorChain) chain;
        Request request = realChain.request();
        // StreamAllocation 对象在执行拦截器之前为 null，在第一个拦截器中实例化 StreamAllocation 对象，并在接下来的拦截器中传递
        StreamAllocation streamAllocation = realChain.streamAllocation();

        // We need the network to satisfy this request. Possibly for validating a conditional GET.
        boolean doExtensiveHealthChecks = !request.method().equals("GET");
        // 见 2.1
        HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
        RealConnection connection = streamAllocation.connection();

        return realChain.proceed(request, streamAllocation, httpCodec, connection);
    }
}
```


## 2.1 StreamAllocation#newStream

```
public HttpCodec newStream(OkHttpClient client, boolean doExtensiveHealthChecks) {
    int connectTimeout = client.connectTimeoutMillis();
    int readTimeout = client.readTimeoutMillis();
    int writeTimeout = client.writeTimeoutMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();
    try {
        RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
                writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);
        HttpCodec resultCodec = resultConnection.newCodec(client, this);
        synchronized (connectionPool) {
            codec = resultCodec;
            return resultCodec;
        }
    } catch (IOException e) {
        throw new RouteException(e);
    }
}
```


## 2.2 StreamAllocation#findHealthyConnection


```
private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
                                             int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
        throws IOException {
    // 直到找到健康的连接        
    while (true) {
        // 见 2.3 
        RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
                connectionRetryEnabled);
        synchronized (connectionPool) {
            if (candidate.successCount == 0) {
                return candidate;
            }
        }
        if (!candidate.isHealthy(doExtensiveHealthChecks)) {
            noNewStreams();
            continue;
        }
        return candidate;
    }
}
```
该方法会寻找健康的连接并返回，如果没有找到会继续寻找。


## 2.3 findConnection


```
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
                                      boolean connectionRetryEnabled) throws IOException {
    Route selectedRoute;
    // 对连接池进行加锁，防止多个线程获得线程池中同一个连接对象。
    synchronized (connectionPool) {
        // Attempt to use an already-allocated connection.  尝试拿到已存在的 connection ，直接返回
        RealConnection allocatedConnection = this.connection;
        if (allocatedConnection != null && !allocatedConnection.noNewStreams) {
            return allocatedConnection;
        }
        // 从 connectionpool 连接池中拿到 connection 直接返回
        Internal.instance.get(connectionPool, address, this, null);
        if (connection != null) {
            return connection;
        }
        selectedRoute = route;
    }
    // 如果需要路由选择，那么进行一次路由选择后，再次寻找可用的连接对象。
    if (selectedRoute == null) {
        selectedRoute = routeSelector.next();
    }
    RealConnection result;
    synchronized (connectionPool) {
        if (canceled) throw new IOException("Canceled");
        // Now that we have an IP address, make another attempt at getting a connection from the pool.
        // This could match due to connection coalescing.
        Internal.instance.get(connectionPool, address, this, selectedRoute);
        if (connection != null) {
            route = selectedRoute;
            return connection;
        }
        // 如果上面两部操作还是不能找到对应的 Connection ，那么就新建 Connection.
        // 可以异步打断将要执行的握手动作
        route = selectedRoute;
        refusedStreamCount = 0;
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result);
    }
    //进行 TCP + TLS 握手，这是阻塞操作
    result.connect(connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled);
    routeDatabase().connected(result.route());
    Socket socket = null;
    synchronized (connectionPool) {
        // Pool the connection.
        Internal.instance.put(connectionPool, result);
        // 如果另外的一个多路复用的连接，拥有和这个连接通过的 IP，那么关闭这个连接，复用已经存在而连接
        if (result.isMultiplexed()) {
            socket = Internal.instance.deduplicate(connectionPool, address, this);
            result = connection;
        }
    }
    closeQuietly(socket);
    return result;
}
```


这个寻找连接的过程是十分复杂的，主要有以下几个步骤：

1. 如果存在可用的连接，则直接则直接复用，否则继续向下执行；
2. 如果连接池中存在可用的连接，那么则直接复用，否则继续向下执行；
3. 经过以上两个步骤，依旧没有获得可用连接，那么更换路由，继续寻找可用连接；
4. 更换路由后，如果连接池中存在可用连接，则使用该连接，否则继续执行；
5. 新建连接。
6. 连接执行 TCP + TLS 握手



OkHttpClient
```
static {
    Internal.instance = new Internal() {
        @Override
        public void addLenient(Headers.Builder builder, String line) {
            builder.addLenient(line);
        }
        @Override
        public void addLenient(Headers.Builder builder, String name, String value) {
            builder.addLenient(name, value);
        }
        @Override
        public void setCache(Builder builder, InternalCache internalCache) {
            builder.setInternalCache(internalCache);
        }
        @Override
        public boolean connectionBecameIdle(
                ConnectionPool pool, RealConnection connection) {
            return pool.connectionBecameIdle(connection);
        }
        @Override
        public RealConnection get(ConnectionPool pool, Address address,
                                  StreamAllocation streamAllocation, Route route) {
            return pool.get(address, streamAllocation, route);
        }
        @Override
        public boolean equalsNonHost(Address a, Address b) {
            return a.equalsNonHost(b);
        }
        @Override
        public Socket deduplicate(
                ConnectionPool pool, Address address, StreamAllocation streamAllocation) {
            return pool.deduplicate(address, streamAllocation);
        }
        @Override
        public void put(ConnectionPool pool, RealConnection connection) {
            pool.put(connection);
        }
        @Override
        public RouteDatabase routeDatabase(ConnectionPool connectionPool) {
            return connectionPool.routeDatabase;
        }
        @Override
        public int code(Response.Builder responseBuilder) {
            return responseBuilder.code;
        }
        @Override
        public void apply(ConnectionSpec tlsConfiguration, SSLSocket sslSocket, boolean isFallback) {
            tlsConfiguration.apply(sslSocket, isFallback);
        }
        @Override
        public HttpUrl getHttpUrlChecked(String url)
                throws MalformedURLException, UnknownHostException {
            return HttpUrl.getChecked(url);
        }
        @Override
        public StreamAllocation streamAllocation(Call call) {
            return ((RealCall) call).streamAllocation();
        }
        @Override
        public Call newWebSocketCall(OkHttpClient client, Request originalRequest) {
            return new RealCall(client, originalRequest, true);
        }
    };
}
```


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