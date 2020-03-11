---
title: HttpURLConnection 源码相关
tags:
---


https://blog.csdn.net/ming54ming/article/details/85089066

http://frodoking.github.io/2015/03/12/android-okhttp/



## 实例代码

实例代码：

```
URL url = new URL("https://certs.cac.washington.edu/CAtest/");
HttpsURLConnection urlConnection = (HttpsURLConnection)url.openConnection();
//connect()方法不必显式调用，当调用conn.getInputStream()方法时内部也会自动调用connect方法
urlConnection.connect();
//调用getInputStream方法后，服务端才会收到完整的请求，并阻塞式地接收服务端返回的数据
InputStream in = urlConnection.getInputStream();
```

其中 url.openConnection 其实是调用了 com.android.okhttp.HttpHandler.openConnection() 的方法，但是 HttpHandler 的源代码为 `/external/okhttp/android/main/java/com/squareup/okhttp/HttpHandler.java`，AOSP 项目对其做了映射，虽然包不相同，但是执行同一个文件。

## 具体流程




### 2.1 HttpHandler.openConnection()


```
    @Override protected URLConnection openConnection(URL url, Proxy proxy) throws IOException {
        if (url == null || proxy == null) {
            throw new IllegalArgumentException("url == null || proxy == null");
        }
        // OkUrlFactory#open，见 2.3
        return newOkUrlFactory(proxy).open(url);
    }
```


```
    protected OkUrlFactory newOkUrlFactory(Proxy proxy) {
        // 见 2.2
        OkUrlFactory okUrlFactory = createHttpOkUrlFactory(proxy);
        // 对于通过 java.net.URL 创建的 HttpURL 连接，Android 使用一个连接池，该连接池知道默认网络更改，以便在默认网络更改时不再重复使用池连接。
        okUrlFactory.client().setConnectionPool(configAwareConnectionPool.get());
        return okUrlFactory;
    }
```

### 2.2 HttpHandler#createHttpOkUrlFactory

创建 OkUrlFactory 对象
```
    public static OkUrlFactory createHttpOkUrlFactory(Proxy proxy) {
        // 在 Android 中创建 OkhttpClient 对象，对应 HttpUrlconnection 对象
        OkHttpClient client = new OkHttpClient();

        ...
        ...

        // 一系列对 OkhttpClient 对象的设置

        // OkHttp requires that we explicitly set the response cache.
        OkUrlFactory okUrlFactory = new OkUrlFactory(client);

        // Use the installed NetworkSecurityPolicy to determine which requests are permitted over
        // http.
        OkUrlFactories.setUrlFilter(okUrlFactory, CLEARTEXT_FILTER);

        ResponseCache responseCache = ResponseCache.getDefault();
        if (responseCache != null) {
            AndroidInternal.setResponseCache(okUrlFactory, responseCache);
        }
        return okUrlFactory;
    }

```

### 2.3 OkUrlFactory#open


```
  HttpURLConnection open(URL url, Proxy proxy) {
    String protocol = url.getProtocol();
    OkHttpClient copy = client.copyWithDefaults();
    copy.setProxy(proxy);

    if (protocol.equals("http")) return new HttpURLConnectionImpl(url, copy, urlFilter);
    if (protocol.equals("https")) return new HttpsURLConnectionImpl(url, copy, urlFilter);
    throw new IllegalArgumentException("Unexpected protocol: " + protocol);
  }
```

即最终根据协议名称获得 HttpURLConnectionImpl 或者 HttpsURLConnectionImpl 实例对象，进行接下来的操作。

### 2.4 HttpURLConnectionImpl#connect

```
  @Override public final void connect() throws IOException {
    // 初始化 HttpEngine，见 2.5 
    initHttpEngine();
    boolean success;
    do {
      // 执行接下来的操作，见 2.7  
      success = execute(false);
    } while (!success);
  }
```
### 2.5 HttpURLConnectionImpl#initHttpEngine

```
  private void initHttpEngine() throws IOException {
    if (httpEngineFailure != null) {
      throw httpEngineFailure;
    } else if (httpEngine != null) {
      return;
    }
    connected = true;
    try {
      if (doOutput) {
        if (method.equals("GET")) {
          // they are requesting a stream to write to. This implies a POST method
          method = "POST";
        } else if (!HttpMethod.permitsRequestBody(method)) {
          throw new ProtocolException(method + " does not support writing");
        }
      }
      // If the user set content length to zero, we know there will not be a request body.
      // 获得 HTTPEngine 实例对象，见 2.6
      httpEngine = newHttpEngine(method, null, null, null);
    } catch (IOException e) {
      httpEngineFailure = e;
      throw e;
    }
  }
```
 
