---
title: Activity 启动流程
tags:
---

Activity启动发起后，通过Binder最终交由system进程中的AMS来完成，则启动流程如下图：

# Activity


## startActivity

```
public void startActivity(Intent intent) {
    this.startActivity(intent, (Bundle)null);
}
public void startActivity(Intent intent, Bundle options) {
    if (options != null) {
        this.startActivityForResult(intent, -1, options);
    } else {
        this.startActivityForResult(intent, -1);
    }
}
```


## startActivityForResult

```
    public void startActivityForResult(Intent intent, int requestCode, Bundle options) {
        if (this.mParent == null) {
            ActivityResult ar = this.mInstrumentation.execStartActivity(this, this.mMainThread.getApplicationThread(), this.mToken, this, intent, requestCode, options);
            if (ar != null) {
                this.mMainThread.sendActivityResult(this.mToken, this.mEmbeddedID, requestCode, ar.getResultCode(), ar.getResultData());
            }

            if (requestCode >= 0) {
                this.mStartedActivity = true;
            }
        } else if (options != null) {
            this.mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            this.mParent.startActivityFromChild(this, intent, requestCode);
        }

    }
```

execStartActivity 主要参数：

* this.mMainThread.getApplicationThread()

```
final ActivityThread.ApplicationThread mAppThread = new ActivityThread.ApplicationThread();
public ActivityThread.ApplicationThread getApplicationThread() {
    return this.mAppThread;
}

```
* this.mToken

```
private IBinder mToken;
```

## Instrumentation#execStartActivity

```
public Instrumentation.ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target, Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread)contextThread;
    if (this.mActivityMonitors != null) {
        synchronized(this.mSync) {
            int N = this.mActivityMonitors.size();
            for(int i = 0; i < N; ++i) {
                Instrumentation.ActivityMonitor am = (Instrumentation.ActivityMonitor)this.mActivityMonitors.get(i);
                if (am.match(who, (Activity)null, intent)) {
                    ++am.mHits;
                    // 
                    if (am.isBlocking()) {
                        return requestCode >= 0 ? am.getResult() : null;
                    }
                    break;
                }
            }
        }
    }
    try {
        intent.setAllowFds(false);
        intent.migrateExtraStreamToClipData();
        int result = ActivityManagerNative.getDefault().startActivity(whoThread, intent, intent.resolveTypeIfNeeded(who.getContentResolver()), token, target != null ? target.mEmbeddedID : null, requestCode, 0, (String)null, (ParcelFileDescriptor)null, options);
        // 检查 activity 是否启动成功
        checkStartActivityResult(result, intent);
    } catch (RemoteException var14) {
    }
    return null;
}
```

核心代码：

```
int result = ActivityManagerNative.getDefault().startActivity(whoThread, intent, intent.resolveTypeIfNeeded(who.getContentResolver()), token, target != null ? target.mEmbeddedID : null, requestCode, 0, (String)null, (ParcelFileDescriptor)null, options);
```

具体 `ActivityManagerNative.getDefault()` 方法如下：

```
public static IActivityManager getDefault() {
    return (IActivityManager)gDefault.get();
}

private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");
        IActivityManager am = ActivityManagerNative.asInterface(b);
        return am;
    }
};
```

```
public static IActivityManager asInterface(IBinder obj) {
    if (obj == null) {
        return null;
    } else {
        IActivityManager in = (IActivityManager)obj.queryLocalInterface("android.app.IActivityManager");
        return (IActivityManager)(in != null ? in : new ActivityManagerProxy(obj));
    }
}
```
其中的单例的实现：

```
public abstract class Singleton<T> {
    private T mInstance;

    public Singleton() {
    }

    protected abstract T create();

    public final T get() {
        synchronized(this) {
            if (this.mInstance == null) {
                this.mInstance = this.create();
            }

            return this.mInstance;
        }
    }
}
```

可见 `ActivityManagerNative.getDefault()` 返回对象 ActivityManagerProxy。


> 这里的疑问？ 为什么是 ActivityManagerProxy 而不是 ActivityManagerNative


## ActivityManagerProxy#startActivity

```
public int startActivity(IApplicationThread caller, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, String profileFile, ParcelFileDescriptor profileFd, Bundle options) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken("android.app.IActivityManager");
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    intent.writeToParcel(data, 0);
    data.writeString(resolvedType);
    data.writeStrongBinder(resultTo);
    data.writeString(resultWho);
    data.writeInt(requestCode);
    data.writeInt(startFlags);
    data.writeString(profileFile);
    if (profileFd != null) {
        data.writeInt(1);
        profileFd.writeToParcel(data, 1);
    } else {
        data.writeInt(0);
    }
    if (options != null) {
        data.writeInt(1);
        options.writeToParcel(data, 0);
    } else {
        data.writeInt(0);
    }
    this.mRemote.transact(3, data, reply, 0);
    reply.readException();
    int result = reply.readInt();
    reply.recycle();
    data.recycle();
    return result;
}
```

AMP 经过 binder ipc 后，进入 ActivityManagerNative（AMN），接下来的程序就进入了 system_server 进程，继续执行接下来的步骤。

通过 AIDL 的相关知识，那么接下来执行的方法为：AMN#onTransact，具体可以参见：[AIDL 浅析]()。


## AMN#onTransact

