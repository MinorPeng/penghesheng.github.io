---
title: Android Broadcast源码分析
tag: Android

---

<meta name="referrer" content="no-referrer" />



[TOC]

# Broadcast源码分析

[源码脑图](http://naotu.baidu.com/file/f218c472d4ef7b60b1ad1f5d9dfa5646?token=3e3820b1e30ce0a2)

## 注册Broadcast

### 动态注册

我们通过`registerReceiver(BroadcastReceiver receiver, IntentFilter filter)`方法来动态注册一个广播，跟进看源码

首先跟Activity和Service一样，都是到ContextWrapper中

```java
@Override
public Intent registerReceiver(
    BroadcastReceiver receiver, IntentFilter filter) {
    return mBase.registerReceiver(receiver, filter);
}

@Override
public Intent registerReceiver(
    BroadcastReceiver receiver, IntentFilter filter, int flags) {
    return mBase.registerReceiver(receiver, filter, flags);
}

@Override
public Intent registerReceiver(
    BroadcastReceiver receiver, IntentFilter filter,
    String broadcastPermission, Handler scheduler) {
    return mBase.registerReceiver(receiver, filter, broadcastPermission,
            scheduler);
}

@Override
public Intent registerReceiver(
    BroadcastReceiver receiver, IntentFilter filter,
    String broadcastPermission, Handler scheduler, int flags) {
    return mBase.registerReceiver(receiver, filter, broadcastPermission,
            scheduler, flags);
}
```

可以很直观的看见，根据传入的参数不同，registerReceiver有多个重载方法，不过这并没有什么影响，因为在ContextWrapper这个类中并没有具体的操作，所以还是得到ContextImpl中看看

接着看看ContextImpl中得registerReceiver方法

```java
@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
    return registerReceiver(receiver, filter, null, null);
}

@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
        int flags) {
    return registerReceiver(receiver, filter, null, null, flags);
}

@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
        String broadcastPermission, Handler scheduler) {
    return registerReceiverInternal(receiver, getUserId(),
            filter, broadcastPermission, scheduler, getOuterContext(), 0);
}

@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
        String broadcastPermission, Handler scheduler, int flags) {
    return registerReceiverInternal(receiver, getUserId(),
            filter, broadcastPermission, scheduler, getOuterContext(), flags);
}
```

代码中也是根据参数得不同重载了几个registerReceiver方法，不过这没有什么影响，最后都会调用5个参数的方法，有些则会调用4个参数的方法。但这并没有什么影响，最后的结果都是调用registerReceiverInternal这个方法

接着看看ContextImpl的registerReceiverInternal方法

```java
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
        IntentFilter filter, String broadcastPermission,
        Handler scheduler, Context context, int flags) {
    IIntentReceiver rd = null;
    if (receiver != null) {
        if (mPackageInfo != null && context != null) {
            if (scheduler == null) {
                scheduler = mMainThread.getHandler();
            }
            //将BroadcastReceiver转换为LoadedApk.ReceiverDispatcher.InnerRecever
            rd = mPackageInfo.getReceiverDispatcher(
                receiver, context, scheduler,
                mMainThread.getInstrumentation(), true);
        } else {
            if (scheduler == null) {
                scheduler = mMainThread.getHandler();
            }
            //当LoadedApk和Context为空的时候直接创建LoadedApk.ReceiverDispatcher对象
            rd = new LoadedApk.ReceiverDispatcher(
                    receiver, context, scheduler, null, true).getIIntentReceiver();
        }
    }
    try {
        //最后通过AMS注册
        final Intent intent = ActivityManager.getService().registerReceiver(
                mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
                broadcastPermission, userId, flags);
        if (intent != null) {
            intent.setExtrasClassLoader(getClassLoader());
            intent.prepareToEnterProcess();
        }
        return intent;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

在这个方法中，首先会通过mPackageInfo来将BroadcastReceiver转换为InnerReceiver，mPackageInfo是LoadedApk对象，那我们看看LoadedApk的getReceiverDispatcher方法

```java
public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
        Context context, Handler handler,
        Instrumentation instrumentation, boolean registered) {
    synchronized (mReceivers) {
        LoadedApk.ReceiverDispatcher rd = null;
        ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
        if (registered) {
            map = mReceivers.get(context);
            if (map != null) {
                rd = map.get(r);
            }
        }
        if (rd == null) {
            //ReceiverDispatcher在这里创建，同时返回一个InnerReceiver对象
            rd = new ReceiverDispatcher(r, context, handler,
                    instrumentation, registered);
            if (registered) {
                if (map == null) {
                    map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                    mReceivers.put(context, map);
                }
                map.put(r, rd);
            }
        } else {
            rd.validate(context, handler);
        }
        rd.mForgotten = false;
        return rd.getIIntentReceiver();
    }
}
```

首先会创建一个ReceiverDispatcher对象，这个类跟ServiceDispatcher的作用是一样的。接着就通过ReceiverDispatcher将BroadcastReceiver转换为IIntentReceiver。IIntentReceiver是一个Binder，不直接使用BroadcastReceiver是因为可能会跨进程通信，BroadcastReceiver是不能进行跨进程传递，跟SeriviceConnection基本相似.IItentReceiver具体实现是LoadApk.ReceiverDispatcher.InnerReceiver，类似于InnerConnection。最后返回了ReceiverDispatcher持有的InnerReceiver

那么接着回到之前ContextImpl的registerReceiverInternal方法，接着往下面看。我们可以看到，不管怎样，都会创建一个LoadedApk.ReceiverDispather对象，最后通过ActivityManager.getService().registerReceiver注册，ActivityManager.getService()就是AMS，这个应该很熟悉了吧

接着看AMS的registerReceiver方法

```java
public Intent registerReceiver(IApplicationThread caller, String callerPackage,
        IIntentReceiver receiver, IntentFilter filter, String permission, int userId,
        int flags) {
    enforceNotIsolatedCaller("registerReceiver");
    ...

    synchronized (this) {
        ...
        ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
        if (rl == null) {
            rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                    userId, receiver);
            ...
            //存储
            mRegisteredReceivers.put(receiver.asBinder(), rl);
        } else if (rl.uid != callingUid) {
            ...
        } else if (rl.pid != callingPid) {
            ...
        } else if (rl.userId != userId) {
            ...
        }
        //实例化BroadcastFilter，BroadcastFilter继承自IntentFilter
        BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                permission, callingUid, userId, instantApp, visibleToInstantApps);
        rl.add(bf);
        ...
        //将远程的InnerReceiver对象和IntentFilter对象存储起来，这样注册就完了
        mReceiverResolver.addFilter(bf);

        // Enqueue broadcasts for all existing stickies that match
        // this filter.
        //由于粘性广播注册后就可以接收，所以这里是粘性广播的处理
        if (allSticky != null) {
            ...
        }

        return sticky;
    }
}
```

这个方法的代码还是优点多，在重要的地方写了注释，广播就这样注册完了（刚开始看的时候也懵逼，就这样完了？），确实是这样就完了

### 静态注册

静态注册是在AndroidManifest中注册的，通过PMS进行解析，具体的请看PMS的源码分析

## 发送和接收Broadcast

我们通过`sendBroadcast(Intent intent)`方法来发送一个广播，在onReceive方法中接收广播，那么来看看具体怎么实现的

首先还是ContextWrapper的sendBroadcast()方法

```java
@Override
public void sendBroadcast(Intent intent) {
    mBase.sendBroadcast(intent);
}
```

那么接下来，还是到ContextImpl中看看sendBroadcast()方法吧

```java
@Override
public void sendBroadcast(Intent intent) {
    warnIfCallingFromSystemProcess();
    String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
    try {
        intent.prepareToLeaveProcess(this);
        ActivityManager.getService().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, null,
                Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                getUserId());
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

代码很简短，ContextImpl马上就把工作交给了AMS去了

那么接着看AMS的broadcastIntent方法吧

```java
public final int broadcastIntent(IApplicationThread caller,
        Intent intent, String resolvedType, IIntentReceiver resultTo,
        int resultCode, String resultData, Bundle resultExtras,
        String[] requiredPermissions, int appOp, Bundle bOptions,
        boolean serialized, boolean sticky, int userId) {
    enforceNotIsolatedCaller("broadcastIntent");
    synchronized(this) {
        intent = verifyBroadcastLocked(intent);

        final ProcessRecord callerApp = getRecordForAppLocked(caller);
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        //交给了内部方法
        int res = broadcastIntentLocked(callerApp,
                callerApp != null ? callerApp.info.packageName : null,
                intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                requiredPermissions, appOp, bOptions, serialized, sticky,
                callingPid, callingUid, userId);
        Binder.restoreCallingIdentity(origId);
        return res;
    }
}
```

在这个方法中也没有做太多的事，就是获取了一些需要的信息，接着就调用了内部的broadcastIntentLocked方法

接着看AMS的broadcastIntentLocked方法

```java
final int broadcastIntentLocked(ProcessRecord callerApp,
        String callerPackage, Intent intent, String resolvedType,
        IIntentReceiver resultTo, int resultCode, String resultData,
        Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
        boolean ordered, boolean sticky, int callingPid, int callingUid, int userId) {
    intent = new Intent(intent);

    ...

    // By default broadcasts do not go to stopped apps.
    //默认情况下，广播不会发给已经停止的应用
    intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);

    // If we have not finished booting, don't allow this to launch new processes.
    ...

    // Make sure that the user who is receiving this broadcast is running.
    // If not, we will just skip it. Make an exception for shutdown broadcasts
    // and upgrade steps.
    ...

    // Verify that protected broadcasts are only being sent by system code,
    // and that system code is only sending protected broadcasts.
    final String action = intent.getAction();
    //确保收保护的广播是由系统发出的
    final boolean isProtectedBroadcast;
    try {
        isProtectedBroadcast = AppGlobals.getPackageManager().isProtectedBroadcast(action);
    } catch (RemoteException e) {
        Slog.w(TAG, "Remote exception", e);
        return ActivityManager.BROADCAST_SUCCESS;
    }

    final boolean isCallerSystem;
    switch (UserHandle.getAppId(callingUid)) {
        case ROOT_UID:
        case SYSTEM_UID:
        case PHONE_UID:
        case BLUETOOTH_UID:
        case NFC_UID:
            isCallerSystem = true;
            break;
        default:
            isCallerSystem = (callerApp != null) && callerApp.persistent;
            break;
    }

    // First line security check before anything else: stop non-system apps from
    // sending protected broadcasts.
    ...

    // Add to the sticky list if requested.
    //粘性广播的处理
    ...

    // Figure out who all will receive this broadcast.
    //查找出匹配的广播接收者并经过过滤
    List receivers = null;
    List<BroadcastFilter> registeredReceivers = null;
    // Need to resolve the intent to interested receivers...
    if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)
             == 0) {
        receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
    }
    if (intent.getComponent() == null) {
        if (userId == UserHandle.USER_ALL && callingUid == SHELL_UID) {
            // Query one target user at a time, excluding shell-restricted users
            for (int i = 0; i < users.length; i++) {
                if (mUserController.hasUserRestriction(
                        UserManager.DISALLOW_DEBUGGING_FEATURES, users[i])) {
                    continue;
                }
                List<BroadcastFilter> registeredReceiversForUser =
                        mReceiverResolver.queryIntent(intent,
                                resolvedType, false /*defaultOnly*/, users[i]);
                if (registeredReceivers == null) {
                    registeredReceivers = registeredReceiversForUser;
                } else if (registeredReceiversForUser != null) {
                    registeredReceivers.addAll(registeredReceiversForUser);
                }
            }
        } else {
            registeredReceivers = mReceiverResolver.queryIntent(intent,
                    resolvedType, false /*defaultOnly*/, userId);
        }
    }

    ...

    // Merge into one list.
    ...
    while (ir < NR) {
        if (receivers == null) {
            receivers = new ArrayList();
        }
        receivers.add(registeredReceivers.get(ir));
        ir++;
    }

    ...
    //将筛选出来的广播接收者添加到BroadcastQueue中，BroadcastQueue会将广播发送给相应的广播接收者
    if ((receivers != null && receivers.size() > 0)
            || resultTo != null) {
        //获取创建好的BroadcastQueue，在AMS中的全局变量，有前台和后台
        BroadcastQueue queue = broadcastQueueForIntent(intent);
        BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                callerPackage, callingPid, callingUid, callerInstantApp, resolvedType,
                requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
                resultData, resultExtras, ordered, sticky, false, userId);

        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing ordered broadcast " + r
                + ": prev had " + queue.mOrderedBroadcasts.size());
        if (DEBUG_BROADCAST) Slog.i(TAG_BROADCAST,
                "Enqueueing broadcast " + r.intent.getAction());

        final BroadcastRecord oldRecord =
                replacePending ? queue.replaceOrderedBroadcastLocked(r) : null;
        if (oldRecord != null) {
            // Replaced, fire the result-to receiver.
            ...
        } else {
            //添加到BroadcastQueue中
            queue.enqueueOrderedBroadcastLocked(r);
            //通过BroadcastQueue进行发送
            queue.scheduleBroadcastsLocked();
        }
    } else {
       ...
    }

    return ActivityManager.BROADCAST_SUCCESS;
}
```

这个方法中的代码有几百行，说明这个方法的重要性。在这个方法中，先会根据intent-filter查找出匹配的广播接收者并经过一系列的条件过滤，最终添加到BroadcastQueue中，通过BroadcastQueue来发送。这就说明了在这个方法中仅仅是将符合条件的广播接收者筛选出来，放到BroadcastQueue中，最终的发送还是通过BroadcastQueue来实现的

接着看BroadcastQueue.scheduleBroadcastsLocked()

```java
public void scheduleBroadcastsLocked() {
    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
            + mQueueName + "]: current="
            + mBroadcastsScheduled);

    if (mBroadcastsScheduled) {
        return;
    }
    //通过Handler来发送
    mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
    mBroadcastsScheduled = true;
}
```

mHanlder是根据创建BroadcastQueue时传入的AMS的MainHandler来创建的BroadcastHandler，持有的是AMS的MainHandler的looper。BroadcastHandler就在BroadcastQueue中

接着看看BroadcastQueue.BroadcastHanlder的handleMessage方法

```java
private final class BroadcastHandler extends Handler {
    public BroadcastHandler(Looper looper) {
        super(looper, null, true);
    }

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case BROADCAST_INTENT_MSG: {
                if (DEBUG_BROADCAST) Slog.v(
                        TAG_BROADCAST, "Received BROADCAST_INTENT_MSG");
                //接着调用
                processNextBroadcast(true);
            } break;
            case BROADCAST_TIMEOUT_MSG: {
                synchronized (mService) {
                    broadcastTimeoutLocked(true);
                }
            } break;
        }
    }
}
```

这个方法中处理了两种消息，其中一种是超时的，另外一种就是我们要发送的广播消息了，然后接着又调用了BroadcastQueue的processNextBroadcast方法

接着看BroadcastQueue的processNextBroadcast方法

```java
final void processNextBroadcast(boolean fromMsg) {
    synchronized(mService) {
        BroadcastRecord r;

        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "processNextBroadcast ["
                + mQueueName + "]: "
                + mParallelBroadcasts.size() + " parallel broadcasts, "
                + mOrderedBroadcasts.size() + " ordered broadcasts");

        mService.updateCpuStats();

        if (fromMsg) {
            mBroadcastsScheduled = false;
        }

        // First, deliver any non-serialized broadcasts right away.
        while (mParallelBroadcasts.size() > 0) {
            r = mParallelBroadcasts.remove(0);
            r.dispatchTime = SystemClock.uptimeMillis();
            r.dispatchClockTime = System.currentTimeMillis();
            ...
            final int N = r.receivers.size();
            ...
            for (int i=0; i<N; i++) {
                Object target = r.receivers.get(i);
                if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                        "Delivering non-ordered on [" + mQueueName + "] to registered "
                        + target + ": " + r);
                //无序广播存储在mParallelBroadcasts中，系统会遍历并将其中的广播发送给它们所有的接收者
                deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
            }
            addBroadcastToHistoryLocked(r);
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Done with parallel broadcast ["
                    + mQueueName + "] " + r);
        }

        // Now take care of the next serialized one...

        // If we are waiting for a process to come up to handle the next
        // broadcast, then do nothing at this point.  Just in case, we
        // check that the process we're waiting for still exists.
        ...
        // Get the next receiver...
        int recIdx = r.nextReceiver++;

        ...

        final BroadcastOptions brOptions = r.options;
        final Object nextReceiver = r.receivers.get(recIdx);

        if (nextReceiver instanceof BroadcastFilter) {
            // Simple case: this is a registered receiver who gets
            // a direct call.
            ...
            if (r.receiver == null || !r.ordered) {
                // The receiver has already finished, so schedule to
                // process the next one.
                if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Quick finishing ["
                        + mQueueName + "]: ordered="
                        + r.ordered + " receiver=" + r.receiver);
                r.state = BroadcastRecord.IDLE;
                //下一个接收者接着调用该方法
                scheduleBroadcastsLocked();
            } else {
                ...
            }
            return;
        }

        // Hard case: need to instantiate the receiver, possibly
        // starting its application process to host it.

        ...
    }
}
```

这个方法中的代码也有很多，但是我们发送标准的广播就在方法开始的前面几行，所以后面的代码都是对其他情况的处理，以及下一个接收者的方法调用。好了，我们重点关注前面的循环，无序广播存储在mParallelBroadcasts中，系统会遍历并将其中的广播通过deliverToRegisteredReceiverLocked方法发送给它们所有的接收者

接着看BroadcastQueue的deliverToRegisteredReceiverLocked方法

```java
private void deliverToRegisteredReceiverLocked(BroadcastRecord r,
        BroadcastFilter filter, boolean ordered, int index) {
    boolean skip = false;
    ...
    try {
        if (DEBUG_BROADCAST_LIGHT) Slog.i(TAG_BROADCAST,
                "Delivering to " + filter + " : " + r);
        if (filter.receiverList.app != null && filter.receiverList.app.inFullBackup) {
            // Skip delivery if full backup in progress
            // If it's an ordered broadcast, we need to continue to the next receiver.
            if (ordered) {
                skipReceiverLocked(r);
            }
        } else {
            //就是这里了
            performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                    new Intent(r.intent), r.resultCode, r.resultData,
                    r.resultExtras, r.ordered, r.initialSticky, r.userId);
        }
        if (ordered) {
            r.state = BroadcastRecord.CALL_DONE_RECEIVE;
        }
    } catch (RemoteException e) {
        Slog.w(TAG, "Failure sending broadcast " + r.intent, e);
        if (ordered) {
            r.receiver = null;
            r.curFilter = null;
            filter.receiverList.curBroadcast = null;
            if (filter.receiverList.app != null) {
                filter.receiverList.app.curReceivers.remove(r);
            }
        }
    }
}
```

好吧，这个方法的代码也不少，不过前面大多数都是if，说明是一些情况的处理，那么直接看到这个方法的最后，try里面有`performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver, new Intent(r.intent), r.resultCode, r.resultData, r.resultExtras, r.ordered, r.initialSticky, r.userId);`注意方法名，Receive，不是Receiver了，所以从这里开始，我们就要开始准备**接收**了哦

------

小分割线，开始准备接收广播…..

------

接着看BroadcastQueue的performReceiveLocked方法

```java
void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
        Intent intent, int resultCode, String data, Bundle extras,
        boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
    // Send the intent to the receiver asynchronously using one-way binder calls.
    if (app != null) {
        if (app.thread != null) {
            // If we have an app thread, do the call through that so it is
            // correctly ordered with other one-way calls.
            try {
                //终于看到了熟悉的ApplicationThread了
                app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                        data, extras, ordered, sticky, sendingUser, app.repProcState);
            ...
            } catch (RemoteException ex) {
                ...
            }
        } else {
            ...
        }
    } else {
        receiver.performReceive(intent, resultCode, data, extras, ordered,
                sticky, sendingUser);
    }
}
```

由于接收广播会调起应用程序，所以app.thread不为null，所以就会执行到ApplicationThread中了

接着看ApplicationThread的scheduleRegisteredReceiver方法

```java
public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
        int resultCode, String dataStr, Bundle extras, boolean ordered,
        boolean sticky, int sendingUser, int processState) throws RemoteException {
    updateProcessState(processState, false);
    receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
            sticky, sendingUser);
}
```

这里代码就比较少了，通过IIntentReceiver调用其performReceive方法，我们知道这个receiver实际上是InnerReceiver，那么接下来就是通过InnerReceiver来实现接收了

接着看LoadedApk.ReceiverDiaptcher.InnerReceiver的performReceive方法

```java
@Override
public void performReceive(Intent intent, int resultCode, String data,
        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    final LoadedApk.ReceiverDispatcher rd;
    if (intent == null) {
        Log.wtf(TAG, "Null intent received");
        rd = null;
    } else {
        //拿到ReceiverDispatcher
        rd = mDispatcher.get();
    }
    ...
    if (rd != null) {
        //接着调用ReceiverDispatcher的performReceive方法
        rd.performReceive(intent, resultCode, data, extras,
                ordered, sticky, sendingUser);
    } else {
        ...
    }
}
```

在InnerReceiver中会先拿到ReceiverDispatcher，接着调用其performReceive方法

接着看ReceiverDispatcher的performReceive方法

```java
public void performReceive(Intent intent, int resultCode, String data,
        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    //创建Args实例，内部实现了Runnable
    final Args args = new Args(intent, resultCode, data, extras, ordered,
            sticky, sendingUser);
    if (intent == null) {
        Log.wtf(TAG, "Null intent received");
    } else {
        if (ActivityThread.DEBUG_BROADCAST) {
            int seq = intent.getIntExtra("seq", -1);
            Slog.i(ActivityThread.TAG, "Enqueueing broadcast " + intent.getAction()
                    + " seq=" + seq + " to " + mReceiver);
        }
    }
    //mActivityThread就是ActivityThread中的H这个Handler，然后通过post方法来执行这个Args对象的run方法中的逻辑
    if (intent == null || !mActivityThread.post(args.getRunnable())) {
        if (mRegistered && ordered) {
            IActivityManager mgr = ActivityManager.getService();
            if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                    "Finishing sync broadcast to " + mReceiver);
            args.sendFinished(mgr);
        }
    }
}
```

在这里会先创建一个Args实例，这个类实现了Runnable，然后通过mActivityThread的post方法执Args的run方法，mActivityThread就是ActivityThread的H

接着就看看Args中Runnable

```java
public final Runnable getRunnable() {
    return () -> {
        final BroadcastReceiver receiver = mReceiver;
        final boolean ordered = mOrdered;

        ...

        final IActivityManager mgr = ActivityManager.getService();
        final Intent intent = mCurIntent;
        ...

        mCurIntent = null;
        mDispatched = true;
        mPreviousRunStacktrace = new Throwable("Previous stacktrace");
        if (receiver == null || intent == null || mForgotten) {
            if (mRegistered && ordered) {
                if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                        "Finishing null broadcast to " + mReceiver);
                sendFinished(mgr);
            }
            return;
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastReceiveReg");
        try {
            ClassLoader cl = mReceiver.getClass().getClassLoader();
            intent.setExtrasClassLoader(cl);
            intent.prepareToEnterProcess();
            setExtrasClassLoader(cl);
            //在这里就调用onReceive方法
            receiver.setPendingResult(this);
            receiver.onReceive(mContext, intent);
        } catch (Exception e) {
            ...
        }

        if (receiver.getPendingResult() != null) {
            finish();
        }
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    };
}
```

在这个方法中我们见到了熟悉的onReceive方法，最终就会回调到我们写onReceive方法那里。所以发送广播后，系统经过一定条件的筛选，最后回调到我们自己写的onReceive方法

## 感谢任玉刚大神的**Android开发艺术探索**