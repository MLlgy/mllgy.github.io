---
title: Rxjava 源码学习(一):基本流程
tags:
---


版本：Rxjava2.2.8 

首先看一下最简单的例子，具体查看其内部实现：


```
Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("one");
        emitter.onNext("two");
        emitter.onComplete();
    }
}).subscribe(new Observer<String>() {
    @Override
    public void onSubscribe(Disposable d) {
        System.out.println("onSubscribe:" + d.toString());
    }
    @Override
    public void onNext(String s) {
        System.out.println("onNext：" + s);
    }
    @Override
    public void onError(Throwable e) {
        System.out.println("Throwable：" + e.getMessage());
    }
    @Override
    public void onComplete() {
        System.out.println("onComplete：");
    }
});
```
打印日志为如下，可以看到事件接收的顺序和事件发送的顺序相同。
```
onSubscribe:CreateEmitter{null}
onNext：one
onNext：two
onComplete
```

### create


```
public static <T> Observable<T> create(ObservableOnSubscribe<T> source)
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```
source 即为我们在使用 Rxjava 的 create 中传入的匿名对象，在上文中为 ObservableOnSubscribe，其方法 -- subscribe 的参数为事件发生器，可以用来发送事件，事件发生器为整个事件流的驱动。

其中涉及了 ObservableCreate，由于这个类对流程进行十分重要，所以我们查看一下其具体实现：

```
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);

        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }

    static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {

        private static final long serialVersionUID = -3434801548987643227L;

        final Observer<? super T> observer;

        CreateEmitter(Observer<? super T> observer) {
            this.observer = observer;
        }
        ...
        ...
    }

    static final class SerializedEmitter<T>
    extends AtomicInteger
    implements ObservableEmitter<T> {
        ...
        ...
    }
}
```
具体的类层级结构为：ObservableCreate 内部包含两个内部类，分别为：CreateEmitter、SerializedEmitter，两者均继承了 ObservableEmitter，是事件发生器，为整个事件流的起点，但是具体如果发送事件，继续向下分析。

在 subscribeActual 方法中，我们可以看到创建了  CreateEmitter 对象，并通过以下调用：

```
observer.onSubscribe(parent);
source.subscribe(parent);
```

以上代码使 Obervable、Observer 与时间发射器分别产生关联，是事件流得以进行下去的关键。其实 `observer.onSubscribe(parent)` 即为在使用 Rxjava 过程的 Observer 中的 `onSubscribe(Disposable d)`，而 `source.subscribe(parent)` 即为 即为在使用 Rxjava 过程的 Observable 中的 `public void subscribe(ObservableEmitter<String> emitter)`，以上全是在方法 subscribeActual 中调用的，具体 subscribeActual 什么时候调用，查看下面的分析。


**组装**

通过调用 RxJavaPlugins.onAssembly 去组装 Observable，具体代码：

```
public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    Function<? super Observable, ? extends Observable> f = onObservableAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```

至于 onObservableAssembly 是什么没看懂，但是不影响对流程的分析，在此方法中，我们可以认为直接返回传入的 Observable 对象。

**订阅**


Observable 对象通过 subscribe 与 Observer 产生调用关系。


```
public final void subscribe(Observer<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        // 同样，我们可以认为直接返回 observer，对流程分析无影响 
        observer = RxJavaPlugins.onSubscribe(this, observer);
        ...
        // 在此处 ObservableOnSubscribe#subscribeActual
        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        RxJavaPlugins.onError(e);
        ...
        throw npe;
    }
}
```

就是在此时通过 ObservableOnSubscribe#subscribeActual 方法，从而使 Observer、Observable 分别与事件发生器发生关联。从上面知道 subscribeActual 中调用了 `source.subscribe(parent)`，其实为新建 ObservableOnSubscribe 对象的 subscribe 方法，从而完成回调 Rxjava 使用过程中自定义的事件发生器：


```
Observable.create(new ObservableOnSubscribe<String>() {
    // source.subscribe(parent) 方法最终会回调到此处
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("one");
        emitter.onNext("two");
        emitter.onComplete();
    }
}).subscribe(new Observer<String>() {
    ...
});
```

**发布事件**



既然 Observer、Observable 分别和事件发生器(Emitter) 产生关联，并且通过回调来到事件发射现场，那么具体查看是如何发生事件，以及观察者如何对每个事件是如何调用的。

具体以 CreateEmitter 为例：


```
    static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {

        private static final long serialVersionUID = -3434801548987643227L;

        final Observer<? super T> observer;

        CreateEmitter(Observer<? super T> observer) {
            this.observer = observer;
        }

        @Override
        public void onNext(T t) {
            if (t == null) {
                onError(new NullPointerException("onNext called with null....."));
                return;
            }
            if (!isDisposed()) {
                observer.onNext(t);
            }
        }

        @Override
        public void onError(Throwable t) {
            if (!tryOnError(t)) {
                RxJavaPlugins.onError(t);
            }
        }

        ...

        @Override
        public void onComplete() {
            if (!isDisposed()) {
                try {
                    observer.onComplete();
                } finally {
                    dispose();
                }
            }
        }

        @Override
        public void setDisposable(Disposable d) {
            DisposableHelper.set(this, d);
        }

        @Override
        public void setCancellable(Cancellable c) {
            setDisposable(new CancellableDisposable(c));
        }
        ...
        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }

        ...
    }
```

