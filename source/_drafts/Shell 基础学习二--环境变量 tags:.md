---
title: Shell 基础学习二--环境变量
tags:
---



### 环境变量配置文件

要想对配置文件的修改想要永久生效，需要写入配置文件。

那环境变量配置文件的作用是什么？



环境变量配置文件主要是定义对系统操作环境生效的系统默认环境变量，比如 PATH、PSI、HISTSIZE、HOSTNAME 等默认环境变量。



#### one


* source 命令

环境变量的配置后需要重新登录才会生效，操作起来比较麻烦，使用 source 命令可以使修改后的配置立即生效。


source 配置文件 
或 
. 配置文件


以上两者等效。



#### 系统中的环境变量配置文件


```
/etc/profile
/etc/profile.d/*.sh
/etc/bashrc
以上对所有登录用户有效
~/.bash_profile
~/.bashrc
以上两个文件只对当前用户有效
```


#### 环境变量配置文件的调用顺序


 登录：

![](/../images/2019_09_20_01.png)
 

按照顺序查看每个配置文件的作用


##### /etc/profile 的作用

* USER 变量
* LOGNAME 变量
* MAIL 变量
* PATH 变量
* HOSTNAME 变量
* HISTNAME 变量
* umask
* 调用 /etc/profile.d/*.sh 文件

##### ~/.bash_profile 的作用

* 调用了 ~/.bashrc 文件
* 在PATH变量后面加入了“:$HOME/bin” 这个目录

##### ~/.bashrc 的作用

* 定义默认别名
* 调用 /etc/bashrc


#### /etc/bashrc 的作用

* PS1 变量
* umask
* PATH 变量
* 调用 /etc/profile.d/*.sh 文文件