---
title: 深入浅出计算机组成原理(二)-CPU是如何执行指令的
date: 2019-05-15 18:04:03
tags: [深入浅出计算机组成原理(徐文浩)]
---


代码变成指令后，是一条一条顺序执行的。

在代码的执行过程中，CPU 中的寄存器扮演着举足轻重的角色。

在硬件上，寄存器有锁存器或触发器组成。

![常见寄存器](https://static001.geekbang.org/resource/image/cd/6f/cdba5c17a04f0dd5ef05b70368b9a96f.jpg)


每种寄存器有不同的功能，而且每种寄存器都在指令执行过程中扮演重要角色。

<!-- more -->

* PC 寄存器(指令地址寄存器)：用来存放下一条需要执行的计算机指令的 **内存地址**。
* 指令寄存器：用来存放正在执行的指令。
* 条件码寄存器：用里面的一个个标志位，用来存在 CPU 进行计算或逻辑运算的结果。

以下图片展示如何根据机器码执行指令；

![执行指令](https://static001.geekbang.org/resource/image/ad/8a/ad91b005e97959d571bbd2a0fa30b48a.jpeg)

我们可以看到，执行程序时，CPU 会根据 PC 寄存器中的内存地址，将指定内存的指令码读取到指令寄存器中，指令长度自增，开始顺序执行下一个指令，执行上面相同的操作。

以上可见程序的指令码是在内存中顺序保存的，也会一条一条加载。


### 指令码跳转执行


程序的指令码在内存中是顺序执行的，那么在程序执行跳转逻辑是，机器指令是如何执行的。

```
// test.c
#include <time.h>
#include <stdlib.h>
int main()
{
  srand(time(NULL));
  int r = rand() % 2;
  int a = 10;
  if (r == 0)
  {
    a = 1;
  } else {
    a = 2;
  } 

```
编译命令：
```
gcc -g -c test.c
objdump -d -M intel -S test.o 
```

在 Mac 上对应的汇编码如下，但是我没有看懂，所以直接参看原博客：


```

1 // test.c
26:   99  cltd
27:   b9 02 00 00 00  movl    $2, %ecx
2c:   f7 f9   idivl   %ecx
2e:   89 55 f8    movl    %edx, -8(%rbp)
; int a = 10;
31:   c7 45 f4 0a 00 00 00    movl    $10, -12(%rbp)
; if (r == 0)
38:   83 7d f8 00     cmpl    $0, -8(%rbp)
3c:   0f 85 0c 00 00 00   jne 12 <_main+0x4e>
; a = 1;
42:   c7 45 f4 01 00 00 00    movl    $1, -12(%rbp)
; } else {
49:   e9 07 00 00 00  jmp 7 <_main+0x55>
; a = 2;
4e:   c7 45 f4 02 00 00 00    movl    $2, -12(%rbp)
; }
55:   8b 45 fc    movl    -4(%rbp), %eax
58:   48 83 c4 10     addq    $16, %rsp
5c:   5d  popq    %rbp
5d:   c3  retq
```

 原博客汇编码：

```
if (r == 0)
3b:   83 7d fc 00             cmp    DWORD PTR [rbp-0x4],0x0
3f:   75 09                   jne    4a <main+0x4a>
{
a = 1;
41:   c7 45 f8 01 00 00 00    mov    DWORD PTR [rbp-0x8],0x1
48:   eb 07                   jmp    51 <main+0x51>
}
else
{
a = 2;
4a:   c7 45 f8 02 00 00 00    mov    DWORD PTR [rbp-0x8],0x2
51:   b8 00 00 00 00          mov    eax,0x0
} 
```

重点关注以下两行(r == 0 的编译结果)：


```
3b:   83 7d fc 00             cmp    DWORD PTR [rbp-0x4],0x0
3f:   75 09                   jne    4a <main+0x4a>
```
判断当条件成立时，跳转到操作数为 4a 的指令，进行接下来的指令执行。

执行过程图：

![跳转指令执行](https://static001.geekbang.org/resource/image/b4/fa/b439cebb2d85496ad6eef2f61071aefa.jpeg)


所以代码变成指令后，是一条一条顺序执行的，这个顺序执行是基于跳转的顺序执行。


--- 
常见指令解释：


cmp DWORD PTR [rbp-0x4],0x0

    从寄存器地址为[rbp-0x4]取出值与 0x0 比较

mov DWORD PTR [rbp-0x8],0x2

    将 0x2 存入寄存器地址为 [rbp-0x8] 的内存中


jne 指令，是 jump if not equal 