```
// 传入的 code 为 3 
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    switch(code) {
        case 3:
            data.enforceInterface("android.app.IActivityManager");
            token = data.readStrongBinder();
            app = ApplicationThreadNative.asInterface(token);
            intent = (Intent)Intent.CREATOR.createFromParcel(data);
            resultWho = data.readString();
            resultTo = data.readStrongBinder();
            resultWho = data.readString();
            i = data.readInt();
            fl = data.readInt();
            profileFile = data.readString();
            profileFd = data.readInt() != 0 ? data.readFileDescriptor() : null;
            options = data.readInt() != 0 ? (Bundle)Bundle.CREATOR.createFromParcel(data) : null;
            result = this.startActivity(app, intent, resultWho, resultTo, resultWho, i, fl, profileFile, profileFd, options);
            reply.writeNoException();
            reply.writeInt(result);
            return true;
    }
    ...
}
```

最终会调用 AMS 的 startActivity

## AMS#startActivity

```
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}
```

## AMS#startActivityAsUser

```
@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
            true /*validateIncomingUser*/);
}
```


```
public final int startActivityAsUser(IApplicationThread caller, String calling
        Intent intent, String resolvedType, IBinder resultTo, String resultWho
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId
        boolean validateIncomingUser) {
    enforceNotIsolatedCaller("startActivity");
    userId = mActivityStartController.checkTargetUser(userId, validateIncoming
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUs
    // TODO: Switch to user app stacks here.
    return mActivityStartController.obtainStarter(intent, "startActivityAsUser
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            // 设置 mRequest.mayWait = true，在下一步执行中有重要作用
            .setMayWait(userId)
            // 构建 ActivityStart 对象，执行 execute 方法
            .execute();
}
```

## ActivityStart#execute
```
int execute() {
    try {
        // TODO(b/64750076): Look into passing request directly to these methods to allow
        // for transactional diffs and preprocessing.
        // 在上一步中设置 mRequest.mayWait = true
        if (mRequest.mayWait) {
            return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                    mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                    mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                    mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                    mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup);
        } else {
            return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                    mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                    mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                    mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                    mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                    mRequest.outActivity, mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup);
        }
    } finally {
        onExecutionComplete();
    }
```

## ActivityStarter#startActivityMayWait

```
private int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, Intent intent, String resolvedType,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, WaitResult outResult,
        Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
        int userId, TaskRecord inTask, String reason,
        boolean allowPendingRemoteAnimationRegistryLookup) {
    // 创建新的 Intent 对象，避免修改原始 Intent 对象被修改
    // Save a copy in case ephemeral needs it
    final Intent ephemeralIntent = new Intent(intent);
    // Don't modify the client's object!
    intent = new Intent(intent);

    //收集Intent所指向的Activity信息, 当存在多个可供选择的Activity,则直接向用户弹出resolveActivity
    ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId,
            0 /* matchFlags */,
                    computeResolveFilterUid(
                            callingUid, realCallingUid, mRequest.filterCallingUid));
    // Collect information about the target of the Intent.
    // 收集关于目标 Intent 的信息
    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);
    synchronized (mService) {

        final long origId = Binder.clearCallingIdentity();
        if (aInfo != null &&
                (aInfo.applicationInfo.privateFlags
                        & ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0 &&
                mService.mHasHeavyWeightFeature) {
            // This may be a heavy-weight process!  Check to see if we already
            // have another, different heavy-weight process running.
            //  heavy-weight 进程的处理流程
            if (aInfo.processName.equals(aInfo.applicationInfo.packageName)) {
               ....
            }
        }
        final ActivityRecord[] outRecord = new ActivityRecord[1];
        // 执行下一步
        int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                allowPendingRemoteAnimationRegistryLookup);
        Binder.restoreCallingIdentity(origId);
        if (stack.mConfigWillChange) {
           // 不会进入该分支
        }
        if (outResult != null) {
            // 不会进入该分支
        }
        return res;
    }
}
```

该过程主要功能：通过resolveActivity来获取ActivityInfo信息, 然后再进入ASS.startActivityLocked().


## ActivityStackSupervisor#resolveIntent

