---
title: Android Activity的启动过程（API27 源码分析）
tag: Android源码
category: Android

---

<meta name="referrer" content="no-referrer" />



[TOC]

# Activity的启动过程（API27 源码分析）

## 正常启动

[Activity启动过程源码分析思维导图](http://naotu.baidu.com/file/a824b4941efefafb032f507b78b9feaf?token=0b7f9970bfba3410)

### 介绍

- Instrumentation：每个Activity都持有Instrumentation对象的一个引用，但是整个进程只会存在一个Instrumentation对象。 Instrumentation这个类里面的方法大多数和Application和Activity有关，**这个类就是完成对Application和Activity初始化和生命周期的工具类。**可以说它就是Activity生命周期的一个大管家
- ActivityManagerService：简称AMS，不仅负责系统中所有Activity的生命周期，还负责其他几个组件的管理，可以说这个服务贯穿了四大组件
- ActivityStarter：Activity启动的intent和flag等的控制和解释
- ActivityStarterSupervisor：ActivityStackSupervisor是ActivityStack的总管。4.4中默认引入了两个ActivityStack，一个叫Home stack，放Launcher和systemui，id为0；另一个是Applicationstack，放App的Activity，id可能是任意值
- ActivityStack：Activity堆栈，其中的ActivityRecord是通过TaskRecord这一层间接地被管理着
- ActivityThread：依赖于UI线程，这个app启动的最开始的地方，在这里你可以找到熟悉的**main()**方法，Application也是在这里创建和初始化的，并且主活动的启动也是在这里的，包括主线程Handler
- ApplicationThread：ActivityThread的一个内部类，你可以在这里找到四大组件的创建和初始化，它是和ActivityManagerService沟通的桥梁
- Instrumentation：用来监控应用程序和系统的交互

### startActivity正常启动分析

通过startActivity(intent)来启动活动，跟进源码看一下

首先Activity类里面重载了多个startActivity()方法，参数不同而已

```java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    //options是启动活动需要添加的附加条件，一般为null
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
}
```

不管options是不是null，判断过后都会调用到Activity类中的`startActivityForResult(@RequiresPermission Intent intent, int requestCode, @Nullable Bundle options)`

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode, @Nullable Bundle options) {
    //mParent是ActivityGroup，用来嵌套多个子Activity，现已弃用，所以一般为null
    if (mParent == null) {
        //判断、转移启动活动的选项
        options = transferSpringboardActivityOptions(options);
        //调用了Instrumentation中的execStartActivity
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            //当ActivityResult不为null的时候，通过mMainThread发送
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        if (requestCode >= 0) {
            ...
            mStartedActivity = true;
        }
        cancelInputsAndStartExitTransition(options);
    } else {
        //有ActivityGroup的情况，暂不分析
        if (options != null) {
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
}
```

在获取Instrumentation.ActivityResult的实例时，调用了Instrumentation中的execStartActivity，先看传入的参数有this（上下文），mMainThread.getApplicationThread()（mMainThread是一个ActivityThread，调用此方法获取了ApplicationThread，ApplicationThread是ActivityThread的一个内部类），mToken（IBinder，用于跨进程通信），this（目标活动），intent，requestCode（前面传入的，没有传入的时候是-1），options（前面传入的）

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    Uri referrer = target != null ? target.onProvideReferrer() : null;
    if (referrer != null) {
        intent.putExtra(Intent.EXTRA_REFERRER, referrer);
    }
    //当活动监视器不为null的一些配置
    if (mActivityMonitors != null) {
        ...
    }
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        //获取启动Activity的结果
        int result = ActivityManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        //检查启动Activity的结果
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

代码中，通过ActivityManager.getService().startActivity来获取启动Activity的结果。那么Activity启动的真正实现就在这里了。先是通过ActivityManager这个管理类来获取Service

```java
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();
}

//单例模式获取IActivityManager
private static final Singleton<IActivityManager> IActivityManagerSingleton =
        new Singleton<IActivityManager>() {
            @Override
            protected IActivityManager create() {
                //获取IBinder型service
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                //通过IActivityManager.Stub绑定service
                final IActivityManager am = IActivityManager.Stub.asInterface(b);
                return am;
            }
        };
```

可以看到，ActivityManager中的getService通过单例模式来获取了IActivityManager的实例am，这个am拥有着从ServiceManager那里获取的Service。这个am其实就是ActivityManagerService（简称AMS），因为AMS继承了IActivityManager.Stub这个内部类，所以最后Activity的启动过程实际到了AMS中了，所以startActivity是AMS中的方法了

在看AMS之前先简单看一下checkStartActivityResult方法

```java
public static void checkStartActivityResult(int res, Object intent) {
    if (!ActivityManager.isStartResultFatalError(res)) {
        return;
    }

    switch (res) {
        case ActivityManager.START_INTENT_NOT_RESOLVED:
        case ActivityManager.START_CLASS_NOT_FOUND:
            if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                //这个就是当待启动的Activity没有在AndroidManifest中注册就会抛出的异常
                throw new ActivityNotFoundException(
                        "Unable to find explicit activity class "
                        + ((Intent)intent).getComponent().toShortString()
                        + "; have you declared this activity in your AndroidManifest.xml?");
            throw new ActivityNotFoundException(
                    "No Activity found to handle " + intent);
        case ActivityManager.START_PERMISSION_DENIED:
            throw new SecurityException("Not allowed to start activity "
                    + intent);
        case ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT:
            throw new AndroidRuntimeException(
                    "FORWARD_RESULT_FLAG used while also requesting a result");
        case ActivityManager.START_NOT_ACTIVITY:
            throw new IllegalArgumentException(
                    "PendingIntent is not an activity");
        case ActivityManager.START_NOT_VOICE_COMPATIBLE:
            throw new SecurityException(
                    "Starting under voice control not allowed for: " + intent);
        case ActivityManager.START_VOICE_NOT_ACTIVE_SESSION:
            throw new IllegalStateException(
                    "Session calling startVoiceActivity does not match active session");
        case ActivityManager.START_VOICE_HIDDEN_SESSION:
            throw new IllegalStateException(
                    "Cannot start voice activity on a hidden session");
        case ActivityManager.START_ASSISTANT_NOT_ACTIVE_SESSION:
            throw new IllegalStateException(
                    "Session calling startAssistantActivity does not match active session");
        case ActivityManager.START_ASSISTANT_HIDDEN_SESSION:
            throw new IllegalStateException(
                    "Cannot start assistant activity on a hidden session");
        case ActivityManager.START_CANCELED:
            throw new AndroidRuntimeException("Activity could not be started for "
                    + intent);
        default:
            throw new AndroidRuntimeException("Unknown error code "
                    + res + " when starting " + intent);
    }
}
```

可以看到确实是对于启动Activity结果的检查，抛出了这么多异常
还是回到我们之前的AMS的startActivity方法

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}

@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    ...
    // mActivityStarter是一个控制Activity怎样启动的管理类，此处调用ActivityStarter的startActivityMayWait
    return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, bOptions, false, userId, null, "startActivityAsUser");
}
```

先调用AMS类里面的startActivityAsUser，接着mActivityStarter.startActivityMayWait，而mActivityStarter是一个ActivityStarter的实例，所以又进入到ActivityStarter中的startActivityMayWait

```java
final int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, Intent intent, String resolvedType,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, WaitResult outResult,
        Configuration globalConfig, Bundle bOptions, boolean ignoreTargetSecurity, int userId,
        TaskRecord inTask, String reason) {
        //一些配置信息和检查
        ...
        //获取res值
        int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor,
                resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, outRecord, inTask,
                reason);
        ...
        return res;
    }
}
```

函数返回的是res，而res是调用ActivityStarter本类中的startActivityLocked方法得到

```java
int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
        ActivityRecord[] outActivity, TaskRecord inTask, String reason) {
    ...
    //调用startActivity获取ActivityResult
    mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
            aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
            callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
            options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
            inTask);
    ...
    return mLastStartActivityResult != START_ABORTED ? mLastStartActivityResult : START_SUCCESS;
}
```

根据mLastStartActivityResult进行返回，而mLastStartActivityResult是通过startActivity得到，所以接着看ActivityStarter本类中的startActivity

```java
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
        ActivityRecord[] outActivity, TaskRecord inTask) {
    ...
    //返回的是startActivity的返回值
    return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags, true,
            options, inTask, outActivity);
}
```

接着又调用ActivityStarter本类中的startActivity

```java
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity) {
    int result = START_CANCELED;
    try {
        mService.mWindowManager.deferSurfaceLayout();
        //要返回的result，通过startActivityUnchecked
        result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, outActivity);
    } finally {
        ...
    }

    postStartActivityProcessing(r, result, mSupervisor.getLastStack().mStackId,  mSourceRecord,
            mTargetStack);

    return result;
}
```

接着调用的是ActivityStarter本类中的startActivityUnchecked

```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity) {
    //初始化ActivityStarter的成员变量，包括了四种启动模式和启动flags
    setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
            voiceInteractor);
    //计算启动的flags，包括启动模式
    computeLaunchingTaskFlags();
    //计算原栈
    computeSourceStack();
    //设置flag
    mIntent.setFlags(mLaunchFlags);

    ActivityRecord reusedActivity = getReusableIntentActivity();

    final int preferredLaunchStackId =
            (mOptions != null) ? mOptions.getLaunchStackId() : INVALID_STACK_ID;
    final int preferredLaunchDisplayId =
            (mOptions != null) ? mOptions.getLaunchDisplayId() : DEFAULT_DISPLAY;
    //此处就开始判断启动的模式了
    if (reusedActivity != null) {
        // When the flags NEW_TASK and CLEAR_TASK are set, then the task gets reused but
        // still needs to be a lock task mode violation since the task gets cleared out and
        // the device would otherwise leave the locked task.
        if (mSupervisor.isLockTaskModeViolation(reusedActivity.getTask(),
                (mLaunchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                        == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))) {
            mSupervisor.showLockTaskToast();
            Slog.e(TAG, "startActivityUnchecked: Attempt to violate Lock Task Mode");
            return START_RETURN_LOCK_TASK_MODE_VIOLATION;
        }

        if (mStartActivity.getTask() == null) {
            mStartActivity.setTask(reusedActivity.getTask());
        }
        if (reusedActivity.getTask().intent == null) {
            // This task was started because of movement of the activity based on affinity...
            // Now that we are actually launching it, we can assign the base intent.
            reusedActivity.getTask().setIntent(mStartActivity);
        }

        // This code path leads to delivering a new intent, we want to make sure we schedule it
        // as the first operation, in case the activity will be resumed as a result of later
        // operations.
        if ((mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                || isDocumentLaunchesIntoExisting(mLaunchFlags)
                || mLaunchSingleInstance || mLaunchSingleTask) {
            final TaskRecord task = reusedActivity.getTask();

            // In this situation we want to remove all activities from the task up to the one
            // being started. In most cases this means we are resetting the task to its initial
            // state.
            final ActivityRecord top = task.performClearTaskForReuseLocked(mStartActivity,
                    mLaunchFlags);

            // The above code can remove {@code reusedActivity} from the task, leading to the
            // the {@code ActivityRecord} removing its reference to the {@code TaskRecord}. The
            // task reference is needed in the call below to
            // {@link setTargetStackAndMoveToFrontIfNeeded}.
            if (reusedActivity.getTask() == null) {
                reusedActivity.setTask(task);
            }

            if (top != null) {
                if (top.frontOfTask) {
                    // Activity aliases may mean we use different intents for the top activity,
                    // so make sure the task now has the identity of the new intent.
                    top.getTask().setIntent(mStartActivity);
                }
                deliverNewIntent(top);
            }
        }

        sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, reusedActivity);

        reusedActivity = setTargetStackAndMoveToFrontIfNeeded(reusedActivity);

        final ActivityRecord outResult =
                outActivity != null && outActivity.length > 0 ? outActivity[0] : null;

        // When there is a reused activity and the current result is a trampoline activity,
        // set the reused activity as the result.
        if (outResult != null && (outResult.finishing || outResult.noDisplay)) {
            outActivity[0] = reusedActivity;
        }

        if ((mStartFlags & START_FLAG_ONLY_IF_NEEDED) != 0) {
            // We don't need to start a new activity, and the client said not to do anything
            // if that is the case, so this is it!  And for paranoia, make sure we have
            // correctly resumed the top activity.
            resumeTargetStackIfNeeded();
            return START_RETURN_INTENT_TO_CALLER;
        }
        setTaskFromIntentActivity(reusedActivity);

        if (!mAddingToTask && mReuseTask == null) {
            // We didn't do anything...  but it was needed (a.k.a., client don't use that
            // intent!)  And for paranoia, make sure we have correctly resumed the top activity.
            resumeTargetStackIfNeeded();
            if (outActivity != null && outActivity.length > 0) {
                outActivity[0] = reusedActivity;
            }

            return START_TASK_TO_FRONT;
        }
    }
    ...
    //判断栈顶的同一个活动，检查是否需要只启动一次
    final ActivityStack topStack = mSupervisor.mFocusedStack;
    final ActivityRecord topFocused = topStack.topActivity();
    final ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(mNotTop);
    final boolean dontStart = top != null && mStartActivity.resultTo == null
            && top.realActivity.equals(mStartActivity.realActivity)
            && top.userId == mStartActivity.userId
            && top.app != null && top.app.thread != null
            && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
            || mLaunchSingleTop || mLaunchSingleTask);
    if (dontStart) {
        //resume栈顶的活动
        // For paranoia, make sure we have correctly resumed the top activity.
        topStack.mLastPausedActivity = null;
        if (mDoResume) {
            //做resume的事情，而不是新建一个活动
            mSupervisor.resumeFocusedStackTopActivityLocked();
        }
        ActivityOptions.abort(mOptions);
        if ((mStartFlags & START_FLAG_ONLY_IF_NEEDED) != 0) {
            // We don't need to start a new activity, and the client said not to do
            // anything if that is the case, so this is it!
            return START_RETURN_INTENT_TO_CALLER;
        }

        deliverNewIntent(top);

        // Don't use mStartActivity.task to show the toast. We're not starting a new activity
        // but reusing 'top'. Fields in mStartActivity may not be fully initialized.
        mSupervisor.handleNonResizableTaskIfNeeded(top.getTask(), preferredLaunchStackId,
                preferredLaunchDisplayId, topStack.mStackId);

        return START_DELIVERED_TO_TOP;
    }

    boolean newTask = false;
    final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
            ? mSourceRecord.getTask() : null;

    //判断是否新建一个任务栈
    int result = START_SUCCESS;
    if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
            && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
        newTask = true;
        result = setTaskFromReuseOrCreateNewTask(
                taskToAffiliate, preferredLaunchStackId, topStack);
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
    ...
    //mDoResume是在方法开始时初始化，doResume的值又是在第一次startActivity中传递的true
    if (mDoResume) {
        final ActivityRecord topTaskActivity =
                mStartActivity.getTask().topRunningActivityLocked();
        if (!mTargetStack.isFocusable()
                || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                && mStartActivity != topTaskActivity)) {
            //检查不能Resume时
        } else {
            ...
            //mSupervisor时ActivityStarterSupervisor的实例
            mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                    mOptions);
        }
    } else {
        mTargetStack.addRecentActivityLocked(mStartActivity);
    }
    ...
    return START_SUCCESS;
}
```

如果没有异常情况的话，这个方法的正确返回值会是START_SUCCESS，在此之前，mDoResume为true，mSupervisor.resumeFocusedStackTopActivityLocked执行，所以启动活动的任务又到了ActivityStarterSupervisor中，接着看ActivityStarterSupervisor的resumeFocusedStackTopActivityLocked方法

```java
boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
    ...
    //重点在这里
    if (targetStack != null && isFocusedStack(targetStack)) {
        return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }
    ...
    return false;
}
```

resumeFocusedStackTopActivityLocked中经过判断后，又调用了ActivityStack中的resumeTopActivityUncheckedLocked方法

```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        ...
        boolean result = false;
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            //这里是重点
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        ...
        return result;
    }
```

接着又调用了ActivityStack本类中的resumeTopActivityInnerLocked

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ...
    final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
    ...
    if (next.app != null && next.app.thread != null) {
        ...
    } else {
        // Whoops, need to restart this activity!
        //如果活动没有被启动过
        if (!next.hasBeenLaunched) {
            next.hasBeenLaunched = true;
        } else {
            if (SHOW_APP_STARTING_PREVIEW) {
                next.showStartingWindow(null /* prev */, false /* newTask */,
                        false /* taskSwich */);
            }
            if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Restarting: " + next);
        }
        if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Restarting " + next);
        //这是重点
        mStackSupervisor.startSpecificActivityLocked(next, true, true);
    }

    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
    return true;
}
```

接着调用了ActivityStarterSupervisor的startSpecificActivityLocked方法

```java
void startSpecificActivityLocked(ActivityRecord r,
        boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
            r.info.applicationInfo.uid, true);

    r.getStack().setLaunchTime(r);

    if (app != null && app.thread != null) {
        try {
            ...
            //这是重点
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }
        ...
    }

    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
            "activity", r.intent.getComponent(), false, false, true);
}
```

当启动活动的Application不为null，接着调用了ActivityStarterSupervisors本类的realStartActivityLocked

```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
        boolean andResume, boolean checkConfig) throws RemoteException {
    ...
    try {
        ...
        try {
            ...
            //此处重点
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info,
                    // TODO: Have this take the merged configuration instead of separate global
                    // and override configs.
                    mergedConfiguration.getGlobalConfiguration(),
                    mergedConfiguration.getOverrideConfiguration(), r.compat,
                    r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                    r.persistentState, results, newIntents, !andResume,
                    mService.isNextTransitionForward(), profilerInfo);
            ...
        } catch (RemoteException e) {
            if (r.launchFailed) {
                ...
            }
            ...
            r.launchFailed = true;
            app.activities.remove(r);
            throw e;
        }
    } finally {
        endDeferResume();
    }
    ...
    return true;
}
```

看到这里，重点是app.thread.scheduleLaunchActivity，安排启动活动。而app.thread是IApplicationThread，而IApplicationThread的实现类其实就是ActivityThread的内部类ApplicationThread，接着看ApplicationThread的scheduleLaunchActivity方法

### ActivityThread

ApplicationThread的scheduleLaunchActivity方法

```java
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
        CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
        int procState, Bundle state, PersistableBundle persistentState,
        List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
        boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

    updateProcessState(procState, false);

    ActivityClientRecord r = new ActivityClientRecord();
    //中间是一些设置
    ...
    //好吧，这里是重点
    sendMessage(H.LAUNCH_ACTIVITY, r);
}
```

最后通过sendMessage来发送消息，连续两次的方法重载

```java
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
    if (DEBUG_MESSAGES) Slog.v(
        TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
        + ": " + arg1 + " / " + obj);
    Message msg = Message.obtain();
    msg.what = what;
    msg.obj = obj;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    if (async) {
        msg.setAsynchronous(true);
    }
    //这是重点，mH是H的实例，H就是一个Hanlder的子类
    mH.sendMessage(msg);
}
```

mH是H的实例，而H是继承于Hanlder的

```java
private class H extends Handler {
    ...
}
```

所以最后调用了mH的sendMessage方法将一个Message对象添加到消息队列。由于是Handler，那么最终也应该mH的handleMessage方法处理消息

```java
public void handleMessage(Message msg) {
    if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
    switch (msg.what) {
        //第一个就是启动Activity
        case LAUNCH_ACTIVITY: {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
            final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

            r.packageInfo = getPackageInfoNoCheck(
                    r.activityInfo.applicationInfo, r.compatInfo);
            handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        } break;
        ...
        case PAUSE_ACTIVITY: {
            ...
        } break;
        ...
        case STOP_ACTIVITY_SHOW: {
            ...
        } break;
        ...
        case RESUME_ACTIVITY:
            ...
            break;
        ...
        case DESTROY_ACTIVITY:
            ...
            break;
        ...
    }
    Object obj = msg.obj;
    if (obj instanceof SomeArgs) {
        ((SomeArgs) obj).recycle();
    }
    if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + codeToString(msg.what));
}
```

接着调用ApplicationThread的handleLaunchActivity

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    ...
    WindowManagerGlobal.initialize();
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        Bundle oldState = r.state;
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);

        if (!r.activity.mFinished && r.startsNotResumed) {
            ...
            performPauseActivityIfNeeded(r, reason);
            ...
            if (r.isPreHoneycomb()) {
                r.state = oldState;
            }
        }
    } else {
        ...
    }
}
```

