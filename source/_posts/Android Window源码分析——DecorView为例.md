---
title: Android Window源码分析——DecorView为例
tag: Android源码
category: Android

---

<meta name="referrer" content="no-referrer" />



[TOC]

# Window源码分析——DecorView为例

在Activity创建完成后，通过`attach()`方法将Context、Application等绑定到Activity中，同时实例化一个PhoneWindow，并建立和WindowManager的关联，接着回调`onCreate()`方法，同时我们这里设置的`setContentView()`会调用到Activity的`setContentView()`，接着会调用其Window的`setContentView()`，最后就调用到PhoneWindow中，在PhoneWindow如果没有初始化DecorView就会先进行初始化DecorView，然后通过LayoutInflater去`inflate()`我们设置的view，在LayoutInflater中通过xml解析器，根据标签递归解析我们的布局，然后生成对应的view添加到DecorView中的parentView（这个是用于放置我们设置的view的地方），这样就得到了整个View树保存在DecorView中（DecorView保存在Windows中）；

在ActivityThread的handleResumeActivity方法中，通过获取到WindowManager，然后在后面通过addView方法，将decor添加进去，WindowManager的实现类是WindowManagerImpl

## window的添加

通过WindowManager的addView实现添加过程

```java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

mGlobal是WindowManagerGlobal对象，从这里看出addView并不是WindowManagerImpl来做的，而是交给了WindowManagerGlobal（典型的桥接模式）

```java
private final ArrayList<View> mViews = new ArrayList<View>();
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
private final ArrayList<WindowManager.LayoutParams> mParams =
        new ArrayList<WindowManager.LayoutParams>();
//存储那些正在被删除的View对象（已经调用removeView方法但是删除操作还未完成）
private final ArraySet<View> mDyingViews = new ArraySet<View>();

...
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    //检查参数是否合法
    ...

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        ...
    }

    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        ...
        //创建一个ViewRootImpl的实例
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        //mViews存储的是所有Window对应的View
        mViews.add(view);
        //mRoots存储的是所有Window对应的ViewRootImpl
        mRoots.add(root);
        //mParams存储的是所有Window对应的布局参数
        mParams.add(wparams);
        // do this last because it fires off messages to start doing things
        try {
            //通过setView建立关联
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            ...
        }
    }
}
```

在这里就将ViewRootImpl对象和DecorView对象建立关联

接着看ViewRootImpl的setView方法

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;

                mAttachInfo.mDisplayState = mDisplay.getState();
                mDisplayManager.registerDisplayListener(mDisplayListener, mHandler);

                mViewLayoutDirectionInitial = mView.getRawLayoutDirection();
                mFallbackEventHandler.setView(view);
                mWindowAttributes.copyFrom(attrs);
                ...

                if (view instanceof RootViewSurfaceTaker) {
                    mSurfaceHolderCallback =
                            ((RootViewSurfaceTaker)view).willYouTakeTheSurface();
                    if (mSurfaceHolderCallback != null) {
                        mSurfaceHolder = new TakenSurfaceHolder();
                        mSurfaceHolder.setFormat(PixelFormat.UNKNOWN);
                        mSurfaceHolder.addCallback(mSurfaceHolderCallback);
                    }
                }

                ...
                mAttachInfo.mRootView = view;
                mAttachInfo.mScalingRequired = mTranslator != null;
                mAttachInfo.mApplicationScale =
                        mTranslator == null ? 1.0f : mTranslator.applicationScale;
                if (panelParentView != null) {
                    mAttachInfo.mPanelParentWindowToken
                            = panelParentView.getApplicationWindowToken();
                }
                mAdded = true;
                int res; /* = WindowManagerImpl.ADD_OKAY; */

                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
                //请求layout
                requestLayout();
                ...
                try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
                    //通过WindowSession去完成添加
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                } catch (RemoteException e) {
                    ...
                } finally {
                    ...
                }
                ...
            }
        }
    }
```

这个方法很长，后面要看的代码可能都比较长，所以要耐心。先看看`requestLayout();`

```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        //View绘制的入口
        scheduleTraversals();
    }
}
```

这里有View绘制的入口，里面就是View的三大流程了。接着回到上一个setView方法，在requestLayout方法后面，会通过WindowSession最终来完成Window的添加过程