```
// startFlags = 0; profilerInfo = null; userId代表caller UserId
ActivityInfo resolveActivity(Intent intent, String resolvedType, int startFlags, ProfilerInfo profilerInfo, int userId) {
    ActivityInfo aInfo;// Note: This method should only be called from {@link startActivity}.
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity) {
    setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
            voiceInteractor);
    computeLaunchingTaskFlags();
    computeSourceStack();
    mIntent.setFlags(mLaunchFlags);
    ActivityRecord reusedActivity = getReusableIntentActivity();
    int preferredWindowingMode = WINDOWING_MODE_UNDEFINED;
    int preferredLaunchDisplayId = DEFAULT_DISPLAY;
    if (mOptions != null) {
        preferredWindowingMode = mOptions.getLaunchWindowingMode();
        preferredLaunchDisplayId = mOptions.getLaunchDisplayId();
    }
    // windowing mode and preferred launch display values from {@link LaunchParams} take
    // priority over those specified in {@link ActivityOptions}.
    if (!mLaunchParams.isEmpty()) {
        if (mLaunchParams.hasPreferredDisplay()) {
            preferredLaunchDisplayId = mLaunchParams.mPreferredDisplayId;
        }
        if (mLaunchParams.hasWindowingMode()) {
            preferredWindowingMode = mLaunchParams.mWindowingMode;
        }
    }
    if (reusedActivity != null) {
        // When the flags NEW_TASK and CLEAR_TASK are set, then the task gets reused but
        // still needs to be a lock task mode violation since the task gets cleared out and
        // the device would otherwise leave the locked task.
        if (mService.getLockTaskController().isLockTaskModeViolation(reusedActivity.getTask
                (mLaunchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                        == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))) {
            Slog.e(TAG, "startActivityUnchecked: Attempt to violate Lock Task Mode");
            return START_RETURN_LOCK_TASK_MODE_VIOLATION;
        }
        // True if we are clearing top and resetting of a standard (default) launch mode
        // ({@code LAUNCH_MULTIPLE}) activity. The existing activity will be finished.
        final boolean clearTopAndResetStandardLaunchMode =
                (mLaunchFlags & (FLAG_ACTIVITY_CLEAR_TOP | FLAG_ACTIVITY_RESET_TASK_IF_NEED
                        == (FLAG_ACTIVITY_CLEAR_TOP | FLAG_ACTIVITY_RESET_TASK_IF_NEEDED)
                && mLaunchMode == LAUNCH_MULTIPLE;
        // If mStartActivity does not have a task associated with it, associate it with the
        // reused activity's task. Do not do so if we're clearing top and resetting for a
        // standard launchMode activity.
        if (mStartActivity.getTask() == null && !clearTopAndResetStandardLaunchMode) {
            mStartActivity.setTask(reusedActivity.getTask());
        }
        if (reusedActivity.getTask().intent == null) {
            // This task was started because of movement of the activity based on affinity.
            // Now that we are actually launching it, we can assign the base intent.
            reusedActivity.getTask().setIntent(mStartActivity);
        }
        // This code path leads to delivering a new intent, we want to make sure we schedul
        // as the first operation, in case the activity will be resumed as a result of late
        // operations.
        if ((mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                || isDocumentLaunchesIntoExisting(mLaunchFlags)
                || isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
            final TaskRecord task = reusedActivity.getTask();
            // In this situation we want to remove all activities from the task up to the o
            // being started. In most cases this means we are resetting the task to its ini
            // state.
            final ActivityRecord top = task.performClearTaskForReuseLocked(mStartActivity,
                    mLaunchFlags);
            // The above code can remove {@code reusedActivity} from the task, leading to t
            // the {@code ActivityRecord} removing its reference to the {@code TaskRecord}.
            // task reference is needed in the call below to
            // {@link setTargetStackAndMoveToFrontIfNeeded}.
            if (reusedActivity.getTask() == null) {
                reusedActivity.setTask(task);
            }
            if (top != null) {
                if (top.frontOfTask) {
                    // Activity aliases may mean we use different intents for the top activ
                    // so make sure the task now has the identity of the new intent.
                    top.getTask().setIntent(mStartActivity);
                }
                deliverNewIntent(top);
            }
        }
        mSupervisor.sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, reusedActivi
        reusedActivity = setTargetStackAndMoveToFrontIfNeeded(reusedActivity);
        final ActivityRecord outResult =
                outActivity != null && outActivity.length > 0 ? outActivity[0] : null;
        // When there is a reused activity and the current result is a trampoline activity,
        // set the reused activity as the result.
        if (outResult != null && (outResult.finishing || outResult.noDisplay)) {
            outActivity[0] = reusedActivity;
        }
        if ((mStartFlags & START_FLAG_ONLY_IF_NEEDED) != 0) {
            // We don't need to start a new activity, and the client said not to do anythin
            // if that is the case, so this is it!  And for paranoia, make sure we have
            // correctly resumed the top activity.
            resumeTargetStackIfNeeded();
            return START_RETURN_INTENT_TO_CALLER;
        }
        if (reusedActivity != null) {
            setTaskFromIntentActivity(reusedActivity);
            if (!mAddingToTask && mReuseTask == null) {
                // We didn't do anything...  but it was needed (a.k.a., client don't use th
                // intent!)  And for paranoia, make sure we have correctly resumed the top 
                resumeTargetStackIfNeeded();
                if (outActivity != null && outActivity.length > 0) {
                    outActivity[0] = reusedActivity;
                }
                return mMovedToFront ? START_TASK_TO_FRONT : START_DELIVERED_TO_TOP;
            }
        }
    }
    if (mStartActivity.packageName == null) {
        final ActivityStack sourceStack = mStartActivity.resultTo != null
                ? mStartActivity.resultTo.getStack() : null;
        if (sourceStack != null) {
            sourceStack.sendActivityResultLocked(-1 /* callingUid */, mStartActivity.result
                    mStartActivity.resultWho, mStartActivity.requestCode, RESULT_CANCELED,
                    null /* data */);
        }
        ActivityOptions.abort(mOptions);
        return START_CLASS_NOT_FOUND;
    }
    // If the activity being launched is the same as the one currently at the top, then
    // we need to check if it should only be launched once.
    final ActivityStack topStack = mSupervisor.mFocusedStack;
    final ActivityRecord topFocused = topStack.getTopActivity();
    final ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(mNotTop);
    final boolean dontStart = top != null && mStartActivity.resultTo == null
            && top.realActivity.equals(mStartActivity.realActivity)
            && top.userId == mStartActivity.userId
            && top.app != null && top.app.thread != null
            && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
            || isLaunchModeOneOf(LAUNCH_SINGLE_TOP, LAUNCH_SINGLE_TASK));
    if (dontStart) {
        // For paranoia, make sure we have correctly resumed the top activity.
        topStack.mLastPausedActivity = null;
        if (mDoResume) {
            mSupervisor.resumeFocusedStackTopActivityLocked();
        }
        ActivityOptions.abort(mOptions);
        if ((mStartFlags & START_FLAG_ONLY_IF_NEEDED) != 0) {
            // We don't need to start a new activity, and the client said not to do
            // anything if that is the case, so this is it!
            return START_RETURN_INTENT_TO_CALLER;
        }
        deliverNewIntent(top);
        // Don't use mStartActivity.task to show the toast. We're not starting a new activi
        // but reusing 'top'. Fields in mStartActivity may not be fully initialized.
        mSupervisor.handleNonResizableTaskIfNeeded(top.getTask(), preferredWindowingMode,
                preferredLaunchDisplayId, topStack);
        return START_DELIVERED_TO_TOP;
    }
    boolean newTask = false;
    final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
            ? mSourceRecord.getTask() : null;
    // Should this be considered a new task?
    int result = START_SUCCESS;
    if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
            && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
        newTask = true;
        result = setTaskFromReuseOrCreateNewTask(taskToAffiliate, topStack);
    } else if (mSourceRecord != null) {
        result = setTaskFromSourceRecord();
    } else if (mInTask != null) {
        result = setTaskFromInTask();
    } else {
        // This not being started from an existing activity, and not part of a new task...
        // just put it in the top task, though these days this case should never happen.
        setTaskToCurrentTopOrCreateNewTask();
    }
    if (result != START_SUCCESS) {
        return result;
    }
    mService.grantUriPermissionFromIntentLocked(mCallingUid, mStartActivity.packageName,
            mIntent, mStartActivity.getUriPermissionsLocked(), mStartActivity.userId);
    mService.grantEphemeralAccessLocked(mStartActivity.userId, mIntent,
            mStartActivity.appInfo.uid, UserHandle.getAppId(mCallingUid));
    if (newTask) {
        EventLog.writeEvent(EventLogTags.AM_CREATE_TASK, mStartActivity.userId,
                mStartActivity.getTask().taskId);
    }
    ActivityStack.logStartActivity(
            EventLogTags.AM_CREATE_ACTIVITY, mStartActivity, mStartActivity.getTask());
    mTargetStack.mLastPausedActivity = null;
    mSupervisor.sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, mStartActivity);
    mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransitio
            mOptions);
    if (mDoResume) {
        final ActivityRecord topTaskActivity =
                mStartActivity.getTask().topRunningActivityLocked();
        if (!mTargetStack.isFocusable()
                || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                && mStartActivity != topTaskActivity)) {
            // If the activity is not focusable, we can't resume it, but still would like t
            // make sure it becomes visible as it starts (this will also trigger entry
            // animation). An example of this are PIP activities.
            // Also, we don't want to resume activities in a task that currently has an ove
            // as the starting activity just needs to be in the visible paused state until 
            // over is removed.
            mTargetStack.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
            // Go ahead and tell window manager to execute app transition for this activity
            // since the app transition will not be triggered through the resume channel.
            mService.mWindowManager.executeAppTransition();
        } else {
            // If the target stack was not previously focusable (previous top running activ
            // on that stack was not visible) then any prior calls to move the stack to the
            // will not update the focused stack.  If starting the new activity now allows 
            // task stack to be focusable, then ensure that we now update the focused stack
            // accordingly.
            if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                mTargetStack.moveToFront("startActivityUnchecked");
            }
            mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                    mOptions);
        }
    } else if (mStartActivity != null) {
        mSupervisor.mRecentTasks.add(mStartActivity.getTask());
    }
    mSupervisor.updateUserStackLocked(mStartActivity.userId, mTargetStack);
    mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTask(), preferredWindowing
            preferredLaunchDisplayId, mTargetStack);
    return START_SUCCESS;
}
    try {
        // 执行下一步
        ResolveInfo rInfo = AppGlobals.getPackageManager().resolveIntent(intent, resolvedType, 66560, userId);
        aInfo = rInfo != null ? rInfo.activityInfo : null;
    } catch (RemoteException var8) {
        aInfo = null;
    }
    if (aInfo != null) {
        intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
        //对于非system进程，根据flags来设置相应的信息
        if ((startFlags & 2) != 0 && !aInfo.processName.equals("system")) {
            this.mService.setDebugApp(aInfo.processName, true, false);
        }
        if ((startFlags & 4) != 0 && !aInfo.processName.equals("system")) {
            this.mService.setOpenGlTraceApp(aInfo.applicationInfo, aInfo.processName);
        }
        if (profilerInfo != null && !aInfo.processName.equals("system")) {
            this.mService.setProfileApp(aInfo.applicationInfo, aInfo.processName, profilerInfo);
        }
    }
    return aInfo;
}
```

