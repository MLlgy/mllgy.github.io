---
title: Glide4
tags:
---

https://mp.weixin.qq.com/s/el0IMGn75QtdPTBk7f5DbA




## Glide 初始化


Glide.with(context)

```
  public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
  }
```

getRetriever(activity)

1. 初始化 Glide
2. 初始化 Engine
3. 初始化 通过 GlideBuild 获得 Glide 实例
4. 获得 RequestManagerRetriever 对象


* 生命周期的感知

getRetriever(activity).get(activity)

```
  public RequestManager get(@NonNull FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      // 获得 FragmentManager 对象
      FragmentManager fm = activity.getSupportFragmentManager();
      return supportFragmentGet(
          activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }
```

```
private RequestManager supportFragmentGet(
    @NonNull Context context,
    @NonNull FragmentManager fm,
    @Nullable Fragment parentHint,
    boolean isParentVisible) {
  // 获得相应的 Fragment
  SupportRequestManagerFragment current =
      getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
  // 生成 RequestManager 对象
  RequestManager requestManager = current.getRequestManager();
  if (requestManager == null) {
    // TODO(b/27524013): Factor out this Glide.get() call.
    Glide glide = Glide.get(context);
    requestManager =
        factory.build(
            glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
    current.setRequestManager(requestManager);
  }
  return requestManager;
}
```

最终获得  RequestManager 对象。



### Glide是怎么依赖生命周期动态控制加载图片的呢 


在上面的步骤中，获得的 SupportRequestManagerFragment 对象中，存在 ActivityFragmentLifecycle 属性，在 SupportRequestManagerFragment 的生命周期均调用了 ActivityFragmentLifecycle 的对应方法。


SupportRequestManagerFragment
```
@Override
  public void onStart() {
    super.onStart();
    lifecycle.onStart();
  }

  @Override
  public void onStop() {
    super.onStop();
    lifecycle.onStop();
  }
```

ActivityFragmentLifecycle#onStart
```
void onStart() {
    isStarted = true;
    for (LifecycleListener lifecycleListener : Util.getSnapshot(lifecycleListeners)) {
      lifecycleListener.onStart();
    }
  }
````

而 RequestManager，TargetTracker和Target（继承）实现了该接口，以 

```
 @Override
  public void onStart() {
    resumeRequests();
    targetTracker.onStart();
  }
```


因为 SupportRequestManagerFragment 是根据所在页面传入的 Context 生成的，Fragment 可以感知所在 Activity 的生命周期，最后通过requestTracker循环找到对应的Request对象，然后调用对应的处理方法从而达到了根据生命周期动态控制图片加载的目的，从而实现 Glide 的生命周期感知。



### 以上获得了 RequestManager、Glide、Engine 实例对象


Glide.with(context)  获得了 RequestManager 实例变量，而 Glide.with(context).load("") 就相当于 RequestManager.load("")。


RequestManager.load("")：

1. 初始化 RequestBuilder 对象，并在初始化过程中初始化 RequestListeners
2. 调用了 RequestBuilder.load("") 方法。

### Glide.with(context).load("").into(mImageView)

标题最终实现了 RequestBuilder.into(mImageView);

 RequestBuilder#into(mImageView)
```
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    Util.assertMainThread();
    Preconditions.checkNotNull(view);

    RequestOptions requestOptions = this.requestOptions;
    if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      // Clone in this method so that if we use this RequestBuilder to load into a View and then
      // into a different target, we don't retain the transformation applied based on the previous
      // View's scale type.
      //  根据 ImageView 的 ScaleType 进行相应的设置
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          requestOptions = requestOptions.clone().optionalFitCenter();
          break;
        case FIT_XY:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case CENTER:
        case MATRIX:
        default:
          // Do nothing.
      }
    }

    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions);
  }
```

### 缓存的读取

缓存分为弱引用缓存和Lru 缓存。


SingleRequest.begin


