---
title: Shell 基础学习七-- 流程控制之多分支case条件语句
tags:
---


case语句和if...elif...else语句一样都是多 分支条件语句，不过和if多分支条件语句 不同的是，case语句只能判断一种条件关 系，而if语句可以判断多种条件关系

```
case $变量名 in 
    "值1")
        如果变量的值等于值1，则执行程序1
        ;; 
    "值2")
        如果变量的值等于值2，则执行程序2
        ;; 
        ...省略其他分支... 
    *)
        如果变量的值都不是以上的值，则执行此程序
        ;; 
esac
```

示例：

```
#!/bin/bash

echo 'if you want to sh,please input 1'
echo 'if you want to gz,please input 2'
echo 'if you want to cd,please input 3'


read -t 30 -p "please input your chose:" cho

case "$cho" in
    "1")
        echo "sh"
        ;;
    "2")
        echo "gz"
        ;;
    "3")
        echo "cd"
        ;;
    *)
        echo "error"

esac
```






touch local.properties
echo "sdk.dir=/Users/daojia/Library/Android/sdk" >> local.properties

chmod a+x gradlew
./gradlew clean
./gradlew assembleRelease




cp -a /Users/daojia/Documents/jiagubao360 ./

cd jiagubao360/jiagu/
#yangxl_dj@163.com  yxl66080101 daojia123
read  -t 30 -p "输入登录名: " name
read  -t 30 -p "输入登录密码: " pass
read  -t 30 -p "输入签名密码: " aspass
java -jar jiagu.jar -login $name  #pass

java -jar jiagu.jar  -importsign ../../app/sheep.jks $aspass sheep $aspass

java -jar jiagu.jar  -config -x86

java -jar jiagu.jar -jiagu ../../app/build/outputs/apk/release/*.apk ./ -autosign

rm ../../app/build/outputs/apk/release/*release.apk

cp -a *.apk ../../app/build/outputs/apk/release/