## PackageManagerService#resolveIntent

AppGlobals.getPackageManager()经过函数层层调用，获取的是ApplicationPackageManager对象。经过binder IPC调用，最终会调用PackageManagerService对象。故此时调用方法为PMS.resolveIntent().


```
public ResolveInfo resolveIntent(Intent intent, String resolvedType,
        int flags, int userId) {
    return resolveIntentInternal(intent, resolvedType, flags, userId, false,
            Binder.getCallingUid());
}
```


## resolveIntentInternal

```
/**
 * Normally instant apps can only be resolved when they're visible to the caller.
 * However, if {@code resolveForStart} is {@code true}, all instant apps are visible
 * since we need to allow the system to start any installed application.
 */
// 通常的，仅当安装的 App 对调用方是可见的，那么该 App 才是可以被解析的。
// 但是如果 resolveForStart 为 true，那么所以安装的 App 对它都是可见的，所以系统 App
// 才可以启动任何已安装的 App。
private ResolveInfo resolveIntentInternal(Intent intent, String resolvedType,
        int flags, int userId, boolean resolveForStart, int filterCallingUid) {
    try {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "resolveIntent");
        if (!sUserManager.exists(userId)) return null;
        final int callingUid = Binder.getCallingUid();
        flags = updateFlagsForResolve(flags, userId, intent, filterCallingUid, resolveForStart);
        mPermissionManager.enforceCrossUserPermission(callingUid, userId,
                false /*requireFullPermission*/, false /*checkShell*/, "resolve intent");
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "queryIntentActivities");
        // 获得相应的 Activity 组件，并保存在 Intent 中
        final List<ResolveInfo> query = queryIntentActivitiesInternal(intent, resolvedType,
                flags, filterCallingUid, userId, resolveForStart, true /*allowDynamicSplits*/);
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        final ResolveInfo bestChoice =
                chooseBestActivity(intent, resolvedType, flags, query, userId);
        return bestChoice;
    } finally {
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
}
```

