---
title: 'adb shell top '
date: 2020-06-16 18:08:54
tags:
---


### 0x0001 adb shell top

执行如下命令，查看该命令的用途：

`adb shell top --help`

打印结果：
```
usage: top [-Hbq] [-k FIELD,] [-o FIELD,] [-s SORT] [-n NUMBER] [-m LINES] [-d SECONDS] [-p PID,] [-u USER,]

Show process activity in real time.// 实时显示进行信息

-H	Show threads // 展示线程
-k	Fallback sort FIELDS (default -S,-%CPU,-ETIME,-PID)
-o	Show FIELDS (def PID,USER,PR,NI,VIRT,RES,SHR,S,%CPU,%MEM,TIME+,CMDLINE)
-O	Add FIELDS (replacing PR,NI,VIRT,RES,SHR,S from default)
-s	Sort by field number (1-X, default 9)
-b	Batch mode (no tty)
-d	Delay SECONDS between each cycle (default 3)// 设置刷新频率，默认 3 秒
-m	Maximum number of tasks to show // 展示最大的几项
-n	Exit after NUMBER iterations // n 次刷新后结束
-p	Show these PIDs
-u	Show these USERs
-q	Quiet (no header lines)

Cursor LEFT/RIGHT to change sort, UP/DOWN move list, space to force update, R to reverse sort, Q to exit.

```

执行 adb shell top，结果为：


![](adb-shell-top/2020_06_16_02.png)

其中列表属性含义见下表：

|PID|USER|PR|NI|VIRT|RES|SHR|S|%CPU|%MEM|TIME+|
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--
|进程id|进程所有者|进程优先级|nice值。负值表示高优先级，正值表示低优先级|进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES|进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA|共享内存大小，单位kb|进程状态。D=不可中断的睡眠状态 R=运行 S=睡眠 T=跟踪/停止 Z=僵尸进程|上次更新到现在的CPU时间占用百分比|进程使用的物理内存百分比|进程使用的CPU时间总计，单位1/100秒|
