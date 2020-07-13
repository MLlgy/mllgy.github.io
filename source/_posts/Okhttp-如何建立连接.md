---
title: Okhttp 如何建立连接
date: 2020-06-15 15:42:02
tags: [Okhttp3,源码解析]
---

# 1. ConnectInterceptor

在 Okhttp 中，由拦截器组成的链，在 Okhttp 发起请求、响应过程中启动驱动作用，而 ConnectInterceptor 在作为链中的一员，起到建立网络连接的作用，此篇主要阐述 Okhttp 是如何建立连接的。

<!-- more --> 

以下为代码分析。

# 2. ConnectInterceptor 实现细节

以下为 ConnectInterceptor 的相关源代码：

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
        // 获得已经建立的连接
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
        // 寻找可用的的连接，见 2.2
        RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
                writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);
        // 最终返回 Http1Codec 或 Http2Codec 实例对象，该实例对象中持有 Socket 的引用
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
    // 进入循环，直到找到可用的连接        
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
该方法会寻找可用的连接并返回，如果没有找到会继续寻找。

## 2.3 StreamAllocation#findConnection


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
        // 从 connectionpool 连接池中拿到 connection 直接返回，见 2.4
        Internal.instance.get(connectionPool, address, this, null);
        if (connection != null) {
            return connection;
        }
        selectedRoute = route;
    }
    // 如果上面的两步皆拿不到 RealConnection 对象，则进行接下来的操作。

    // 如果需要路由选择，那么进行一次路由选择后，再次寻找可用的连接对象。
    if (selectedRoute == null) {
        selectedRoute = routeSelector.next();
    }
    RealConnection result;
    // 获取 connectionPool  同步锁
    synchronized (connectionPool) {
        if (canceled) throw new IOException("Canceled");
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
    //获得连接后，进行 TCP + TLS 握手，这是阻塞操作，具体见 2.5
    result.connect(connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled);
    // RouteDatabase：失败路由集合
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
4. 更换路由后，如果连接池中存在可用连接，则使用该连接，否则继续向下执行；
5. 新建连接。
6. 连接执行 TCP + TLS 握手


## 2.4 OkHttpClient 代码块

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
## 2.5 RealConnection#connect

```
public void connect(
        int connectTimeout, int readTimeout, int writeTimeout, boolean connectionRetryEnabled) {
RouteException routeException = null;
    List<ConnectionSpec> connectionSpecs = route.address().connectionSpecs();
    ConnectionSpecSelector connectionSpecSelector = new ConnectionSpecSelector(connectionSpecs);
    // sslSocketFactory 是否为 null
    if (route.address().sslSocketFactory() == null) {
        if (!connectionSpecs.contains(ConnectionSpec.CLEARTEXT)) {
            throw new RouteException(new UnknownServiceException(
                    "CLEARTEXT communication not enabled for client"));
        }
        String host = route.address().url().host();
        if (!Platform.get().isCleartextTrafficPermitted(host)) {
            throw new RouteException(new UnknownServiceException(
                    "CLEARTEXT communication to " + host + " not permitted by network security policy"));
        }
    }
    while (true) {
        try {
            // 如果此路由通过 HTTP 代理隧道 HTTPS，则返回 true。
            if (route.requiresTunnel()) {
                // 见 2.6 
                connectTunnel(connectTimeout, readTimeout, writeTimeout);
            } else {
                // 见 2.9 
                connectSocket(connectTimeout, readTimeout);
            }
            // 在经过以上步骤后，建立了连接，判断是否建立 SSL 连接，见 2.11，至此连接建立成功
            establishProtocol(connectionSpecSelector);
            break;
        } catch (IOException e) {
            closeQuietly(socket);
            closeQuietly(rawSocket);
            socket = null;
            rawSocket = null;
            source = null;
            sink = null;
            handshake = null;
            protocol = null;
            http2Connection = null;
            if (routeException == null) {
                routeException = new RouteException(e);
            } else {
                routeException.addConnectException(e);
            }
            if (!connectionRetryEnabled || !connectionSpecSelector.connectionFailed(e)) {
                throw routeException;
            }
        }
    }
    if (http2Connection != null) {
        synchronized (connectionPool) {
            allocationLimit = http2Connection.maxConcurrentStreams();
        }
    }
}
```
判断网络请求是否为 Https:
```
// 是否是 HTTPS 请求
public boolean requiresTunnel() {
    return address.sslSocketFactory != null && proxy.type() == Proxy.Type.HTTP;
}
```

Address#sslSocketFactory
```
//在 HTTPS 请求时返回 SSLSocketFactory 实例对象，否则的话返回 null
public @Nullable
SSLSocketFactory sslSocketFactory() {
    return sslSocketFactory;
}
```

## 2.6 RealConnection#connectTunnel

建立HTTPS 代理隧道。

```
private void connectTunnel(int connectTimeout, int readTimeout, int writeTimeout)
        throws IOException {
    // 见 2.7
    Request tunnelRequest = createTunnelRequest();
    HttpUrl url = tunnelRequest.url();
    int attemptedConnections = 0;
    int maxAttempts = 21;
    while (true) {
        if (++attemptedConnections > maxAttempts) {
            throw new ProtocolException("Too many tunnel connections attempted: " + maxAttempts);
        }
        // 见 2.9
        connectSocket(connectTimeout, readTimeout);
        // 在建立的 Socket 的基础上，建立 HTTPS 请求，见 2.8
        tunnelRequest = createTunnel(readTimeout, writeTimeout, tunnelRequest, url);
        if (tunnelRequest == null) break; // Tunnel successfully created.
        closeQuietly(rawSocket);
        rawSocket = null;
        sink = null;
        source = null;
    }
}
```
通过隧道建立 HTTPS 连接，代理服务器可以校验请求的授权，并且可以关闭连接。

至于什么是隧道，《网络是怎么连接的》 有这样的解释：

所谓隧道，就类似于套接字之间建立的TCP连接。在TCP连接中，我们从一侧的出口（套接字）放入数据，数据就会原封不动地从另一个出口出来，隧道也是如此。也就是说，我们将包含头部在内的整个包从隧道的一头扔进去，这个包就会原封不动地从隧道的另一头出来，就好像在网络中挖了一条地道，网络包从这个地道里穿过去一样。

隧道有几种实现方式， TCP 连接是一种方式，基于封装也是一种方式，具体可以查看《网络是怎么连接的》- 4.3 接入网中使用的 PPP 和隧道。


可以看到建立隧道的过程也是基于 Socket 完成的，在此过程建立起连接，获得 source、sink 属性，从而在 createTunnel 设置相应的属性，从而完成隧道的建立，具体可以查看 2.8。

## 2.7 RealConnection#createTunnelRequest

```
private Request createTunnelRequest() {
    return new Request.Builder()
            .url(route.address().url())
            .header("Host", Util.hostHeader(route.address().url(), true))
            .header("Proxy-Connection", "Keep-Alive") // For HTTP/1.0 proxies like Squid.
            .header("User-Agent", Version.userAgent())
            .build();
}
```
返回通过 HTTP 代理创建 TLS 隧道的请求。隧道请求中的所有内容。未加密发送到代理服务器，因此隧道仅包含最小标头集。这样可以避免将潜在的敏感数据（如 HTTP Cookie）发送到未加密的代理。

## 2.8 createTunnel

要通过HTTP代理建立HTTPS连接，请发送一个未加密的连接请求来创建代理连接。如果代理需要授权，可能需要重新尝试。

```
private Request createTunnel(int readTimeout, int writeTimeout, Request tunnelRequest,
                             HttpUrl url) throws IOException {
    // Make an SSL Tunnel on the first message pair of each SSL + proxy connection.
    // 在每个 SSL 和代理连接的第一个消息对上建立 SSL 隧道
    String requestLine = "CONNECT " + Util.hostHeader(url, true) + " HTTP/1.1";
    while (true) {
        Http1Codec tunnelConnection = new Http1Codec(null, null, source, sink);
        // 设置读写的时间限制
        source.timeout().timeout(readTimeout, MILLISECONDS);
        sink.timeout().timeout(writeTimeout, MILLISECONDS);
        tunnelConnection.writeRequest(tunnelRequest.headers(), requestLine);
        tunnelConnection.finishRequest();
        Response response = tunnelConnection.readResponseHeaders(false)
                .request(tunnelRequest)
                .build();
        // The response body from a CONNECT should be empty, but if it is not then we should consume
        // it before proceeding.
        long contentLength = HttpHeaders.contentLength(response);
        if (contentLength == -1L) {
            contentLength = 0L;
        }
        Source body = tunnelConnection.newFixedLengthSource(contentLength);
        Util.skipAll(body, Integer.MAX_VALUE, TimeUnit.MILLISECONDS);
        body.close();
        switch (response.code()) {
            case HTTP_OK:
                if (!source.buffer().exhausted() || !sink.buffer().exhausted()) {
                    throw new IOException("TLS tunnel buffered too many bytes!");
                }
                return null;
            case HTTP_PROXY_AUTH:
                tunnelRequest = route.address().proxyAuthenticator().authenticate(route, response);
                if (tunnelRequest == null)
                    throw new IOException("Failed to authenticate with proxy");
                if ("close".equalsIgnoreCase(response.header("Connection"))) {
                    return tunnelRequest;
                }
                break;
            default:
                throw new IOException(
                        "Unexpected response code for CONNECT: " + response.code());
        }
    }
}
```


## 2.9 RealConnection#connectSocket

```
private void connectSocket(int connectTimeout, int readTimeout) throws IOException {
    Proxy proxy = route.proxy();
    Address address = route.address();
    rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
            ? address.socketFactory().createSocket()
            : new Socket(proxy);
    rawSocket.setSoTimeout(readTimeout);
    try {
        // 建立 Socket 连接，见 2.10
        Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
    } catch (ConnectException e) {
        ConnectException ce = new ConnectException("Failed to connect to " + route.socketAddress());
        ce.initCause(e);
        throw ce;
    }
    try {
        source = Okio.buffer(Okio.source(rawSocket));
        sink = Okio.buffer(Okio.sink(rawSocket));
    } catch (NullPointerException npe) {
        if (NPE_THROW_WITH_NULL.equals(npe.getMessage())) {
            throw new IOException(npe);
        }
    }
}
```
## 2.10 Platform#connectSocket

```
public void connectSocket(Socket socket, InetSocketAddress address,
                          int connectTimeout) throws IOException {
    socket.connect(address, connectTimeout);
}
```
建立 Socket 连接。

Platform 有不同实现类： AndroidPlatform、 Jdk9Platform 等，在 AndroidPlatform 中对 Socket 的连接做个各种保护措施。


## 2.11 RealConnection#establishProtocol

```
private void establishProtocol(ConnectionSpecSelector connectionSpecSelector) throws IOException {
    // 如果为 Http 则返回
    if (route.address().sslSocketFactory() == null) {
        protocol = Protocol.HTTP_1_1;
        socket = rawSocket;
        return;
    }
    // Https 连接 tls、ssl,见 2.12
    connectTls(connectionSpecSelector);
    if (protocol == Protocol.HTTP_2) {
        socket.setSoTimeout(0); 
        http2Connection = new Http2Connection.Builder(true)
                .socket(socket, route.address().url().host(), source, sink)
                .listener(this)
                .build();
        http2Connection.start();
    }
}
```

## 2.12 RealConnection#connectTls

```
private void connectTls(ConnectionSpecSelector connectionSpecSelector) throws IOException {
    Address address = route.address();
    // 获得设置的 SSLSocketFactory，若没有设置 SSLSocketFactory，
    //则使用系统默认的 SSLSocketFactory，见 2.13
    SSLSocketFactory sslSocketFactory = address.sslSocketFactory();
    boolean success = false;
    SSLSocket sslSocket = null;
    try {
        // Create the wrapper over the connected socket.
        // 包装已经建立的 socket 连接，形成 SSL Socket
        sslSocket = (SSLSocket) sslSocketFactory.createSocket(
                rawSocket, address.url().host(), address.url().port(), true /* autoClose */);
        // Configure the socket's ciphers(密码), TLS versions, and extensions.
        ConnectionSpec connectionSpec = connectionSpecSelector.configureSecureSocket(sslSocket);
        if (connectionSpec.supportsTlsExtensions()) {
            Platform.get().configureTlsExtensions(
                    sslSocket, address.url().host(), address.protocols());
        }
        // Force handshake. This can throw!
        // 开启 SSL 握手
        sslSocket.startHandshake();
        Handshake unverifiedHandshake = Handshake.get(sslSocket.getSession());
        // 验证该套接字对目标主机的证书是可以接受的;
        if (!address.hostnameVerifier().verify(address.url().host(), sslSocket.getSession())) {
        }
        // 检查证书提供的标签
        address.certificatePinner().check(address.url().host(),
                unverifiedHandshake.peerCertificates());
        // Success! Save the handshake and the ALPN protocol.
        String maybeProtocol = connectionSpec.supportsTlsExtensions()
                ? Platform.get().getSelectedProtocol(sslSocket)
                : null;
        socket = sslSocket;
        source = Okio.buffer(Okio.source(socket));
        sink = Okio.buffer(Okio.sink(socket));
        handshake = unverifiedHandshake;
        protocol = maybeProtocol != null
                ? Protocol.get(maybeProtocol)
                : Protocol.HTTP_1_1;
        success = true;
    } catch (AssertionError e) {
    } finally {
        if (sslSocket != null) {
            Platform.get().afterHandshake(sslSocket);
        }
        if (!success) {
            closeQuietly(sslSocket);
        }
    }
}
```


## 2.13 OkHttpClient 构造函数

```
OkHttpClient(Builder builder) {
    ...
    this.connectionSpecs = builder.connectionSpecs;
    boolean isTLS = false;
    for (ConnectionSpec spec : connectionSpecs) {
        isTLS = isTLS || spec.isTls();
    }
    if (builder.sslSocketFactory != null || !isTLS) {
        this.sslSocketFactory = builder.sslSocketFactory;
        this.certificateChainCleaner = builder.certificateChainCleaner;
    } else {
        X509TrustManager trustManager = systemDefaultTrustManager();
        this.sslSocketFactory = systemDefaultSslSocketFactory(trustManager);
        this.certificateChainCleaner = CertificateChainCleaner.get(trustManager);
    }
    ...
}
// 系统默认的 X509TrustManager
private X509TrustManager systemDefaultTrustManager() {
    try {
        TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(
                TrustManagerFactory.getDefaultAlgorithm());
        trustManagerFactory.init((KeyStore) null);
        TrustManager[] trustManagers = trustManagerFactory.getTrustManagers();
        if (trustManagers.length != 1 || !(trustManagers[0] instanceof X509TrustManager)) {
            throw new IllegalStateException("Unexpected default trust managers:"
                    + Arrays.toString(trustManagers));
        }
        return (X509TrustManager) trustManagers[0];
    } catch (GeneralSecurityException e) {
        throw new AssertionError(); // The system has no TLS. Just give up.
    }
}
// 系统默认的 SSLSocketFactory
private SSLSocketFactory systemDefaultSslSocketFactory(X509TrustManager trustManager) {
    try {
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, new TrustManager[]{trustManager}, null);
        return sslContext.getSocketFactory();
    } catch (GeneralSecurityException e) {
        throw new AssertionError(); // The system has no TLS. Just give up.
    }
}
```

当然，Okhttp 也可以配置自己的证书，具体可以查看 [Android Https相关完全解析 当OkHttp遇到Https](https://blog.csdn.net/lmj623565791/article/details/48129405)


## 总结


此次 Okhttp 建立网络请求的一步步操作算是明了了，但是这只是牵涉到具体流程，具体细节还需要根据需要仔细研究。

可以看到的是最终网络连接的建立还是需要 Socket ，这是网络连接的基础。

后补：

TCP、UDP 协议也是对 Socket 的封装，具体可以查看 《网络是怎么连接的》 这本书。


