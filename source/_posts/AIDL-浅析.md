---
title: AIDL 浅析
date: 2019-08-07 18:01:49
tags: [AIDL,Android]
---

AIDL 是理解 Android 系统不可避免的知识点。

0x0001 自定义 AIDL

为了更加直观的展示相关内容，我们通过具体示例来展示相关的细节。

自定义一个 aidl 文件，里面定义方法(如：MyAidl.aidl)，AS 会帮我们生产对于的类文件(MyAidl.java)。

<!-- more -->

1. 建立 java 同级目录 aidl：

![](/../images/2019_08_06_01.png)

2. 自定义 Aidl 文件

建立与 java 目录相同的包层级结构。


定义该该过程使用到的 Java 实体类，由于类对象会在 IPC 中使用，所以 **类需要实现序列化**。

```
public class Book implements Parcelable {
    private int bookId;
    private String bookName;

    .....
}
```

定义 Book.aidl(需要保证 Book.java 和 Book.aidl 在相同的包层级结构)
```
parcelable Book;
```

定义 IBookManager.aidl 文件，添加相关方法

```
interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}
```

此时 aidl 的文件目录如下：

![](/../images/2019_08_06_02.png)


3. AS build 目录下生成对应的 Java 文件

此处不会生成 Book.aidl 的 Java 文件，因为已经有 Book 类。


![](/../images/2019_08_06_03.png)


将 build 文件中的 IBookManager.java 拷贝出来，源码如下： [IBookManager.java](https://github.com/leeGYPlus/AidlDemo/blob/master/app/src/main/java/com/mk/aidldemo/server/IBookManager2.java)


IBookManager 的内部层级结构：
```
public interface IBookManager extends android.os.IInterface {

    ......
    ......

    public static abstract class Stub extends android.os.Binder implements com.mk.aidldemo.IBookManager {
        
        ......
        ......

        private static class Proxy implements com.mk.aidldemo.IBookManager {
            ......
            ......
        }
    }
}

```




### 0X0002 流程分析

为什么不生成 3 个文件(一个接口、两个类)，而是放在了一个文件中，这是因为当多个 AIDL 类时， Stub 和  Proxy 就会重名或者多个类会显得比较繁杂，而把它们放在各自的 AIDL 类中，就会比较容易区分。

下面分析如何进行跨进程通信。


起决定性作用的是 Stub 的 asInterface 方法和 onTranscact 方法，首先通过一个示意图大致了解其过程。


![](/../images/2019_08_07_01.jpg)

1. 对于 Client 端，作为 AIDL 的使用端，我们开启相关的业务：

```
IBookManager.asInterface(IBinder 对象).addBook(Book(countId, "Book $countId"))
```

> 这个 Binder 对象就是在 bindService 时，相关 Service 中的 onBinder 方法的返回对象。

asInterface 方法主要是判断参数，也就是 IBinder 对象，**是和与自己同处一个进程**：
* 是，则直接转换、直接使用，则接下来的操作与 Binder 跨进程无关。
* 否，则会把这个 IBinder 对象包装成一个 Proxy 对象，这时调用的 Stub 的方法，间接调用 Proxy 的相应方法。

```
public static IBookManager2 asInterface(android.os.IBinder obj) {
    if ((obj == null)) {
        return null;
    }
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin != null) && (iin instanceof IBookManager2))) {
        return ((IBookManager2) iin);
    }
    return new Stub.Proxy(obj);
}
```

2. 在 Proxy 中调用相关的方法，会使用 Pracelable 数据来准备数据，把函数名、函数的参数都写入 _data,使用 _reply 来接收函数的返回值，使用 Binder 的 transact 方法，把数据传给 Binder 的 Server 端。

```
public void addBook(com.mk.aidldemo.Book book) throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        if ((book != null)) {
            _data.writeInt(1);
            book.writeToParcel(_data, 0);
        } else {
            _data.writeInt(0);
        }
        Log.e("process proxy add", ProcessUtils.getCurrentProcessName());
        // mRemote 对象为构建 Proxy 对象时传入
        mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
        _reply.readException();
    } finally {
        _reply.recycle();
        _data.recycle();
    }
}
```


3. Server 端通过 onTransact 方法来接收 Client 传过来的数据(包括函数名称、函数的参数、函数的标识)，找到指定的函数，就相应的数据传入，得到结果并将结果写回。


### 0x0003 具体分析

针对 Binder 跨进程通信机制，在每次通信过程中都需要有 Binder Client 端和 Binder Server 端。


在上文的流程图中，可以看到  Stub 为相应的 Binder Server 端，即为 Service 所在的进程中，我们通过加入 Log 日志，查看相应的操作执行哪个进程。


具体 Log 打点查看源码: [GitHub 源码](https://github.com/leeGYPlus/AidlDemo/blob/master/app/src/main/java/com/mk/aidldemo/MainActivity.kt)

```
E/process binderService: com.mk.aidldemo
E/process add: com.mk.aidldemo
E/process proxy add: com.mk.aidldemo
E/process add: com.mk.aidldemo
E/process proxy add: com.mk.aidldemo

E/process service addBook: com.mk.aidldemo:remote
E/process onTransact add: com.mk.aidldemo:remote
```


可以看到在进程 `com.mk.aidldemo:remote` 中执行的操作有：onTransact 和 Server 中实例化 Binder 中的方法，即为 Binder Server 端，其他均处于 Binder Client 端。


这其中的关键方法有 mRemote.transact  和 onTransact。


**onTransact**

这个方法运行在 **服务端中的 Binder线程池** 中，当客户端发起跨进程请求时，远程请求会通过 `系统底层封装` 后交由此方法来处理。该方法的原型为`publicBooleanonTransact(intcode,android.os.Parceldata,android.os.Parcelreply,intflags)`。服务端通过 code 可以确定客户端所请求的目标方法是什么，接着从data中取出目标方法所需的参数（如果目标方法有参数的话），然后执行目标方法。当目标方法执行完毕后，就向 reply 中写入返回值（如果目标方法有返回值的话），onTransact 方法的执行过程就是这样的。需要注意的是，如果此方法返回false，那么客户端的请求会失败，因此我们可以利用这个特性来做权限验证，毕竟我们也不希望随便一个进程都能远程调用我们的服务。

**transact**

Proxy#getBookList 这个方法运行在 **客户端**，当客户端远程调用此方法时，它的内部实现是这样的：首先创建该方法所需要的输入型Parcel对象_data、输出型Parcel对象_reply和返回值对象List；然后把该方法的参数信息写入_data中（如果有参数的话）；**接着调用 transact 方法来发起RPC（远程过程调用）请求，同时当前线程挂起**； 然后 **服务端的onTransact 方法会被调用**，直到 RPC 过程返回后，当前线程继续执行，并从_reply中取出RPC过程的返回结果；最后返回_reply中的数据。


这两个方法都为 Binder 的方法，至于底层是如何实现 RPC 实现了，需学习相关细节，期待。


----

知识链接：

[Android 插件化开发指南](http://product.dangdang.com/25325752.html)

[Android 开发艺术探索](http://product.dangdang.com/23766472.html)