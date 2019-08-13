---
title: Android Service源码分析
tag: Android

---

<meta name="referrer" content="no-referrer" />



[TOC]

# Service源码分析

[Service脑图](http://naotu.baidu.com/file/2b9806029008a9bbead841f4893d12d3?token=f34d86e3c5c4eaf2)

## Service的启动过程

我们通过`startService(Intent service);`方法来启动一个Service，跟进源码。调用的是ContextWrapper.startService()

```java
@Override
public ComponentName startService(Intent service) {
    return mBase.startService(service);
}
```

这里并没有做什么事情，ContextWrapper是这里用到的一种桥接模式，所有相关函数（Activity相关、Service等）的调用都会先到这个类，但实际还是Context来实现的。调用mBase的startService方法，mBase是一个Context对象，而ContextImpl才是Context的具体实现类，是在Activity创建的时候通过attach关联起来的。

接下来看看ContextImpl的startService方法

```java
@Override
public ComponentName startService(Intent service) {
    //如果系统进程直接调用该方法，就会抛出警告
    warnIfCallingFromSystemProcess();
    //false，不需要前台进程；mUser是一个UserHandle
    return startServiceCommon(service, false, mUser);
}

private ComponentName startServiceCommon(Intent service, boolean requireForeground,
        UserHandle user) {
    try {
        //Service的验证
        validateServiceIntent(service);
        //离开进程准备
        service.prepareToLeaveProcess(this);
        //重点，通过AMS来startService，返回需要的ComPonentName
        //mMainThread.getApplicationThread()主线程的ApplicationThread
        ComponentName cn = ActivityManager.getService().startService(
            mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                        getContentResolver()), requireForeground,
                        getOpPackageName(), user.getIdentifier());
        ...
        return cn;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

ContextImpl的startService方法调用后，会接着调用内部的startServiceCommon方法，在startServiceCommon中先进行了一些少数的准备，接着调用ActivityManager.getService().startService()方法，ActivityManager.getService()是IActivityManager，其实例就是ActivityManagerService对象

接着看AMS（ActivityManagerService)的startService方法

```java
@Override
public ComponentName startService(IApplicationThread caller, Intent service,
        String resolvedType, boolean requireForeground, String callingPackage, int userId)
        throws TransactionTooLargeException {
    //一些检验
    ...
    synchronized(this) {
        //进程id
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        ComponentName res;
        try {
            //mServices是一个ActivityServices对象
            res = mServices.startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid,
                    requireForeground, callingPackage, userId);
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
        return res;
    }
}
```

在AMS的startService方法中，在一个synchronized代码块中，获取了需要用到的进程id、Binder标识等，然后通过mServices这个ActivityServices对象，调用了其startServiceLocked()方法，ActivityServices跟ActivityStarter的作用是类似的，用来辅助AMS管理Service的类，例如缓存、启动、绑定、停止等

接着看ActivityServices的startServiceLocked方法

```java
ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
        int callingPid, int callingUid, boolean fgRequired, String callingPackage, final int userId)
        throws TransactionTooLargeException {
    ...
    //检索Service
    ServiceLookupResult res =
        retrieveServiceLocked(service, resolvedType, callingPackage,
                callingPid, callingUid, userId, true, callerFg, false);
    ...
    //获取ServiceRecord，一个Service的记录，一直贯穿整个启动过程
    ServiceRecord r = res.record;

    ...

    //权限检查
    if (mAm.mPermissionReviewRequired) {
        if (!requestStartTargetPermissionsReviewIfNeededLocked(r, callingPackage,
                callingUid, service, callerFg, userId)) {
            return null;
        }
    }

    ...

    final ServiceMap smap = getServiceMapLocked(r.userId);

    ...
    //接着调用内部的startServiceInnerLocked方法
    ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
    return cmp;
}
```

startServiceLocked方法前面基本都是一些检查、检索，到最后又调用了内部的startServiceInnerLocked方法

接着看ActivityServices的startServiceInnerLocked方法

```java
ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
        boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
    ServiceState stracker = r.getTracker();
    if (stracker != null) {
        stracker.setStarted(true, mAm.mProcessStats.getMemFactorLocked(), r.lastActivity);
    }
    r.callStart = false;
    synchronized (r.stats.getBatteryStats()) {
        r.stats.startRunningLocked();
    }
    //后续的工作完成
    String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
    if (error != null) {
        return new ComponentName("!!", error);
    }

    //需要为前台Service的处理
    if (r.startRequested && addToStarting) {
        ...
    } else if (callerFg || r.fgRequired) {
        smap.ensureNotStartingBackgroundLocked(r);
    }

    return r.name;
}
```

这里面并没有做太多的工作，具体的启动过程也没有

所以要接着看ActivityServices的bringUpServiceLocked方法

```java
private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
        boolean whileRestarting, boolean permissionsReviewRequired)
        throws TransactionTooLargeException {
    //检验，其他情况的处理
    ...

    // Service is now being launched, its package can't be stopped.
    ...

    final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
    final String procName = r.processName;
    String hostingType = "service";
    ProcessRecord app;

    if (!isolated) {
        app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
        if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                    + " app=" + app);
        if (app != null && app.thread != null) {
            try {
                app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
                //真正启动的地方
                realStartServiceLocked(r, app, execInFg);
                return null;
            } catch (TransactionTooLargeException e) {
                throw e;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting service " + r.shortName, e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }
    } else {
        ...
    }

    // Not running -- get it started, and enqueue this service record
    // to be executed when the app comes up.
    ...

    return null;
}
```

bringUpServiceLocked方法中做了一些其他情况的处理，但真正启动Service却是在realStartServiceLocked方法中

接着看ActivityServices.realStartServiceLocked方法

```java
private final void realStartServiceLocked(ServiceRecord r,
        ProcessRecord app, boolean execInFg) throws RemoteException {
    ...

    final boolean newService = app.services.add(r);
    bumpServiceExecutingLocked(r, execInFg, "create");
    mAm.updateLruProcessLocked(app, false, null);
    updateServiceForegroundLocked(r.app, /* oomAdj= */ false);
    mAm.updateOomAdjLocked();

    boolean created = false;
    try {
        if (LOG_SERVICE_START_STOP) {
            String nameTerm;
            int lastPeriod = r.shortName.lastIndexOf('.');
            nameTerm = lastPeriod >= 0 ? r.shortName.substring(lastPeriod) : r.shortName;
            EventLogTags.writeAmCreateService(
                    r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
        }
        synchronized (r.stats.getBatteryStats()) {
            r.stats.startLaunchedLocked();
        }
        mAm.notifyPackageUse(r.serviceInfo.packageName,
                             PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
        app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
        //通过ApplicationThread来创建一个Service
        app.thread.scheduleCreateService(r, r.serviceInfo,
                mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                app.repProcState);
        r.postNotification();
        created = true;
    } catch (DeadObjectException e) {
        Slog.w(TAG, "Application dead when creating service " + r);
        mAm.appDiedLocked(app);
        throw e;
    } finally {
        if (!created) {
            // Keep the executeNesting count accurate.
            final boolean inDestroying = mDestroyingServices.contains(r);
            //出错时停止Service的一些操作
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);

            // Cleanup.
            if (newService) {
                app.services.remove(r);
                r.app = null;
            }

            // Retry.
            if (!inDestroying) {
                scheduleServiceRestartLocked(r, false);
            }
        }
    }

    if (r.whitelistManager) {
        app.whitelistManager = true;
    }
    //这里就是后面要分析的绑定过程
    requestServiceBindingsLocked(r, execInFg);

    updateServiceClientActivitiesLocked(app, null, true);

    // If the service is in the started state, and there are no
    // pending arguments, then fake up one so its onStartCommand() will
    // be called.
    ...
    //通过该方法来调用Service其他的方法，如onStartCommand()
    sendServiceArgsLocked(r, execInFg, true);

    ...
}
```

realStartServiceLocked包括了Service的很多操作，先是通过ApplicationThread.scheduleCreateService来创建了Service，接着通过requestServiceBindingsLocked方法进行绑定，再通过sendServiceArgsLocked方法进行其他方法的调用，如onStartCommand()。在sendServiceArgsLocked方法中，又会调用ActivityServices.sendServiceArgsLocked()方法，接着通过ApplicationThread.scheduleServiceArgs()方法，最后调用了Service的onStartCommand方法。

由于分析的是启动过程，所以这里主要看`app.thread.scheduleCreateService(r, r.serviceInfo， mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo), app.repProcState);`这行代码，app.thread是IApplicationThread，它的实现类是ApplicationThread，这个类是ActivityThread的内部类

接着在ActivityThread中看ApplicationThread的scheduleCreateService方法

```java
public final void scheduleCreateService(IBinder token,
            ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
        updateProcessState(processState, false);
        CreateServiceData s = new CreateServiceData();
        s.token = token;
        s.info = info;
        s.compatInfo = compatInfo;
        //发送CREATE_SERVICE的消息创建Service
        sendMessage(H.CREATE_SERVICE, s);
    }
```

scheduleCreateService方法代码很少也很简单，创建了一个CreateServiceData的对象，这个对象就是创建Service时需要的信息，然后调用sendMessage方法

接着看ActivityThread的sendMessage方法

```java
private void sendMessage(int what, Object obj) {
    sendMessage(what, obj, 0, 0, false);
}

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
    mH.sendMessage(msg);
}
```

第一个sendMessage方法没啥，后面又重载了一个，所以重点在第二个sendMessage方法，基本上四大组件最后都会在ActivityThread中通过这个函数发送消息。这里，就是重新打包了一下Message，然后通过mH来发送。mH是一个Handler，是ActivityThread内部的一个类，继承自Hanlder。发送完消息后，自然要在H中的来处理消息。

接着看H.handleMessage方法

```java
public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
           ...
           //这里主要留着关于Service的，包括create、bind、unbind、stop等
            case CREATE_SERVICE:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj)));
                handleCreateService((CreateServiceData)msg.obj);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
            case BIND_SERVICE:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceBind");
                handleBindService((BindServiceData)msg.obj);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
            case UNBIND_SERVICE:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceUnbind");
                handleUnbindService((BindServiceData)msg.obj);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
            case SERVICE_ARGS:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceStart: " + String.valueOf(msg.obj)));
                handleServiceArgs((ServiceArgsData)msg.obj);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
            case STOP_SERVICE:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceStop");
                handleStopService((IBinder)msg.obj);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
            ...
        }
        ...
    }
}
```

前面我们发送的消息是CREATE_SERVICE，所以就会调用`handleCreateService((CreateServiceData)msg.obj);`
在这里可以看见，处理了很多消息，关于Activity、Window等各种的，所以这个H很重要。

接着看ActivityThread的handleCreateService()方法

```java
private void handleCreateService(CreateServiceData data) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();

    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    Service service = null;
    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        //这里就创建了Service对象，通过类加载器
        service = (Service) cl.loadClass(data.info.name).newInstance();
    } catch (Exception e) {
        if (!mInstrumentation.onException(service, e)) {
            throw new RuntimeException(
                "Unable to instantiate service " + data.info.name
                + ": " + e.toString(), e);
        }
    }

    try {
        if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
        //创建了ContextImpl对象
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);

        Application app = packageInfo.makeApplication(false, mInstrumentation);
        //和Application等关联
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManager.getService());
        //create
        service.onCreate();
        //放入Service的列表缓存
        mServices.put(data.token, service);
        try {
            //通过AMS来通知创建完成
            ActivityManager.getService().serviceDoneExecuting(
                    data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(service, e)) {
            throw new RuntimeException(
                "Unable to create service " + data.info.name
                + ": " + e.toString(), e);
        }
    }
}
```

到这里，终于看见了Service的创建了，这里的代码很简单。拿到类加载器器，接着通过类加载器创建了Serivce，然后创建ContextImpl对象通过Service的attach关联，最后调用Service的onCreate。

## Service的绑定过程

首先`bindService(Intent service, ServiceConnection conn, int flags);`，接着就是ContentWrapper中bindService方法

```java
@Override
public boolean bindService(Intent service, ServiceConnection conn,
        int flags) {
    return mBase.bindService(service, conn, flags);
}
```

没啥看的，跟Service创建讲的一样，最终还是在ContextImpl中

接着看ContentImpl中的bindService方法

```java
@Override
public boolean bindService(Intent service, ServiceConnection conn,
        int flags) {
    warnIfCallingFromSystemProcess();
    return bindServiceCommon(service, conn, flags, mMainThread.getHandler(),
            Process.myUserHandle());
}
```

然而这里面并没有什么太多的东西，需要注意的一点就是mMainThread.getHandler()，mMainThread是一个ActivityThread对象，getHandler返回的是ActivityThread中的H，返回了bindServiceCommon()

接着看ContextImpl.bindServiceCommon方法

```java
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
        handler, UserHandle user) {
    // Keep this in sync with DevicePolicyManager.bindDeviceAdminServiceAsUser.
    IServiceConnection sd;
    if (conn == null) {
        throw new IllegalArgumentException("connection is null");
    }
    if (mPackageInfo != null) {
        //在这里将ServiceConnection转换为了ServiceDispatcher.InnerConnection对象，且初始化了ServiceDispatcher对象
        sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
    } else {
        throw new RuntimeException("Not supported in system context");
    }
    validateServiceIntent(service);
    try {
        IBinder token = getActivityToken();
        if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                && mPackageInfo.getApplicationInfo().targetSdkVersion
                < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
            flags |= BIND_WAIVE_PRIORITY;
        }
        service.prepareToLeaveProcess(this);
        //还是通过AMS来绑定
        int res = ActivityManager.getService().bindService(
            mMainThread.getApplicationThread(), getActivityToken(), service,
            service.resolveTypeIfNeeded(getContentResolver()),
            sd, flags, getOpPackageName(), user.getIdentifier());
        if (res < 0) {
            throw new SecurityException(
                    "Not allowed to bind to service " + service);
        }
        return res != 0;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

在这里，首先是将客户端的ServiceConnection对象转换为ServiceDispatcher.InnerConnection对象，之所以不能直接使用ServiceConnection对象是因为服务的绑定有可能是跨进程的，跨进程必须借助Binder，而ServiceDispatcher.InnerConnection刚好充当了Binder的角色。ServiceDispatcher起着链接ServiceConnection和InnerConnection的作用（在取消绑定的时候ServiceDispatcher也会用到）这个过程是由LoadedApk的getServiceDispatcher方法完成。

```java
public final IServiceConnection getServiceDispatcher(ServiceConnection c,
        Context context, Handler handler, int flags) {
    synchronized (mServices) {
        LoadedApk.ServiceDispatcher sd = null;
        //ServiceConnection和ServiceDispatcher是一一对应的
        ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);
        if (map != null) {
            if (DEBUG) Slog.d(TAG, "Returning existing dispatcher " + sd + " for conn " + c);
            sd = map.get(c);
        }
        if (sd == null) {
            //如果这个ServiceConnection对应的ServiceDispathcer没有就创建一个
            sd = new ServiceDispatcher(c, context, handler, flags);
            if (DEBUG) Slog.d(TAG, "Creating new dispatcher " + sd + " for conn " + c);
            if (map == null) {
                map = new ArrayMap<>();
                mServices.put(context, map);
            }
            map.put(c, sd);
        } else {
            sd.validate(context, handler);
        }
        return sd.getIServiceConnection();
    }
}
```

系统会首先查找是否存在相同的ServiceConnection，不存在就重新创建一个ServiceDispathcer对象并存储在mServices（`private final ArrayMap<Context, ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>> mServices = new ArrayMap<>();`）中，在SerivceDispatcher的内部又保存了ServiceConnection和InnerConnection。当Service和客户端建立连接后，会通过InnerConnection来调用ServiceConnection中的onServiceConnected方法，又可能跨进程

言归正传，最后还是得通过AMS来进行绑定

接着看AMS的bindService方法

```java
public int bindService(IApplicationThread caller, IBinder token, Intent service,
        String resolvedType, IServiceConnection connection, int flags, String callingPackage,
        int userId) throws TransactionTooLargeException {
    enforceNotIsolatedCaller("bindService");
    ...

    synchronized(this) {
        return mServices.bindServiceLocked(caller, token, service,
                resolvedType, connection, flags, callingPackage, userId);
    }
}
```

这里AMS并没有做什么实际的事，交给了mServices，mServices是一个ActivityServices对象，调用了其bindServiceLocked方法

接着看ActivityServices的bindServiceLocked方法

```java
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
        String resolvedType, final IServiceConnection connection, int flags,
        String callingPackage, final int userId) throws TransactionTooLargeException {
    ...
    final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
    ...

    ActivityRecord activity = null;
    if (token != null) {
        activity = ActivityRecord.isInStackLocked(token);
        if (activity == null) {
            Slog.w(TAG, "Binding with unknown activity: " + token);
            return 0;
        }
    }

    ...
    //检索Service，获取其状态
    ServiceLookupResult res =
        retrieveServiceLocked(service, resolvedType, callingPackage, Binder.getCallingPid(),
                Binder.getCallingUid(), userId, true, callerFg, isBindExternal);
    ...
    ServiceRecord s = res.record;

    boolean permissionsReviewRequired = false;

    // If permissions need a review before any of the app components can run,
    // we schedule binding to the service but do not start its process, then
    // we launch a review activity to which is passed a callback to invoke
    // when done to start the bound service's process to completing the binding.
    if (mAm.mPermissionReviewRequired) {
        if (mAm.getPackageManagerInternalLocked().isPermissionsReviewRequired(
                s.packageName, s.userId)) {

            permissionsReviewRequired = true;

            // Show a permission review UI only for binding from a foreground app
            if (!callerFg) {
                Slog.w(TAG, "u" + s.userId + " Binding to a service in package"
                        + s.packageName + " requires a permissions review");
                return 0;
            }

            final ServiceRecord serviceRecord = s;
            final Intent serviceIntent = service;

            RemoteCallback callback = new RemoteCallback(
                    new RemoteCallback.OnResultListener() {
                @Override
                public void onResult(Bundle result) {
                    synchronized(mAm) {
                        final long identity = Binder.clearCallingIdentity();
                        try {
                            if (!mPendingServices.contains(serviceRecord)) {
                                return;
                            }
                            // If there is still a pending record, then the service
                            // binding request is still valid, so hook them up. We
                            // proceed only if the caller cleared the review requirement
                            // otherwise we unbind because the user didn't approve.
                            if (!mAm.getPackageManagerInternalLocked()
                                    .isPermissionsReviewRequired(
                                            serviceRecord.packageName,
                                            serviceRecord.userId)) {
                                try {
                                    //接着调用该方法进行绑定
                                    bringUpServiceLocked(serviceRecord,
                                            serviceIntent.getFlags(),
                                            callerFg, false, false);
                                } catch (RemoteException e) {
                                    /* ignore - local call */
                                }
                            } else {
                                //取消绑定
                                unbindServiceLocked(connection);
                            }
                        } finally {
                            Binder.restoreCallingIdentity(identity);
                        }
                    }
                }
            });

            final Intent intent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                    | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
            intent.putExtra(Intent.EXTRA_PACKAGE_NAME, s.packageName);
            intent.putExtra(Intent.EXTRA_REMOTE_CALLBACK, callback);

            if (DEBUG_PERMISSIONS_REVIEW) {
                Slog.i(TAG, "u" + s.userId + " Launching permission review for package "
                        + s.packageName);
            }

            mAm.mHandler.post(new Runnable() {
                @Override
                public void run() {
                    mAm.mContext.startActivityAsUser(intent, new UserHandle(userId));
                }
            });
        }
    }

    final long origId = Binder.clearCallingIdentity();

    try {
        if (unscheduleServiceRestartLocked(s, callerApp.info.uid, false)) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "BIND SERVICE WHILE RESTART PENDING: "
                    + s);
        }

        if ((flags&Context.BIND_AUTO_CREATE) != 0) {
            s.lastActivity = SystemClock.uptimeMillis();
            if (!s.hasAutoCreateConnections()) {
                // This is the first binding, let the tracker know.
                ServiceState stracker = s.getTracker();
                if (stracker != null) {
                    stracker.setBound(true, mAm.mProcessStats.getMemFactorLocked(),
                            s.lastActivity);
                }
            }
        }

        mAm.startAssociationLocked(callerApp.uid, callerApp.processName, callerApp.curProcState,
                s.appInfo.uid, s.name, s.processName);
        // Once the apps have become associated, if one of them is caller is ephemeral
        // the target app should now be able to see the calling app
        mAm.grantEphemeralAccessLocked(callerApp.userId, service,
                s.appInfo.uid, UserHandle.getAppId(callerApp.uid));

        AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
        ConnectionRecord c = new ConnectionRecord(b, activity,
                connection, flags, clientLabel, clientIntent);

        IBinder binder = connection.asBinder();
        ArrayList<ConnectionRecord> clist = s.connections.get(binder);
        if (clist == null) {
            clist = new ArrayList<ConnectionRecord>();
            s.connections.put(binder, clist);
        }
        clist.add(c);
        b.connections.add(c);
        if (activity != null) {
            if (activity.connections == null) {
                activity.connections = new HashSet<ConnectionRecord>();
            }
            activity.connections.add(c);
        }
        b.client.connections.add(c);
        if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
            b.client.hasAboveClient = true;
        }
        if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {
            s.whitelistManager = true;
        }
        if (s.app != null) {
            updateServiceClientActivitiesLocked(s.app, c, true);
        }
        clist = mServiceConnections.get(binder);
        if (clist == null) {
            clist = new ArrayList<ConnectionRecord>();
            mServiceConnections.put(binder, clist);
        }
        clist.add(c);

        if ((flags&Context.BIND_AUTO_CREATE) != 0) {
            s.lastActivity = SystemClock.uptimeMillis();
            if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                    permissionsReviewRequired) != null) {
                return 0;
            }
        }

        if (s.app != null) {
            if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                s.app.treatLikeActivity = true;
            }
            if (s.whitelistManager) {
                s.app.whitelistManager = true;
            }
            // This could have made the service more important.
            mAm.updateLruProcessLocked(s.app, s.app.hasClientActivities
                    || s.app.treatLikeActivity, b.client);
            mAm.updateOomAdjLocked(s.app, true);
        }

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bind " + s + " with " + b
                + ": received=" + b.intent.received
                + " apps=" + b.intent.apps.size()
                + " doRebind=" + b.intent.doRebind);

        if (s.app != null && b.intent.received) {
            // Service is already running, so we can immediately
            // publish the connection.
            try {
                c.conn.connected(s.name, b.intent.binder, false);
            } catch (Exception e) {
                Slog.w(TAG, "Failure sending service " + s.shortName
                        + " to connection " + c.conn.asBinder()
                        + " (in " + c.binding.client.processName + ")", e);
            }

            // If this is the first app connected back to this binding,
            // and the service had previously asked to be told when
            // rebound, then do so.
            if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                requestServiceBindingLocked(s, b.intent, callerFg, true);
            }
        } else if (!b.intent.requested) {
            requestServiceBindingLocked(s, b.intent, callerFg, false);
        }

        getServiceMapLocked(s.userId).ensureNotStartingBackgroundLocked(s);

    } finally {
        Binder.restoreCallingIdentity(origId);
    }

    return 1;
}
```

在这里又调用了bringUpServiceLocked方法，这个方法是不是很眼熟？在前面启动Service的时候，也会调用到这个方法，所以后面的就跟Service的启动一样了。但是我们前面讲创建的时候留了一个东西，就是Service创建成功后，绑定的方法我们并没有讲。那么就这前面的讲。

调用了ActivityServices的bringUpServiceLocked方法后，又会接着调用ActivityServices.realStartServiceLocked()，这个方法前面启动Service时也讲到过

我们再看看这个方法

```java
private final void realStartServiceLocked(ServiceRecord r,
        ProcessRecord app, boolean execInFg) throws RemoteException {
    ...
    r.app = app;
    r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

    final boolean newService = app.services.add(r);
    bumpServiceExecutingLocked(r, execInFg, "create");
    mAm.updateLruProcessLocked(app, false, null);
    updateServiceForegroundLocked(r.app, /* oomAdj= */ false);
    mAm.updateOomAdjLocked();

    boolean created = false;
    try {
        ...
        //创建Service
        app.thread.scheduleCreateService(r, r.serviceInfo,
                mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                app.repProcState);
        r.postNotification();
        created = true;
    } catch (DeadObjectException e) {
        Slog.w(TAG, "Application dead when creating service " + r);
        mAm.appDiedLocked(app);
        throw e;
    } finally {
        ...
    }

    ...

    //这里就是我们前面留着的绑定Service
    requestServiceBindingsLocked(r, execInFg);

    updateServiceClientActivitiesLocked(app, null, true);

    ...
    //通过该方法来调用Service其他的方法，如onStartCommand()
    sendServiceArgsLocked(r, execInFg, true);
    ...
}
```

这个方法很熟悉了吧，但是接下来我们就不去看Service怎么创建的了。我们这里主要看Service怎样绑定的

接着ActivityServices的requestServiceBindingsLocked方法

```java
private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)
        throws TransactionTooLargeException {
    for (int i=r.bindings.size()-1; i>=0; i--) {
        IntentBindRecord ibr = r.bindings.valueAt(i);
        if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
            break;
        }
    }
}
```

这里面并没有做太多的事，遍历所有的IntentBindRecord，接着还是调用重载的requestServiceBindingsLocked方法

接着看这个重载的requestServiceBindingsLocked方法

```java
private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
        boolean execInFg, boolean rebind) throws TransactionTooLargeException {
    if (r.app == null || r.app.thread == null) {
        // If service is not currently running, can't yet bind.
        return false;
    }
    if (DEBUG_SERVICE) Slog.d(TAG_SERVICE, "requestBind " + i + ": requested=" + i.requested
            + " rebind=" + rebind);
    if ((!i.requested || rebind) && i.apps.size() > 0) {
        try {
            bumpServiceExecutingLocked(r, execInFg, "bind");
            r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            //熟悉的app.thread又来了
            r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                    r.app.repProcState);
            if (!rebind) {
                i.requested = true;
            }
            i.hasBound = true;
            i.doRebind = false;
        } catch (TransactionTooLargeException e) {
            // Keep the executeNesting count accurate.
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while binding " + r, e);
            final boolean inDestroying = mDestroyingServices.contains(r);
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);
            throw e;
        } catch (RemoteException e) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while binding " + r);
            // Keep the executeNesting count accurate.
            final boolean inDestroying = mDestroyingServices.contains(r);
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);
            return false;
        }
    }
    return true;
}
```

在这个方法中，从ServiceRecord中拿到ApplicationThread，然后调用scheduleBindService方法

接着看ActivityThread.ApplicationThread.scheduleBindService()

```java
public final void scheduleBindService(IBinder token, Intent intent,
            boolean rebind, int processState) {
        updateProcessState(processState, false);
        BindServiceData s = new BindServiceData();
        s.token = token;
        s.intent = intent;
        s.rebind = rebind;

        if (DEBUG_SERVICE)
            Slog.v(TAG, "scheduleBindService token=" + token + " intent=" + intent + " uid="
                    + Binder.getCallingUid() + " pid=" + Binder.getCallingPid());
        sendMessage(H.BIND_SERVICE, s);
    }
```

熟悉的一切，跟Service的创建是一样的。在这里会调用ActivityThread的sendMessage()方法，然后重载一次，通过mH这个Handler的sendMessage发送。然后在H中的handleMessage中处理消息，接着又调用ActivityThread.handleBindService()

接着看看ActivityThread.handleBindService()这个方法

```java
private void handleBindService(BindServiceData data) {
    //获取之前创建的Service
    Service s = mServices.get(data.token);
    if (DEBUG_SERVICE)
        Slog.v(TAG, "handleBindService s=" + s + " rebind=" + data.rebind);
    if (s != null) {
        try {
            data.intent.setExtrasClassLoader(s.getClassLoader());
            data.intent.prepareToEnterProcess();
            try {
                if (!data.rebind) {
                    //绑定
                    IBinder binder = s.onBind(data.intent);
                    //通知客户端绑定成功
                    ActivityManager.getService().publishService(
                            data.token, data.intent, binder);
                } else {
                    s.onRebind(data.intent);
                    ActivityManager.getService().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                }
                ensureJitEnabled();
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(s, e)) {
                throw new RuntimeException(
                        "Unable to bind to service " + s
                        + " with " + data.intent + ": " + e.toString(), e);
            }
        }
    }
}
```

到这里，Service通过onBind方法，就绑定完成了。然后系统会通知客户端，接着看看如何通知客户端的

AMS.publishService方法

```java
public void publishService(IBinder token, Intent intent, IBinder service) {
    // Refuse possible leaked file descriptors
    if (intent != null && intent.hasFileDescriptors() == true) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

    synchronized(this) {
        if (!(token instanceof ServiceRecord)) {
            throw new IllegalArgumentException("Invalid service token");
        }
        mServices.publishServiceLocked((ServiceRecord)token, intent, service);
    }
}
```

从代码中可以看出，最终还是交到了ActivityServices中处理

接着看ActivityServices的publishServiceLocked方法

```java
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
    final long origId = Binder.clearCallingIdentity();
    try {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "PUBLISHING " + r
                + " " + intent + ": " + service);
        if (r != null) {
            Intent.FilterComparison filter
                    = new Intent.FilterComparison(intent);
            IntentBindRecord b = r.bindings.get(filter);
            if (b != null && !b.received) {
                b.binder = service;
                b.requested = true;
                b.received = true;
                for (int conni=r.connections.size()-1; conni>=0; conni--) {
                    ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
                    for (int i=0; i<clist.size(); i++) {
                        ConnectionRecord c = clist.get(i);
                        ...
                        try {
                            //这里才是重点
                            c.conn.connected(r.name, service, false);
                        } catch (Exception e) {
                            Slog.w(TAG, "Failure sending service " + r.name +
                                  " to connection " + c.conn.asBinder() +
                                  " (in " + c.binding.client.processName + ")", e);
                        }
                    }
                }
            }

            serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
        }
    } finally {
        Binder.restoreCallingIdentity(origId);
    }
}
```

c.conn是ServiceDispatcher.InnerConnection

所以接着看看LoadedApk.ServiceDispatcher.InnerConnection的connected方法

```java
public void connected(ComponentName name, IBinder service, boolean dead)
        throws RemoteException {
    LoadedApk.ServiceDispatcher sd = mDispatcher.get();
    if (sd != null) {
        sd.connected(name, service, dead);
    }
}
```

在这里可以看出，先是从mDispatcher中获取了ServiceDispatcher，然后调用了其connected方法。mDispatcher中存储着之前在ContextImpl.bindService()中将ServiceConnection转换为ServiceDispatcher.InnerConnection处创建的ServiceDispatcher

接着看LoadedApk.ServiceDispatcher的connected方法

```java
public void connected(ComponentName name, IBinder service, boolean dead) {
    if (mActivityThread != null) {
        mActivityThread.post(new RunConnection(name, service, 0, dead));
    } else {
        doConnected(name, service, dead);
    }
}
```

通过mActivityThread post了一个RunConnection对象，mActivityThread就是一个Handler，是ActivityThread中的H（在ContextImpl的bindService中，调用bindServiceCommon时，通过mMainThread.getHandler获取，也就是在这里开始传入的），RunConnection就通过H发送到了主线程

简单看看RunConnection

```java
private final class RunConnection implements Runnable {
    RunConnection(ComponentName name, IBinder service, int command, boolean dead) {
        mName = name;
        mService = service;
        mCommand = command;
        mDead = dead;
    }

    public void run() {
        if (mCommand == 0) {
            doConnected(mName, mService, mDead);
        } else if (mCommand == 1) {
            doDeath(mName, mService);
        }
    }

    final ComponentName mName;
    final IBinder mService;
    final int mCommand;
    final boolean mDead;
}
```

在这里会调用ServiceDispatcher的doConnected方法

接着看一些LoadedApk.ServiceDispatcher.doConnected

```java
public void doConnected(ComponentName name, IBinder service, boolean dead) {
    ServiceDispatcher.ConnectionInfo old;
    ServiceDispatcher.ConnectionInfo info;

    ...

    // If there was an old service, it is now disconnected.
    if (old != null) {
        mConnection.onServiceDisconnected(name);
    }
    if (dead) {
        mConnection.onBindingDied(name);
    }
    // If there is a new service, it is now connected.
    if (service != null) {
        mConnection.onServiceConnected(name, service);
    }
}
```

由于ServiceDispatcher保存右ServiceConnection，所以可以很方便的调用ServiceConnection的onServiceConnected方法

## Service的取消绑定

从`unbindService(ServiceConnection conn);`开始，先是ContextWrapper的unbindService方法

```java
@Override
public void unbindService(ServiceConnection conn) {
    mBase.unbindService(conn);
}
```

然后接着还是ContextImpl中的unBindService方法

```java
@Override
public void unbindService(ServiceConnection conn) {
    if (conn == null) {
        throw new IllegalArgumentException("connection is null");
    }
    if (mPackageInfo != null) {
        //将ServiceConnection转换回IServiceConnection
        IServiceConnection sd = mPackageInfo.forgetServiceDispatcher(
                getOuterContext(), conn);
        try {
            //AMS中的unbindService
            ActivityManager.getService().unbindService(sd);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    } else {
        throw new RuntimeException("Not supported in system context");
    }
}
```

在这里先是通过LoadedApk.forgetServiceDispatcher方法将ServiceConnection转换回IServiceConnection，并对ServiceDispatcher做一些预处理，如remove掉这个ServiceConnection匹配的ServiceDispatcher，接着还是到了AMS中

接着看AMS的unbindService方法

```java
public boolean unbindService(IServiceConnection connection) {
    synchronized (this) {
        return mServices.unbindServiceLocked(connection);
    }
}
```

然后又交给了ActivityServices

接着看ActivityServices.unbindServiceLocked方法

```java
boolean unbindServiceLocked(IServiceConnection connection) {
    IBinder binder = connection.asBinder();
    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "unbindService: conn=" + binder);
    ArrayList<ConnectionRecord> clist = mServiceConnections.get(binder);
    ...

    final long origId = Binder.clearCallingIdentity();
    try {
        while (clist.size() > 0) {
            ConnectionRecord r = clist.get(0);
            //这么重要的remove
            removeConnectionLocked(r, null, null);
            ...

            if (r.binding.service.app != null) {
                if (r.binding.service.app.whitelistManager) {
                    updateWhitelistManagerLocked(r.binding.service.app);
                }
                // This could have made the service less important.
                if ((r.flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                    r.binding.service.app.treatLikeActivity = true;
                    mAm.updateLruProcessLocked(r.binding.service.app,
                            r.binding.service.app.hasClientActivities
                            || r.binding.service.app.treatLikeActivity, null);
                }
                mAm.updateOomAdjLocked(r.binding.service.app, false);
            }
        }

        mAm.updateOomAdjLocked();

    } finally {
        Binder.restoreCallingIdentity(origId);
    }

    return true;
}
```

看重要的，就知道下面是`removeConnectionLocked(r, null, null);`这个方法

接着看ActivityServices的removeConnectionLocked方法

```java
void removeConnectionLocked(
    ConnectionRecord c, ProcessRecord skipApp, ActivityRecord skipAct) {
    IBinder binder = c.conn.asBinder();
    AppBindRecord b = c.binding;
    ServiceRecord s = b.service;
    ArrayList<ConnectionRecord> clist = s.connections.get(binder);
    if (clist != null) {
        clist.remove(c);
        if (clist.size() == 0) {
            s.connections.remove(binder);
        }
    }
    b.connections.remove(c);
    if (c.activity != null && c.activity != skipAct) {
        if (c.activity.connections != null) {
            c.activity.connections.remove(c);
        }
    }
    if (b.client != skipApp) {
        b.client.connections.remove(c);
        if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
            b.client.updateHasAboveClientLocked();
        }
        // If this connection requested whitelist management, see if we should
        // now clear that state.
        if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {
            s.updateWhitelistManager();
            if (!s.whitelistManager && s.app != null) {
                updateWhitelistManagerLocked(s.app);
            }
        }
        if (s.app != null) {
            updateServiceClientActivitiesLocked(s.app, c, true);
        }
    }
    clist = mServiceConnections.get(binder);
    if (clist != null) {
        clist.remove(c);
        if (clist.size() == 0) {
            mServiceConnections.remove(binder);
        }
    }

    mAm.stopAssociationLocked(b.client.uid, b.client.processName, s.appInfo.uid, s.name);

    if (b.connections.size() == 0) {
        b.intent.apps.remove(b.client);
    }

    if (!c.serviceDead) {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Disconnecting binding " + b.intent
                + ": shouldUnbind=" + b.intent.hasBound);
        if (s.app != null && s.app.thread != null && b.intent.apps.size() == 0
                && b.intent.hasBound) {
            try {
                bumpServiceExecutingLocked(s, false, "unbind");
                if (b.client != s.app && (c.flags&Context.BIND_WAIVE_PRIORITY) == 0
                        && s.app.setProcState <= ActivityManager.PROCESS_STATE_RECEIVER) {
                    // If this service's process is not already in the cached list,
                    // then update it in the LRU list here because this may be causing
                    // it to go down there and we want it to start out near the top.
                    mAm.updateLruProcessLocked(s.app, false, null);
                }
                mAm.updateOomAdjLocked(s.app, true);
                b.intent.hasBound = false;
                // Assume the client doesn't want to know about a rebind;
                // we will deal with that later if it asks for one.
                b.intent.doRebind = false;
                //ApplicationThread，又是这个方式调用
                s.app.thread.scheduleUnbindService(s, b.intent.intent.getIntent());
            } catch (Exception e) {
                Slog.w(TAG, "Exception when unbinding service " + s.shortName, e);
                serviceProcessGoneLocked(s);
            }
        }

        // If unbound while waiting to start, remove the pending service
        mPendingServices.remove(s);

        if ((c.flags&Context.BIND_AUTO_CREATE) != 0) {
            boolean hasAutoCreate = s.hasAutoCreateConnections();
            if (!hasAutoCreate) {
                if (s.tracker != null) {
                    s.tracker.setBound(false, mAm.mProcessStats.getMemFactorLocked(),
                            SystemClock.uptimeMillis());
                }
            }
            bringDownServiceIfNeededLocked(s, true, hasAutoCreate);
        }
    }
}
```

这个方法里面有太多的remove了，感兴趣的可以自己去分析分析。我们看重点`s.app.thread.scheduleUnbindService(s, b.intent.intent.getIntent());`，这里是多么熟悉，又是到ApplicationThread中

接着看ApplicationThread的scheduleUnbindService方法

```java
public final void scheduleUnbindService(IBinder token, Intent intent) {
    BindServiceData s = new BindServiceData();
    s.token = token;
    s.intent = intent;

    sendMessage(H.UNBIND_SERVICE, s);
}
```

后面的就都是一样的，在ActivityThread中辗转反则，最后通过ActivityThread的H发送，在H中处理消息，然后会调用ActivityThread的handleUnbindServicef方法

接着看这个方法

```java
private void handleUnbindService(BindServiceData data) {
    //获取Service
    Service s = mServices.get(data.token);
    if (s != null) {
        try {
            data.intent.setExtrasClassLoader(s.getClassLoader());
            data.intent.prepareToEnterProcess();
            //调用unBind
            boolean doRebind = s.onUnbind(data.intent);
            try {
                if (doRebind) {
                    //如果需要重新绑定
                    ActivityManager.getService().unbindFinished(
                            data.token, data.intent, doRebind);
                } else {
                    //已经取消绑定
                    ActivityManager.getService().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                }
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(s, e)) {
                throw new RuntimeException(
                        "Unable to unbind to service " + s
                        + " with " + data.intent + ": " + e.toString(), e);
            }
        }
    }
}
```

注释很清楚了，先获取Service，接着调用Service的onUnbind方法，接着看看AMS的serviceDoneExecuting方法吧

```java
public void serviceDoneExecuting(IBinder token, int type, int startId, int res) {
    synchronized(this) {
        if (!(token instanceof ServiceRecord)) {
            Slog.e(TAG, "serviceDoneExecuting: Invalid service token=" + token);
            throw new IllegalArgumentException("Invalid service token");
        }
        //又去了ActivityServices
        mServices.serviceDoneExecutingLocked((ServiceRecord)token, type, startId, res);
    }
}
```

接着看ActivityServices的serviceDoneExecutingLocked方法

```java
void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res) {
    boolean inDestroying = mDestroyingServices.contains(r);
    if (r != null) {
        ...
        final long origId = Binder.clearCallingIdentity();
        //接着调用
        serviceDoneExecutingLocked(r, inDestroying, inDestroying);
        Binder.restoreCallingIdentity(origId);
    } else {
        Slog.w(TAG, "Done executing unknown service from pid "
                + Binder.getCallingPid());
    }
}
```

接着看ActivityServices的serviceDoneExecutingLocked重载方法

```java
private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
        boolean finishing) {
    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "<<< DONE EXECUTING " + r
            + ": nesting=" + r.executeNesting
            + ", inDestroying=" + inDestroying + ", app=" + r.app);
    else if (DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
            "<<< DONE EXECUTING " + r.shortName);
    r.executeNesting--;
    if (r.executeNesting <= 0) {
        if (r.app != null) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                    "Nesting at 0 of " + r.shortName);
            r.app.execServicesFg = false;
            r.app.executingServices.remove(r);
            if (r.app.executingServices.size() == 0) {
                if (DEBUG_SERVICE || DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                        "No more executingServices of " + r.shortName);
                mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
            } else if (r.executeFg) {
                // Need to re-evaluate whether the app still needs to be in the foreground.
                for (int i=r.app.executingServices.size()-1; i>=0; i--) {
                    if (r.app.executingServices.valueAt(i).executeFg) {
                        r.app.execServicesFg = true;
                        break;
                    }
                }
            }
            if (inDestroying) {
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                        "doneExecuting remove destroying " + r);
                mDestroyingServices.remove(r);
                r.bindings.clear();
            }
            mAm.updateOomAdjLocked(r.app, true);
        }
        r.executeFg = false;
        if (r.tracker != null) {
            r.tracker.setExecuting(false, mAm.mProcessStats.getMemFactorLocked(),
                    SystemClock.uptimeMillis());
            if (finishing) {
                r.tracker.clearCurrentOwner(r, false);
                r.tracker = null;
            }
        }
        if (finishing) {
            if (r.app != null && !r.app.persistent) {
                r.app.services.remove(r);
                if (r.whitelistManager) {
                    updateWhitelistManagerLocked(r.app);
                }
            }
            r.app = null;
        }
    }
}
```

又是各种remove，remove的消息

## Service的停止

从`stopService(Intent name);`开始，然后是ContextWrapper中

```java
@Override
public boolean stopService(Intent name) {
    return mBase.stopService(name);
}
```

然后具体的实现是在ContextImpl中

```java
@Override
public boolean stopService(Intent service) {
    warnIfCallingFromSystemProcess();
    return stopServiceCommon(service, mUser);
}
```

接着调用了ContextImpl内部的stopServiceCommon方法

```java
private boolean stopServiceCommon(Intent service, UserHandle user) {
    try {
        //这都是一些准备
        validateServiceIntent(service);
        service.prepareToLeaveProcess(this);
        //还是AMS
        int res = ActivityManager.getService().stopService(
            mMainThread.getApplicationThread(), service,
            service.resolveTypeIfNeeded(getContentResolver()), user.getIdentifier());
        if (res < 0) {
            throw new SecurityException(
                    "Not allowed to stop service " + service);
        }
        return res != 0;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

做一些准备后，又到了AMS中的stopService

```java
@Override
public int stopService(IApplicationThread caller, Intent service,
        String resolvedType, int userId) {
    enforceNotIsolatedCaller("stopService");
    // Refuse possible leaked file descriptors
    if (service != null && service.hasFileDescriptors() == true) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

    synchronized(this) {
        return mServices.stopServiceLocked(caller, service, resolvedType, userId);
    }
}
```

好吧，AMS还是交给了ActivityServices
接着看看ActivityServices的stopServiceLocked方法

```java
int stopServiceLocked(IApplicationThread caller, Intent service,
        String resolvedType, int userId) {
    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "stopService: " + service
            + " type=" + resolvedType);

    final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
    if (caller != null && callerApp == null) {
        throw new SecurityException(
                "Unable to find app for caller " + caller
                + " (pid=" + Binder.getCallingPid()
                + ") when stopping service " + service);
    }

    // If this service is active, make sure it is stopped.
    //确保这个Service是要停止的检查
    ServiceLookupResult r = retrieveServiceLocked(service, resolvedType, null,
            Binder.getCallingPid(), Binder.getCallingUid(), userId, false, false, false);
    if (r != null) {
        if (r.record != null) {
            final long origId = Binder.clearCallingIdentity();
            try {
                //内部方法调用
                stopServiceLocked(r.record);
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
            return 1;
        }
        return -1;
    }

    return 0;
}
```

接着还是调用ActivityServices内部的stopServiceLocked方法

```java
private void stopServiceLocked(ServiceRecord service) {
    ...
    //继续
    bringDownServiceIfNeededLocked(service, false, false);
}
```

接着还是调用ActivityServices的bringDownServiceIfNeededLocked方法

```java
private final void bringDownServiceIfNeededLocked(ServiceRecord r, boolean knowConn,
        boolean hasConn) {
    ...

    bringDownServiceLocked(r);
}
```

接着还是调用ActivityServices的bringDownServiceLocked方法

```java
private final void bringDownServiceLocked(ServiceRecord r) {
    ...

    // Tell the service that it has been unbound.
    if (r.app != null && r.app.thread != null) {
        for (int i=r.bindings.size()-1; i>=0; i--) {
            IntentBindRecord ibr = r.bindings.valueAt(i);
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bringing down binding " + ibr
                    + ": hasBound=" + ibr.hasBound);
            if (ibr.hasBound) {
                try {
                    bumpServiceExecutingLocked(r, false, "bring down unbind");
                    mAm.updateOomAdjLocked(r.app, true);
                    ibr.hasBound = false;
                    ibr.requested = false;
                    //停止之前先取消绑定
                    r.app.thread.scheduleUnbindService(r,
                            ibr.intent.getIntent());
                } catch (Exception e) {
                    Slog.w(TAG, "Exception when unbinding service "
                            + r.shortName, e);
                    serviceProcessGoneLocked(r);
                }
            }
        }
    }

    ...

    if (r.app != null) {
        synchronized (r.stats.getBatteryStats()) {
            r.stats.stopLaunchedLocked();
        }
        r.app.services.remove(r);
        if (r.whitelistManager) {
            updateWhitelistManagerLocked(r.app);
        }
        if (r.app.thread != null) {
            updateServiceForegroundLocked(r.app, false);
            try {
                bumpServiceExecutingLocked(r, false, "destroy");
                mDestroyingServices.add(r);
                r.destroying = true;
                mAm.updateOomAdjLocked(r.app, true);
                //停止
                r.app.thread.scheduleStopService(r);
            } catch (Exception e) {
                Slog.w(TAG, "Exception when destroying service "
                        + r.shortName, e);
                serviceProcessGoneLocked(r);
            }
        } else {
            if (DEBUG_SERVICE) Slog.v(
                TAG_SERVICE, "Removed service that has no process: " + r);
        }
    } else {
        if (DEBUG_SERVICE) Slog.v(
            TAG_SERVICE, "Removed service that is not running: " + r);
    }

    ...
}
```

这个方法中的代码有很多，但是关键还是`.app.thread.scheduleStopService(r);`这行代码

接着看ApplicationThread中的scheduleStopService方法

```java
public final void scheduleStopService(IBinder token) {
    sendMessage(H.STOP_SERVICE, token);
}
```

好吧，熟悉的一切，跟前面的启动、绑定、取消绑定一样，最后调用H处理消息，然后调用ActivityThread的handleStopService方法

```java
private void handleStopService(IBinder token) {
    //从列表中移除Service
    Service s = mServices.remove(token);
    if (s != null) {
        try {
            if (localLOGV) Slog.v(TAG, "Destroying service " + s);
            //销毁Service
            s.onDestroy();
            s.detachAndCleanUp();
            Context context = s.getBaseContext();
            if (context instanceof ContextImpl) {
                final String who = s.getClassName();
                ((ContextImpl) context).scheduleFinalCleanup(who, "Service");
            }

            QueuedWork.waitToFinish();

            try {
                //通知完成停止操作
                ActivityManager.getService().serviceDoneExecuting(
                        token, SERVICE_DONE_EXECUTING_STOP, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(s, e)) {
                throw new RuntimeException(
                        "Unable to stop service " + s
                        + ": " + e.toString(), e);
            }
            Slog.i(TAG, "handleStopService: exception for " + token, e);
        }
    } else {
        Slog.i(TAG, "handleStopService: token=" + token + " not found.");
    }
    //Slog.i(TAG, "Running services: " + mServices);
}
```

先是从mServices中移除了要停止的Service，然后调用Service的onDestory销毁，最后通过AMS来发出通知（`ActivityManager.getService().serviceDoneExecuting(token, SERVICE_DONE_EXECUTING_STOP, 0, 0);`）后面的就跟取消绑定一样了