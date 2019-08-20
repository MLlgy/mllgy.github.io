---
title: 从缓存文件的角度帮你理解 Okhttp3 缓存原理
date: 2019-08-19 16:07:38
tags: [Okhttp3,Okhttp3 缓存文件]
---



> 本文以一个不同的角度来解读 Okhttp3 实现缓存功能的思路，即：对于对于的缓存空间(文件夹)中的缓存文件的生成时机、不同时期下个文件的状态、不同时期下日志文件读写。通过这些方法来真正理解 Okhttp3 的缓存功能。如果你理解 DiskLrcCache 开源库的设计，那么对于 Okhttp3 的缓存实现你就已经掌握了，因为前者以后者为基础，你甚至没有看本文的必要。

# 1. 需要了解的概念

缓存功能的实现，理所当然的涉及文件的读写操作、缓存机制方案的设计。Okhttp3 缓存功能的实现涉及到 Okio 和 DiskLruCache，在阐述具体缓存流程之前，我们需要了解两者的一些基本概念。

<!-- more -->


## 1.2 Okio

Okio 中有两个关键的接口: **Sink** 和 **Source** ，对比 Java 中 I/O 流概念，我们可以把 Sink 看作 OutputStream , 把 Source 看作 InputStream 。

其具体实现非本文重点，有兴趣自己可以查看源码。

## 1.1 DiskLruCache