通过performLaunchActivity来获取Activity对象

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }

    ComponentName component = r.intent.getComponent();
    if (component == null) {
        component = r.intent.resolveActivity(
            mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }

    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
    }
    //获取一个Context
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        //获取类加载器
        java.lang.ClassLoader cl = appContext.getClassLoader();
        //通过mInstrumentation，Instrumentation中的newActivity方法是通过反射来获取一个实例的
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        ...
    }

    try {
        //获取Application
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        ...
        if (activity != null) {
            //获取标题
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            if (r.overrideConfig != null) {
                config.updateFrom(r.overrideConfig);
            }
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                    + r.activityInfo.name + " with config " + config);
            Window window = null;
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                //获取window
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }
            appContext.setOuterContext(activity);
            //配置信息
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback);

            if (customIntent != null) {
                activity.mIntent = customIntent;
            }
            r.lastNonConfigurationInstances = null;
            checkAndBlockForNetworkAccess();
            activity.mStartedActivity = false;
            //获取theme
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            activity.mCalled = false;
            if (r.isPersistable()) {
                //开始调用onCreate方法，最后回调成重写的onCreate方法
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                //开始调用onCreate方法
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onCreate()");
            }
            r.activity = activity;
            r.stopped = true;
            if (!r.activity.mFinished) {
                //最终回调到Activity中的onStart
                activity.performStart();
                r.stopped = false;
            }
            if (!r.activity.mFinished) {
                if (r.isPersistable()) {
                    if (r.state != null || r.persistentState != null) {
                        //最后会回调成onRestoreInstanceState方法，对你没猜错
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                r.persistentState);
                    }
                } else if (r.state != null) {
                    //最后会回调成onRestoreInstanceState方法，对你没猜错
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                }
            }
            if (!r.activity.mFinished) {
                activity.mCalled = false;
                if (r.isPersistable()) {
                    //最后会回调成onPostCreate方法
                    mInstrumentation.callActivityOnPostCreate(activity, r.state,
                            r.persistentState);
                } else {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onPostCreate()");
                }
            }
        }
        r.paused = true;
        //入栈
        mActivities.put(r.token, r);

    } catch (SuperNotCalledException e) {
        throw e;

    } catch (Exception e) {
        ...
    }
    return activity;
}
```

进行了一些Activity必须的一些配置，然后都会调用mInstrumentation.callActivityOnCreate方法，即Instrumentation中的callActivityOnCreate

```java
public void callActivityOnCreate(Activity activity, Bundle icicle,
        PersistableBundle persistentState) {
    prePerformCreate(activity);
    //这是重点
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}
```

接着又调用了Activity的performCreate方法，这时就开始回到最初的Activity中了，在Activity中，不管performCreate重载了几种，最后都会调用performCreate有两个参数的方法

```java
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    mCanEnterPictureInPicture = true;
    restoreHasCurrentPermissionRequest(icicle);
    if (persistentState != null) {
        //这是重点，onCreate出现了
        onCreate(icicle, persistentState);
    } else {
        //这是重点，onCreate出现了
        onCreate(icicle);
    }
    mActivityTransitionState.readState(icicle);

    mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
            com.android.internal.R.styleable.Window_windowNoDisplay, false);
    mFragments.dispatchActivityCreated();
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
}
```

可以看见，源码中都会调用Activity中的onCreate方法

```java
public void onCreate(@Nullable Bundle savedInstanceState,
        @Nullable PersistableBundle persistentState) {
    onCreate(savedInstanceState);
}

