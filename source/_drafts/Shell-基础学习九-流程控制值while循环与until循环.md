---
title: Shell 基础学习九-流程控制值while循环与until循环
tags:
---


### while 循环

```
while [ 条件判断语句]
    do
        执行语句
    done
```


示例：


```
#!/bin/basah
i=1
s=0
while [ $i -le 100]
    do
        s=$(($i+$s))
        i=$(($i+1))
    done
echo "total num is: $s"
```



### until 循环


```
until [终止条件]
    do
    done
```


示例：

```
#!/bin/basah
i=1
s=0
until [ $i -gt 100]
    do
        s=$(($i+$s))
        i=$(($i+1))
    done
echo "total num is: $s"
```