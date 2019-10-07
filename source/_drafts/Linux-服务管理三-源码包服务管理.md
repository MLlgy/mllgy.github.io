---
title: Linux 服务管理三--源码包服务管理
tags:
---


### 源码包的安装和启动


使用绝对路径，调用启动脚本来启动，是不同源码包的启动脚本不同，可以查看源码包的安装说明，查看启动方法。

例如：

> /usr/local/apache2/bin/apachectl start|stop


### 源码包的自启动


>vi /etc/rc.d/rc.local

加入如下操作：


> /usr/local/apache2/bin/apachectl start


### 如何使源码包服务被服务管理命令识别(不推荐，如何混淆系统服务与源码服务)


* 让源码包的apache服务能被service命令管理
启动:建立连接

> ln -s /usr/local/apache2/bin/apachectl /etc/init.d/apache



* 让源码包的apache服务能被chkconfig与 ntsysv命令管理自启动
> vi /etc/init.d/apache

添加以下内容：
```
# chkconfig: 35 86 76
#指定httpd脚本可以被chkconfig命令管理。格式是: chkconfig: 运行级别 启动顺序 关闭顺序
# description: source package apache
#说明，内容随意
```

执行命令，把源码包 apache 加入chkconfig命令

>chkconfig --add apache 