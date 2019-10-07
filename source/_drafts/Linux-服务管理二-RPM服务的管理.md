---
title: Linux 服务管理二-RPM服务的管理
tags:
---


### 独立服务的管理
RPM 包配置安装服务的默认目录：
```
/etc/init.d/:启动脚本位置
/etc/sysconfig/:初始化环境配置文件位置  /etc/:配置文件位置
/etc/xinetd.conf:xinetd配置文件
/etc/xinetd.d/:基于xinetd服务的启动脚本  /var/lib/:服务产生的数据放在这里
/var/log/:日志
```



#### 独立服务的启动
> /etc/init.d/独立服务名 start|stop|status|restart|

其实所有的服务位于 /etc/init.d/ 目录中。

> service 独立服务名 start|stop|restart|status

#### 独立服务的自启动

> chkconfig [--level 运行级别] [独立服务名] [on|off]

> 修改/etc/rc.d/rc.local文件(推荐)

> 使用ntsysv命令管理自启动



### 基于 xinetd 服务的管理


xinetd ：超级守护进程


验证操作：

> yum -y install xinetd
> yum -y install telnet-server


### 2. xinetd服务的启动

> vi /etc/xinetd.d/telnet
由于不常用，具体使用在查询。

重启xinetd服务
> service xinetd restart

### 3. xinetd服务的自启动 
 
1. 方式一
> chkconfig telnet on
2. 方式二
ntsysv

xinetd 的启动和自启动状态会保持一致，会被同时修改。