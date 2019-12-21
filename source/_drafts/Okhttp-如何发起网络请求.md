---
title: Okhttp 如何发起网络请求
tags:
---


# 


OkHttp 如何发起网络请求，其核心业务存在于 CallServerInterceptor 拦截中，通过源码查看如果进行网络请求。


# 在 

在 ConnectInterceptor 拦截器中，客户端和服务器建立了 Socket 连接，那么 CallServerInterceptor 就是通过对该 Socket 连接进行读写，从而完成网络交互，具体流程如下。

## 2.1 ConnectInterceptor


```
public final class CallServerInterceptor implements Interceptor {
    private final boolean forWebSocket;

    public CallServerInterceptor(boolean forWebSocket) {
        this.forWebSocket = forWebSocket;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        RealInterceptorChain realChain = (RealInterceptorChain) chain;
        // 获取在 ConnectInterceptor 中建立的 HttpCodec 对象，此处一以 HTTP/1.1 分析，即获取的对象为 Http1Code 对象，网络交互的主角
        HttpCodec httpCodec = realChain.httpStream();
        StreamAllocation streamAllocation = realChain.streamAllocation();
        RealConnection connection = (RealConnection) realChain.connection();
        Request request = realChain.request();

        long sentRequestMillis = System.currentTimeMillis();
        // 写入 HTTP 请求头, 见 2.2
        httpCodec.writeRequestHeaders(request);

        Response.Builder responseBuilder = null;
        if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
            // 如果在请求头中包含 "Expect: 100-continue",那么在传输请求体之前需要等待 "HTTP/1.1 100 Continue" 的响应，
            // 如果没有获得该响应，就不会发送请求体，活的到 4xx 的响应。
            if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
                httpCodec.flushRequest();
                responseBuilder = httpCodec.readResponseHeaders(true);
            }

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
        }
        // 刷新 Soket 流，写入请求数据
        httpCodec.finishRequest();

        if (responseBuilder == null) {
            // 获得响应头，见 2.3 
            responseBuilder = httpCodec.readResponseHeaders(false);
        }

        // 构建响应的基本内容，但是此时 response 空的 body 为 null，见 2.5
        Response response = responseBuilder
                .request(request)
                .handshake(streamAllocation.connection().handshake())
                .sentRequestAtMillis(sentRequestMillis)
                .receivedResponseAtMillis(System.currentTimeMillis())
                .build();

        int code = response.code();
        if (forWebSocket && code == 101) {
            // 为 response 构建空报文主题
            response = response.newBuilder()
                    .body(Util.EMPTY_RESPONSE)
                    .build();
        } else {
            // 从 Socket 中读取 body 内容，用户构建 response
            response = response.newBuilder()
                    //openResponseBody 返回读取响应体的流
                    .body(httpCodec.openResponseBody(response))
                    .build();
        }
        // 至此完成 Response 的构建，在拦截器链中向下传递
        return response;
    }
}
```


## 2.2 Http1Code#writeRequest

```
public void writeRequest(Headers headers, String requestLine) throws IOException {
    if (state != STATE_IDLE) throw new IllegalStateException("state: " + state);
    sink.writeUtf8(requestLine).writeUtf8("\r\n");
    for (int i = 0, size = headers.size(); i < size; i++) {
        sink.writeUtf8(headers.name(i))
                .writeUtf8(": ")
                .writeUtf8(headers.value(i))
                .writeUtf8("\r\n");
    }
    sink.writeUtf8("\r\n");
    state = STATE_OPEN_REQUEST_BODY;
}
```

## 2.3 readResponseHeaders

```
public Response.Builder readResponseHeaders(boolean expectContinue) throws IOException {
    if (state != STATE_OPEN_REQUEST_BODY && state != STATE_READ_RESPONSE_HEADERS) {
        throw new IllegalStateException("state: " + state);
    }
    try {
        // 由于网络请求遵循 HTTP 协议，即响应有报文由响应首部、空行、报文主体构成，那么在此处获取响应的首部的状态行，比如 HTTP/1.1 200 OK
        StatusLine statusLine = StatusLine.parse(source.readUtf8LineStrict());
        Response.Builder responseBuilder = new Response.Builder()
                .protocol(statusLine.protocol)
                .code(statusLine.code)
                .message(statusLine.message)
                // readHeaders 获得响应首部内容，见 2.4 
                .headers(readHeaders());
        if (expectContinue && statusLine.code == HTTP_CONTINUE) {
            return null;
        }
        state = STATE_OPEN_RESPONSE_BODY;
        return responseBuilder;
    } catch (EOFException e) {
        // Provide more context if the server ends the stream before sending a response.
        IOException exception = new IOException("unexpected end of stream on " + streamAllocation);
        exception.initCause(e);
        throw exception;
    }
}
```

## 2.4 readHeaders

```
public Headers readHeaders() throws IOException {
    Headers.Builder headers = new Headers.Builder();
    // parse the result headers until the first blank line
    for (String line; (line = source.readUtf8LineStrict()).length() != 0; ) {
        Internal.instance.addLenient(headers, line);
    }
    return headers.build();
}
```

该方法返回具体的响应首部,比如：

```
Server: Apache-Coyote/1.1
Cache-Control: private
Expires: Thu, 01 Jan 1970 08:00:00 CST
Set-Cookie: JSESSIONID=E6DDBD7C8FF99A0B3109CE12126E300E; Path=/; Secure; HttpOnly
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Thu, 12 Dec 2019 10:41:58 GMT
```

## 2.5 responseBuilder.xx.xx.build

经过此步骤，此时 response 的对象的具体包含的属性值如下(response.toString()）:
```
Response{protocol=http/1.1, code=200, message=OK, url=https://wanandroid.com/wxarticle/chapters/json}
```

## openResponseBody
```
public ResponseBody openResponseBody(Response response) throws IOException {
    Source source = getTransferStream(response);
    return new RealResponseBody(response.headers(), Okio.buffer(source));
}
```