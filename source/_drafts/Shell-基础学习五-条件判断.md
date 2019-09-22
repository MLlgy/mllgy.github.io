---
title: Shell 基础学习五--条件判断
tags:
---


### 1. 两种判断格式


* 方式一
> test -e /root/install.log


* 方式二
> [ -e /root/install.log ]

在 Shell 中多采用这种方式进行判断。

### 2. 按照文件类型进行判断


![](/../images/2019_09_22_01.png)


用蓝色字体标记的判断较为常用。



> [ -d /root ] && echo "yes" || echo "no"

第一个判断命令如果正确执行，则打印“yes”，否则打 印“no”




### 3. 按照文件权限进行判断

![](/../images/2019_09_22_02.png)


>  [ -w student.txt ] && echo "yes" || echo "no"


判断文件是拥有写权限的，但是此种方式不会区别用户组。

### 4. 两个文件之间的比较



![](/../images/2019_09_22_03.png)

简单理解：

nt: new to
ot: old to
ef: equal file


测试：

> ln /root/student.txt /tmp/stu.txt

创建个硬链接吧

>[ /root/student.txt -ef /tmp/stu.txt ] && echo "yes" || echo "no" yes


用test测试下。

### 5. 两个整数之间的比较


![](/../images/2019_09_22_04.png)

>[ 23 -ge 22 ] && echo "yes" || echo "no" yes



### 6. 两个字符串间的比较

![](/../images/2019_09_22_05.png)

>  name=sc

给name变量赋值

>[ -z "$name" ] && echo "yes" || echo "no" no

```
aa=11
bb=22
[ "$aa" == "bb" ] && echo "yes" || echo "no" no
```


### 7. 多重条件的判断

![](/../images/2019_09_22_07.png)


```
aa=11
[ -n "$aa" -a "$aa" -gt 23 ] && echo "yes" || echo "no"
no
```

```
 aa=24
[ -n "$aa" -a "$aa" -gt 23 ] && echo "yes" || echo "no" yes
```