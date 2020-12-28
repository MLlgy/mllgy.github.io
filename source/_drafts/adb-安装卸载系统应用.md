---
title: adb 安装卸载系统应用
tags:
---


## 卸载

adb root 
adb disable-vertity
adb reboot
adb root 
adb remount
adb shell
su
cd /system/app
rm xxx.apk
rm -rf /data/data/xxx.apk/ 删除对应 apk 的文件目录
adb reboot

## 安装

adb root
adb remount
adb push xxx.apk /system/app
adb reboot （系统应用需要系统签名）