### 2.6 HttpURLConnectionImpl#newHttpEngine

```
  private HttpEngine newHttpEngine(String method, StreamAllocation streamAllocation,
      RetryableSink requestBody, Response priorResponse)
      throws MalformedURLException, UnknownHostException {
    // OkHttp's Call API requires a placeholder body; the real body will be streamed separately.
    // 构建网络请求数据的请求体
    RequestBody placeholderBody = HttpMethod.requiresRequestBody(method)
        ? EMPTY_REQUEST_BODY
        : null;
    // 接下来的操作，和使用 Okhttp 发起网络请求的步骤一致
    URL url = getURL();
    HttpUrl httpUrl = Internal.instance.getHttpUrlChecked(url.toString());
    Request.Builder builder = new Request.Builder()
        .url(httpUrl)
        .method(method, placeholderBody);
    Headers headers = requestHeaders.build();
    for (int i = 0, size = headers.size(); i < size; i++) {
      builder.addHeader(headers.name(i), headers.value(i));
    }
    ...
    ...
    // 为 builder 对象设置各种相应的参数
    Request request = builder.build();
    // If we're currently not using caches, make sure the engine's client doesn't have one.
    OkHttpClient engineClient = client;
    if (Internal.instance.internalCache(engineClient) != null && !getUseCaches()) {
      engineClient = client.clone().setCache(null);
    }
    // 最终返回 HttpEngine 实例对象
    return new HttpEngine(engineClient, request, bufferRequestBody, true, false, streamAllocation,
        requestBody, priorResponse);
  }
```

### 2.7 HttpURLConnectionImpl#execute



```
 private boolean execute(boolean readResponse) throws IOException {
    boolean releaseConnection = true;
    if (urlFilter != null) {
      urlFilter.checkURLPermitted(httpEngine.getRequest().url());
    }
    try {
      // 发起网络请求，见 2.8 
      httpEngine.sendRequest();
      Connection connection = httpEngine.getConnection();
      if (connection != null) {
        route = connection.getRoute();
        handshake = connection.getHandshake();
      } else {
        route = null;
        handshake = null;
      }
      if (readResponse) {
        // 如果需要读取响应，则读取网络响应
        httpEngine.readResponse();
      }
      releaseConnection = false;

      return true;
    } catch (RequestException e) {
      // An attempt to interpret a request failed.
      IOException toThrow = e.getCause();
      httpEngineFailure = toThrow;
      throw toThrow;
    } catch (RouteException e) {
      // The attempt to connect via a route failed. The request will not have been sent.
      HttpEngine retryEngine = httpEngine.recover(e);
      if (retryEngine != null) {
        releaseConnection = false;
        httpEngine = retryEngine;
        return false;
      }

      // Give up; recovery is not possible.
      IOException toThrow = e.getLastConnectException();
      httpEngineFailure = toThrow;
      throw toThrow;
    } catch (IOException e) {
      // An attempt to communicate with a server failed. The request may have been sent.
      HttpEngine retryEngine = httpEngine.recover(e);
      if (retryEngine != null) {
        releaseConnection = false;
        httpEngine = retryEngine;
        return false;
      }

      // Give up; recovery is not possible.
      httpEngineFailure = e;
      throw e;
    } finally {
      // We're throwing an unchecked exception. Release any resources.
      if (releaseConnection) {
        StreamAllocation streamAllocation = httpEngine.close();
        streamAllocation.release();
      }
    }
  }
```

发起一次请求，并读取响应数据。如果请求成功返回数据，则返回 true，如果需要重新发起请求，则返回 false。


### 2.8 HttpEngine#sendRequest

