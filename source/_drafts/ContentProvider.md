---
title: ContentProvider
tags:
---


### ContentProvider 的 URI


URI，全称为 Universal Resource Identifier，即通用资源标志符，通过它用来唯一标志某个资源在网络中的位置，它的结构和我们常见的HTTP形式URL是一样的，其实我们可以把常见的 HTTP 形式的 UR L看成是 URI 结构的一个实例，URI 是在更高一个层次上的抽象。

在 Android 中，定义了 URI 用来访问 ContentProvider 的 URI 结构，通常由四部分组成，具体如下：


[content://][xxx][/item][/123]

|------A------|-----------------B-------------------|---C---|---D--|

A组件称为Scheme，它固定为content://，表示它后面的路径所表示的资源是由Content Provider来提供的。
B组件称为Authority，它唯一地标识了一个特定的Content Provider，因此，这部分内容一般使用Content Provider所在的package来命名，使得它是唯一的。
C组件称为资源路径，它表示所请求的资源的类型，这部分内容是可选的。如果我们自己所实现的Content Provider只提供一种类型的资源访问，那么这部分内部就可以忽略；如果我们自己实现的Content Provider同时提供了多种类型的资源访问，那么这部分内容就不可以忽略了。
D组件称为资源ID，它表示所请求的是一个特定的资源，它通常是一个数字，它唯一地标志了某一种资源下的一个特定的实例。


    
### MIME 类型

MIME（Multipurpose Internet Mail Extensions）类型以及格式等，这些常量是第三方应用程序访问时要使用到的。


每个MIME类型由两部分组成，前面是数据的大类别，后面定义具体的种类。