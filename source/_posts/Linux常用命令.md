---
title: Linux
date: 2020-07-14 16:38:29
tags:
---

## lsof

lsof -i 显示所有的连接

lsof -i | grep 8080

找到使用 8080 端口的进程信息。

kill -9 pid

杀死进程 pid 