```java
try {
    mOrigWindowType = mWindowAttributes.type;
    mAttachInfo.mRecomputeGlobalAttributes = true;
    collectViewAttributes();
    //通过WindowSession去完成添加
    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
            getHostVisibility(), mDisplay.getDisplayId(),
            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
            mAttachInfo.mOutsets, mInputChannel);
} catch (RemoteException e) {
    ...
} finally {
    ...
}
```

mWindowSession的类型是IWindowSession，是一个Binder对象，真正的实现类是Session，所以Window的添加过程是一次IPC调用

接着看Session的addToDisplay方法

```java
@Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
        Rect outOutsets, InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
            outContentInsets, outStableInsets, outOutsets, outInputChannel);
}
```

mService的类型是WMS（WindowManagerService），Window的添加请求就交给WMS处理了，在WMS内部会为每一个应用保留一个单独的Session

## window的删除

window的删除过程跟添加过程一样，先是通过WindowManagerImpl，最后交给WindowManagerGlobal

```java
@Override
public void removeView(View view) {
    mGlobal.removeView(view, false);
}
```

接着看WindowManagerGlobal的removeView方法

```java
public void removeView(View view, boolean immediate) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }

    synchronized (mLock) {
        int index = findViewLocked(view, true);
        View curView = mRoots.get(index).getView();
        removeViewLocked(index, immediate);
        if (curView == view) {
            return;
        }

        throw new IllegalStateException("Calling with view " + view
                + " but the ViewAncestor is attached to " + curView);
    }
}
```

先是通过findViewLocked查找待删除的View的索引，再调用removeViewLocked做进一步删除

```java
//就是通过ArrayList遍历查找
private int findViewLocked(View view, boolean required) {
    final int index = mViews.indexOf(view);
    if (required && index < 0) {
        throw new IllegalArgumentException("View=" + view + " not attached to window manager");
    }
    return index;
}

private void removeViewLocked(int index, boolean immediate) {
    ViewRootImpl root = mRoots.get(index);
    View view = root.getView();

    if (view != null) {
        InputMethodManager imm = InputMethodManager.getInstance();
        if (imm != null) {
            imm.windowDismissed(mViews.get(index).getWindowToken());
        }
    }
    boolean deferred = root.die(immediate);
    if (view != null) {
        view.assignParent(null);
        if (deferred) {
            mDyingViews.add(view);
        }
    }
}
```

removeViewLocked是通过ViewRootImpl来完成删除操作的，在WindowManager中提供了两种删除接口removeView和removeViewImmediate，分别表示异步和同步删除（主要使用异步删除）。先是拿去添加的时候存储的对应的ViewRootImpl，接着通过ViewRootImpl的die方法来完成

```java
boolean die(boolean immediate) {
    // Make sure we do execute immediately if we are in the middle of a traversal or the damage
    // done by dispatchDetachedFromWindow will cause havoc on return.
    if (immediate && !mIsInTraversal) {
        doDie();
        return false;
    }

    if (!mIsDrawing) {
        destroyHardwareRenderer();
    } else {
        Log.e(mTag, "Attempting to destroy the window while drawing!\n" +
                "  window=" + this + ", title=" + mWindowAttributes.getTitle());
    }
    mHandler.sendEmptyMessage(MSG_DIE);
    return true;
}
```

在异步删除的情况下，die方法只是发送了一个请求删除的消息，最后通过ViewRootImpl的Handler处理此消息并调用doDie方法；如果是同步删除，就直接调用doDie方法

先看看这个消息的处理吧

```java
@Override
public void handleMessage(Message msg) {
    switch (msg.what) {
        ...
        case MSG_DIE:
            doDie();
            break;
        ...
    }
}
```

接着看看doDie方法吧

