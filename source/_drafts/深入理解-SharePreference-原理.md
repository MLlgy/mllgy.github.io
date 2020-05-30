---
title: 深入理解 SharePreference 原理
tags:
---
> 本文对 SharePreference 原理的分析主要参考 Gityuan [全面剖析SharedPreferences](http://gityuan.com/2017/06/18/SharedPreferences/)，相关源码均来自线上看源码最好的网站 [Android 社区](https://www.androidos.net.cn/android/9.0.0_r8/xref/frameworks/base/core/java/android/app/SharedPreferencesImpl.java)，以及自己的一点点心得，毕竟还是站在巨人的肩膀的才可以看的更远。


## 1. 简介

在日常开发过程中，使用 SharedPreferences 进行少量数据的本地持久化存储操作，是会频繁使用到的，具体的读、写操作如下：


### 读操作
```
SharedPreferences sharepre = getSharedPreferences("test", MODE_PRIVATE);
boolean isChecked = sharepre.getBoolean("test", false);
```

### 写操作
```
SharedPreferences.Editor editor = getSharedPreferences("test", MODE_PRIVATE).edit();
editor.putBoolean("test", true);
editor.commit();
```

## 2. SharedPreferences 基本原理

个人看法，无论何种技术想要完成持久化，最终还是要将数据写到文件(或者文件范畴)上，像 RAM 这种随用随走不留一丝痕迹的家伙，最终还是要靠 ROM。

SharedPreferences 能够实现本地持久话存储，其本质还是对本地文件(`/data/data/<app-packagename>/shared_prefs/`)进行读写，其中涉及的主关键类有：`ContextImpl`、`SharedPreferencesImpl`、`EditorImpl`，结合 Gityuan 的博文以及相关源码进行学习。

### 2.1 ContextImpl 中的关键点

理解 ContextImpl 涉及三个变量，对理解 SharePreference 的实现有很大的帮助，它们分别为：

**1. private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;**

该变量以应用的包名为 `key`，`value` 是以 SP 文件为 key，以 `SharedPreferencesImpl` 为 value 的 Map，有了这个变量就可以根据包名获得应用内所有的 `SharedPreferencesImpl` 实例以及对应的 `File`。

通过包名和文件名的双重限制，可以获得可以操作该文件的 `SharedPreferencesImpl` 的实例对象，得以实现 SP 的持久化功能，具体可以查看 `ContextImpl` 相应源码。


需要特别注意的是 `sSharedPrefsCache` 为静态成员变量，所以它是进程内唯一的，并且该变量由 `ContextImpl.class` 锁保。

**2. private File mPreferencesDir;**

`mPreferencesDir`:是指 SP 所在目录, 是指 `/data/data/<app-packagename>/shared_prefs/`。


**3. private ArrayMap<String, File> mSharedPrefsPaths;**

以文件名为 key，以对应 File 为 vaule 的 Map 结构，记录所有 SP 操作的文件，最终生成的文件名为 `mPreferencesDir + string` 的 xml 文件,比如 `/data/data/<app-packagename>/shared_prefs/test.xml`。

### 2.2 SharedPreferencesImpl 关键成员变量

在 ContextImpl 初始化 SharedPreferencesImpl 的过程中，会调用 loadFromDisk 方法，这个方法的主要功能是 **将对应文件的内容加载到内存中，以便开发者可以进行读写操作。**

```
SharedPreferencesImpl(File file, int mode) {
    ......
    startLoadFromDisk();
    ......
}
private void startLoadFromDisk() {
    synchronized (mLock) {
        mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}
```

### 2.3 loadFromDisk()

通过锁对象 `mLock` 进行同步操作:

```
private final Object mLock = new Object();
````
loadFromDisk 关键代码如下：

```
private void loadFromDisk() {
    synchronized (mLock) {
        if (mLoaded) {
            return;
        }
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }
    ......
    Map<String, Object> map = null;
    StructStat stat = null;
    Throwable thrown = null;
    try {
        stat = Os.stat(mFile.getPath());
        if (mFile.canRead()) {
            BufferedInputStream str = null;
            try {
                str = new BufferedInputStream(new FileInputStream(mFile), 16 * 1024);
                // 解析 Xml 文件，将相应的值包存到 Map 中
                map = (Map<String, Object>) XmlUtils.readMapXml(str);
            } catch (Exception e) {
                Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
            } finally {
                IoUtils.closeQuietly(str);
            }
        }
    }
    ......
    synchronized (mLock) {
        mLoaded = true;
        mThrowable = thrown;
        try {
            if (thrown == null) {
                if (map != null) {
                    //从文件读取的信息保存到 mMap
                    mMap = map;
                    //更新修改时间
                    mStatTimestamp = stat.st_mtim;
                    //更新文件大小
                    mStatSize = stat.st_size;
                } else {
                    mMap = new HashMap<>();
                }
            }
        } catch (Throwable t) {
            mThrowable = t;
        } finally {
             //将文件加载到内存中后，唤醒处于等待状态的线程，进行读写操作
            mLock.notifyAll();
        }
    }
}
```

至此之后，SP 通过类似 getString API 获得相应的数据，具体查询数据的逻辑可以参见下文。

### 2.3 查询数据

在进行数据查询是，如 `loadFromDisk` 没有执行完成, 则会阻塞查询操作;当数据加载完成, 通过 `mLock.notifyAll()` 方法， 则直接从 `mMap` 来查询相应数据，具体逻辑可以查看 `awaitLoadedLocked` 方法。

```
public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        // 检测是否从文件中加载完毕内容，当 loadFromDisk 加载完毕，通过 mLock.notifyAll() 唤醒处于等待状态的线程，接下来的操作才得以执行。
        awaitLoadedLocked();
        // 通过 mMap 获取相应的值
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}
```


### 2.4 awaitLoadedLocked

```
@GuardedBy("mLock")
private void awaitLoadedLocked() {
    if (!mLoaded) {
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    while (!mLoaded) {
        try {
            //如果还没有将内容从文件中加载完毕，则进入等待状态
            mLock.wait();
        } catch (InterruptedException unused) {
        }
    }
    if (mThrowable != null) {
        throw new IllegalStateException(mThrowable);
    }
}
其中 mLoaded 变量是在 loadFromDisk 完成加载文件中的内容后置为 true 的。

```

### 2.5 写入数据

```

public final class EditorImpl implements Editor {
    private final Map<String, Object> mModified = Maps.newHashMap();
    private boolean mClear = false;

    //插入数据
    public Editor putString(String key, @Nullable String value) {
        synchronized (this) {
            //插入数据, 先暂存到mModified对象
            mModified.put(key, value);
            return this;
        }
    }
    //移除数据
    public Editor remove(String key) {
        synchronized (this) {
            mModified.put(key, this);
            return this;
        }
    }

    //清空全部数据
    public Editor clear() {
        synchronized (this) {
            mClear = true;
            return this;
        }
    }
}
```

可以看到调用 putString 方法仅仅是修改 mModified 和 mClear 的相应的数据，最终需要调用 `commit()` 或者 `apply()`，才会把数据真正的更新到 SharedPreferencesImpl 中，从而持久化存储到 xml 文件中。

关于数据提交，有 commit 和 apply 两种方式。

### 2.5 commit

**SharedPreferencesImpl$EditorImpl#commit()**

```
public boolean commit() {
    //将数据更新到内存，将上面 mModified 中的数据更新到内存中，之后将数据同步到 SharedPreferencesImpl 中的文件
    MemoryCommitResult mcr = commitToMemory();
    //将内存数据同步到文件，在该方法中通过 writeToFIle 将数据写到文件中。
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null);
    try {
        //进入等待状态, 直到写入文件的操作完成
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    }
    //通知监听则, 并在主线程回调onSharedPreferenceChanged()方法
    notifyListeners(mcr);
    // 返回文件操作的结果数据
    return mcr.writeToDiskResult;
}
```

### 2.6 commitToMemory

```
private MemoryCommitResult commitToMemory() {
    long memoryStateGeneration;
    List<String> keysModified = null;
    Set<OnSharedPreferenceChangeListener> listeners = null;
    Map<String, Object> mapToWriteToDisk;
    synchronized (SharedPreferencesImpl.this.mLock) {
        if (mDiskWritesInFlight > 0) {
            mMap = new HashMap<String, Object>(mMap);
        }
        mapToWriteToDisk = mMap;
        mDiskWritesInFlight++;
        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            keysModified = new ArrayList<String>();
            listeners = new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
        }
        synchronized (mEditorLock) {
            boolean changesMade = false;
            //如果 mClear 为 true，则直接清除数据
            if (mClear) {
                if (!mapToWriteToDisk.isEmpty()) {
                    changesMade = true;
                    mapToWriteToDisk.clear();
                }
                mClear = false;
            }
            // 上面知道读写操作的均是操作 mModified，此处将 mModified 中数据复制给 mapToWriteToDisk，最终创建 MemoryCommitResult 对象
            for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                // this 是一个特殊的值，具体查看 remove(String key) 方法
                if (v == this || v == null) {
                    if (!mapToWriteToDisk.containsKey(k)) {
                        continue;
                    }
                    mapToWriteToDisk.remove(k);
                } else {
                    if (mapToWriteToDisk.containsKey(k)) {
                        Object existingValue = mapToWriteToDisk.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    mapToWriteToDisk.put(k, v);
                }
                //changesMade 代表数据是否有改变,在 key/value 发生改变时，则设置此变量为 true
                changesMade = true;
                if (hasListeners) {
                    keysModified.add(k);
                }
            }
            mModified.clear();
            if (changesMade) {
                mCurrentMemoryStateGeneration++;
            }
            memoryStateGeneration = mCurrentMemoryStateGeneration;
        }
    }

    return new MemoryCommitResult(memoryStateGeneration, keysModified, listeners,
            mapToWriteToDisk);
}
```

### 2.7 enqueueDiskWrite

enqueueDiskWrite 方法将数据更新到 xml 文件中，其中具体逻辑如下：

```
    private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        final boolean isFromSyncCommit = (postWriteRunnable == null);

        final Runnable writeToDiskRunnable = new Runnable() {
                @Override
                public void run() {
                    synchronized (mWritingToDiskLock) {
                        writeToFile(mcr, isFromSyncCommit);
                    }
                    synchronized (mLock) {
                        mDiskWritesInFlight--;
                    }
                    if (postWriteRunnable != null) {
                        postWriteRunnable.run();
                    }
                }
            };

        // 使用 commit 提交数据会执行该方法，而 apply 不会执行该方法
        if (isFromSyncCommit) {
            boolean wasEmpty = false;
            synchronized (mLock) {
                wasEmpty = mDiskWritesInFlight == 1;
            }
            if (wasEmpty) {
                writeToDiskRunnable.run();
                return;
            }
        }
        // 使用 apply 方法提交数据会执行该方法，commit 不会执行此操作，所以 apply 会在单独的线程中执行。
        QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
    }