```
private @NonNull List<ResolveInfo> queryIntentActivitiesInternal(Intent intent,
        String resolvedType, int flags, int filterCallingUid, int userId,
        boolean resolveForStart, boolean allowDynamicSplits) {
    
    ....
    final String pkgName = intent.getPackage();
    ComponentName comp = intent.getComponent();
    if (comp == null) {
        if (intent.getSelector() != null) {
            intent = intent.getSelector();
            comp = intent.getComponent();
        }
    }
    flags = updateFlagsForResolve(flags, userId, intent, filterCallingUid, resolveForStart,
            comp != null || pkgName != null /*onlyExposedExplicitly*/);
    // comp 不为 null
    if (comp != null) {
        final List<ResolveInfo> list = new ArrayList<ResolveInfo>(1);
        final ActivityInfo ai = getActivityInfo(comp, flags, userId);
        if (ai != null) {
            if (!blockResolution) {
                final ResolveInfo ri = new ResolveInfo();
                ri.activityInfo = ai;
                list.add(ri);
            }
        }
        return applyPostResolutionFilter(
                list, instantAppPkgName, allowDynamicSplits, filterCallingUid, resolveForStart,
                userId, intent);
    }
    
}

```
ASS.resolveActivity()方法的核心功能是找到相应的Activity组件，并保存到intent对象。


## ActivityStart#startActivity

执行`ActivityStackSupervisor#resolveIntent` 后，继续流程