```java
void doDie() {
    checkThread();
    if (LOCAL_LOGV) Log.v(mTag, "DIE in " + this + " of " + mSurface);
    synchronized (this) {
        if (mRemoved) {
            return;
        }
        mRemoved = true;
        if (mAdded) {
            //通过此方法进行真正的删除逻辑
            dispatchDetachedFromWindow();
        }

        if (mAdded && !mFirst) {
            destroyHardwareRenderer();

            if (mView != null) {
                int viewVisibility = mView.getVisibility();
                boolean viewVisibilityChanged = mViewVisibility != viewVisibility;
                if (mWindowAttributesChanged || viewVisibilityChanged) {
                    // If layout params have been changed, first give them
                    // to the window manager to make sure it has the correct
                    // animation info.
                    try {
                        if ((relayoutWindow(mWindowAttributes, viewVisibility, false)
                                & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
                            //windowSessin的结束
                            mWindowSession.finishDrawing(mWindow);
                        }
                    } catch (RemoteException e) {
                    }
                }
                //surface的释放
                mSurface.release();
            }
        }

        mAdded = false;
    }
    //刷新mRoots、mParams、mDyingViews中的数据
    WindowManagerGlobal.getInstance().doRemoveView(this);
}
```

在这里除了真正删除的逻辑，还有一些其他删除后的操作，接着看看这个dispatchDetachedFromWindow真正删除的地方吧

```java
void dispatchDetachedFromWindow() {
    if (mView != null && mView.mAttachInfo != null) {
        mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(false);
        //调用View的dispatchDetachedFromWindow，在内部会调用View的onDetachedFromWindow以及onDetachedFromWindowwInternal方法
        mView.dispatchDetachedFromWindow();
    }
    //垃圾回收的相关工作
    mAccessibilityInteractionConnectionManager.ensureNoConnection();
    mAccessibilityManager.removeAccessibilityStateChangeListener(
            mAccessibilityInteractionConnectionManager);
    mAccessibilityManager.removeHighTextContrastStateChangeListener(
            mHighContrastTextManager);
    removeSendWindowContentChangedCallback();

    destroyHardwareRenderer();

    setAccessibilityFocus(null, null);

    mView.assignParent(null);
    mView = null;
    mAttachInfo.mRootView = null;

    mSurface.release();

    if (mInputQueueCallback != null && mInputQueue != null) {
        mInputQueueCallback.onInputQueueDestroyed(mInputQueue);
        mInputQueue.dispose();
        mInputQueueCallback = null;
        mInputQueue = null;
    }
    if (mInputEventReceiver != null) {
        mInputEventReceiver.dispose();
        mInputEventReceiver = null;
    }
    try {
        //通过Session的remove方法删除Window，这同样是一个IPC过程，最终调用WMS的removeWindow方法
        mWindowSession.remove(mWindow);
    } catch (RemoteException e) {
    }

    // Dispose the input channel after removing the window so the Window Manager
    // doesn't interpret the input channel being closed as an abnormal termination.
    if (mInputChannel != null) {
        mInputChannel.dispose();
        mInputChannel = null;
    }

    mDisplayManager.unregisterDisplayListener(mDisplayListener);

    unscheduleTraversals();
}
```

## window的更新

WindowMangerImpl的updateViewLayout方法

```java
@Override
public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.updateViewLayout(view, params);
}
```

接着WindowManagerGlobal的updateViewLayout方法

```java
public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;

    view.setLayoutParams(wparams);

    synchronized (mLock) {
        int index = findViewLocked(view, true);
        ViewRootImpl root = mRoots.get(index);
        mParams.remove(index);
        mParams.add(index, wparams);
        root.setLayoutParams(wparams, false);
    }
}
```

首先需要更新View的LayoutParams并替换掉旧的，接着再更新ViewRootImpl中的LayoutParams，通过ViewRootImpl的setLayoutParams来实现。

```java
void setLayoutParams(WindowManager.LayoutParams attrs, boolean newView) {
    synchronized (this) {
        final int oldInsetLeft = mWindowAttributes.surfaceInsets.left;
        final int oldInsetTop = mWindowAttributes.surfaceInsets.top;
        final int oldInsetRight = mWindowAttributes.surfaceInsets.right;
        final int oldInsetBottom = mWindowAttributes.surfaceInsets.bottom;
        final int oldSoftInputMode = mWindowAttributes.softInputMode;
        final boolean oldHasManualSurfaceInsets = mWindowAttributes.hasManualSurfaceInsets;
        ...
        applyKeepScreenOnFlag(mWindowAttributes);

        if (newView) {
            mSoftInputMode = attrs.softInputMode;
            requestLayout();
        }
        ...

        mWindowAttributesChanged = true;
        scheduleTraversals();
    }
}
```

