---
title: makefile 的基本了解
tags:
---


## Makefile 的规则


```
target ... : prerequisites ...  
    command  
    ...  
    ...
```

* target

target 为一个目标文件，可以是目标文件、可执行文件、标签。

* prerequisites

生成 target 所需要的文件或者目标。

* common 

make 需要执行的命令。


target 这一个或多个的目标文件依赖于 prerequisites 中的文件，其生成规则定义在 command 中。


make 并不管命令是怎么工作的，他只管执行所定义的命令。
make 会比较 targets 文件和 prerequisites 文件的修改日期，如果 prerequisites 文件的日期要比targets 文件的日期要新，或者 target 不存在的话，那么，make 就会执行后续定义的命令。


## 定义变量


在 Makefile 中定义变量以及使用：


```
# 定义变量
object = .....
# 使用变量
${object} 或 $(object)
```


## Makefile 可以自动推导

make 找到 test.o，那么它可以自动推导 test.c 为 test.o 的依赖文件，那么在变下 makefile
时可以省略相应文件名的书写。



```
test.o : test.c defs.h
# 以上省略 test.c，如下：
test.o : defs.h
```

以上为 makefile 的隐晦规则。



## Makefile 里有什么


1. 显示规则


2. 隐晦规则

3. 定义变量

4. 文件指示

    * 引用其他的 makefile 文件，类比 C 中的 include
    * 在某些情况下指定 makefile 的有效部分，类比 #if
  
5. 注释


## 引用其他的 makefile

使用 `include` 关键字引入其他关键字。

在 `include` 前面可以有一些空字符，但是绝不能是[Tab]键开始。`include` 和可以用一个或多个空格隔开。

```
include <filename>
```


实例：

```
include foo.make *.mk $(bar)
```

**执行**：

make 命令开始时，会把找寻 include 所引入的其它 Makefile，并把其内容安置在当前的位置。


如果你想让 make 不会理那些无法读取的文件，而继续执行，你可以在include前加一个减号“-”

```
  -include <filename>
```

## make 工作方式


GNU的make工作时的执行步骤入下：（想来其它的make也是类似）

1. 读入所有的Makefile。
2. 读入被include的其它Makefile。
3. 初始化文件中的变量。
4. 推导隐晦规则，并分析所有规则。
5. 为所有的目标文件创建依赖关系链。
6. 根据依赖关系，决定哪些目标要重新生成。
7. 执行生成命令。

## Makefile 书写规则

规则包含两个部分，一个是依赖关系，一个是生成目标的方法。

Makefile 中只应该有一个最终目标，在 Makefile 中的目标可能会有很多，**但是第一条规则中的目标** 将被确立为最终的目标。


### 规则书写


规则语法：

```
targets : prerequisites  
    command  
    ...
```

或是这样：
```
targets : prerequisites ; command  
    command  
    ...
```


targets 即为目标，目标基本上是一个文件，也有可能是多个文件。

prerequisites 为目标依赖的一个或多个文件。


common 单独一行是必须以 Tab 键开头，否则要用分号分隔。



示例：
```
# foo模块  
foo.o : foo.c defs.h       # foo.0 为我们的目标，foo.c、defs.h  为依赖文件  
    cc -c -g foo.c # 命令
```

### 在规则中使用通配符

在 makefile 中可以使用三个通配符：*、？、[]。


```
foo.o : *.c *.h
    cc -c -g *.c

# 定义变量 object
object = *.o

# 使用变量,wildcard 为 makefile 中的关键字
objects = $(wildcard *.o)
```

### 在 makefile 中搜索文件


一个大的项目，会将源码按照业务逻辑放入不同的文件夹中，当 make 需要去找寻文件的依赖关系时，你可以在文件前加上路径，但最好的方法是把一个路径告诉 make，让make在自动去找。


makefile 中定义了特殊的变量 -- VPATH，如果存在该变量时，make 首先会在当前目录寻找，找不到的话，回去 VPATH 指定的目录中寻找。

```
# 多个目录由 : 标记
VPATH = src:../src
```

同时 makefile 中还存在 vpath 关键字，较之更加灵活,vpath 中可以使用 %。


```
# 在 ../src 目录下搜索以  .c 结尾的文件
vpath %.c ../src
```


### 伪目标


“伪目标”不是文件，所以make无法生成它的依赖关系和决定它是否要执行。

显式声明一个目标是伪目标：

```
.PHONY : clean
clean:
    rm *.c
```

### 多目标


makefile 中可以有多个目标，有时候多个目标依赖同一个文件，并且它们执行的命令大致相同，那么可以将它们合并：

```

bigoutput : text.g  
    generate text.g -big > bigoutput  
littleoutput : text.g  
    generate text.g -little > littleoutput
````

合并为：


```
bigoutput littleoutput : text.g  
    generate text.g -$(subst output,,$@) > $@
```


-$(subst output,,$@) 的含义：


* $: 表示执行一个 makefile 函数
* subst：函数名，后面为参数
* $@： 表示目标的集合(bigoutput littleoutput 组成的集合)，依次取出，并执行命令。

关于函数，见下文。

### 静态模式

静态模式可以更加容易地定义多目标的规则.

```
<targets ...>: <target-pattern>: <prereq-patterns ...>  
    <commands>  
    ...