protected void onCreate(@Nullable Bundle savedInstanceState) {
    if (DEBUG_LIFECYCLE) Slog.v(TAG, "onCreate " + this + ": " + savedInstanceState);
    ...
}
```

没错，这就是你在Activity中重写的方法onCreate

------

## APP启动过程（从点击桌面开始）

### 介绍

我们知道从桌面上点击一个应用图标时，就会启动到对应的应用中，并且启动的应用首先展示的Activity就是我们每个apk里面在AndroidManifest中注册的主活动。

在我们分析从桌面点击启动APP之前先了解一些概念：

- Launcher：其实我们常说的桌面本身就是一个APP，一般手机上默认的桌面就是一个系统级的APP；这个桌面就是我们需要了解的Lancher应用程序，它会展示所有的当前系统安装了的应用程序和其相关的信息（具体了解Launcher请移步[Android 系统启动流程](http://blog.penghesheng.cn/2019/04/25/Android Activity启动过程/)）
- Launcher进程：Launcher所在的进程
- APP进程：启动的APP所在的进程，我们知道每一个APP都是独立运行在一个Android虚拟机中的，也是在一个独立的进程中
- zygote：这也是一个进程，译为*受精卵*，Android是基于Linux系统的，而在Linux中，所有的进程都是由init进程直接或者是间接fork出来的，zygote进程也不例外。在Android系统里，开启新进程的方式都是通过fork第一个Zygote进程来实现的。所以Dalvik虚拟机、应用程序进程以及SystemServer都是Zygote的子进程
- SystemServer：运行系统的关键服务的一个进程，它和Zygote是Android系统FragmeWork两个非常重要的进程；SystemServer中开启了很多重要的服务，如ActivityManagerService、PackageManagerService、WindowManagerService等
- ActivityManagerService：简称AMS，不仅负责系统中所有Activity的生命周期，还负责其他几个组件的管理，可以说这个服务贯穿了四大组件
- APP进程：待启动应用程序所在的进程

整个启动过程呢，大致就是下面的图咯（是后面的博文中的图呢）

[![app启动图](https://upload-images.jianshu.io/upload_images/3985563-b7edc7b70c9c332f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/960/format/webp)](https://upload-images.jianshu.io/upload_images/3985563-b7edc7b70c9c332f.png?imageMogr2/auto-orient/strip|imageView2/2/w/960/format/webp)app启动图

### 桌面点击过程

从前面我们了解到了，从桌面点击一个应用图标，其实就是去启动一个Activity，又由于Launcher本身就是一个应用程序，我们启动的又是一个应用程序，所以这将是一个跨进程的启动Activity，同时这将是一个普通的APP进程从无到有的过程；那么接下来我们就分析这个过程

### Launcher启动Activity

Launcher的源码在系统源码的packages/apps/Launcher3下（PS：虽然现在有Launcher、Launcher2、Launcher3三个Launcher，但是都不影响代码分析，这里主要以Launcher3来看源码）

首先看看Launcher的定义吧

```java
/**
 * Default launcher application.
 */