Okhttp3 中 DiskLruCache 与JakeWharton 大神的 [DiskLruCache](https://github.com/JakeWharton/DiskLruCache) 指导思想一致，但是具体细节不同，比如前者使用 Okio 进行 IO 操作，更加高效。

在 DiskLruCache 有几个重要概念，了解它们，才能对 DiskLruCache 的实现原理有基本的认识。

为了能够表达的更加直观，我们看一下一张图片进行缓存时缓存文件的具体内容：
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/1969719-74b5b0b5c7ff7db3?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

### 1.2.1 日志文件 journal

该文件为 DiskLruCache 内部的日志文件，对 cache 的每一次操作都对应一条日志，并写入到 journal 文件中，同时也可以通过 journal 文件的分析创建 cache。

打开上图中 journal 文件，具体内容为：

```
libcore.io.DiskLruCache
1
201105
2

DIRTY 0e39614b6f9e1f83c82cf663e453a9d7
CLEAN 0e39614b6f9e1f83c82cf663e453a9d7 4687 14596
```

在 DiskLruCache.java 类中，我们可以看到对 journal 文件内容的描述，在这里自己对其简单翻译，有兴趣的朋友可以看 JakeWharton 的描述: [DiskLruCache](https://github.com/JakeWharton/DiskLruCache/blob/master/src/main/java/com/jakewharton/disklrucache/DiskLruCache.java)。
```
文件的前五行构成头部，格式一般固定。
第一行: 常量 -- libcore.io.DiskLruCache ；
第二行: 硬盘缓存版本号 --  1
第三行: 应用版本号 -- 201105
第四行: 一个有意义的值 -- 2
第五行: 空白行

头部后的每一行都是 Cache 中 Entry 状态的一条记录。
每条记录的信息包括: 状态值(DIRTY CLEAN READ REMOVE) 缓存信息entry的key值 状态相关的值(可选)。 

下面对记录的状态进行说明：
DIRTY: 该状态表明一个 entry 正在被创建或更新。每一个成功的 DIRTY 操作记录后应该 CLEAN 或 REMOVE 操作记录，否则被临时创建的文件会被删除。
CLEAN: 该状态表明一个 entry 已经被成功的创建，并且可以被读取，后面记录了对应两个文件文件(具体哪两个文件后面会谈到)的字节数。
READ: 该状态表明正在跟踪 LRU 的访问。
REMOVE: 该状态表明entry被删除了。
```

需要注意的是在这里 DIRTY 并不是 “脏”、“脏数据” 的意思，而是这个数据的状态不为最终态、稳定态，该文件现在正在被操作，
而 CLEAN 并不是数据被清除，而是表示该文件的操作已经完成。同时在后续的 dirtyFiles 和 cleanFiles 也表示此含义。 

关于日志文件在整个缓存系统中的作用，在后续过程中用到它的时候在具体阐述。


### 1.2.2 DiskLruCache.Entry

每个 DiskLruCache.Entry 对象代表对每一个 URl 在缓存中的操作对象，该类成员变量的具体含义如下：
```
private final class Entry {
        final String key; // Entry 的 key
        final long[] lengths; // key.0 key.1 文件字节数的数组
        final File[] cleanFiles; // 稳定的文件数组
        final File[] dirtyFiles;// 正在执行操作的文件数组
        boolean readable;// 如果该条目被提交了，为 true
        Editor currentEditor;// 正在执行的编辑对象，在没有编辑时为 null
        long sequenceNumber;// 编辑条目的最近提交的序列号
        
        ...
        ...
}
```

具体操作在缓存实现流程中阐述。

### 1.2.3 DiskLruCache.SnapShot

此类为缓存的快照，为缓存空间中特定时刻的缓存的状态、内容，该类成员变量的具体含义：
```
public final class Snapshot implements Closeable {
        private final String key;
        private final long sequenceNumber; // 编辑条目的最近提交的序列号
        private final Source[] sources;// 缓存中 key.0 key.1 文件的 Okio 输入流
        private final long[] lengths;// 对应 Entry 中的 lengths，为文件字节大小
        
        ...
        ...
}
```
### 1.2.3 DiskLruCache.Editor

该类为 DiskLruCache 的编辑器，顾名思义该类是对 DiskLruCache 执行的一系列操作，如：abort() 、 commit() 等。

**Entry publish 的含义是什么？？？？？**
# 2. 缓存实现的有关流程

简单介绍了几个概念，在这一节具体查看一下缓存实现的具体流程。在这之前我们需要明确一下几个前提：
1. OkhttpClient 设置支持缓存。
2. 网络请求头部中的字段设置为支持缓存(Http 协议首部字段值对缓存的实现有影响，具体看参见 [图解 HTTP](https://item.jd.com/11449491.html)、[HTTP 权威指南](https://item.jd.com/11056556.html))。

**由多个拦截器构成的拦截器链是 Okhttp3 网络请求的执行关键，可以说整个网络请求能够正确的执行是有整个链驱动的 (责任链模式)。仿照 RxJava 是事件驱动的，那么 Okhttp3 是拦截器驱动的。**

关于缓存功能实现的拦截器为 **CacheInterceptor**, CacheInterceptor 位于拦截器链中间位置，那么以执行下一个拦截器为界将缓存流程分为两部分：
1. 触发之后拦截器之前的操作
2. 触发之后拦截器之后的操作

即以 `networkResponse = chain.proceed(networkRequest);` 为分界

### 1. 触发之后拦截器之前的操作
```
Response cacheCandidate = cache != null
                ? cache.get(chain.request())// 执行 DiskLruCache#initialize()
                : null;//本地缓存

        long now = System.currentTimeMillis();
        // 缓存策略
        CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
        //策略中的请求
        Request networkRequest = strategy.networkRequest;
        ////策略中的响应
        Response cacheResponse = strategy.cacheResponse;

        if (cache != null) {
            cache.trackResponse(strategy);
        }

        if (cacheCandidate != null && cacheResponse == null) {
            closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
        }

        //缓存和网络皆为空，返回code 为504 的响应
        // If we're forbidden from using the network and the cache is insufficient, fail.
        if (networkRequest == null && cacheResponse == null) {
            return new Response.Builder()
                    .request(chain.request())
                    .protocol(Protocol.HTTP_1_1)
                    .code(504)
                    .message("Unsatisfiable Request (only-if-cached)")
                    .body(Util.EMPTY_RESPONSE)
                    .sentRequestAtMillis(-1L)
                    .receivedResponseAtMillis(System.currentTimeMillis())
                    .build();
        }

        // If we don't need the network, we're done.  缓存策略请求为null，则使用缓存
        if (networkRequest == null) {
            return cacheResponse.newBuilder()
                    .cacheResponse(stripBody(cacheResponse))
                    .build();
        }

```


#### 1.1 日志文件的初始化

当执行如下代码时会按照调用链执行相关逻辑：

```
Response cacheCandidate = cache != null
                ? cache.get(chain.request())// 执行 DiskLruCache#initialize()
                : null;//本地缓存
```

首先检查在缓存中是否存在该 request 对应的缓存数据，如果有的话就返回 Response，如果没有就置 null。

调用链来到以下方法：

```
@Nullable
Response get(Request request) {
    String key = key(request.url());
    DiskLruCache.Snapshot snapshot;
    Entry entry;
    try {
        snapshot = cache.get(key);// 在这里会执行 
        ...
    return response;
}
```
在 `snapshot = cache.get(key);` 处执行相应的初始化操作。

在此过程中执行一个特别重要的操作，需要对缓存中的 journal 系列日志文件(包括 journal journal.bak) 进行新建、重建、读取等操作，具体查看源码：

```
// DiskLruCache#initialize()
public synchronized void initialize() throws IOException {
        assert Thread.holdsLock(this);

        if (initialized) {// 代码 1 
            return; // Already initialized.
        }
        // If a bkp file exists, use it instead. journal文件备份是否存在
        if (fileSystem.exists(journalFileBackup)) {// 代码 2
            // If journal file also exists just delete backup file.
            if (fileSystem.exists(journalFile)) {
                fileSystem.delete(journalFileBackup);
            } else {
                fileSystem.rename(journalFileBackup, journalFile);
            }
        }
        // Prefer to pick up where we left off.
        if (fileSystem.exists(journalFile)) {
            try {
                readJournal();// 代码 3
                processJournal(); // 代码 4
                initialized = true; // 代码 5
                return;
            } catch (IOException journalIsCorrupt) {
                Platform.get().log(WARN, "DiskLruCache " + directory + " is corrupt: "
                        + journalIsCorrupt.getMessage() + ", removing", journalIsCorrupt);
            }

            // The cache is corrupted, attempt to delete the contents of the directory. This can throw and
            // we'll let that propagate out as it likely means there is a severe filesystem problem.
            try {
                delete();
            } finally {
                closed = false;
            }
        }
        rebuildJournal();// 代码 6
        initialized = true;// 代码 7
    }
```
##### 1. App 启动后的初始化 

在启动 App 是标志位 `initialized = false`，那么由 `代码 1` 可知此时需要执行初始化操作。
```
if (initialized) {// 代码 1 
    return; // Already initialized.
}
```

###### 1.1  若 journal 日志文件存在 



如果存在 journal.bak 那么将该文件重命名为 journal。

接下来对 journal 日志文件所做的操作如 `代码 3、4 、5`  所示，具体作用做如下阐述。`代码 3 ` 要做的是读取日志文件 journal 并根据日志内容初始化 `LinkedHashMap<String, Entry> lruEntries` 中的元素，DiskLruCache 正是通过 LinkedHashMap 来实现 LRU 功能的。我们看一下 readJournal() 的具体代码:

```
private void readJournal() throws IOException {
        BufferedSource source = Okio.buffer(fileSystem.source(journalFile));
        try {
            String magic = source.readUtf8LineStrict();
            String version = source.readUtf8LineStrict();
            String appVersionString = source.readUtf8LineStrict();
            String valueCountString = source.readUtf8LineStrict();
            String blank = source.readUtf8LineStrict();
            if (!MAGIC.equals(magic)
                    || !VERSION_1.equals(version)
                    || !Integer.toString(appVersion).equals(appVersionString)
                    || !Integer.toString(valueCount).equals(valueCountString)
                    || !"".equals(blank)) {
                throw new IOException("unexpected journal header: [" + magic + ", " + version + ", "
                        + valueCountString + ", " + blank + "]");
            }

            int lineCount = 0;
            while (true) {// 不断执行如下操作，直到文件尾部，结束如下操作
                try {
                    readJournalLine(source.readUtf8LineStrict());
                    lineCount++;
                } catch (EOFException endOfJournal) {
                    break;
                }
            }
            redundantOpCount = lineCount - lruEntries.size();

            // If we ended on a truncated line, rebuild the journal before appending to it.
            if (!source.exhausted()) {
                rebuildJournal();
            } else {
                journalWriter = newJournalWriter();
            }
        } finally {
            Util.closeQuietly(source);
        }
    }
```
在方法的开始读取 journal 日志文件的头部做基本的判断，如不满足要求则抛出异常。接下来在 该方法中通过方法 -- `readJournalLine(source.readUtf8LineStrict());` 读取 journal 日志文件的每一行，根据日志文件的每一行生成 Entry 存入 lruEntries 中用来实现 LRU 功能。
```
  private void readJournalLine(String line) throws IOException {
        ...
        ...
        // 一顿操作得到 key 的值
        
        // 根据日志文件中 key 值获得或者生成 Entry，存入 lruEntries 中
        Entry entry = lruEntries.get(key);
        if (entry == null) {
            entry = new Entry(key);
            lruEntries.put(key, entry);
        }

        if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {
            String[] parts = line.substring(secondSpace + 1).split(" ");
            entry.readable = true;
            entry.currentEditor = null;
            entry.setLengths(parts);
        } else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {
            entry.currentEditor = new Editor(entry);
        } else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {
            // This work was already done by calling lruEntries.get().
        } else {
            throw new IOException("unexpected journal line: " + line);
        }
    }
```

readJournal() 执行完毕后相当于对 lruEntries 进行初始化。lruEntries 元素的个数等于该 App 在此缓存文件夹下缓存文件的个数。在此过程中如果 lruEntries 中没有此行日志中的 key 对应的 Entry 对象，因为现在为进入 App 中的对缓存空间的初始化，所以都需要新建该类的对象：

```
 // 根据日志文件中 key 值获得或者生成 Entry，存入 lruEntries 中
    Entry entry = lruEntries.get(key);
        if (entry == null) {
        entry = new Entry(key);
        lruEntries.put(key, entry);
    }
```
新建 Entry 对象的过程对于整个缓存体系的构建也十分重要，代码如下：
```
Entry(String key) {
    this.key = key;

    lengths = new long[valueCount];
    cleanFiles = new File[valueCount];
    dirtyFiles = new File[valueCount];

    // The names are repetitive so re-use the same builder to avoid allocations.
    //名称是重复的，所以要重复使用相同的构建器以避免分配
    StringBuilder fileBuilder = new StringBuilder(key).append('.');
    int truncateTo = fileBuilder.length();
    for (int i = 0; i < valueCount; i++) {
        fileBuilder.append(i);
        cleanFiles[i] = new File(directory, fileBuilder.toString()); // key.0 key.1
        fileBuilder.append(".tmp");
        dirtyFiles[i] = new File(directory, fileBuilder.toString());// key.0.tmp key.1.tmp
        fileBuilder.setLength(truncateTo);
        }
    }
```
新建对象过程中会根据 valueCount = 2; 的值定义缓存文件分别为 key.0、key.1、key.0.tmp、key.1.tmp ,其中 key.0 为稳定状态下的请求的 mate 数据，key.1 为稳定状态下的缓存数据，而 key.0.tmp、key.1.tmp 分别为 mate 数据和缓存数据的临时文件,此时并不会真正的新建文件。

**在这里需要明确的是 cleanFiles 和 dirtyFiles 都是 Entry 的成员变量，也就是说是通过 Entry 的对象对两者进行读取并进行相关操作的。**
   
   
processJournal() 方法实现了缓存文件夹下删除无用的文件。
```
private void processJournal() throws IOException {
    fileSystem.delete(journalFileTmp);
    for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
        Entry entry = i.next();
        if (entry.currentEditor == null) {
            for (int t = 0; t < valueCount; t++) {
                size += entry.lengths[t];
            }
        } else {
            entry.currentEditor = null;
            for (int t = 0; t < valueCount; t++) {
                fileSystem.delete(entry.cleanFiles[t]);
                fileSystem.delete(entry.dirtyFiles[t]);
            }
            i.remove();
        }
    }
}
```

> **何为无用的文件 ？**
>
>如果文件夹下存在 `entry.currentEditor != null;` 的文件，说明此文件为处在编辑状态下，但是此时的时机为刚打开 App 后的初始化状态，所有的文件均不应该处在编辑状态，所以此状态下的文件即为无用的文件，需要被删除。


执行完毕后标志位 initialized 置位为 true 并中断执行 (return;) 返回操作去执行其他操作。

###### 1.2  若 journal 日志文件不存在

若 journal 日志文件不存在，那么不会执行 代码 2、3、4、5 直接执行代码 6 --  rebuildJournal() ，具体执行操作如下：

```
synchronized void rebuildJournal() throws IOException {
        if (journalWriter != null) {
            journalWriter.close();
        }
        //产生 journal.tmp 文件
        BufferedSink writer = Okio.buffer(fileSystem.sink(journalFileTmp));
        try {// 写入 journal 文件内容
            writer.writeUtf8(MAGIC).writeByte('\n');
            writer.writeUtf8(VERSION_1).writeByte('\n');
            writer.writeDecimalLong(appVersion).writeByte('\n');
            writer.writeDecimalLong(valueCount).writeByte('\n');
            writer.writeByte('\n');

            /**
             *  将 lruEntries 的值重新写入 journal 文件
             */
            for (Entry entry : lruEntries.values()) {
                if (entry.currentEditor != null) { // 当前的 editor 不为 null 说明当前 journal 为非稳定态
                    writer.writeUtf8(DIRTY).writeByte(' ');
                    writer.writeUtf8(entry.key);
                    writer.writeByte('\n');
                } else {
                    writer.writeUtf8(CLEAN).writeByte(' ');
                    writer.writeUtf8(entry.key);
                    entry.writeLengths(writer);
                    writer.writeByte('\n');
                }
            }
        } finally {
            writer.close();
        }
        // journal.tmp --> journal
        if (fileSystem.exists(journalFile)) {
            fileSystem.rename(journalFile, journalFileBackup);
        }
        fileSystem.rename(journalFileTmp, journalFile);
        fileSystem.delete(journalFileBackup);

        journalWriter = newJournalWriter();
        hasJournalErrors = false;
        mostRecentRebuildFailed = false;
    }
```

十分重要的操作为 ： Okio.buffer(fileSystem.sink(journalFileTmp)); ,因为此时 journal 不存在，那么此行代码执行的操作正是新建journal 临时文件 --  journal.tmp ,写入文件头部文件后将 journal.tmp 重命名为 journal 。前文解析 journal 文件内容的含义，此处代码正好可以作为印证。


#### 1.2 初始化后

经过初始化后最终获取 DiskLruCache 快照 DiskLruCache$Snapshot 对象，并进行相关包装返回 Response 对象为缓存中的Response 对象。


```
 @Nullable
    Response get(Request request) {
        ...
        ...
        try {
            snapshot = cache.get(key);// 在这里会执行 initialize(),进行一次初始化
            if (snapshot == null) {
                return null;
            }
        ...
        ...
        Response response = entry.response(snapshot);

        ...
        ...

        return response;
    }
```

至此,以上即为进入 CacheInterceptor 后的第一步操作，说实话工作量真是大，开启了 Debug 模式 n 遍才稍微把基本流程搞明白。
```
Response cacheCandidate = cache != null
                ? cache.get(chain.request())// 执行 DiskLruCache#initialize() ，会对 journal 文件进行一些操作
                : null;//本地缓存
```


#### 1.3 缓存策略

缓存策略的获取主要涉及代码如下：

```
CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
```
具体执行代码位置：
`CacheStrategy#getCandidate()`，由于具体业务逻辑比较容易理解，根据缓存响应、请求中头部关于缓存的字段进行相关判断，得出缓存策略，在这里不做过多阐释。

### 2. 触发之后拦截器之后的操作

触发之后的拦截器后，进行相关的一系列操作，根据责任链模式逻辑还是会最终回来，接着此拦截器的逻辑继续执行。此时整个请求的状态为已经成功得到网络响应，那么我们要做的就是对网络响应进行缓存，具体代码如下：

```
if (cache != null) {
    if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
    // Offer this request to the cache.
    CacheRequest cacheRequest = cache.put(response);// 将 response 写入内存中，此时进行的步骤： 创建 0.tmp(已经写入数据) 和 1.tmp(尚未写入数据)
        return cacheWritingResponse(cacheRequest, response);
        }

    if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
            cache.remove(networkRequest);
        } catch (IOException ignored) {
            // The cache cannot be written.
        }
    }
}
```

跟随 CacheRequest cacheRequest = cache.put(response); 执行如下逻辑：


```
CacheRequest put(Response response) {
        ...
        ...
        
        //由Response对象构建一个Entry对象,Entry是Cache的一个内部类
        Entry entry = new Entry(response);
        DiskLruCache.Editor editor = null;// disk 缓存的编辑
        try {
            editor = cache.edit(key(response.request().url()));// key(response.request().url()) 根据 URL生成唯一 key
            if (editor == null) {
                return null;
            }
            //把这个entry写入
            //方法内部是通过Okio.buffer(editor.newSink(ENTRY_METADATA));获取到一个BufferedSink对象，随后将Entry中存储的Http报头数据写入到sink流中。
            entry.writeTo(editor);// 触发生成 0.tmp
            //构建一个CacheRequestImpl对象，构造器中通过editor.newSink(ENTRY_BODY)方法获得Sink对象
            return new CacheRequestImpl(editor);// 触发生成 1.tmp
        } catch (IOException e) {
            abortQuietly(editor);
            return null;
        }
    }

```
Cache#writeTo()
```
// 写入 0.tmp 数据 // 写入 的dirtyfile 文件的 buffersink 输出流
public void writeTo(DiskLruCache.Editor editor) throws IOException {
    BufferedSink sink = Okio.buffer(editor.newSink(ENTRY_METADATA));//新建 key.0.tmp
    // TODO: 在这里出现了 0.tmp
    sink.writeUtf8(url)
            .writeByte('\n');
    ....
}
```
非常明显的操作在此处创建了 key.0.tmp 文件，并写入数据，此处写入的数据为 mate 数据

```
CacheRequestImpl(final DiskLruCache.Editor editor) {
    this.editor = editor;
    this.cacheOut = editor.newSink(ENTRY_BODY);// 在这里生成 1.tmp
    this.body = new ForwardingSink(cacheOut) {
        @Override
        public void close() throws IOException {
            synchronized (Cache.this) {
                if (done) {
                    return;
                }
                done = true;
                writeSuccessCount++;
            }
            super.close();
            editor.commit();//最终调用了此函数，0.tmp 1.tmp --》 key.0  key.1 
        }
    };
}
```

在初始化 CacheRequestImpl 对象时创建了 key.1.tmp 文件。

执行如上操作后回到 CacheInterceptor 执行 cacheWritingResponse() 方法：

```
private Response cacheWritingResponse(final CacheRequest cacheRequest, Response response)
            throws IOException {
        // Some apps return a null body; for compatibility we treat that like a null cache request.
        if (cacheRequest == null) return response;
        Sink cacheBodyUnbuffered = cacheRequest.body();
        if (cacheBodyUnbuffered == null) return response;

        final BufferedSource source = response.body().source();
        final BufferedSink cacheBody = Okio.buffer(cacheBodyUnbuffered);

        Source cacheWritingSource = new Source() {
            boolean cacheRequestClosed;

            @Override
            public long read(Buffer sink, long byteCount) throws IOException {
                long bytesRead;
                try {
                    bytesRead = source.read(sink, byteCount);
                } catch (IOException e) {
                    if (!cacheRequestClosed) {
                        cacheRequestClosed = true;
                        cacheRequest.abort(); // Failed to write a complete cache response.
                    }
                    throw e;
                }

                if (bytesRead == -1) {
                    if (!cacheRequestClosed) {
                        cacheRequestClosed = true;
                        cacheBody.close(); // The cache response is complete!
                    }
                    return -1;
                }

                sink.copyTo(cacheBody.buffer(), sink.size() - bytesRead, bytesRead);
                cacheBody.emitCompleteSegments();
                return bytesRead;
            }

            @Override
            public Timeout timeout() {
                return source.timeout();
            }

            @Override
            public void close() throws IOException {
                if (!cacheRequestClosed
                        && !discard(this, HttpCodec.DISCARD_STREAM_TIMEOUT_MILLIS, MILLISECONDS)) {
                    cacheRequestClosed = true;
                    cacheRequest.abort();
                }
                source.close();
            }
        };

        return response.newBuilder()
                .body(new RealResponseBody(response.headers(), Okio.buffer(cacheWritingSource)))
                .build();
```
执行一系列操作，使用 Okio 这个库不断的向 key.1.tmp 写入数据，具体操作过程实在是太过繁杂，而且牵涉到 Okio 库原理，自己在这么短时间无法理清具体流程。

**对于数据写入的切入点自己还没有很好的认识，在何处真正进行写文件操作自己只能够通过 Debug 知道其走向，但是对其原理还没有理解。**


最后会执行 CacheRequestImpl 对象的close 方法，

```
CacheRequestImpl(final DiskLruCache.Editor editor) {
            this.editor = editor;
            this.cacheOut = editor.newSink(ENTRY_BODY);//在这里生成 1.tmp
            this.body = new ForwardingSink(cacheOut) {
                @Override
                public void close() throws IOException {
                    synchronized (Cache.this) {
                        if (done) {
                            return;
                        }
                        done = true;
                        writeSuccessCount++;
                    }
                    super.close();
                    editor.commit();// 最终调用了此函数，0.tmp 1.tmp -> key.0  key.1 
                }
            };
        }

```
执行 editor.commit(); 该方法会调用的 completeEdit()。


```
synchronized void completeEdit(Editor editor, boolean success) throws IOException {
        Entry entry = editor.entry;
        if (entry.currentEditor != editor) {
            throw new IllegalStateException();
        }

        // If this edit is creating the entry for the first time, every index must have a value.
        if (success && !entry.readable) {
            for (int i = 0; i < valueCount; i++) {
                if (!editor.written[i]) {
                    editor.abort();
                    throw new IllegalStateException("Newly created entry didn't create value for index " + i);
                }
                if (!fileSystem.exists(entry.dirtyFiles[i])) {
                    editor.abort();
                    return;
                }
            }
        }
        // key.0.tmp key.1.tmp --> key.0 key.1
        for (int i = 0; i < valueCount; i++) {
            File dirty = entry.dirtyFiles[i];
            if (success) {
                if (fileSystem.exists(dirty)) {
                    File clean = entry.cleanFiles[i];
                    fileSystem.rename(dirty, clean);
                    long oldLength = entry.lengths[i];
                    long newLength = fileSystem.size(clean);
                    entry.lengths[i] = newLength;
                    size = size - oldLength + newLength;
                }
            } else {
                fileSystem.delete(dirty);
            }
        }

       ....
    }

```

该方法中最终会将 key.0.tmp 、key.1.tmp 分别 重命名为 key.0 、key.1 ，这两个文件分别为两个文件的稳定状态，同时更新 journal 日志记录。



----

至此 Okhttp3 实现缓存功能的大致流程基本结束，但是其中还是有很多的逻辑和细节是自己没有发现和不能理解的，其源码还是需要不断的去阅读去理解，需要对其中的实现、思想有进一步的体会。


