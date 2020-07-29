---
title: Https 如何反抓包
tags: [Android,网络安全]
date: 2020-07-29 16:54:01
---

## 0x0001 方案一

其实要想明白如何实现针对 HTTPS 的反抓包操作，那么需要理解 HTTPS TLS 建立连接的过程，该过程网上到处都有，此处不再陈述，大致过程见如下简图：

<!-- more -->

![](HTTPS-反抓包/2020_07_29_01.png)

关于步骤二证书的验证过程的是需要与服务器交互的，在这里为了流程图的清晰，省略该流程，特此标注，下图的验证过程也是如此。

Charles、Fidder 等抓包工具的大致原理，这些工具为中介者，大致过程见如下简图：

![](HTTPS-反抓包/2020_07_29_02.png)

基于 Charles 的实现原理，那么可以子签名 CA 证书，那么这样的话上图步骤 4 将无法通过，所以后续的抓包动作无法进行，具体的自签名证书可以查看 ：[通过 HTTPS 和 SSL 确保安全](https://developer.android.google.cn/training/articles/security-ssl?hl=zh_cn#java)


## 0x0002 方案二: 

**检测是否进行了 WiFi 代理**

```
private static boolean isWifiProxy() {
    final boolean IS_ICS_OR_LATER = Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH;
    String proxyAddress;
    int proxyPort;
    if (IS_ICS_OR_LATER) {
        proxyAddress = System.getProperty("http.proxyHost");
        String portStr = System.getProperty("http.proxyPort");
        proxyPort = Integer.parseInt((portStr != null ? portStr : "-1"));
    } else {
        proxyAddress = android.net.Proxy.getHost(mContext);
        proxyPort = android.net.Proxy.getPort(mContext);
    }
    return (!TextUtils.isEmpty(proxyAddress)) && (proxyPort != -1);
}
```

如果进行了 Wifi 代理，可以对用户进行提醒或者直接不执行网络情况，但是这样会存在问题：用户使用 VPN 代理，那么就不能进行网络请求(网络搜文，个人没有验证)。

**OkHttp 设置无代理方式**

```
OkHttpClient.Builder okHttpClient = new OkHttpClient.Builder();
okHttpClient.proxy(Proxy.NO_PROXY);
```

---

**知识链接：**

[通过 HTTPS 和 SSL 确保安全](https://developer.android.google.cn/training/articles/security-ssl?hl=zh_cn#java)

[关于Android 抓包 与 反抓包](https://www.jianshu.com/p/0b2a69447404)：关于 OKhttp 的自定义 TLS 

[第二章 连接管理](https://www.cnblogs.com/loveyakamoz/archive/2011/07/21/2112832.html)

[HTTPS工作原理以及Android中如何防止抓包](https://juejin.im/post/5d8c593ee51d45783544b9ac):在代码中配置做定义 CA 证书，可以有效的防止抓包：是这样的吗？

[Android 网络安全：如何避免 Okhttp 的 HTTPS 请求被抓包](https://www.jianshu.com/p/11577eb0ce2d) 同样指向了上面的结论：配置指定的 CA 证书


[部分APP无法代理抓包的原因及解决方法](https://cloud.tencent.com/developer/article/1490033)