```

### 2.8 writeToFile

```
private void writeToFile(MemoryCommitResult mcr, booleanisFromSyncCommit) {
    long startTime = 0;
    long existsTime = 0;
    long backupExistsTime = 0;
    long outputStreamCreateTime = 0;
    long writeTime = 0;
    long fsyncTime = 0;
    long setPermTime = 0;
    long fstatTime = 0;
    long deleteTime = 0;
    // 一些文件或者文件备份校验动作
    try {
        // 文件输出流
        FileOutputStream str = createFileOutputStrea(mFile);
        if (DEBUG) {
            outputStreamCreateTime = SystemcurrentTimeMillis();
        }
        if (str == null) {
            mcr.setDiskWriteResult(false, false);
            return;
        }
        // 通过文件输出流将 mcr.mapToWriteToDisk 的数据写入到件，至此将数据持久化文件上
        XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);
        writeTime = System.currentTimeMillis();
        FileUtils.sync(str);
    mcr.setDiskWriteResult(false, false);
}
```
至此，数据才算真正的被持久化到文件上。

### 2.10 apply

```

public void apply() {
    final long startTime = System.currentTimeMillis();
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            @Override
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }
            }
        };
    // 添加 awaitCommit 到 QueuedWork
    QueuedWork.addFinisher(awaitCommit);
    Runnable postWriteRunnable = new Runnable() {
            @Override
            public void run() {
                awaitCommit.run();
                //awaitCommit 执行完毕后，从QueuedWork中移除
                QueuedWork.removeFinisher(awaitCommit);
            }
        };
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);
}
```
在 Queue 中存在关键代码：

```
// 创建新的线程
HandlerThread handlerThread = new HandlerThread("queued-work-looper",Process.THREAD_PRIORITY_FOREGROUND);
handlerThread.start();
sHandler = new QueuedWorkHandler(handlerThread.getLooper());
```

 最终 apply 中的 Runnable 会加入在一个单独的线程的 Looper 中，也会在新的线程中执行相应的操作。


通过可以看到 commit 和 apply 的不同大概有两点：

1. commit 有返回值，标识提交数据是否成功，而 apply 无返回值；
2. apply 会调用 ` QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit)` 在单独的线程中执行相关的逻辑，而 commit 不会；


## 3. 总结

> 此部分为 COPY G大

**apply 与commit的对比**

1. apply没有返回值, commit有返回值能知道修改是否提交成功
2. apply是将修改提交到内存，再异步提交到磁盘文件; commit是同步的提交到磁盘文件；
3. 多并发的提交commit时，需等待正在处理的commit数据更新到磁盘文件后才会继续往下执行，从而降低效率; 而apply只是原子更新到内存，后调用apply函数会直接覆盖前面内存数据，从一定程度上提高很多效率。


**获取SP与Editor**:

1. getSharedPreferences()是从ContextImpl.sSharedPrefsCache唯一的SPI对象;
2. edit()每次都是创建新的EditorImpl对象.

**优化建议:**

1. 强烈建议不要在sp里面存储特别大的key/value, 有助于减少卡顿/anr(大的对象在读写文件时间变长)
2. 请不要高频地使用apply, 尽可能地批量提交();commit直接在主线程操作, 更要注意了
3. 不要使用MODE_MULTI_PROCESS;
4. 高频写操作的key与高频读操作的key可以适当地拆分文件, 用于减少同步锁竞争（可以分为两个文件）;
5. 不要一上来就执行getSharedPreferences().edit(), 应该分成两大步骤来做, 中间可以执行其他代码（因为获取 SP 对象需要加载文件到内存，这个步骤需要一定时间）。
6. 不要连续多次edit(), 应该获取一次获取edit(),然后多次执行putxxx(), 减少内存波动; 经常看到大家喜欢封装方法, 结果就导致这种情况的出现.
7. 每次commit时会把全部的数据更新的文件, 所以整个文件是不应该过大的, 影响整体性能;