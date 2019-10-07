---
title: Shell 基础学习六 -- 流程控制之if 语句
tags:
---


### 1. 单分支 if 条件语句


```
if [ 条件判断式 ];then 
    程序
fi
```
或者

```
if [ 条件判断式 ] 
    then
        程序 
fi
```
注意点：

* [ 条件判断式 ]就是使用test命令判断，所 以中括号和条件判断式之间必须有 **空格**。

示例:判断分区使用率

```
#!/bin/bash #统计根分区使用率
# Author: )
rate=$(df -h | grep "/dev/sda3" | awk '{print $5}' | cut -d "%" - f1)
#把根分区使用率作为变量值赋予变量rate
if [ $rate -ge 80 ]
   then
echo "Warning! /dev/sda3 is full!!" fi

```


### 2. 双分支 if 条件语句


```
if [ 条件判断式] 
    then
        条件成立时，执行的程序 
    else
        条件不成立时，执行的另一个程序 
fi
```

示例： 备份 mysql 数据库

```
#!/bin/bash
#备份mysql数据库。
ntpdate asia.pool.ntp.org &>/dev/null 
#同步系统时间
date=$(date +%y%m%d) 
#把当前系统时间按照“年月日”格式赋予变量date
size=$(du -sh /var/lib/mysql) 
#统计mysql数据库的大小，并把大小赋予size变量

if [ -d /tmp/dbbak ]
   then
        echo "Date : $date!" > /tmp/dbbak/dbinfo.txt
        echo "Data size : $size" >> /tmp/dbbak/dbinfo.txt
        cd /tmp/dbbak
        tar -zcf mysql-lib-$date.tar.gz /var/lib/mysql &>/dev/null
        rm -rf /tmp/dbbak/dbinfo.txt
   else
        mkdir /tmp/dbbak
        echo "Date : $date!" > /tmp/dbbak/dbinfo.txt
        echo "Data size : $size" >> /tmp/dbbak/dbinfo.txt
        cd /tmp/dbbak
        tar -zcf mysql-lib-$date.tar.gz /var/lib/mysql &>/dev/null
        rm -rf /tmp/dbbak/dbinfo.txt
fi
```


实例二： 判断 appach 是否启动


```
#!/bin/bash
port=$(nmap -sT 192.168.1.156 | grep tcp | grep http | awk '{print $2}')
#使用nmap命令扫描服务器，并截取apache服务的状态，赋予变量port 
if [ "$port" == "open" ]
    then
        echo “$(date) httpd is ok!” >> /tmp/autostart-acc.log
   else
        /etc/rc.d/init.d/httpd start &>/dev/null
        echo "$(date) restart httpd !!" >> /tmp/autostart-err.log 
fi
```


### 3. 多分支 if 语句


```
if [ 条件判断式1 ] 
    then
        当条件判断式1成立时，执行程序1 
    elif [ 条件判断式2 ]
    then 
        当条件判断式2成立时，执行程序2
    „省略更多条件... 
else
    当所有条件都不成立时，最后执行此程序 
fi
```

示例：判断输入的内容

```
 #!/bin/bash #判断用户输入的是什么文件
# Author: shenchao (E-mail: shenchao@lampbrother.net) 
read -p "Please input a filename: " file
#接收键盘的输入，并赋予变量file 
if [ -z "$file" ]
#判断file变量是否为空 
    then
        echo "Error,please input a filename" 
        exit 1
 
elif [ ! -e "$file" ] 
#判断file的值是否存在
    then
        echo "Your input is not a file!" exit 2
elif [ -f "$file" ] 
#判断file的值是否为普通文件
   then
        echo "$file is a regulare file!"
elif [ -d "$file" ]
 #判断file的值是否为目录文件
   then
        echo "$file is a directory!"
else
        echo "$file is an other file!"
fi
```