---
title: Diaglog 源码解析
tags:
---


### 0x0001 创建 Dialog 

```
Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
    // 传入为 true
    if (createContextThemeWrapper) {
        if (themeResId == ResourceId.ID_NULL) {
            final TypedValue outValue = new TypedValue();
            context.getTheme().resolveAttribute(R.attr.dialogTheme, outValue, true);
            themeResId = outValue.resourceId;
        }
        // 为 Dialog 创建 Context 对象
        mContext = new ContextThemeWrapper(context, themeResId);
    } else {
        mContext = context;
    }
    // 获取 WindowManagerImpl 对象，见 0x0002 
    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    // 创建 Window 对象
    final Window w = new PhoneWindow(mContext);
    mWindow = w;
    w.setCallback(this);
    w.setOnWindowDismissedCallback(this);
    w.setOnWindowSwipeDismissedCallback(() -> {
        if (mCancelable) {
            cancel();
        }
    });
    // 为 window 设置 wms 对象
    w.setWindowManager(mWindowManager, null, null);
    w.setGravity(Gravity.CENTER);
    mListenersHandler = new ListenersHandler(this);
}
```



### 0x0002 getSystemService()


*/frameworks/base/core/java/android/app/ContextImpl.java*
```
@Override
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name);
}
```

*/frameworks/base/core/java/android/app/SystemServiceRegistry.java*


```
static {
    ......

    registerService(Context.WINDOW_SERVICE, WindowManager.class,new CachedServiceFetcher<WindowManager>() {
        @Override
        public WindowManager createService(ContextImpl ctx) {
            return new WindowManagerImpl(ctx);
        }});
    ......

}
```
```
private static <T> void registerService(String serviceName, Class<T> serviceClass,ServiceFetcher<T> serviceFetcher) {
    SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
    SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
}
```


```
public static Object getSystemService(ContextImpl ctx, String name) {
    ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
    return fetcher != null ? fetcher.getService(ctx) : null;
}
```

通过以上步骤，获得 WindowManagerImpl 实例对象。


### 0x0003 show()


```
public void show() {
    // mShowing 为 true ，表示正在展示
    if (mShowing) {
        if (mDecor != null) {
            if (mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
                mWindow.invalidatePanelMenu(Window.FEATURE_ACTION_BAR);
            }
            mDecor.setVisibility(View.VISIBLE);
        }
        return;
    }
    mCanceled = false;
    // 是否已经创建
    if (!mCreated) {
        dispatchOnCreate(null);
    } else {
        // 由于屏幕配置信息容易发生变化，所以需要更新 Window 的配置信息
        mWindow.getDecorView().dispatchConfigurationChanged(config);
    }
    onStart();
    // 获得 DecorView
    mDecor = mWindow.getDecorView();
    ....
    WindowManager.LayoutParams l = mWindow.getAttributes();
    ....
    // 见 0x00004 
    mWindowManager.addView(mDecor, l);
    if (restoreSoftInputMode) {
        l.softInputMode &=
                ~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
    }
    mShowing = true;
    sendShowMessage();
}
```


### 0x0x004 WindowManagerImpl#addView

```
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

### 0x00005 WindowManagerGlobal#

```
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        //如果 parentWindow 为 null，应用会设置该 view 硬件加速
        final Context context = view.getContext();
        if (context != null
                && (context.getApplicationInfo().flags
                        & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }
    ViewRootImpl root;
    View panelParentView = null;
    // 同步锁
    synchronized (mLock) {
        // Start watching for system property changes.
        if (mSystemPropertyUpdater == null) {
            mSystemPropertyUpdater = new Runnable() {
                @Override public void run() {
                    synchronized (mLock) {
                        for (int i = mRoots.size() - 1; i >= 0; --i) {
                            mRoots.get(i).loadSystemProperties();
                        }
                    }
                }
            };
            SystemProperties.addChangeCallback(mSystemPropertyUpdater);
        }
        int index = findViewLocked(view, false);
        .....
        .....
        // 新建 ViewRootImpl 对象
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        // 将 view、viewrootimpl、wparams 添加到相应集合中，本地保存
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
        // do this last because it fires off messages to start doing things
        // 最后执行该操作，因为它会发送信息去执行相应操作
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            if (index >= 0) {
                removeViewLocked(index, true);
            }
            throw e;
        }
    }
}
```

### 0x00006 ViewRootImpl#setView

```
 public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
                ......
                mWindowAttributes.copyFrom(attrs);
                ......
                try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
                    // 关于 mWindowSession 的获取，可以查看 0x0007
                    // addToDisplay，见 0x0009
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
                } catch (RemoteException e) {
                    ......
                    throw new RuntimeException("Adding window failed", e);
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }
                ......
                ......
                
                if (DEBUG_LAYOUT) Log.v(mTag, "Added window " + mWindow);
                if (res < WindowManagerGlobal.ADD_OKAY) {
                    mAttachInfo.mRootView = null;
                    mAdded = false;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    switch (res) {
                        case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                        case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- token " + attrs.token
                                    + " is not valid; is your activity running?");
                        case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- token " + attrs.token
                                    + " is not for an application");
                        // 省略其他 case
                    }
                    throw new RuntimeException(
                            "Unable to add window -- unknown error code " + res);
                }
                .......
                // 初始化屏幕输入系统
                CharSequence counterSuffix = attrs.getTitle();
                mSyntheticInputStage = new SyntheticInputStage();
                InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
                InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                        "aq:native-post-ime:" + counterSuffix);
                InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
                InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                        "aq:ime:" + counterSuffix);
                InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
                InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                        "aq:native-pre-ime:" + counterSuffix);

                mFirstInputStage = nativePreImeStage;
                mFirstPostImeInputStage = earlyPostImeStage;
                mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
            }
        }
    }