在ViewRootImpl中会通过scheduleTraversals来进行重绘，ViewRootImpl还会通过WindowSession来更新Window的视图，最终由WMS的relayoutWindow实现，也是一个IPC过程

## Window、DecorView、Activity的关系

通过类加载器创建Activity后，设置了一个null的Window，然后通过attach方法，将相关的东西跟Activity关联（包括Application、Window等）

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...

    ActivityInfo aInfo = r.activityInfo;
    ...

    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        //通过类加载器实例化一个Activity
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        ...
    } catch (Exception e) {
       ...
    }

    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        ...
        if (activity != null) {
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            ...
            //声明一个Window
            Window window = null;
            //Activity创建后，由于前面没有设置mPendingRemoveWindow这些东西，所以下面的判断不会执行
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }
            appContext.setOuterContext(activity);
            //通过Activity的attach方法进行关联
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback);

            ...
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            activity.mCalled = false;
            if (r.isPersistable()) {
                //回调onCreate方法
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            ...
            r.activity = activity;
            r.stopped = true;
            if (!r.activity.mFinished) {
                activity.performStart();
                r.stopped = false;
            }
            if (!r.activity.mFinished) {
                if (r.isPersistable()) {
                    if (r.state != null || r.persistentState != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                r.persistentState);
                    }
                } else if (r.state != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                }
            }
            if (!r.activity.mFinished) {
                activity.mCalled = false;
                if (r.isPersistable()) {
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

        mActivities.put(r.token, r);

    } catch (SuperNotCalledException e) {
        throw e;

    } catch (Exception e) {
        ...
    }
    return activity;
}
```

在Activity的attach方法中，会实例化一个PhoneWindow，通过`context.getSystemService(Context.WINDOW_SERVICE)`获取WindowManager的实例

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback) {
    attachBaseContext(context);

    mFragments.attachHost(null /*parent*/);
    //实例化PhoneWindow
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    //window的一些设置
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }
    if (info.uiOptions != 0) {
        mWindow.setUiOptions(info.uiOptions);
    }
    mUiThread = Thread.currentThread();

    mMainThread = aThread;
    mInstrumentation = instr;
    mToken = token;
    mIdent = ident;
    mApplication = application;
    mIntent = intent;
    mReferrer = referrer;
    mComponent = intent.getComponent();
    mActivityInfo = info;
    mTitle = title;
    mParent = parent;
    mEmbeddedID = id;
    mLastNonConfigurationInstances = lastNonConfigurationInstances;
    if (voiceInteractor != null) {
        if (lastNonConfigurationInstances != null) {
            mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
        } else {
            mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                    Looper.myLooper());
        }
    }
    //给Window设置WindowManager，通过context来获取WindowManager的实例
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;

    mWindow.setColorMode(info.colorMode);
}
```

先看一下怎么获取WindowManager实例的，ContextImpl中的getSystemService方法

```java
@Override
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name);
}
```

直接返回了SystemServiceRegistry的getSystemService方法的返回值

```java
final class SystemServiceRegistry {
    ...
    //HashMap缓存，，Service
    private static final HashMap<Class<?>, String> SYSTEM_SERVICE_NAMES =
            new HashMap<Class<?>, String>();
    private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
            new HashMap<String, ServiceFetcher<?>>();
    private static int sServiceCacheSize;

    // Not instantiable.
    private SystemServiceRegistry() { }

    static {
        ...
        registerService(Context.ACTIVITY_SERVICE, ActivityManager.class,
                new CachedServiceFetcher<ActivityManager>() {
            @Override
            public ActivityManager createService(ContextImpl ctx) {
                return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
            }});
        ...

        registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
                new CachedServiceFetcher<LayoutInflater>() {
            @Override
            public LayoutInflater createService(ContextImpl ctx) {
                return new PhoneLayoutInflater(ctx.getOuterContext());
            }});
        ...

        registerService(Context.WINDOW_SERVICE, WindowManager.class,
                new CachedServiceFetcher<WindowManager>() {
            @Override
            public WindowManager createService(ContextImpl ctx) {
                return new WindowManagerImpl(ctx);
            }});

       ...
    }
    //根据字段获取
    public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }

    ...

    private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
        //缓存
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    }
}
```