public class Launcher extends BaseDraggingActivity implements LauncherExterns,
        LauncherModel.Callbacks, LauncherProviderChangeListener, UserEventDelegate{
		...
        }
```

Launcher是通过继承这个BaseDraggingActivity的，通过名字就知道这是个能够拖放的BaseActivity。通过查看它的源码，也的确是这样，BaseDraggingActivity是继承自BaseActivity的，然后在BaseActivity的基础上扩展了一些功能，最主要的就是拖放了。好了，这些都不是我们的重点了，我们着重关注Launcher的点击事件。在Launcher中无法找到点击应用图标响应点击事件的处理，在`createShortcut()`方法中我们能知道，在Launcher添加这些应用图标的时候都设置了ClickListener，而这个Listener是`ItemClickHandler.INSTANCE`

```java
public class Launcher extends BaseDraggingActivity implements LauncherExterns,
        LauncherModel.Callbacks, LauncherProviderChangeListener, UserEventDelegate{
	...
    //这里就是创建桌面上显示的View
	View createShortcut(ShortcutInfo info) {
        return createShortcut((ViewGroup) mWorkspace.getChildAt(mWorkspace.getCurrentPage()), info);
    }

    public View createShortcut(ViewGroup parent, ShortcutInfo info) {
        //加载对应的app icon，设置响应的参数 如监听事件
        BubbleTextView favorite = (BubbleTextView) LayoutInflater.from(parent.getContext())
                .inflate(R.layout.app_icon, parent, false);
        favorite.applyFromShortcutInfo(info);
        favorite.setOnClickListener(ItemClickHandler.INSTANCE);
        favorite.setOnFocusChangeListener(mFocusHandler);
        return favorite;
    }
	...
}
```

接着我们直接看整个`ItemClickHandler.INSTANCE`是什么

```java
public class ItemClickHandler {
    /**
     * Instance used for click handling on items
     */
    public static final OnClickListener INSTANCE = ItemClickHandler::onClick;

