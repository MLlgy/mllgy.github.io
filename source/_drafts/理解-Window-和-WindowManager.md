---
title: 理解 Window 和 WindowManager
tags:
---


### one
Window:桌面上类似浮动窗的东西。

Window 是一个抽象类，具体实现为 PhoneWindow.

使用 WindowManager 创建 Window 。
 

WindowManager 是外界访问 Window 的入口。

WindowManager 和 WindowManagerService 的交互是一个 IPC 过程。

Android 中所以的视图都是通过 Window 实现的：Activity、Dialog、Toast，都是依附在 Window 上，所以 Window 是 View 的直接管理者。


### two

Window 的 Type 属性表示 Window 的三种类型：

* 应用 Window(层级：1~99，对应 WindowManager.LayoutParams.type )
* 子 Window(不会单独存在，依附在特定父 Window 中，比如 Dialog)(层级：1000~1999)
* 系统 Window(Toast、系统状态栏)(层级：2000~2999)


### WindowManager

实现了 ViewManager

常用的方法：增加 View、更新 View、删除 View。

可以创建一个 Window，并向其中添加 View、更新 View、删除 View。


### Window 的内部机制

Window 是一个抽象的概念，**每一个 Window 都对应一个 View 和一个 ViewRootImpl**，**Window 和 View 之间通过 ViewRootImpl 来建立联系**，因此 **Window 是不存在的，以 View 的形式存在**。 
 

**Window 的添加过程**

`WindowManager` 只是一个接口，真正的实现为 `WindowManagerImpl，而` `WindowManagerImpl` 的操作通过 `WindowManagerGlobal` 代理类来完成。

`WindowManagerGlobal` 中 addView 的重要步骤：

1. 检查参数是否合法，如果是子 Window 那么需要调整一些 **布局参数**

2. **创建 ViewRootImpl** 并将 View 添加到列表中。

WindowManagerGlobal 中几个重要的集合：

```
//mViews 存储的是所有 Window 对应的 View
private final ArrayList<View> mViews = new ArrayList<View>();
//mRoots 存储的是所有 Window 对应的 ViewRootImpl
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
//mParams 存储的是所有 Window 对应的布局参数
private final ArrayList<WindowManager.LayoutParams> mParams =
        new ArrayList<WindowManager.LayoutParams>();
//mDyingViews 存储的是所有正在被删除的 View 对象
private final ArraySet<View> mDyingViews = new ArraySet<View>();
```

3. **由 ViewRootImpl 来更新界面**,并完成 Window 的添加过程。

调用过程：
`WindowManagerGlobal#addView` ->` ViewRootImpl#setView`

```
root.setView(view, wparams, panelParentView);
```
`ViewRootImpl#setView:`

```
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    ...
    // 更新界面
    requestLayout();
    ...
    // WindowSession 完成 Window 的添加过程，这是一个 IPC 过程
    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
}

public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        // View 的绘制入口
        scheduleTraversals();
    }
}
```

mWindowSession 的类型是 IWindowSession，它是一个 Binder 对象，真正的实现类是 Session，所以 Window 的添加过程是一次 IPC 调用。

在 Session 中会通过 WMS 来实现 Window 的添加过程，由 WMS 去处理，在 WMS 中为每一个应用保留一个 Session。

```
public int addToDisplay(IWindowwindow,intseq,WindowManager.LayoutParams attrs,int viewVisibility,int displayId,Rect outContentInsets,InputChannel outInputChannel){ 
    // WMS 中实现
    return mService.addWindow(this,window,seq,attrs,viewVisibility,displayId,outContentInsets,outInputChannel);
}
```

如此以来，Window 的添加请求就交给 WMS 处理了，在WMS 内部会为每一个应用保留一个 Session 对象，为每一个应用实现添加 Window 的功能。


至于 WMS 中如何实现 addWindow 同时也会是 IPC 过程，而且牵涉比较广，可以参见源码。

### Window 删除过程


WindowManagerImpl#WindowManagerGlobal#removeView -> ViewRootImpl#die -> ViewRootImpl#removeView(异步删除)/ViewRootImpl#removeViewImmediate(同步删除)-> doDie() -> dispatchDetachedFromWindow

dispatchDetachedFromWindow 的工作：

1. 垃圾回收工作：清除数据和消息、移除回调
2. 通过 Session 的 remove 方法删除 Window：IPC 过程，最终调用 WMS 的 removeWindow 方法。
3. dispatchDetachedFromWindow 中调用其他方法或回调，执行相应的回收资源操作。
4. 调用 WindowManagerGlobal 的 doRemoveView 方法刷新数据，更新 mRoot、mParams、mDyingViews，从中删除 Window 相应元素。


### Window 更新过程

WindowManagerGlobal#updateViewLayout 

1. 更新 View 的 LayoutParams并替换为老的 LayoutParams
2. 更新 ViewRootImpl 中的 LayoutParams
3. ViewRootParams 调用 scheduleTraversals 方法对 View 重新布局，同时执行 WMS 的 relayoutWindow 更新 Window 视图。


### Window 的创建过程



#### Activity 的 Window 创建过程（androidx1.0.2 版本代码）


* 第一阶段：Window 的创建

![](/source/images/2019_12_03_04.png)

* 第二阶段：Activity 视图是怎么关联到 Window 上的


在相应的 Activity 的 oncreate 方法中做如下调用：Activity#setContengView 

```
public void setContentView(@LayoutRes int layoutResID) {
    // 获得的 Window 实例为 PhoneWindow
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```
PhoneWindow#setContentView

```
    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```


---





在 Activity 中调用 setContentView(int resId) 后的调用关系如下：

![](/source/images/2019_12_03_05.png)

```
public void setContentView(int resId) {
    ensureSubDecor();
    ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mOriginalWindowCallback.onContentChanged();
}
```

`android.R.id.content` 标识的布局为 FrameLayout，通过 `LayoutInflater.from(mContext).inflate(resId, contentParent);` 的调用，最终 resId 所代表的布局被添加到 `android.R.id.content` 所标示的布局中，通过回调 Activity 的 onContentChanged 通知 Activity 视图发生了改变。

而在 ` ensureSubDecor();` 的调用中关联 Window 和 Activity 最外层的 ViewGroup，至此 Activity 的视图成功关联到 Window，关键代码为：

```
mWindow.setContentView(subDecor)
```