首先getSystemService只是根据对应的name字符串从缓存SYSTEM_SERVICE_FETCHERS中获取对应的服务，而在这个类中，后面又有一个registerService方法，将注册的Service都放进了缓存。通过查看这个类发现，首先SystemServiceRegistry是一个静态类，其次，有一个static块，在这个static块中，你会发现很多registerService方法，而这些注册的Service是很多的Manager，慢慢看这个static块，你会发现，有很多的熟悉的Manager，例如ActivityManager、BatteryManager、AudioManager、BluetoothManager等等，很多你都在日常开发中用到过，当然了，这里面也有我们要找的WindowManager，在这个registerService方法中，各种Manager都进行了实例化（new）。所以最后通过getSystemService和字段获取的就是对应的Service，那么我们这里就获取了WindowManagerImpl（WindowManager的实现类）

有点跑题了，回到Activity的attach方法，在这里有了PhoneWindow的实例、WindowManager的实例，但是还并没有将DecorView和PhoneWindow进行关联（DecorView是系统内置的一个布局，这个有问题的自己查咯），那我们接着看ActivityThread中创建完Activity后干了什么吧

在ActivityThread的performLaunchActivity后，回到上一个方法handleLaunchActivity方法

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    ...
    WindowManagerGlobal.initialize();

    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        Bundle oldState = r.state;
        //创建完后，接着调用了resume方法，这就是要调用的onResume哦
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);

        if (!r.activity.mFinished && r.startsNotResumed) {
            ...
            performPauseActivityIfNeeded(r, reason);
            ...
        }
    } else {
        // If there was an error, for any reason, tell the activity manager to stop us.
        try {
            ActivityManager.getService()
                .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                        Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```

在创建Activity完后，又调用了ActivityThread的handleResumeActivity，这个方法其实就会onResume方法的，那么来看看吧

```java
final void handleResumeActivity(IBinder token,
        boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    //获取缓存的ActivityClientRecord（记录Activity相关的信息咯）
    ActivityClientRecord r = mActivities.get(token);
    ...

    // TODO Push resumeArgs into the activity for consideration
    r = performResumeActivity(token, clearHide, reason);

    if (r != null) {
        final Activity a = r.activity;
        ...
        //如果是第一次创建Activity，那么这个Window肯定是null的咯，还记得之前创建的时候，说了ActivityClientRecord基本没什么设置了吗
        if (r.window == null && !a.mFinished && willBeVisible) {
            //既然是null，那就获取Activity的Window吧，这个就是attach的时候实例化的PhoneWindow哦
            r.window = r.activity.getWindow();
            //这里获取DecorView，可是前面好像没有创建DecorView啊？接着看后面的源码哦
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            //获取WindowManager，没错，这就是我们前面跑题去看的Manager，获取了WindowManagerImpl的实例
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            //由于PhoneWindow中的Window是null，这里肯定不会执行这个判断的啦
            if (r.mPreserveWindow) {
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                // Normally the ViewRoot sets up callbacks with the Activity
                // in addView->ViewRootImpl#setView. If we are instead reusing
                // the decor view we have to notify the view root that the
                // callbacks may have changed.
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    //哎呀，addView了哦
                    wm.addView(decor, l);
                } else {
                    ...
                    a.onWindowAttributesChanged(l);
                }
            }
        ...
        } else if (!willBeVisible) {
            if (localLOGV) Slog.v(
                TAG, "Launch " + r + " mStartedActivity set");
            r.hideForNow = true;
        }
        ...
        // The window is now visible if it has been added, we are not
        // simply finishing, and we are not starting another activity.
        if (!r.activity.mFinished && willBeVisible
                && r.activity.mDecor != null && !r.hideForNow) {
            ...
            r.activity.mVisibleFromServer = true;
            mNumVisibleActivities++;
            if (r.activity.mVisibleFromClient) {
                //最终会通过调用Activity的makeVisible进行显示，才能被看见
                r.activity.makeVisible();
            }
        }
    } else {
        // If an exception was thrown when trying to resume, then
        // just end this activity.
        try {
            ActivityManager.getService()
                .finishActivity(token, Activity.RESULT_CANCELED, null,
                        Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```

要记得抓重点哦，这个方法代码很多，但逻辑很简单，直接看`if (r.window == null && !a.mFinished && willBeVisible)`这个判断咯，其实就是根据是否已经添加了Window来判断，执行的就是两个逻辑（两种情况），第一次肯定还没有添加Window，那么就需要添加，就先获取PhoneWindow实例，然后获取DecorView，最后通过WindowManager的addView方法添加；另一种就是已经添加的情况，就是单纯的resume了

好了，我们先来看看怎么获取DecorView的吧（毕竟我们前面并没有去创建它）

我们知道，我们在一个Activity的onCreate方法中通常都要写这么一行`setContentView()`，通过这行代码就将我们要设置的布局显示出来，我们知道，我们设置的这个布局是DecorView中contentParent的一个子View；而在Activity创建后，resume之前，是先会调用onCreate方法的，所以我们看看Activity的setContentView方法（注意一点，AppCompateActivity的setContentView要多几个过程）

```java
public void setContentView(@LayoutRes int layoutResID) {
    //获取到window，来设置View
    getWindow().setContentView(layoutResID);
    //初始化window的actionBar
    initWindowDecorActionBar();
}
```

这里获取的Window就是前面通过attach时，创建的PhoneWindow，那么接着看PhoneWindow的setContentView方法

```java
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
        //通过LayoutInflater加载资源布局，添加到DecorView的mContentParent中
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

mContentParent是一个ViewGroup类型对象，当它为空，就执行installDecor方法创建DecorView，这个mContentParent可能是DecorView，也可能是它的子View（没太懂）

那么接着看看这个installDecor方法吧

```java
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        //通过generateDecor获取通常的DecorView
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        //通过generateLayout获取mContentParent
        mContentParent = generateLayout(mDecor);
        ...
        }
    }
}
```

首先，进行mDecor判断，为null的时候调用generateDecor创建一个，接着由于mContentParent，也会调用generateLayout创建一个，后面部分可以看到对默认标题、菜单等的一些设置处理。

接着看generateDecor方法

```java
protected DecorView generateDecor(int featureId) {
    ...
    Context context;
    if (mUseDecorContext) {
        Context applicationContext = getContext().getApplicationContext();
        if (applicationContext == null) {
            context = getContext();
        } else {
            context = new DecorContext(applicationContext, getContext().getResources());
            if (mTheme != -1) {
                context.setTheme(mTheme);
            }
        }
    } else {
        context = getContext();
    }
    return new DecorView(context, featureId, this, getAttributes());
}
```

获取Context，然后实例化了一个DecorView，同时也将PhoneWindow和DecorView关联起来，在DecorView的构造方法中会进行这个操作

```java
DecorView(Context context, int featureId, PhoneWindow window,
        WindowManager.LayoutParams params) {
    super(context);
    mFeatureId = featureId;

    mShowInterpolator = AnimationUtils.loadInterpolator(context,
            android.R.interpolator.linear_out_slow_in);
    mHideInterpolator = AnimationUtils.loadInterpolator(context,
            android.R.interpolator.fast_out_linear_in);

    mBarEnterExitDuration = context.getResources().getInteger(
            R.integer.dock_enter_exit_duration);
    mForceWindowDrawsStatusBarBackground = context.getResources().getBoolean(
            R.bool.config_forceWindowDrawsStatusBarBackground)
            && context.getApplicationInfo().targetSdkVersion >= N;
    mSemiTransparentStatusBarColor = context.getResources().getColor(
            R.color.system_bar_background_semi_transparent, null /* theme */);

    updateAvailableWidth();
    //将DecorView添加到PhoneWindow
    setWindow(window);

    updateLogTag(params);

    mResizeShadowSize = context.getResources().getDimensionPixelSize(
            R.dimen.resize_shadow_size);
    initResizingPaints();
}

...

void setWindow(PhoneWindow phoneWindow) {
    mWindow = phoneWindow;
    Context context = getContext();
    if (context instanceof DecorContext) {
        DecorContext decorContext = (DecorContext) context;
        decorContext.setPhoneWindow(mWindow);
    }
}
```

DecorContext就是DecorView的不依赖于活动的上下文

创建好DecorView并将之于PhoneWindow关联，然后回创建mContentParent，接着看PhoneWindow的generateDecor方法

```java
protected ViewGroup generateLayout(DecorView decor) {
    // Apply data from current theme.

    TypedArray a = getWindowStyle();
    //很多标签检查，设置标签等等
    ...

    // Inflate the window decor.
    //开始加载默认的布局
    int layoutResource;
    int features = getLocalFeatures();
    // System.out.println("Features: 0x" + Integer.toHexString(features));
    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        layoutResource = R.layout.screen_swipe_dismiss;
        setCloseOnSwipeEnabled(true);
    } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogTitleIconsDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_title_icons;
        }
        // XXX Remove this once action bar supports these features.
        removeFeature(FEATURE_ACTION_BAR);
        // System.out.println("Title Icons!");
    } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
            && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
        // Special case for a window with only a progress bar (and title).
        // XXX Need to have a no-title version of embedded windows.
        layoutResource = R.layout.screen_progress;
        // System.out.println("Progress!");
    } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
        // Special case for a window with a custom title.
        // If the window is floating, we need a dialog layout
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogCustomTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_custom_title;
        }
        // XXX Remove this once action bar supports these features.
        removeFeature(FEATURE_ACTION_BAR);
    } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
        // If no other features and not embedded, only need a title.
        // If the window is floating, we need a dialog layout
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
            layoutResource = a.getResourceId(
                    R.styleable.Window_windowActionBarFullscreenDecorLayout,
                    R.layout.screen_action_bar);
        } else {
            layoutResource = R.layout.screen_title;
        }
        // System.out.println("Title!");
    } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
        layoutResource = R.layout.screen_simple_overlay_action_mode;
    } else {
        // Embedded, so no decoration is needed.
        layoutResource = R.layout.screen_simple;
        // System.out.println("Simple!");
    }

    mDecor.startChanging();
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    ...

    // Remaining setup -- of background and title -- that only applies
    // to top-level windows.
    //顶级Window的标题和背景设置
    if (getContainer() == null) {
        final Drawable background;
        if (mBackgroundResource != 0) {
            background = getContext().getDrawable(mBackgroundResource);
        } else {
            background = mBackgroundDrawable;
        }
        mDecor.setWindowBackground(background);

        final Drawable frame;
        if (mFrameResource != 0) {
            frame = getContext().getDrawable(mFrameResource);
        } else {
            frame = null;
        }
        mDecor.setWindowFrame(frame);

        mDecor.setElevation(mElevation);
        mDecor.setClipToOutline(mClipToOutline);

        if (mTitle != null) {
            setTitle(mTitle);
        }

        if (mTitleColor == 0) {
            mTitleColor = mTextColor;
        }
        setTitleColor(mTitleColor);
    }

    mDecor.finishChanging();

    return contentParent;
}
```

首先看方法返回的是一个ViewGroup，说明mContentParent是ViewGroup类型的，然后在代码中我们看到了很多默认的资源文件的加载和设置，这就是顶级Window的默认设置，你自己修改后也是在这里生效的

然后我们回过头看看ActivityThread的handleResumeActivity方法中，通过`View decor = r.window.getDecorView();`来获取到我们前面创建的DecorView，那么看看PhoneWindow的getDecorView方法吧

```java
@Override
public final View getDecorView() {
    if (mDecor == null || mForceDecorInstall) {
        installDecor();
    }
    return mDecor;
}
```

在这个方法中先进行了是否为null的判断，所以不管我们前面有没有成功创建DecorView，最后获取到的都不会是null，（所以这里的DecorView到底是前面创建还是后面创建的呢）

好了，到这里呢，DecorView有了，Window也有了，两者也进行了关联，那么我们就回到ActivityThread中的handleResumeActivity方法，在`View decor = r.window.getDecorView();`这行代码后，我们又是获取到了WindowManager，然后在后面通过addView方法，将decor添加进去，这就是Window的添加过程了

然后我们看看Activity的makeVisible方法

```java
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
```

这里也进行了判断，是否添加了Window，然后将DecorView显示