    private static void onClick(View v) {
        // Make sure that rogue clicks don't get through while allapps is launching, or after the
        // view has detached (it's possible for this to happen if the view is removed mid touch).
        if (v.getWindowToken() == null) {
            return;
        }
		//获取Launcher 这个方法是Launcher中的静态方法
        Launcher launcher = Launcher.getLauncher(v.getContext());
        if (!launcher.getWorkspace().isFinishedSwitchingState()) {
            return;
        }

        Object tag = v.getTag();
        if (tag instanceof ShortcutInfo) {
            onClickAppShortcut(v, (ShortcutInfo) tag, launcher);
        } else if (tag instanceof FolderInfo) {
            if (v instanceof FolderIcon) {
                onClickFolderIcon(v);
            }
        } else if (tag instanceof AppInfo) {
            startAppShortcutOrInfoActivity(v, (AppInfo) tag, launcher);
        } else if (tag instanceof LauncherAppWidgetInfo) {
            if (v instanceof PendingAppWidgetHostView) {
                onClickPendingWidget((PendingAppWidgetHostView) v, launcher);
            }
        }
    }
    ...
}
```

`ItemClickHandler.INSTANCE`其实就是ItemClickHandler中的onClick方法，在Launcher中用了一个单例来获取，在onClick方法中会先做一个检查，确保当某个app正在卸载但是图标还没有移除的情况下，这时候的点击事件不能当作一个正常的点击事件。

接着会拿到Launcher的实例，通过点击的的View的tag来判定是ShortcutInfo、FolderInfo、AppInfo、LauncherAppWidgetInfo其中的某一个。

> ShortcutInfo代表的是一个可快捷式点击的图标，这也是Launcher在添加app图标的时候创建的时候设置的参数;
>
> FolderInfo代表的是文件形式的图标，我们都知道在桌面上是可以将多个图标放在一个文件夹中的，这就是描述的这种情况；
>
> AppInfo代表的是应用图标，表示app（没懂和ShortcutInfo的区别）
>
> LauncherAppWidgetInfo代表的是桌面上的一些小控件

不管是ShortcutInfo还是AppInfo，最后都会调用`startAppShortcutOrInfoActivity(View v, ItemInfo item, Launcher launcher)`方法，`onClickAppShortcut()`方法最终也会调用上面的方法

```java
public class ItemClickHandler {
	...
	private static void onClickAppShortcut(View v, ShortcutInfo shortcut, Launcher launcher) {
        if (shortcut.isDisabled()) {
            ...
        }
        // Check for abandoned promise
        if ((v instanceof BubbleTextView) && shortcut.hasPromiseIconUi()) {
            ...
        }
        // Start activities
        startAppShortcutOrInfoActivity(v, shortcut, launcher);
    }
	...
}
```

接着看一下`startAppShortcutOrInfoActivity(View v, ItemInfo item, Launcher launcher)`方法

```java
public class ItemClickHandler {
	...
	private static void startAppShortcutOrInfoActivity(View v, ItemInfo item, Launcher launcher) {
        Intent intent;
        if (item instanceof PromiseAppInfo) {
            PromiseAppInfo promiseAppInfo = (PromiseAppInfo) item;
            intent = promiseAppInfo.getMarketIntent(launcher);
        } else {
            intent = item.getIntent();
        }
        if (intent == null) {
            throw new IllegalArgumentException("Input must have a valid intent");
        }
        if (item instanceof ShortcutInfo) {
            ...
        }
        launcher.startActivitySafely(v, intent, item);
    }
}
```

首先看一下方法传进的参数，View就是点击事件产生的View，ItemInfo是Launcher中的所有Item，ShortcutInfo、AppInfo等都是继承自它的

方法中先检查是不是PromiseAppInfo类型，PromiseAppInfo是继承自AppInfo的，然后通过PromiseAppInfo的getMarketIntent方法来获取Intent，`getMarketIntent()`返回的是`new PackageManagerHelper(context).getMarketIntent(componentName.getPackageName())`

如果是其他类型，比如ShortcutInfo，则直接获取intent

接着如果是ShortcutInfo还需要检查一下

最后通过`launcher.startActivitySafely(v, intent, item);`来启动Activity，launcher就是我们前面获取的Launcher实例

```java
public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
       boolean success = super.startActivitySafely(v, intent, item);
       if (success && v instanceof BubbleTextView) {
           BubbleTextView btv = (BubbleTextView) v;
           btv.setStayPressed(true);
           setOnResumeCallback(btv);
       }
       return success;
   }
```

Launcher中直接调用的是父类BaseDraggingActivity的startActivitySafely方法

```java
public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
       ...
       UserHandle user = item == null ? null : item.user;
       // Prepare intent
       intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
       if (v != null) {
           intent.setSourceBounds(getViewBounds(v));
       }
       try {
           boolean isShortcut = Utilities.ATLEAST_MARSHMALLOW
                   && (item instanceof ShortcutInfo)
                   && (item.itemType == Favorites.ITEM_TYPE_SHORTCUT
                   || item.itemType == Favorites.ITEM_TYPE_DEEP_SHORTCUT)
                   && !((ShortcutInfo) item).isPromise();
           if (isShortcut) {
               // Shortcuts need some special checks due to legacy reasons.
               startShortcutIntentSafely(intent, optsBundle, item);
           } else if (user == null || user.equals(Process.myUserHandle())) {
               // Could be launching some bookkeeping activity
               startActivity(intent, optsBundle);
           } else {
               LauncherAppsCompat.getInstance(this).startActivityForProfile(
                       intent.getComponent(), user, intent.getSourceBounds(), optsBundle);
           }
           getUserEventDispatcher().logAppLaunch(v, intent);
           return true;
       } catch (ActivityNotFoundException|SecurityException e) {
           ...
       }
       return false;
   }