```
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        SafeActivityOptions options,
        boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
        TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup) {
    int err = ActivityManager.START_SUCCESS;
    // Pull the optional Ephemeral Installer-only bundle out of the options early.
    final Bundle verificationBundle
            = options != null ? options.popAppVerificationBundle() : null;
    // 获取调用者的进程记录对象
    ProcessRecord callerApp = null;
    if (caller != null) {
        callerApp = mService.getRecordForAppLocked(caller);
        if (callerApp != null) {
            callingPid = callerApp.pid;
            callingUid = callerApp.info.uid;
        } else {
            err = ActivityManager.START_PERMISSION_DENIED;
        }
    }
    final int userId = aInfo != null && aInfo.applicationInfo != null
            ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;
    ActivityRecord sourceRecord = null;
    ActivityRecord resultRecord = null;
    if (resultTo != null) {
        // 获取调用者所在的 Activity
        sourceRecord = mSupervisor.isInAnyStackLocked(resultTo);
        if (sourceRecord != null) {
            if (requestCode >= 0 && !sourceRecord.finishing) {
                //requestCode = -1 则不进入
                resultRecord = sourceRecord;
            }
        }
    }
    final int launchFlags = intent.getFlags();
    if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
        // activity执行结果的返回由源Activity转换到新Activity, 不需要返回结果则不会进入该分支，具体查看源码

    }
    if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
        //从Intent中无法找到相应的Component
        err = ActivityManager.START_INTENT_NOT_RESOLVED;
    }
    if (err == ActivityManager.START_SUCCESS && aInfo == null) {
        //从Intent中无法找到相应的ActivityInfo
        err = ActivityManager.START_CLASS_NOT_FOUND;
    }
    if (err == ActivityManager.START_SUCCESS && sourceRecord != null
            && sourceRecord.getTask().voiceSession != null) {
        // If this activity is being launched as part of a voice session, we need
        // to ensure that it is safe to do so.  If the upcoming activity will also
        // be part of the voice session, we can only launch it if it has explicitly
        // said it supports the VOICE category, or it is a part of the calling app.
        // 
        if ((launchFlags & FLAG_ACTIVITY_NEW_TASK) == 0
                && sourceRecord.info.applicationInfo.uid != aInfo.applicationInfo.uid) {
            try {
                intent.addCategory(Intent.CATEGORY_VOICE);
                if (!mService.getPackageManager().activitySupportsIntent(
                        intent.getComponent(), intent, resolvedType)) {
                    Slog.w(TAG,
                            "Activity being started in current voice task does not support voice: "
                                    + intent);
                    err = ActivityManager.START_NOT_VOICE_COMPATIBLE;
                }
            } catch (RemoteException e) {
                Slog.w(TAG, "Failure checking voice capabilities", e);
                err = ActivityManager.START_NOT_VOICE_COMPATIBLE;
            }
        }
    }
    if (err == ActivityManager.START_SUCCESS && voiceSession != null) {
        // If the caller is starting a new voice session, just make sure the target
        // is actually allowing it to run this way.
        try {
            if (!mService.getPackageManager().activitySupportsIntent(intent.getComponent(),
                    intent, resolvedType)) {
                Slog.w(TAG,
                        "Activity being started in new voice task does not support: "
                                + intent);
                err = ActivityManager.START_NOT_VOICE_COMPATIBLE;
            }
        } catch (RemoteException e) {
            Slog.w(TAG, "Failure checking voice capabilities", e);
            err = ActivityManager.START_NOT_VOICE_COMPATIBLE;
        }
    }
    final ActivityStack resultStack = resultRecord == null ? null : resultRecord.getStack();
    if (err != START_SUCCESS) {
        if (resultRecord != null) {
            resultStack.sendActivityResultLocked(
                    -1, resultRecord, resultWho, requestCode, RESULT_CANCELED, null);
        }
        SafeActivityOptions.abort(options);
        return err;
    }
    boolean abort = !mSupervisor.checkStartAnyActivityPermission(intent, aInfo, resultWho,
            requestCode, callingPid, callingUid, callingPackage, ignoreTargetSecurity,
            inTask != null, callerApp, resultRecord, resultStack);
    abort |= !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
            callingPid, resolvedType, aInfo.applicationInfo);
    // Merge the two options bundles, while realCallerOptions takes precedence.
    ActivityOptions checkedOptions = options != null
            ? options.getOptions(intent, aInfo, callerApp, mSupervisor)
            : null;
    if (allowPendingRemoteAnimationRegistryLookup) {
        checkedOptions = mService.getActivityStartController()
                .getPendingRemoteAnimationRegistry()
                .overrideOptionsIfNeeded(callingPackage, checkedOptions);
    }
    if (mService.mController != null) {
        try {
            // The Intent we give to the watcher has the extra data
            // stripped off, since it can contain private information.
            Intent watchIntent = intent.cloneFilter();
            abort |= !mService.mController.activityStarting(watchIntent,
                    aInfo.applicationInfo.packageName);
        } catch (RemoteException e) {
            mService.mController = null;
        }
    }
    mInterceptor.setStates(userId, realCallingPid, realCallingUid, startFlags, callingPackage);
    if (mInterceptor.intercept(intent, rInfo, aInfo, resolvedType, inTask, callingPid,
            callingUid, checkedOptions)) {
        // activity start was intercepted, e.g. because the target user is currently in quiet
        // mode (turn off work) or the target application is suspended
        intent = mInterceptor.mIntent;
        rInfo = mInterceptor.mRInfo;
        aInfo = mInterceptor.mAInfo;
        resolvedType = mInterceptor.mResolvedType;
        inTask = mInterceptor.mInTask;
        callingPid = mInterceptor.mCallingPid;
        callingUid = mInterceptor.mCallingUid;
        checkedOptions = mInterceptor.mActivityOptions;
    }
    if (abort) {
        if (resultRecord != null) {
            resultStack.sendActivityResultLocked(-1, resultRecord, resultWho, requestCode,
                    RESULT_CANCELED, null);
        }
        // We pretend to the caller that it was really started, but
        // they will just get a cancel result.
        ActivityOptions.abort(checkedOptions);
        return START_ABORTED;
    }
    // If permissions need a review before any of the app components can run, we
    // launch the review activity and pass a pending intent to start the activity
    // we are to launching now after the review is completed.
    if (mService.mPermissionReviewRequired && aInfo != null) {
        if (mService.getPackageManagerInternalLocked().isPermissionsReviewRequired(
                aInfo.packageName, userId)) {
            IIntentSender target = mService.getIntentSenderLocked(
                    ActivityManager.INTENT_SENDER_ACTIVITY, callingPackage,
                    callingUid, userId, null, null, 0, new Intent[]{intent},
                    new String[]{resolvedType}, PendingIntent.FLAG_CANCEL_CURRENT
                            | PendingIntent.FLAG_ONE_SHOT, null);
            final int flags = intent.getFlags();
            Intent newIntent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
            newIntent.setFlags(flags
                    | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
            newIntent.putExtra(Intent.EXTRA_PACKAGE_NAME, aInfo.packageName);
            newIntent.putExtra(Intent.EXTRA_INTENT, new IntentSender(target));
            if (resultRecord != null) {
                newIntent.putExtra(Intent.EXTRA_RESULT_NEEDED, true);
            }
            intent = newIntent;
            resolvedType = null;
            callingUid = realCallingUid;
            callingPid = realCallingPid;
            rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId, 0,
                    computeResolveFilterUid(
                            callingUid, realCallingUid, mRequest.filterCallingUid));
            aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags,
                    null /*prof..ilerInfo*/);
            if (DEBUG_PERMISSIONS_REVIEW) {
                Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true,
                        true, false) + "} from uid " + callingUid + " on display "
                        + (mSupervisor.mFocusedStack == null
                        ? DEFAULT_DISPLAY : mSupervisor.mFocusedStack.mDisplayId));
            }
        }
    }
    // If we have an ephemeral app, abort the process of launching the resolved intent.
    // Instead, launch the ephemeral installer. Once the installer is finished, it
    // starts either the intent we resolved here [on install error] or the ephemeral
    // app [on install success].
    if (rInfo != null && rInfo.auxiliaryInfo != null) {
        intent = createLaunchIntent(rInfo.auxiliaryInfo, ephemeralIntent,
                callingPackage, verificationBundle, resolvedType, userId);
        resolvedType = null;
        callingUid = realCallingUid;
        callingPid = realCallingPid;
        aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, null /*profilerInfo*/);
    }
    ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
            callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
            resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
            mSupervisor, checkedOptions, sourceRecord);
    if (outActivity != null) {
        outActivity[0] = r;
    }
    if (r.appTimeTracker == null && sourceRecord != null) {
        // If the caller didn't specify an explicit time tracker, we want to continue
        // tracking under any it has.
        r.appTimeTracker = sourceRecord.appTimeTracker;
    }
    final ActivityStack stack = mSupervisor.mFocusedStack;
    // If we are starting an activity that is not from the same uid as the currently resumed
    // one, check whether app switches are allowed.
    if (voiceSession == null && (stack.getResumedActivity() == null
            || stack.getResumedActivity().info.applicationInfo.uid != realCallingUid)) {
        if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid,
                realCallingPid, realCallingUid, "Activity start")) {
            mController.addPendingActivityLaunch(new PendingActivityLaunch(r,
                    sourceRecord, startFlags, stack, callerApp));
            ActivityOptions.abort(checkedOptions);
            return ActivityManager.START_SWITCHES_CANCELED;
        }
    }
    if (mService.mDidAppSwitch) {
        // This is the second allowed switch since we stopped switches,
        // so now just generally allow switches.  Use case: user presses
        // home (switches disabled, switch to home, mDidAppSwitch now true);
        // user taps a home icon (coming from home so allowed, we hit here
        // and now allow anyone to switch again).
        mService.mAppSwitchesAllowedTime = 0;
    } else {
        mService.mDidAppSwitch = true;
    }
    mController.doPendingActivityLaunches(false);
    return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
            true /* doResume */, checkedOptions, inTask, outActivity);
}
```


