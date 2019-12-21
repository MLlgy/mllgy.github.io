---
title: Linux 下常见的命令
tags:
---


* find

find 命令用于查找某个文件或者文件夹

> find . -name "*.java"

该命令用于查找当前目录下的所有扩展名为 Java 的文件。


* grep

 grep 命令为正则表达式匹配命令，用于字符串匹配，比如想要查找 Test.kt 中包含 ”Activity“ 的所有地方：

 > grep "Activity" Test.kt

 很明显的，可以看到 grep 和 find 的区别：find 主要用来查找目录或者文件，而 grep 主要用来查找指定文件中的字符串，字符串可以通过正则表达式描述。


 
