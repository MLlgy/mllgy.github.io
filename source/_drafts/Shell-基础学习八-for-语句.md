---
title: Shell 基础学习八--流程控制之for 语句
tags:
---


### 语法一
```
for 变量 in 值1 值2 值3 ...
    do
        执行语句
    done
```

示例：

```
#!/bin/bash

for time in morning noon afternoon
    do 
        echo "time is $time"
    done
```


示例二：批量解压缩


```
#!/bin/bash
# 批量解压缩脚本
cd /dir
ls *.tar.gz > ls.log

for in ${cat ls.log}
    do
        tar -zxf $i 
        $>/dev/null
    done
rm -rf /dir/ls.log
```


### 语句二

```
for ((初始值；循环控制条件；变量变化)))
    do  
        程序
    done
```

示例：累加
```
#!/bin/bash
s=0
for((i=0;i<=100;i=i+1))
    do
        s=$(($s+$i))
    done
echo "The total num of 1+2+..+100 is : $s" 
```


```
#!/bin/bash

read -p 'plese input user name' -t 30 name
read -p 'plese input user num' -t 30 num
read -p 'plese input user pass' -t30  pass
if [ ! -z "$name" -a ! -z "$num" -a ! -z "$pass"]
    then
        y=$(echo $num | sed 's/[0-9]//g')
        if[ -z "$y"]
        then
        for((i=1;i<="$num";i=i+1))
            do  
                /usr/sbin/useradd $name$i &>/dev/null
                echo $pass | /usr/bin/passwd --stdin "$name$i" &>/dev/null
            done
        fi
fi
```