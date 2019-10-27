---
title: 'Kotlin 核心编程:操作符'
tags:
---



### let


let 的定义：

```
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block(this)
}
```

调用某对象的 let 函数式，该对象会作为函数的参数(以上代码中的 this )，在函数块中可以通过 it 代指该对象，返回值为最后一行或者指定retrun 表达式。