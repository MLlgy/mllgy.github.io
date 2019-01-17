---
title: RxJava Transformer
date: 2019-01-17 17:26:13
tags: [RxJava, Transformer]
---




### 0x81 Observable、Observer 线程切换

在 Retrofit 结合 RxJava 进行开发时，我们可以通过 subscribeOn(...)、observeOn(...) 分别设置被订阅者和订阅者的线程。在此场景中，我们要在应用中的所有请求中执行上面操作，这下重复工作就需要 Transformer 来优化。


### 0x82 Transformer 实现对调用链的处理

我们可能有以下实现方式：

```
abstract class SchedulerTransformer<T> protected constructor(private val subscribeOnScheduler: Scheduler = Schedulers.io(), 
                                                      private val observeOnScheduler: Scheduler = AndroidSchedulers.mainThread()) : ObservableTransformer<T, T>,
        SingleTransformer<T, T>,
        MaybeTransformer<T, T>,
        CompletableTransformer,
        FlowableTransformer<T, T> {

    override fun apply(upstream: Completable): CompletableSource {
        return upstream.subscribeOn(subscribeOnScheduler)
                .observeOn(observeOnScheduler)
    }

    override fun apply(upstream: Flowable<T>): Publisher<T> {
        return upstream.subscribeOn(subscribeOnScheduler)
                .observeOn(observeOnScheduler)
    }

    override fun apply(upstream: Maybe<T>): MaybeSource<T> {
        return upstream.subscribeOn(subscribeOnScheduler)
                .observeOn(observeOnScheduler)
    }

    override fun apply(upstream: Observable<T>): ObservableSource<T> {
        return upstream.subscribeOn(subscribeOnScheduler)
                .observeOn(observeOnScheduler)
    }

    override fun apply(upstream: Single<T>): SingleSource<T> {
        return upstream.subscribeOn(subscribeOnScheduler)
                .observeOn(observeOnScheduler)
    }
}
```
<!--more-->
在具体业务中使用：

```
Observable.create(ObservableOnSubscribe<Int> { emitter -> emitter.onNext(1) }).compose(SchedulerTransformer(Schedulers.io(),AndroidSchedulers.mainThread())).subscribe(...);
```


### 0x83 RxLifeCycle2 中的实现

RxLifeCycle2 中使用相同的机制，在事件处理过程中对针对生命周期做出处理。

```
 //手动设置在activity的destroy中取消订阅,防止内存泄漏
    public static <T> ObservableTransformer<T, T> activityLifecycle(RxAppCompatActivity activity) {
        return observable ->
                observable.compose(activity.bindUntilEvent(ActivityEvent.DESTROY));
    }

    //手动设置在activity的destroy中取消订阅,防止内存泄漏
    public static <T> ObservableTransformer<T, T> activityLifecycle(RxAppCompatActivity activity, ActivityEvent event) {
        return observable ->
                observable.compose(activity.bindUntilEvent(event));
    }

    //手动设置在Fragment的destroy中取消订阅，防止内存泄漏
    public static <T> ObservableTransformer<T, T> fragmentLifecycle(RxFragment fragment) {
        return observable ->
                observable.compose(fragment.bindUntilEvent(FragmentEvent.DESTROY));
    }

```

### 0x84 在 Kotlin 中的实现


```
// SchedulersApply.kt
fun <T> transformerSchedluer(): ObservableTransformer<T, T> =
        ObservableTransformer { upstream -> upstream.subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread()) }
```

直接在调用链中使用：

```
obserable..compose(transformerSchedluer2()).subscribe(....);
```

### 0x85 compose

以上对 Observable 的变换最终插入调用链中，主要是因为 compose(...) 的作用。compose 操作符可以对调用链的原始 Observable 产生作用。

compose() 除了 实现上述对 Observable 进行变换外我们可以做一些其他处理，如 网络请求过程中 Dialog 的显示和隐藏。

```
  public static <T> ObservableTransformer<T, T> loadingDialog(BaseActivity activity, String message) {
        SpotsDialog dialog = DialogUtil.showLoadingDialog(activity, message);
        return observable -> observable
                .doOnSubscribe(disposable -> {
                    if (dialog != null) {
                        dialog.show();
                    }
                })
                .doOnComplete(() -> DialogUtil.dismiss(dialog))
                .doOnError(throwable -> DialogUtil.dismiss(dialog))
                .doOnNext(t -> DialogUtil.dismiss(dialog));
    }

    public static <T> ObservableTransformer<T, T> loadingDialog(BaseActivity activity) {
        return loadingDialog(activity, "");
    }
```