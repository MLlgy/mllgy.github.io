---
title: HandlerThread 相关
tags:
---

### HandlerThread 源码

```
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;
    private @Nullable Handler mHandler;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    /**
     * @param priority The priority to run the thread at. The value supplied must be from {@link android.os.Process} and not from java.lang.Thread.
     */
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    
    /**
     * Call back method that can be explicitly overridden if needed to execute some setup before Looper loops.
     */
    protected void onLooperPrepared() {
    }

    // 此时会对线程相关的 Looper 进行初始化，并 notifyAll()
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
    
    /**
     * This method returns the Looper associated with this thread. If this thread not been started
     * or for any reason isAlive() returns false, this method will return null. If this thread
     * has been started, this method will block until the looper has been initialized.  
     * 该方法返回与此 Thread 相关的 Looper。
     */
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    @NonNull
    public Handler getThreadHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper());
        }
        return mHandler;
    }

    /**
     * Quits the handler thread's looper.
     * @return True if the looper looper has been asked to quit or false if the
     * thread had not yet started running.
     */
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    /**
     * Quits the handler thread's looper safely.
     * @return True if the looper looper has been asked to quit or false if the
     * thread had not yet started running.
     */
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    public int getThreadId() {
        return mTid;
    }
}
```
从 HandlerThread 的字面可以猜想 HandlerThread 是一个和 Handler 、Thread 均相关的类，根据上面源码可以 HandlerThread 是可以使用 Handler 的 Thread。但是与普通 Thread 只执行后台耗时任务不同，HanderThread 内部创建了 **线程相关的消息队列 Looper**，至于为什么是线程相关可以看一下:[ThreadLocal(Jdk1.8) 使用及源码分析](https://leegyplus.github.io/2019/09/19/ThreadLocal(Jdk1.8)%20%E4%BD%BF%E7%94%A8%E5%8F%8A%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)。我们可以着重看一下 HanderThread 的 run 方法：

```
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```

在 run() 方法中通过 Looper.prepare() 创建消息队列，通过 Looper.loop() 来开启队列，有了 Looper 就可以创建 Handler 了。

具体使用：
    外部需要通过 Handler 的消息方式来通知 HandlerThread 执行一个具体的任务。

```
HandlerThread handlerThread = new HandlerThread("HandlerThread");
handlerThread.start();
Handler mHandler = new Handler(handlerThread.getLooper()){
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        Log.d("Log","current thread is "+Thread.currentThread().getName());
    }
};
mHandler.sendEmptyMessage(1);
```

同时自己在 Thread 中使用 Handler：

```
Handler mHandler;
new Thread(new Runnable() {
    @Override
    public void run() {
        Looper.prepare();//Looper初始化
        //Handler初始化 需要注意, Handler初始化传入Looper对象是子线程中缓存的Looper对象
        mHandler = new Handler(Looper.myLooper());
        Looper.loop();//死循环
    }
}).start();
```
当然这种写法不如直接使用 HandlerThread 方便、安全。

### 使用 HandlerThread 实现 IntentService 相关功能

IntentService 是一个在后台任务执行完毕后自动停止的 Service，由于 IntentService 的优先级比较高，所以可以使用它进行高优先级的后台任务。

```
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

    public IntentService(String name) {
        super();
        mName = name;
    }

    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();
        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    
    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }


    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }

    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}
```

可以看到 IntentService 内部封装了 HandlerThread 和 Handler，在 onStartCommand 会调用 onStart，在 onStart 中通过 Handler 对象发送信息，最终


----

[Android 艺术探索]