事件发生器发射事件：
```
emitter.onNext("one");
```

此时会调用到 CreateEmitter#onNext：

```
        @Override
        public void onNext(T t) {
            if (t == null) {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }
            if (!isDisposed()) {
                observer.onNext(t);
            }
        }
```

由于 发生器与自定义 Observer 已经产生关联关系，那么此时就会回调 Observer 的 onNext，也就是我们自定义 Observer 中的而如下代码


```
@Override
public void onNext(String s) {
    System.out.println("onNext：" + s);
}
```

同理 emitter.onComplete()，emitter.onError() 也是如上过程，只不过 onError 会首先判断 error 是否能够自己处理，否则就交给 RxJavaPlugins 处理：

```
@Override
public void onError(Throwable t) {
    if (!tryOnError(t)) {
        RxJavaPlugins.onError(t);
    }
}

@Override
public boolean tryOnError(Throwable t) {
    if (t == null) {
        t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators an
    }
    if (!isDisposed()) {
        try {
            // 最终会调用到自定义  Observer 的 onError 方法
            observer.onError(t);
        } finally {
            dispose();
        }
        return true;
    }
    return false;
}
```
至此，Rxjava 的基本流程分析结束。


事件流可以而上而下进行下去，原因是 Observable.操作符 得到的还是 Observable，通过通过 Observable.subsribe 方法实现订阅关系。

### Map 操作符

map 操作符使用代码：
```
Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("one");
        emitter.onNext("two");
        emitter.onComplete();
    }
}).map(new Function<String, Integer>() {
    @Override
    public Integer apply(String s) throws Exception {
        return Integer.valueOf(s);
    }
}).subscribe(new Observer<Integer>() {
    @Override
    public void onSubscribe(Disposable d) {
        System.out.println("onSubscribe:" + d.toString());
    }
    @Override
    public void onNext(Integer s) {
        System.out.println("onNext：" + s);
    }
    @Override
    public void onError(Throwable e) {
        System.out.println("Throwable：" + e.getMessage());
    }
    @Override
    public void onComplete() {
        System.out.println("onComplete：");
    }
});
```

map 操作符：
```
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
}
```
将 Function 对象 mapper 通过 ObservableMap 传给 ObservableMap，并完成相应的赋值。

```
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;

    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        this.function = function;
    }

    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }

    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;

        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            // 将观察者对象 actual 赋值给 downstream
            super(actual);
            this.mapper = mapper;
        }
        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }
            if (sourceMode != NONE) {
                downstream.onNext(null);
                return;
            }
            U v;
            try {
                // 调用 mapper.apply ，其实是自定义 Function 中的 apply 方法
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
            // 最终调用 Observer 的 onNext 方法
            downstream.onNext(v);
        }

        @Override
        public int requestFusion(int mode) {
            return transitiveBoundaryFusion(mode);
        }
        ....
    }
}
```

接下来的流程和 Rxjava 基本流程基本相同：
在执行产生订阅关系的方法： subscribe 时调用了 ObservableMap#subscribeActual：

```
@Override
public void subscribeActual(Observer<? super U> t) {
    // 此处的 source 为调用 map 操作符的 Observable，即上一步通过 create 创建的 Observable 
    source.subscribe(new MapObserver<T, U>(t, function));
}
```

```
public final void subscribe(Observer<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        observer = RxJavaPlugins.onSubscribe(this, observer);
        ObjectHelper.requireNonNull(observer, "The RxJavaPlugins.onSubscribe hook retu...);
        // 由于多态的存在，此处的 subscribeActual 会调用 ObservableCreate 的subscribeActual
        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
        RxJavaPlugins.onError(e);
        NullPointerException npe = new NullPointerException("Actually not, but can't t
        npe.initCause(e);
        throw npe;
    }
}
```

其中最重要的方法 subscribeActual 调用的为 ObservableCreate 的 subscribeActual 方法，接下来和基本流程一样会调用 ObservableCreate 的 subscribe 从而开启事件的分发，与 Rxjava 基本流程不同的是 map 操作符构建了 MapObserver，完成 MapObserver 的相关操作后，才会最终调用自定义的 Observer 对象。


### 总结


如果把第一个构建的 Observable 标记为 A，把自定义的 Observer 标记为 Z，那么各种操作符会构建不同的 Observer 标记为 B、C、D ....,通过 subscribeActual 方法使 A、B、C、D ... 、Z 形成链式关系，最终由 Observable 对象 A 开启事件分发，将事件通过操作符定义的 Observer 对象 B、C、D ... 进行各自处理，最终传递到 Observer 对象 Z 中，这个事件流得以完成。

```
@Override
public void subscribeActual(Observer<? super U> t) {
    source.subscribe(new XxxxObserver<T, U>(t, function));
}
```




