---
title: 网络协议：HTTP 协议
tags:
---


## HTTP 请求准备

DNS 解析域名。

HTTP 是基于 TCP 协议的，所以首先要进行 TCP 连接。

目前大部分使用的 HTTP 协议都是 HTTP1.1 的，在 1.1 协议中默认开启 keep-alive,，这样建立的 TCP 连接可以在多次请求中复用。

## HTTP 请求的构建


请求格式：

![](https://static001.geekbang.org/resource/image/10/74/10ff27d1032bf32393195f23ef2f9874.jpg)


HTTP 请求报文：

### 请求行


### 首部字段

首部是 key value，通过冒号分隔。



* Charset：表示客户端可以接受的字符集。防止传过来的是另外的字符集，从而导致出现乱码。
* Content-Type：正文的格式




至此，拼凑起了 HTTP 请求的报文格式，接下来，浏览器会把它交给下一层传输层。怎么交给传输层呢？其实还是使用 **Socket**，只不过这部分工作浏览器已经帮我们完成了。


>以前一直不是很确定Keep-Alive的作用, 今天结合tcp的知识, 终于是彻底搞清楚了. 其实就是浏览器访问服务端之后, 一个http请求的底层是tcp连接, tcp连接要经过三次握手之后,开始传输数据, 而且因为http设置了keep-alive,所以单次http请求完成之后这条tcp连接并不会断开, 而是可以让下一次http请求直接使用.当然keep-alive肯定也有timeout, 超时关闭。

## HTTP 请求的发送


HTTP 协议是基于 TCP 协议的，所以它使用面向连接的方式发送请求，通过 stream 二进制流的方式传给对方。当然，到了 TCP 层，它会把二进制流变成一个的报文段发送给服务器。


在发送给每个报文段的时候，都需要对方有一个回应 ACK，来保证报文可靠地到达了对方。如果没有回应，那么 TCP 这一层会进行重新传输，直到可以到达。同一个包有可能被传了好多次，但是 HTTP 这一层不需要知道这一点，因为是 TCP 这一层在埋头苦干。




## HTTP 返回的构建
HTTP 1.1:

![](https://static001.geekbang.org/resource/image/1c/c1/1c2cfd4326d0dfca652ac8501321fac1.jpg)