```

先是一些检查，我们就不贴出来了。然后通过`intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);`来给intent设置FLAG，意味着我们要在一个新的活动栈里启动一个Activity

接着对是不是ShortcutInfo进行了一个判断，因为ShortcutInfo需要一些特殊的检查

接着是当item.user为null的时候，直接进行Activity的启动`startActivity(intent, optsBundle);`

由于我们在Launcher中的是ShortcutInfo类型的，所以这里肯定执行的是内部的`startShortcutIntentSafely(intent, optsBundle, item);`方法

```java
private void startShortcutIntentSafely(Intent intent, Bundle optsBundle, ItemInfo info) {
       ...
       // Could be launching some bookkeeping activity
       startActivity(intent, optsBundle);
      	...
   }
```

重点就是留下来的这一行代码`startActivity(intent, optsBundle);`，这个方法是Activity类中的

```java
   @Override
public void startActivity(Intent intent, @Nullable Bundle options) {
       if (options != null) {
           startActivityForResult(intent, -1, options);
       } else {
           // Note we want to go through this call for compatibility with
           // applications that may have overridden the method.
           startActivityForResult(intent, -1);
       }
   }
```

根据有没有options来调用startActivityForResult方法，-1表示不需要Activity结束后的结果

到了这里，后面很多逻辑都和前面启动Activity一样了，所以后面就着重讲不一样的地方

从这里开始，一直到ActivityStarterSupervisor的startSpecificActivityLocked方法，我们从这里接着看

```java
void startSpecificActivityLocked(ActivityRecord r,
           boolean andResume, boolean checkConfig) {
       // Is this activity's application already running?
       ProcessRecord app = mService.getProcessRecordLocked(r.processName,
               r.info.applicationInfo.uid, true);
       r.getStack().setLaunchTime(r);
       if (app != null && app.thread != null) {
           try {
               ...
               realStartActivityLocked(r, app, andResume, checkConfig);
               return;
           } 
           ...
       }
       mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
               "activity", r.intent.getComponent(), false, false, true);
   }
```

前面我们知道，正常启动Activity时，在这里判断application的时候是不为null的，所以正常启动Activity的时候后面会调用`realStartActivityLocked(r, app, andResume, checkConfig);`方法，并return

但是现在通过Launcher来启动一个应用，没有运行的application，所以`app == null`，所以这里就会执行到`mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0, "activity", r.intent.getComponent(), false, false, true);`方法，mService就是AMS

```java
final ProcessRecord startProcessLocked(String processName,
           ApplicationInfo info, boolean knownToBeDead, int intentFlags,
           String hostingType, ComponentName hostingName, boolean allowWhileBooting,
           boolean isolated, boolean keepIfLarge) {
       return startProcessLocked(processName, info, knownToBeDead, intentFlags, hostingType,
               hostingName, allowWhileBooting, isolated, 0 /* isolatedUid */, keepIfLarge,
               null /* ABI override */, null /* entryPoint */, null /* entryPointArgs */,
               null /* crashHandler */);
   }
```

在AMS中startProcessLocked方法调用了重载的startProcessLocked方法

```java
  final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
          boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
          boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
          String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
      long startTime = SystemClock.elapsedRealtime();
      ProcessRecord app;
      if (!isolated) {
          app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
          ...
      } else {
          // If this is an isolated process, it can't re-use an existing process.
          app = null;
      }
...
      if (app != null && app.pid > 0) {
          ...
      }
...
      if (app == null) {
          checkTime(startTime, "startProcess: creating new process record");
          app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
          ...
      } else {
          ...
      }
      // If the system is not ready yet, then hold off on starting this
      // process until it is.
      ...
      checkTime(startTime, "startProcess: stepping in to startProcess");
      startProcessLocked(
              app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
      checkTime(startTime, "startProcess: done starting proc!");
      return (app.pid != 0) ? app : null;
  }
```

首先根据前面传进来的isolated参数为false，所以必然执行的是`app = getProcessRecordLocked(processName, info.uid, keepIfLarge);`，这行代码是根据processName去获取已经存在的进程；在Activity应用程序中的AndroidManifest.xml配置文件中，我们没有指定Application标签的process属性，系统就会默认使用package的名称（我们可以配置两个应用程序具有相同的uid和package，或者在AndroidManifest.xml配置文件的application标签或者activity标签中显式指定相同的process属性值，这样，不同的应用程序也可以在同一个进程中启动）；由于我们是Launcher启动的，所有会返回null

当`app == null`的时候，会调用`newProcessRecordLocked()`去创建一个进程

接着先看AMS中`newProcessRecordLocked(ApplicationInfo info, String customProcess, boolean isolated, int isolatedUid)`方法

```java
final ProcessRecord newProcessRecordLocked(ApplicationInfo info, String customProcess,
           boolean isolated, int isolatedUid) {
       String proc = customProcess != null ? customProcess : info.processName;
       BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
       final int userId = UserHandle.getUserId(info.uid);
       int uid = info.uid;
       if (isolated) {
           ...
       }
       final ProcessRecord r = new ProcessRecord(stats, info, proc, uid);
       if (!mBooted && !mBooting
               && userId == UserHandle.USER_SYSTEM
               && (info.flags & PERSISTENT_MASK) == PERSISTENT_MASK) {
           r.persistent = true;
           r.maxAdj = ProcessList.PERSISTENT_PROC_ADJ;
       }
       addProcessNameLocked(r);
       return r;
   }
