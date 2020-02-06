---
title: Linux 多用户管理
tags:
---


## UID、GID

区别不同用户的数字称为 UID。


Linux 中的用户分为3 类：普通用户、根用户、系统用户。


普通用户端而 UID 大于 500。


根用户的 UID 为 0。

系统用户的 UID 范围：1~499.


区别不同用户组的 ID 称为 GID。


## 查看 UID、GID 的命令

确定自己的而 UID：id
确认自己所属用户组；groups
查询当前登录的用户：who


## /etc/passwd /etc/shadow


系统用来记录用户名和密码的文件。


## 新增和删除用户


* 新增用户：useradd

useradd user1
useradd -u 555 user2// 指定 UID

useradd -g user1 user3
useradd -d /home/dic4 user2//指定家目录


* 修改密码：passwd

passwd user1


* 修改用户：usermod

* 删除用户：userdel

userdel user// 该命令不会删除用户的家目录和邮件信息。
userdel -r user//会删除


## 新增和删除用户组

groupadd group1
groupdel group1


## 检查用户信息


* 查看用户


users：当前系统有哪些用户
who：展示登录的用户
w: 比 who 更加详细


* 调查用户


finger：显示系统的登录用户

finger user1： 显示用户的详细信息