```

targets: 定义一系列文件，是生成目标的而一个集合。

target-pattern： 目标集模式，比如下例中目标集以 .c 结尾。。

prereq-patterns：依赖集模式，依据目标集模式下生成的目标文件集生成依赖文件集。

```
objects = foo.o bar.o

$(objects): %.o: %.c  
    $(CC) -c $(CFLAGS) $< -o $@
```

$(objects) 表示目标集为 foo.o bar.o 组成的集合。

%.o 表示目标为以 .o 结尾的集合。

%.c 表示取出目标集 %，将依赖集模式定义为 %.c，即 foo.c、bar.c


$<: 依次取出所有依赖文件的集和，即 foo.c bar.c 组成的集合。

$@：依次取出目标集合。



## Makefile 中的命令

make的命令默认是被“/bin/sh”——UNIX的标准Shell解释执行的。

在一条规则中，make 会按照顺序依次执行命令。

### 显示命令

@ 在命令前，那么在执行时这个命令不会被显示出来。

```
@echo 正在编译XXX模块
```
执行时输出：

```
正在编译XXX模块
```
没有 @

```

echo 正在编译XXX模块
```
执行时输出：
```
echo 正在编译XXX模块
正在编译XXX模块
```

### 执行命令


如果你要让上一条命令的结果应用在下一条命令时，你应该将两个命令写在同一行，使用分号分隔这两条命令。

```
exec:
    # 情况一
    cd /home/lele
    pwd
    # 情况二
    cd /home/lele;pwd
```

情况一的打印日志为当前 makefile 目录，而情况二打印日志为： `/home/lele` 。

在执行命令时，如果命令执行错误，那么 make 会终止该规则的执行，也有可能终止所有规则的执行。有时候命令的出错并不代表错误，如删除一个文件，当文件不存在，也不会产生具体意义上的错误，，此时可以在 makefile 命令行之前添加 “-”，标记无论该命令是出不出错都认为是成功的。


## 使用变量


在Makefile中，变量可以使用在“目标”，“依赖目标”，“命令”或是Makefile的其它部分中。

变量在声明时需要给予初值，使用变量时，最好使用 “（）”或者“{}“ 把变量名包起来，如果要使用真实的 $，那么需要用 $$ 表示。


### 另外一种定义变量的方式

#### 1. ：=

可以使用 ':=' 的方式来定义变量，这种方式的意义在于前面的变量不可以使用后面的变量。


```
x = foo
y = $(x) bar
```

此时 y 的值为 foo bar。


```
x := foo
y = $(x) bar
```

此时 y 的值为 bar。


#### 2. ？=


```
x ?= bar
```
x 如果每天被定义过，那么变量 x 的值为 bar。


### 环境变量


### 自动化变量


$@

$<

....

## 条件判断
ifeq、ifneq、ifdef、ifndef

ifeq()
...
else
...
endif


ifneq()
...
else
...
endif

ifdef
...
else
...
endif


## 函数


### 函数语法

函数调用和变量一样使用 $ 来标记。

函数调用语法：

```
$(<function> <arguments>)
```
or

```
${<function> <arguments>}
```

参数之间使用 逗号”，“ 分隔。

### 字符处理函数


* 字符替换函数

$(subst <from>,<to>,<text>)

把字串<text>中的<from>字符串替换成<to>。

* 模式字符串替换函数

$(patsubst <pattern>,<replacement>,<text>)


```
$(patsubst %.c,%.o,x.c.c bar.c)
```
返回结果为 x.c.o bar.o

* 去空格函数

$(strip <string>)

去掉<string>字串中开头和结尾的空字符。

* 查找字符串函数

$(findstring <find>,<in>)

在字串<in>中查找<find>字串,如果找到，那么返回<find>，否则返回空字符串。

* 过滤函数

$(filter <pattern...>,<text>)

以<pattern>模式过滤<text>字符串中的单词，保留符合模式<pattern>的单词。可以有多个模式，返回符合模式<pattern>的字串。


### foreach 函数

$(foreach <var>,<list>,<text>)

把参数<list>中的单词逐一取出放到参数<var>所指定的变量中，然后再执行<text>所包含的表达式。


```
names := a b c d
files := $(foreach n,$(names),$(n).o)
```
$(files)的值是“a.o b.o c.o d.o”


## if 函数

$(if &lt;condition&gt;,&lt;then-part&gt;)

或者 

$(if &lt;condition&gt;,&lt;then-part&gt;,&lt;else-part&gt;)


## call 函数

call 函数是唯一一个可以用来 **创建新的参数化** 的函数。


```
$(call <expression>,<parm1>,<parm2>,<parm3>...)
```

这个表达式中(expression)，你可以定义许多参数，然后你可以用 call 函数来 **向这个表达式传递参数**。

```
reverse =  $(2) $(1)
foo = $(call reverse,a,b) 
```
$(n) 代表第几个参数，上例中 foo 的值为 b a。


## shell 函数

contents := $(shell cat foo)

shell函数把执行操作系统命令后的输出作为函数返回
