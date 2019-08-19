---
title: 初步了解 Dex 文件
tags:
---



将 HelloWorld.java 编译生成 Dex 文件

1. 将 .java 文件编译生成 .class 文件
    > javac HelloWorld.java
2. 将 .class 文件编译成 .dex 文件
    > dx --dex --no-strict --output=HelloWorld.dex /xx/xx/HelloWorld.class(绝对路径)

    会在命令行目录生成 HelloWorld.dex ，不知道为什么自己总是报出 `path not find` 的错误，所以命令行中添加 `--no-strict` 参数。



可以通过以下命令反编译 dex 文件
>  dexdump -d HelloWorld.dex


---
知识链接：

[一篇文章带你搞懂DEX文件的结构](https://blog.csdn.net/sinat_18268881/article/details/55832757)



