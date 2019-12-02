---
title: 记 Okhttp3 下一次网络请求的实现流程
tags:
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