```
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {
    int result = START_CANCELED;
    try {
        mService.mWindowManager.deferSurfaceLayout();
        // 继续执行下一步操作
        result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, outActivity);
    } finally {
        // If we are not able to proceed, disassociate the activity from the task. Leaving an
        // activity in an incomplete state can lead to issues, such as performing operations
        // without a window container.
        final ActivityStack stack = mStartActivity.getStack();
        if (!ActivityManager.isStartResultSuccessful(result) && stack != null) {
            stack.finishActivityLocked(mStartActivity, RESULT_CANCELED,
                    null /* intentResultData */, "startActivity", true /* oomAdj */);
        }
        mService.mWindowManager.continueSurfaceLayout();
    }
    postStartActivityProcessing(r, result, mTargetStack);
    return result;
}
```
## ActivityStart#startActivityUnchecked

```

// Note: This method should only be called from {@link startActivity}.
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity) {
    setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
            voiceInteractor);
    computeLaunchingTaskFlags();
    computeSourceStack();
    mIntent.setFlags(mLaunchFlags);
    ActivityRecord reusedActivity = getReusableIntentActivity();
    int preferredWindowingMode = WINDOWING_MODE_UNDEFINED;
    int preferredLaunchDisplayId = DEFAULT_DISPLAY;
    if (mOptions != null) {
        preferredWindowingMode = mOptions.getLaunchWindowingMode();
        preferredLaunchDisplayId = mOptions.getLaunchDisplayId();
    }
    // windowing mode and preferred launch display values from {@link LaunchParams} take
    // priority over those specified in {@link ActivityOptions}.
    if (!mLaunchParams.isEmpty()) {
        if (mLaunchParams.hasPreferredDisplay()) {
            preferredLaunchDisplayId = mLaunchParams.mPreferredDisplayId;
        }
        if (mLaunchParams.hasWindowingMode()) {
            preferredWindowingMode = mLaunchParams.mWindowingMode;
        }
    }
    if (reusedActivity != null) {
        // When the flags NEW_TASK and CLEAR_TASK are set, then the task gets reused but
        // still needs to be a lock task mode violation since the task gets cleared out and
        // the device would otherwise leave the locked task.
        if (mService.getLockTaskController().isLockTaskModeViolation(reusedActivity.getTask
                (mLaunchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                        == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))) {
            Slog.e(TAG, "startActivityUnchecked: Attempt to violate Lock Task Mode");
            return START_RETURN_LOCK_TASK_MODE_VIOLATION;
        }
        // True if we are clearing top and resetting of a standard (default) launch mode
        // ({@code LAUNCH_MULTIPLE}) activity. The existing activity will be finished.
        final boolean clearTopAndResetStandardLaunchMode =
                (mLaunchFlags & (FLAG_ACTIVITY_CLEAR_TOP | FLAG_ACTIVITY_RESET_TASK_IF_NEED
                        == (FLAG_ACTIVITY_CLEAR_TOP | FLAG_ACTIVITY_RESET_TASK_IF_NEEDED)
                && mLaunchMode == LAUNCH_MULTIPLE;
        // If mStartActivity does not have a task associated with it, associate it with the
        // reused activity's task. Do not do so if we're clearing top and resetting for a
        // standard launchMode activity.
        if (mStartActivity.getTask() == null && !clearTopAndResetStandardLaunchMode) {
            mStartActivity.setTask(reusedActivity.getTask());
        }
        if (reusedActivity.getTask().intent == null) {
            // This task was started because of movement of the activity based on affinity.
            // Now that we are actually launching it, we can assign the base intent.
            reusedActivity.getTask().setIntent(mStartActivity);
        }
        // This code path leads to delivering a new intent, we want to make sure we schedul
        // as the first operation, in case the activity will be resumed as a result of late
        // operations.
        if ((mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                || isDocumentLaunchesIntoExisting(mLaunchFlags)
                || isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
            final TaskRecord task = reusedActivity.getTask();
            // In this situation we want to remove all activities from the task up to the o
            // being started. In most cases this means we are resetting the task to its ini
            // state.
            final ActivityRecord top = task.performClearTaskForReuseLocked(mStartActivity,
                    mLaunchFlags);
            // The above code can remove {@code reusedActivity} from the task, leading to t
            // the {@code ActivityRecord} removing its reference to the {@code TaskRecord}.
            // task reference is needed in the call below to
            // {@link setTargetStackAndMoveToFrontIfNeeded}.
            if (reusedActivity.getTask() == null) {
                reusedActivity.setTask(task);
            }
            if (top != null) {
                if (top.frontOfTask) {
                    // Activity aliases may mean we use different intents for the top activ
                    // so make sure the task now has the identity of the new intent.
                    top.getTask().setIntent(mStartActivity);
                }
                deliverNewIntent(top);
            }
        }
        mSupervisor.sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, reusedActivi
        reusedActivity = setTargetStackAndMoveToFrontIfNeeded(reusedActivity);
        final ActivityRecord outResult =
                outActivity != null && outActivity.length > 0 ? outActivity[0] : null;
        // When there is a reused activity and the current result is a trampoline activity,
        // set the reused activity as the result.
        if (outResult != null && (outResult.finishing || outResult.noDisplay)) {
            outActivity[0] = reusedActivity;
        }
        if ((mStartFlags & START_FLAG_ONLY_IF_NEEDED) != 0) {
            // We don't need to start a new activity, and the client said not to do anythin
            // if that is the case, so this is it!  And for paranoia, make sure we have
            // correctly resumed the top activity.
            resumeTargetStackIfNeeded();
            return START_RETURN_INTENT_TO_CALLER;
        }
        if (reusedActivity != null) {
            setTaskFromIntentActivity(reusActivityStackSupervisor#resolveIntent
                resumeTargetStackIfNeeded();
                if (outActivity != null && outActivity.length > 0) {
                    outActivity[0] = reusedActivity;
                }
                return mMovedToFront ? START_TASK_TO_FRONT : START_DELIVERED_TO_TOP;
            }
        }
    }
    if (mStartActivity.packageName == null) {
        final ActivityStack sourceStack = mStartActivity.resultTo != null
                ? mStartActivity.resultTo.getStack() : null;
        if (sourceStack != null) {
            sourceStack.sendActivityResultLocked(-1 /* callingUid */, mStartActivity.result
                    mStartActivity.resultWho, mStartActivity.requestCode, RESULT_CANCELED,
                    null /* data */);
        }
        ActivityOptions.abort(mOptions);
        return START_CLASS_NOT_FOUND;
    }
    // If the activity being launched is the same as the one currently at the top, then
    // we need to check if it should only be launched once.
    final ActivityStack topStack = mSupervisor.mFocusedStack;
    final ActivityRecord topFocused = topStack.getTopActivity();
    final ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(mNotTop);
    final boolean dontStart = top != null && mStartActivity.resultTo == null
            && top.realActivity.equals(mStartActivity.realActivity)
            && top.userId == mStartActivity.userId
            && top.app != null && top.app.thread != null
            && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
            || isLaunchModeOneOf(LAUNCH_SINGLE_TOP, LAUNCH_SINGLE_TASK));
    if (dontStart) {
        // For paranoia, make sure we have correctly resumed the top activity.
        topStack.mLastPausedActivity = null;
        if (mDoResume) {
            mSupervisor.resumeFocusedStackTopActivityLocked();
        }
        ActivityOptions.abort(mOptions);
        if ((mStartFlags & START_FLAG_ONLY_IF_NEEDED) != 0) {
            // We don't need to start a new activity, and the client said not to do
            // anything if that is the case, so this is it!
            return START_RETURN_INTENT_TO_CALLER;
        }
        deliverNewIntent(top);
        // Don't use mStartActivity.task to show the toast. We're not starting a new activi
        // but reusing 'top'. Fields in mStartActivity may not be fully initialized.
        mSupervisor.handleNonResizableTaskIfNeeded(top.getTask(), preferredWindowingMode,
                preferredLaunchDisplayId, topStack);
        return START_DELIVERED_TO_TOP;
    }
    boolean newTask = false;
    final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
            ? mSourceRecord.getTask() : null;
    // Should this be considered a new task?
    int result = START_SUCCESS;
    if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
            && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
        newTask = true;
        result = setTaskFromReuseOrCreateNewTask(taskToAffiliate, topStack);
    } else if (mSourceRecord != null) {
        result = setTaskFromSourceRecord();
    } else if (mInTask != null) {
        result = setTaskFromInTask();
    } else {
        // This not being started from an existing activity, and not part of a new task...
        // just put it in the top task, though these days this case should never happen.
        setTaskToCurrentTopOrCreateNewTask();
    }
    if (result != START_SUCCESS) {
        return result;
    }
    mService.grantUriPermissionFromIntentLocked(mCallingUid, mStartActivity.packageName,
            mIntent, mStartActivity.getUriPermissionsLocked(), mStartActivity.userId);
    mService.grantEphemeralAccessLocked(mStartActivity.userId, mIntent,
            mStartActivity.appInfo.uid, UserHandle.getAppId(mCallingUid));
    if (newTask) {
        EventLog.writeEvent(EventLogTags.AM_CREATE_TASK, mStartActivity.userId,
                mStartActivity.getTask().taskId);
    }
    ActivityStack.logStartActivity(
            EventLogTags.AM_CREATE_ACTIVITY, mStartActivity, mStartActivity.getTask());
    mTargetStack.mLastPausedActivity = null;
    mSupervisor.sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, mStartActivity);
    mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransitio
            mOptions);
    if (mDoResume) {
        final ActivityRecord topTaskActivity =
                mStartActivity.getTask().topRunningActivityLocked();
        if (!mTargetStack.isFocusable()
                || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                && mStartActivity != topTaskActivity)) {
            // If the activity is not focusable, we can't resume it, but still would like t
            // make sure it becomes visible as it starts (this will also trigger entry
            // animation). An example of this are PIP activities.
            // Also, we don't want to resume activities in a task that currently has an ove
            // as the starting activity just needs to be in the visible paused state until 
            // over is removed.
            mTargetStack.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
            // Go ahead and tell window manager to execute app transition for this activity
            // since the app transition will not be triggered through the resume channel.
            mService.mWindowManager.executeAppTransition();
        } else {
            // If the target stack was not previously focusable (previous top running activ
            // on that stack was not visible) then any prior calls to move the stack to the
            // will not update the focused stack.  If starting the new activity now allows 
            // task stack to be focusable, then ensure that we now update the focused stack
            // accordingly.
            if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                mTargetStack.moveToFront("startActivityUnchecked");
            }
            mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                    mOptions);
        }
    } else if (mStartActivity != null) {
        mSupervisor.mRecentTasks.add(mStartActivity.getTask());
    }
    mSupervisor.updateUserStackLocked(mStartActivity.userId, mTargetStack);
    mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTask(), preferredWindowing
            preferredLaunchDisplayId, mTargetStack);
    return START_SUCCESS;
}
```