```

由于isolated为false，所以不会执行if中的语句，接着就`new ProcessRecord(stats, info, proc, uid)`来创建新的应用程序的进程，然后返回

接着回到上一个方法AMS中的startProcessLocked，接着看到最后`startProcessLocked( app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs)`，这AMS中重载的

```java
private final void startProcessLocked(ProcessRecord app, String hostingType,
           String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
       long startTime = SystemClock.elapsedRealtime();
       if (app.pid > 0 && app.pid != MY_PID) {
           ...
       }
       ...
 	checkTime(startTime, "startProcess: starting to update cpu stats");
       //更新cpu
       updateCpuStats();
       checkTime(startTime, "startProcess: done updating cpu stats");
       try {
           try {
               final int userId = UserHandle.getUserId(app.uid);
               ...
           } catch (RemoteException e) {
               throw e.rethrowAsRuntimeException();
           }

           int uid = app.uid;
           int[] gids = null;
           int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;
           if (!app.isolated) {
               int[] permGids = null;
               try {
                   checkTime(startTime, "startProcess: getting gids from package manager");
                   ....
               } catch (RemoteException e) {
                   throw e.rethrowAsRuntimeException();
               }
			...
           }
           checkTime(startTime, "startProcess: building args");
		...
           //这个标志在启动进程的时候很重要，这里暂就不分析了
           int debugFlags = 0;
           ...
           boolean isActivityProcess = (entryPoint == null);
           //注意这里
           if (entryPoint == null) entryPoint = "android.app.ActivityThread";
           Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                   app.processName);
           checkTime(startTime, "startProcess: asking zygote to start proc");
           ProcessStartResult startResult;
           if (hostingType.equals("webview_service")) {
               ...
           } else {
               //启动进程
               startResult = Process.start(entryPoint,
                       app.processName, uid, uid, gids, debugFlags, mountExternal,
                       app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                       app.info.dataDir, invokeWith, entryPointArgs);
           }
           checkTime(startTime, "startProcess: returned from zygote!");
           Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

           mBatteryStatsService.noteProcessStart(app.processName, app.info.uid);
           checkTime(startTime, "startProcess: done updating battery stats");
		...
           checkTime(startTime, "startProcess: building log message");
           ...
           checkTime(startTime, "startProcess: starting to update pids map");
           ...
           synchronized (mPidsSelfLocked) {
               //存入mPidsSelfLocked这个进程集合里
               this.mPidsSelfLocked.put(startResult.pid, app);
               if (isActivityProcess) {
                   Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
                   msg.obj = app;
                   mHandler.sendMessageDelayed(msg, startResult.usingWrapper
                           ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
               }
           }
           checkTime(startTime, "startProcess: done updating pids map");
       } catch (RuntimeException e) {
           ...
       }
   }
```

这个方法就很重要了，前面我们只是看到了进程的创建，启动进程就是在这个方法中了。这个方法比较长，我们一点一点看

会获取到userId，然后他通过PackageManager去获取gids，如果没有就会创建，然后配置相应的一些参数，最后保存到我们的app里面

重点看一下42行，`startResult = Process.start(entryPoint, app.processName, uid, uid, gids, debugFlags, mountExternal, app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet, app.info.dataDir, invokeWith, entryPointArgs);`，这里会通过Zogyte来启动我们的进程，那么entryPoint又是什么呢？看到33行，`if (entryPoint == null) entryPoint = "android.app.ActivityThread";`，从这里我们就知道，通过Zogyte启动的进程是什么了——启动了ActivityThread，因为我们所通过桌面点击启动的，所以会跨进程启动，然后到这里来启动对应的应用的进程

具体是怎样的，我们看一下Process的start方法

```java
/**
    * Start a new process.
    * ...
    */
   public static final ProcessStartResult start(final String processClass,
                                 final String niceName,
                                 int uid, int gid, int[] gids,
                                 int debugFlags, int mountExternal,
                                 int targetSdkVersion,
                                 String seInfo,
                                 String abi,
                                 String instructionSet,
                                 String appDataDir,
                                 String invokeWith,
                                 String[] zygoteArgs) {
       return zygoteProcess.start(processClass, niceName, uid, gid, gids,
                   debugFlags, mountExternal, targetSdkVersion, seInfo,
                   abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
   }
```

通过注释，我们就知道这个方法是用来启动新的进程

然后接着调用了zygoteProcess的start方法，zygoteProcess就是ZygoteProcess，名如其义

接着看看ZygoteProcess

```java
public final Process.ProcessStartResult start(final String processClass,
                                                 final String niceName,
                                                 int uid, int gid, int[] gids,
                                                 int debugFlags, int mountExternal,
                                                 int targetSdkVersion,
                                                 String seInfo,
                                                 String abi,
                                                 String instructionSet,
                                                 String appDataDir,
                                                 String invokeWith,
                                                 String[] zygoteArgs) {
       try {
           return startViaZygote(processClass, niceName, uid, gid, gids,
                   debugFlags, mountExternal, targetSdkVersion, seInfo,
                   abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
       } ...
   }
```

接着调用了`startViaZygote(processClass, niceName, uid, gid, gids, debugFlags, mountExternal, targetSdkVersion, seInfo, abi, instructionSet, appDataDir, invokeWith, zygoteArgs);`

```java
/**
    * Starts a new process via the zygote mechanism.
    * ...
    */
   private Process.ProcessStartResult startViaZygote(final String processClass,
                                                     final String niceName,
                                                     final int uid, final int gid,
                                                     final int[] gids,
                                                     int debugFlags, int mountExternal,
                                                     int targetSdkVersion,
                                                     String seInfo,
                                                     String abi,
                                                     String instructionSet,
                                                     String appDataDir,
                                                     String invokeWith,
                                                     String[] extraArgs)
                                                     throws ZygoteStartFailedEx {
       ArrayList<String> argsForZygote = new ArrayList<String>();
	...
       synchronized(mLock) {
           return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
       }
   }
```

一系列的配置参数后，调用了`zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);`

```java
/**
    * Sends an argument list to the zygote process, which starts a new child
    * and returns the child's pid. Please note: the present implementation
    * replaces newlines in the argument list with spaces.
    *
    * @throws ZygoteStartFailedEx if process start failed for any reason
    */
   @GuardedBy("mLock")
   private static Process.ProcessStartResult zygoteSendArgsAndGetResult(
           ZygoteState zygoteState, ArrayList<String> args)
           throws ZygoteStartFailedEx {
       try {
           ...
           final BufferedWriter writer = zygoteState.writer;
           final DataInputStream inputStream = zygoteState.inputStream;
           writer.write(Integer.toString(args.size()));
           writer.newLine();
           for (int i = 0; i < sz; i++) {
               String arg = args.get(i);
               writer.write(arg);
               writer.newLine();
           }
           writer.flush();
           // Should there be a timeout on this?
           Process.ProcessStartResult result = new Process.ProcessStartResult();
           // Always read the entire result from the input stream to avoid leaving
           // bytes in the stream for future process starts to accidentally stumble
           // upon.
           result.pid = inputStream.readInt();
           result.usingWrapper = inputStream.readBoolean();

           if (result.pid < 0) {
               throw new ZygoteStartFailedEx("fork() failed");
           }
           return result;
       } ...
   }
```

到这里就是真正的孵化了一个Zygote进程了，并且启动了进程

好了，Zygote启动后，ActivityThread自然也就运行了，接着往后面看

### ActivityThread过程

在java中main方法就是程序的入口了，同样，ActivityThread的main方法就是我们APP的入口了，也就是主活动启动之前

```java
public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    CloseGuard.setEnabled(false);
    Environment.initForCurrentUser();
    // Set the reporter for event logging in libcore
    EventLogger.setReporter(new EventLoggingReporter());
    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);
    Process.setArgV0("<pre-initialized>");
    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }
    // End of event ActivityThreadMain.
    //这里会通知Zogyte进程告知ActivityThread现在已经启动了
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

ActivityThread是final的，也就是说一个APP对应一个ActivityThread，同时了创建了ActivityThread的实例。在main方法中，做了一些准备，如Looper和消息队列。实例化后，调用ActivityThread的attach方法将其实例对象绑定到AMS中