```
 public void sendRequest() throws RequestException, RouteException, IOException {

    Request request = networkRequest(userRequest);

    InternalCache responseCache = Internal.instance.internalCache(client);
    Response cacheCandidate = responseCache != null
        ? responseCache.get(request)
        : null;

    long now = System.currentTimeMillis();
    // 根据是否存在缓存，获得缓存策略
    cacheStrategy = new CacheStrategy.Factory(now, request, cacheCandidate).get();
    networkRequest = cacheStrategy.networkRequest;
    cacheResponse = cacheStrategy.cacheResponse;

    if (responseCache != null) {
      responseCache.trackResponse(cacheStrategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    if (networkRequest != null) {
      // 建立起连接，见 2.9 
      httpStream = connect();
      httpStream.setHttpEngine(this);

      // If the caller's control flow writes the request body, we need to create that stream
      // immediately. And that means we need to immediately write the request headers, so we can
      // start streaming the request body. (We may already have a request body if we're retrying a
      // failed POST.)
      if (callerWritesRequestBody && permitsRequestBody(networkRequest) && requestBodyOut == null) {
        long contentLength = OkHeaders.contentLength(request);
        if (bufferRequestBody) {
          if (contentLength > Integer.MAX_VALUE) {
            throw new IllegalStateException("Use setFixedLengthStreamingMode() or "
                + "setChunkedStreamingMode() for requests larger than 2 GiB.");
          }

          if (contentLength != -1) {
            // Buffer a request body of a known length.
            httpStream.writeRequestHeaders(networkRequest);
            requestBodyOut = new RetryableSink((int) contentLength);
          } else {
            // Buffer a request body of an unknown length. Don't write request
            // headers until the entire body is ready; otherwise we can't set the
            // Content-Length header correctly.
            requestBodyOut = new RetryableSink();
          }
        } else {
          httpStream.writeRequestHeaders(networkRequest);
          requestBodyOut = httpStream.createRequestBody(networkRequest, contentLength);
        }
      }

    } else {
      if (cacheResponse != null) {
        // We have a valid cached response. Promote it to the user response immediately.
        this.userResponse = cacheResponse.newBuilder()
            .request(userRequest)
            .priorResponse(stripBody(priorResponse))
            .cacheResponse(stripBody(cacheResponse))
            .build();
      } else {
        // We're forbidden from using the network, and the cache is insufficient.
        this.userResponse = new Response.Builder()
            .request(userRequest)
            .priorResponse(stripBody(priorResponse))
            .protocol(Protocol.HTTP_1_1)
            .code(504)
            .message("Unsatisfiable Request (only-if-cached)")
            .body(EMPTY_BODY)
            .build();
      }

      userResponse = unzip(userResponse);
    }
  }
```

### 2.10 HttpEngin#connect

```
  private HttpStream connect() throws RouteException, RequestException, IOException {
    boolean doExtensiveHealthChecks = !networkRequest.method().equals("GET");
    // 见 2.11 
    return streamAllocation.newStream(client.getConnectTimeout(),
        client.getReadTimeout(), client.getWriteTimeout(),
        client.getRetryOnConnectionFailure(), doExtensiveHealthChecks);
  }
```

### 2.11 StreamAllocation#newStream

```
public HttpStream newStream(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws RouteException, IOException {
    try {
     // 获得连接，并建立连接
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);

      HttpStream resultStream;
      if (resultConnection.framedConnection != null) {
        resultStream = new Http2xStream(this, resultConnection.framedConnection);
      } else {
        resultConnection.getSocket().setSoTimeout(readTimeout);
        resultConnection.source.timeout().timeout(readTimeout, MILLISECONDS);
        resultConnection.sink.timeout().timeout(writeTimeout, MILLISECONDS);
        resultStream = new Http1xStream(this, resultConnection.source, resultConnection.sink);
      }

      synchronized (connectionPool) {
        resultConnection.streamCount++;
        stream = resultStream;
        return resultStream;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
```

至此，接下来的操作和使用 OkHttp 建立连接的步骤相似，具体参见 [OkHttp 如何建立连接]()


### 2.12 HttpEngin#readNetworkResponse


```
private Response readNetworkResponse() throws IOException {
    httpStream.finishRequest();

    Response networkResponse = httpStream.readResponseHeaders()
        .request(networkRequest)
        .handshake(streamAllocation.connection().getHandshake())
        .header(OkHeaders.SENT_MILLIS, Long.toString(sentRequestMillis))
        .header(OkHeaders.RECEIVED_MILLIS, Long.toString(System.currentTimeMillis()))
        .build();

    if (!forWebSocket) {
      networkResponse = networkResponse.newBuilder()
          .body(httpStream.openResponseBody(networkResponse))
          .build();
    }

    if ("close".equalsIgnoreCase(networkResponse.request().header("Connection"))
        || "close".equalsIgnoreCase(networkResponse.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    return networkResponse;
  }
```

写入响应报文，并从从 HttpStream 中获得响应数据。