```

### 0x0007 ViewRootImpl$mWindowSession

```
final IWindowSession mWindowSession;
public ViewRootImpl(Context context, Display display) {
    mWindowSession = WindowManagerGlobal.getWindowSession();
}
```
*WindowManagerGlobal.getWindowSession()*

```
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                InputMethodManager imm = InputMethodManager.getInstance();
                // 获取 system_server 进程中的 WMS 对象，具体 getWindowManagerService 
                IWindowManager windowManager = getWindowManagerService();
                // 
                sWindowSession = windowManager.openSession(
                        new IWindowSessionCallback.Stub() {
                            @Override
                            public void onAnimatorScaleChanged(float scale) {
                                ValueAnimator.setDurationScale(scale);
                            }
                        },
                        imm.getClient(), imm.getInputContext());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowSession;
    }
}


public static IWindowManager getWindowManagerService() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowManagerService == null) {
            // 跨进程获得 system_server 进程中的 WMS 对象
            sWindowManagerService = IWindowManager.Stub.asInterface(
                    ServiceManager.getService("window"));
            try {
                if (sWindowManagerService != null) {
                    ValueAnimator.setDurationScale(
                            sWindowManagerService.getCurrentAnimatorScale());
                }
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowManagerService;
    }
}
```

可以看到关于 Session 的获取，是需要通过跨进程首先获得 WMS 的实例，然后获得 Session 实例并返回。

### 0x0008 WMS#openSession


```
@Override
public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
        IInputContext inputContext) {
    if (client == null) throw new IllegalArgumentException("null client");
    if (inputContext == null) throw new IllegalArgumentException("null inputContext");
    Session session = new Session(this, callback, client, inputContext);
    return session;
}
```


### 0x0009 Session#addToDisplay

[Session 源码](http://androidxref.com/8.1.0_r33/xref/frameworks/base/services/core/java/com/android/server/wm/Session.java#mService)

```
 @Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
        Rect outOutsets, InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
            outContentInsets, outStableInsets, outOutsets, outInputChannel);
}
```

*WMS.addWindow*

```
public int addWindow(Session session, IWindow client, int seq,
        WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
        Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
        InputChannel outInputChannel) {
    int[] appOp = new int[1];
    int res = mPolicy.checkAddPermission(attrs, appOp);
    if (res != WindowManagerGlobal.ADD_OKAY) {
        return res;
    }
    // 此处的 attr 为调用 addView 方法时的属性值 Window.LayoutParam 
    final int type = attrs.type;
    final DisplayContent displayContent = mRoot.getDisplayContentOrCreate(displayId);
    if (displayContent == null) {
       Slog.w(TAG_WM, "Attempted to add window to a display that does not exist: "
               + displayId + ".  Aborting.");
       return WindowManagerGlobal.ADD_INVALID_DISPLAY;
    }

    ....
    ....

    // 检测 Window 属性
     if (type >= FIRST_SUB_WINDOW && type <= LAST_SUB_WINDOW) {
        parentWindow = windowForClientLocked(null, attrs.token, false);
        if (parentWindow == null) {
            Slog.w(TAG_WM, "Attempted to add window with token that is not a window: "
                  + attrs.token + ".  Aborting.");
            return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
        }
        if (parentWindow.mAttrs.type >= FIRST_SUB_WINDOW
                && parentWindow.mAttrs.type <= LAST_SUB_WINDOW) {
            Slog.w(TAG_WM, "Attempted to add window with token that is a sub-window: "
                    + attrs.token + ".  Aborting.");
            return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
        }
    }

    AppWindowToken atoken = null;
    final boolean hasParent = parentWindow != null;
    // Use existing parent window token for child windows since they go in the same token
    // as there parent window so we can apply the same policy on them.

    // 子窗口和父窗口拥有相同的 token，所以可以复用父窗口的 token，并且它们可以使用相同的协议。
    WindowToken token = displayContent.getWindowToken(
              hasParent ? parentWindow.mAttrs.token : attrs.token);
    // If this is a child window, we want to apply the same type checking rules as the
    // parent window type.
    // 如果该窗口有父窗口，那么子窗口会采用和父窗口相同的检测规则。
    final int rootType = hasParent ? parentWindow.mAttrs.type : type;
    boolean addToastWindowRequiresToken = false;
    if (token == null) {
        if (rootType >= FIRST_APPLICATION_WINDOW && rootType <= LAST_APPLICATION_WINDOW) {
            Slog.w(TAG_WM, "Attempted to add application window with unknown token "
                  + attrs.token + ".  Aborting.");
            return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
        }

        ...
    }

```


这个方法中，包含添加 Window 的逻辑，牵涉多个方面，值的仔细探究，但是不在本篇研究范围内，有机会再回来。最终根据不同情况，返回类似于 ` WindowManagerGlobal.ADD_BAD_APP_TOKEN` 的 int 值，这时候就可以返回 0x00006 了，根据返回值，进行接下来的判断，有可能抛出如下异常：

*ViewRootImpl#setView*

```
switch (res) {
    case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
    case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
        throw new WindowManager.BadTokenException(
                "Unable to add window -- token " + attrs.token
                + " is not valid; is your activity running?");
    case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
        throw new WindowManager.BadTokenException(
                "Unable to add window -- token " + attrs.token
                + " is not for an application");
    // 省略其他 case
}
```

至此将 Dialog 添加 Window，并将 Window 添加到屏幕的大致流程梳理完成，不过这也只是 Android 显示系统中的冰山一角。

### 关于该过程中的一些异常




---





https://www.codercto.com/a/33137.html


[Android Window的添加和显示过程](https://blog.csdn.net/u011228598/article/details/86499975)