```java
private void attach(boolean system) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
        ...
        final IActivityManager mgr = ActivityManager.getService();
        try {
            //看这里
            mgr.attachApplication(mAppThread);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        ...
    } else {
        ...
    }
    ...
}
```

调用AMS的attachApplication将ActivityThread绑定到AMS中，接着看AMS中的attachApplication方法

```java
@Override
public final void attachApplication(IApplicationThread thread) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid);
        Binder.restoreCallingIdentity(origId);
    }
}
```

AMS的attachApplication接着调用了本类中的attachApplicationLocked

```java
private final boolean attachApplicationLocked(IApplicationThread thread,
        int pid) {
    ProcessRecord app;
    long startTime = SystemClock.uptimeMillis();
    //现在，可以根据pid在AMS中拿取到app了
    if (pid != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid);
        }
    } else {
        app = null;
    }
    ...
    try {
        ...
        if (app.instr != null) {
            thread.bindApplication(processName, appInfo, providers,
                    app.instr.mClass,
                    profilerInfo, app.instr.mArguments,
                    app.instr.mWatcher,
                    app.instr.mUiAutomationConnection, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(getGlobalConfiguration()), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked(),
                    buildSerial);
        } else {
            //将ApplicationThread绑定到AMS
            thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                    null, null, null, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(getGlobalConfiguration()), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked(),
                    buildSerial);
        }
        ...
    } catch (Exception e) {
        ...
    }
    ...
    if (normalMode) {
        try {
            //这里是重点
            if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            ...
        }
    }
    ...
    return true;
}
```

thread.bindApplication将ApplicationThread对象绑定到AMs，然后ApplicationThread里面会接着调用ActivityThread的sendMessage，最后通过ActivityThread类的H.sendMessage送到消息队列，然后通过H的handleMessage处理消息，最后就调用到ActivityThread的handleBindApplication完成整个ApplicationThread的绑定。

接着看AMS的attachApplicationLocked的后面，发现还通过mStackSupervisor调用了attachApplicationLocked，又到StackSupervisor中看看attachApplicationLocked

```java
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
    final String processName = app.processName;
    boolean didSomething = false;
    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
        ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
        for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
            ...
            for (int i = 0; i < size; i++) {
                final ActivityRecord activity = mTmpActivityList.get(i);
                if (activity.app == null && app.uid == activity.info.applicationInfo.uid
                        && processName.equals(activity.processName)) {
                    try {
                        //看这里咯
                        if (realStartActivityLocked(activity, app,
                                top == activity /* andResume */, true /* checkConfig */)) {
                            didSomething = true;
                        }
                    } catch (RemoteException e) {
                        ...
                    }
                }
            }
        }
    }
    ...
    return didSomething;
}
```

StackSupervisor中attachApplicationLocked方法的代码不是很多，仔细看看，看到了realStartActivityLocked，咦，这不是前面一个Activity正常启动时会调用的方法吗。没错，这里就开始进入到主活动的启动了，跟着进去看看本类的realStartActivityLocked

```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
        boolean andResume, boolean checkConfig) throws RemoteException {
    ...
    try {
        ...
        try {
            ...
            //通过各种检测，收集参数信息，最后在这里就开始准备启动Activity了
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info,
                    // TODO: Have this take the merged configuration instead of separate global
                    // and override configs.
                    mergedConfiguration.getGlobalConfiguration(),
                    mergedConfiguration.getOverrideConfiguration(), r.compat,
                    r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                    r.persistentState, results, newIntents, !andResume,
                    mService.isNextTransitionForward(), profilerInfo);
            ...
        } catch (RemoteException e) {
            ...
        }
    } finally {
        endDeferResume();
    }
    ...
    return true;
}
```

可以看见，这就是前面分析Activity启动时的部分源码，最后都是调用ApplicationThread来准备启动Activity，后面的调用就跟Activity的启动过程一样了，最后回调到Activity的onCreate，这样，主活动就启动成功了

## 总结

### 正常Activity的启动

1. Activity中重载的startActivity启动方法，最终调用到startActivityForResult方法
2. 调用到Instrumentation中，会检查是否在AndroidManifest中注册，在Instrumentation中会通过单例获取AMS，调用到AMS中
3. 在AMS中又会调用到ActivityStarter，经过一些列调用后，会计算Activity的启动模式，FLAG等信息以及是否在栈顶等，然后保存这些信息
4. 然后又到ActivityStarterSupervisor中，通过ActivityStack判断Activity是否启动过，然后检查Application是否为null，不为null就会进行一个正常后面的启动；但是如果启动的Activity是跨进程的，会先去进行进程的一些孵化操作和启动进程的操作，而不会直接去启动活动
5. 正常的流程接着会通过ApplicationThread来launch一个Activity
6. 在ApplicationThread（ActivityThread的一个内部类）中会通过Handler的方式，发送启动Activity的消息；Handler是ActivityThread维持的主线程Handler，在接收到启动消息后，接着通过ApplicationThread进行后面的操作
7. 在ApplicationThread中，通过获取类加载器，然后通过反射来获取了一个Activity实例；接着给这个Activity绑定Application、创建Window、添加DecorView等，接着会回调onCreate方法（先是调用Instrumentation中，最后回调到了Activity中）

### 桌面点击应用启动

1. 从桌面点击应用图标，从Launcher从开始处理这个点击事件，先是会判断当前点击的是一个可启动的应用还是一个文件夹或者是一个小部件，然后相应对应时间，启动应用按照正常的流程启动一个Activity（当然，首先是Launcher这个活动会进入onPause）
2. 到了ActivityStarterSupervisor中，检查当前要启动的Activity是否跟当前的Activity是一个进程，不是然后就进行进程的孵化
3. 在AMS中，进行一些列的检查，根据包名是否能够查到相应的进程、获取到相关的信息，最终判定需要新孵化一个进程，就会创建一个ProcessRecord来保存相关的信息
4. 在AMS中startProcessLocked方法，进行新进程的一个启动，会通过Zygote来孵化一个新的Zygote自进程，然后启动新的ActivityThread主线程
5. 接着就是要启动APP的ActivityThread在运行中了，在ActivityThread中有一个main方法，在这里会创建ActivityThread的实例，然后绑定Application到AMS中，，准备Handler和Looper等，也会通知之前的进程——当前进程启动了并且ActivityThread启动了
6. 在将ActivityThread绑定到AMS中的同时，会去启动我们要启动APP的主活动，最后也是通过StackSupervisor中的realStartActivityLocked方法来启动这个MainActivity的，然后再后面的就和正常的启动一样了

## 特别感谢

[Android开发艺术探索](http://blog.penghesheng.cn/2019/04/25/Android Activity启动过程/)

[一个APP从启动到主页面显示经历了哪些过程？](https://www.jianshu.com/p/a72c5ccbd150)

[Android 进程启动流程（App 启动）](https://juejin.im/entry/586105ff61ff4b006308096a)

[Android应用程序启动过程源代码分析](https://blog.csdn.net/luoshengyang/article/details/6689748)