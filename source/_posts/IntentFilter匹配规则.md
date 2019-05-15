---
title: IntentFilter匹配规则
date: 2019-04-02 10:28:45
tags:
---


IntentFilter 中的过滤信息包括 action、category、data，为了匹配过滤列表需要同时匹配 action、category、data ，否则匹配失败。

### action 匹配规则

action 是一个字符串，系统定义了一些 action，用户自己也可以定义 action。

action 的匹配规则是：**Intent 中的 action 必须能够和过滤规则中的 action 匹配**，匹配是指两个 action 的字符串完全相同。

一个过滤规则的中可以有多个 action，**只要 Intent 中的任何一个 action 和其中的一个 action 匹配，则可匹配成功**。

action的匹配要求Intent中的action存在且必须和过滤规则中的其中一个action相同，**action 区分大小写**。


```
val intent = Intent("xxaction")
intent.setAction("")
```
<!-- more -->
### category 匹配规则

category 是一个字符串，系统预定义了一些 category，用户也可以定义自己的 category。

category 的匹配规则：Intent 中如果含有 category，那么 Intent **所有的** category 必须和过滤规则中的一个相同。

为 Intent 的添加 category 规则


```
intent.addCategory("xxx");
```

系统在调用 `startActivity` 或 `startActivityForResult` 时会为 Intent 添加 `android.intent.category.DEFAULT` 这个 category,所以在隐式调用 Activity 时需要在清单文件中显示的添加 `android.intent.category.DEFAULT` 这条过滤规则。

Activity 的显示调用：

```
val intent = Intent(this,SecondActivity::class.java)
startActivity(intent)
```
Activity 的隐式调用：

```
val intent = Intent()
intent.setAction("xxx")
intent.addCategory("xxx")
startActivity(intent)
```


```
<activity>
    <intent-filter>   
        <category android:name = "android.intent.category.DEFAULT" //>   
    </intent-filter>       
</activity>
```
### data 的匹配规则

#### data 的结构

data 由两部分组成， mimeType 和 URI。

* mimeType
  
    mimeType 指的是 **媒体类型**，例如：如image/jpeg、audio/mpeg4generic和video/* 等，可以表示图片、文本、视频等不同的媒体类型。

* URI(统一资源标识符)

    URI 的结构

    ```
    <scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>](中括号中表示三者可选)
    ```
    如下面的例子：

    ```
    http://localhost:8080/user/info
    ```

    * Scheme

        URI 模式，比如 http、file、content 等，如果 URI 没有指定有效 scheme，那么整个 URI 都是无效的。
    
    * Host

        URI 的主机名，如果没有指定 host ，那么其他的参数都是无效的，整个 URI 都是无效的。
    
    * Port

        URI 的端口号。当 URI 的 Scheme 和 Host 指定后 Port 才会有效。
    
    * Path、PathPrefix、PathPattern

        三者均表示路径信息。Path 表示完整的路径信息，PathPattern 也表示完整的路径信息，但是其中可以包含通配符 "*" ，表示 0 个或任意多个字符；PathPrefix 表示路径的前缀信息。

#### data 的匹配规则

data 的匹配规则：**Intent 中的 data 数据必须和过滤规则中的某一个 data 完全匹配。**
如下例：

```
val intent = Intent()
intnet.setDataAndType((Uri.parse("file://abc"),"image/png")
```

那么对应的 IntentFilter 中的匹配过滤信息如下：


```
<intent-filter>
    <data android:mimeType="image/*" android:scheme="file" android:host="abc">
</intent-filter>

OR

<intent-filter>
    <data android:mimeType="image/*"/> 
    <data android:scheme="file"/>
    <data android:host="abc"/>
</intent-filter>

// 两组是完全相同的，只是第二组把 URI 的各个部分分开表示。
```
