---
title: Android View系列(一)——View源码分析——DecorView
tag: Android

---

<meta name="referrer" content="no-referrer" />



[TOC]

# View源码分析——DecorView为例

**基本概念**：ViewRoot对应于ViewRootImpl类，它是连接WindowManager和DecorView的纽带。View的三大流程measure、layout、draw都是通过ViewRoot来完成。在ActivityThread中，当Activity对象被创建，会将DecorView添加到Window，同时会创建ViewRootImpl对象，并将两个关联

在Activity创建完成后，通过`attach()`方法将Context、Application等绑定到Activity中，同时实例化一个PhoneWindow，并建立和WindowManager的关联，接着回调`onCreate()`方法，同时我们这里设置的`setContentView()`会调用到Activity的`setContentView()`，接着会调用其Window的`setContentView()`，最后就调用到PhoneWindow中，在PhoneWindow如果没有初始化DecorView就会先进行初始化DecorView，然后通过LayoutInflater去`inflate()`我们设置的view，在LayoutInflater中通过xml解析器，根据标签递归解析我们的布局，然后生成对应的view添加到DecorView中的parentView（这个是用于放置我们设置的view的地方），这样就得到了整个View树保存在DecorView中（DecorView保存在Windows中）；

在ActivityThread的handleResumeActivity方法中，通过获取到WindowManager，然后在后面通过addView方法，将decor添加进去，WindowManager的实现类是WindowManagerImpl，这又是Window的添加过程，在这个过程中，有一个requestLayout方法（相关过程参考我的[Window源码分析](http://penghesheng.cn/?p=162)）

```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        //这就是View绘制的入口
        scheduleTraversals();
    }
}
```

接着看内部的scheduleTraversals方法

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //这里通过
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        //mTraversalRunnable是一个Runnable
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

这个代码看着可能就有点迷茫了，目前也没看太懂，只知道mTraversalRunnable是一个Runnable对象，而在它的run方法呢看到我们后续要分析的方法，所以这里的分析就暂时放放（感觉是通过mChoreographer来调用mTraversalRunnable中的run方法，但是不知道mChoreographer是用来干什么的），**好吧，遇到问题就先放放，欢迎知道的人答疑（期待您的看法）**。

那就接着看mTraversalRunnable中的run方法吧

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
//这个对象还是final的哦
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```

run方法中就只有一行代码，就是调用了ViewRootImpl中的doTraversal方法，接着看

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        ...
        performTraversals();
        ...
    }
}
```

这个方法中呢，重点就只是performTraversals咯，接着看ViewRootImpl中内部的performTraversals方法吧

```java
private void performTraversals() {
    // cache mView since it is used so much below...
    final View host = mView;
    ...
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {
        final Resources res = mView.getContext().getResources();
        ...
        // Ask host how big it wants to be
        //在测量的开始，会去询问到底要多大的空间，对于DecorView来说，就是屏幕的尺寸，desiredWindowWidth、desiredWindowHeight是屏幕的宽和高
        windowSizeMayChange |= measureHierarchy(host, lp, res,
                desiredWindowWidth, desiredWindowHeight);
    }
    ...
    if (mFirst || windowShouldResize || insetsChanged ||
            viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
        ...
        if (!mStopped || mReportNextDraw) {
            ...
            if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                    || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                    updatedConfiguration) {
                ...
                 // Ask host how big it wants to be
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                ....
                //是否需要重新测量
                if (measureAgain) {
                    ...
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                }
                ...
            }
        }
    } else {
        ...
    }

    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    ...
    if (didLayout) {
        performLayout(lp, mWidth, mHeight);
        ...
    }
    ...
    if (!cancelDraw && !newSurface) {
        ...
        performDraw();
    } else {
        ...
    }
    ...
}
```

View的绘制流程是从ViewRoot的performTraversals开始的，这个就是我们View绘制流程的开始（我知道这个方法很关键，但是也不用这么多吧），代码很长，就跟你改不完的bug一样多。好了，赶紧抓重点，在这个方法中我们找到了performMeasure、performLayout、performDraw这三个方法，这好像就是View绘制的三大流程（但好像还有点不像啊），**这三个方法分别完成顶级View（DecorView）的measure、layout、draw过程，在performMeasure中回调用measure方法，在measure方法中又会调用onMeasure方法，在onMeasure方法中则会对所有子View进行measure过程，这时候measure流程就从父容器传到子View中了，这样就完成了一次measure过程。接着子View会重复父容器的过程。performLayout和performDraw的传递流程跟performMeasure基本类似，唯一不同的是performDraw在draw方法是通过dispatchDraw实现的**

接下来就一一分析measure、layout、draw吧

## MeasureSpec

在分析measure之前，先来了解MeasureSpec是做什么的把

MeasureSpec决定了View的测量过程，是measure过程重要的一个类

> MeasureSpec是一个32位int值，高2位代表SpecMode（测量模式），低30位代表SpecSize（在某种测量模式下的测量大小）

```java
public static class MeasureSpec {
    private static final int MODE_SHIFT = 30;
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

    /** @hide */
    @IntDef({UNSPECIFIED, EXACTLY, AT_MOST})
    @Retention(RetentionPolicy.SOURCE)
    public @interface MeasureSpecMode {}

    public static final int UNSPECIFIED = 0 << MODE_SHIFT;
    public static final int EXACTLY     = 1 << MODE_SHIFT;
    public static final int AT_MOST     = 2 << MODE_SHIFT;
    //构建MeasureSpec
    public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                      @MeasureSpecMode int mode) {
        if (sUseBrokenMakeMeasureSpec) {
            return size + mode;
        } else {
            return (size & ~MODE_MASK) | (mode & MODE_MASK);
        }
    }

    public static int makeSafeMeasureSpec(int size, int mode) {
        if (sUseZeroUnspecifiedMeasureSpec && mode == UNSPECIFIED) {
            return 0;
        }
        return makeMeasureSpec(size, mode);
    }

    @MeasureSpecMode
    public static int getMode(int measureSpec) {
        //noinspection ResourceType
        return (measureSpec & MODE_MASK);
    }

    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }

    static int adjust(int measureSpec, int delta) {
        final int mode = getMode(measureSpec);
        int size = getSize(measureSpec);
        if (mode == UNSPECIFIED) {
            // No need to adjust size for UNSPECIFIED mode.
            return makeMeasureSpec(size, UNSPECIFIED);
        }
        size += delta;
        if (size < 0) {
            Log.e(VIEW_LOG_TAG, "MeasureSpec.adjust: new size would be negative! (" + size +
                    ") spec: " + toString(measureSpec) + " delta: " + delta);
            size = 0;
        }
        return makeMeasureSpec(size, mode);
    }
    ...
}
```

MeasureSpec是View的内部类，里面代码并不多。MeasureSpec通过makeMeasureSpec方法将SpecMode和SpecSize打包成一个int值类避免过多的对象内存分配，然后也提供了解包的方法getMode、getSize来分别获取SpecMode和SpecSize。所以MeasureSpec这个类就是一个工具类，用来将SpecMode和SpecSize打包成一个int值，然后通过这个int值和对应的方法，可以获取对应的SpecMode和SpecSize

**SpecMode有三类**:

- UNSPECIFIED
    父容器对View不会有任何限制，要多大给多大，一般用于系统内部，表示一种测量状态
- EXACTLY
    父容器已经检测出View所需要的精确大小，这个时候View的最终大小就是SpecSize指定的值，对应于LayoutParams中的match_parent和具体的数值这两种模式
- AT_MOST
    父容器指定了一个可用大小的SpecSize，View的大小不能大于这个值，具体是什么看不同View的具体实现。对应于LayoutParams中的wrap_content

**MeasureSpec和LayoutParams的关系**：
系统内部是通过MeasureSpec来进行View测量，但我们可以给View设置LayoutParams，在View测量的时候，系统会将LayoutParams在父容器的约束下转换成对应的MeasureSpec，然后再根据这个MeasureSpec来确定View测量后的宽/高。（注意：自身的LayoutParams和父容器一起才能决定View的MeasureSpec，从而决定View的宽/高；有一个例外就是顶级View——DecorView的MeasureSpec是由窗口的尺寸和自身的LayoutParams来决定的），MeasureSpec确定后，onMeasure中就可以确定View的宽/高了

对于DecorView来说，通过performMeasure(childWidthMeasureSpec, childHeightMeasureSpec)来进行测量的，在这之前，会通过`windowSizeMayChange |= measureHierarchy(host, lp, res, desiredWindowWidth, desiredWindowHeight);`这行代码先询问要多大的空间，屏幕的尺寸是否修改，传进的参数是屏幕的尺寸和lp，lp是WindowManager.LayoutParams类型，是窗口的参数

```java
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
        final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
    int childWidthMeasureSpec;
    int childHeightMeasureSpec;
    boolean windowSizeMayChange = false;

    if (DEBUG_ORIENTATION || DEBUG_LAYOUT) Log.v(mTag,
            "Measuring " + host + " in display " + desiredWindowWidth
            + "x" + desiredWindowHeight + "...");

    boolean goodMeasure = false;
    ...
    if (!goodMeasure) {
        //根据屏幕的尺寸获取自身的MeasureSpec
        childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
        childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
            windowSizeMayChange = true;
        }
    }
    ...
    return windowSizeMayChange;
}
```

childWidthMeasureSpec和childHeightMeasureSpec的赋值是通过getTootMeasureSpec方法得到的，根据屏幕的尺寸来获取的，那接着看看这个getRootMeasureSpec方法

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {

    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

这里就是产生MeasureSpec的地方了，LayoutParams.MATCH_PARENT对应就是窗口的大小；LayoutParams.WRAP_CONTENT就是最大模式，大小不定，不能超过窗口的大小；固定大小就是指定大小。

对于DecorView这个顶级View来说，就是通过屏幕的尺寸和窗口的LayoutParams来获取MeasureSpec的

## measure

### ViewGroup的测量，这里以DecorView为例

measure过程从ViewRootImpl的performMeasure开始

```java
private void performTraversals() {
    ...
    //final WindowManager.LayoutParams mWindowAttributes = new WindowManager.LayoutParams();
    WindowManager.LayoutParams lp = mWindowAttributes;
    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
}

private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

首先看一下传进来的参数是什么？

childWidthMeasureSpec和childHeightMeasureSpec都是通过`getRootMeasureSpec()`方法来获取的，这个方法前面我们讲过了（[MeasureSpec](http://blog.penghesheng.cn/2019/04/25/Android View系列(一)——View源码分析/#MeasureSpec)），就是根据传入的size来判定对应的mode，然后组装成一个MeasureSpec。以childWidthMeasureSpec来讲，那么这里我们传递的参数`mWidth`和`lp.width`是什么呢？`mWidth`是通过`mWidth = frame.width();`来获取的，其实就是Window的最大值；`lp.width`是WindowAttributes，代码中注释了具体来源，就是WindowManager.LayoutParams；总的来说，这里由于是DecorView，所以自然就是根据屏幕来获取的这个MeasureSpec

接着看`performMeasure()`，mView是在ViewRootImpl实例化的时候传进的DecorView，DecorView本身也是一个View，接着调用View的measure方法测量child的width和height

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...

    if (forceLayout || needsLayout) {
        ...
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } else {
            long value = mMeasureCache.valueAt(cacheIndex);
            // Casting a long to int drops the high 32 bits, no mask needed
            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
        ...
    }
    ...
}
```

measure方法是一个final方法，就意味着子类不能重写，在View的measure方法中，会去调用onMeasure方法测量自己本身，由于这里分析的View是DecorView，它是一个ViewGroup，所以会调用自己重写的onMeasure方法，然后会去测量子View

### 子View的测量（普通View）

**注意**：因为这里分析的DecorView，DecorView也是一个ViewGroup，会在onMeasure方法中对所有子View进行测量

DecorView的onMeasure方法

```java
 @Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    final DisplayMetrics metrics = getContext().getResources().getDisplayMetrics();
    final boolean isPortrait =
            getResources().getConfiguration().orientation == ORIENTATION_PORTRAIT;

    final int widthMode = getMode(widthMeasureSpec);
    final int heightMode = getMode(heightMeasureSpec);

    boolean fixedWidth = false;
    mApplyFloatingHorizontalInsets = false;
    //如果SpecMode不是EXACTLY的，则需要在这里调整为EXACTLY
    if (widthMode == AT_MOST) {
        final TypedValue tvw = isPortrait ? mWindow.mFixedWidthMinor : mWindow.mFixedWidthMajor;
        if (tvw != null && tvw.type != TypedValue.TYPE_NULL) {
            final int w;
            //根据DecorView属性，计算出DecorView需要的宽度
            if (tvw.type == TypedValue.TYPE_DIMENSION) {
                w = (int) tvw.getDimension(metrics);
            } else if (tvw.type == TypedValue.TYPE_FRACTION) {
                w = (int) tvw.getFraction(metrics.widthPixels, metrics.widthPixels);
            } else {
                w = 0;
            }
            if (DEBUG_MEASURE) Log.d(mLogTag, "Fixed width: " + w);
            final int widthSize = MeasureSpec.getSize(widthMeasureSpec);
            if (w > 0) {
                //根据上面计算出来的需要的宽度生成新的MeasureSpec用于DecorView的测量流程
                widthMeasureSpec = MeasureSpec.makeMeasureSpec(
                        Math.min(w, widthSize), EXACTLY);
                fixedWidth = true;
            } else {
                widthMeasureSpec = MeasureSpec.makeMeasureSpec(
                        widthSize - mFloatingInsets.left - mFloatingInsets.right,
                        AT_MOST);
                mApplyFloatingHorizontalInsets = true;
            }
        }
    }

    mApplyFloatingVerticalInsets = false;
    //如果SpecMode不是EXACTLY的，则需要在这里调整为EXACTLY
    if (heightMode == AT_MOST) {
        final TypedValue tvh = isPortrait ? mWindow.mFixedHeightMajor
                : mWindow.mFixedHeightMinor;
        if (tvh != null && tvh.type != TypedValue.TYPE_NULL) {
            final int h;
            if (tvh.type == TypedValue.TYPE_DIMENSION) {
                h = (int) tvh.getDimension(metrics);
            } else if (tvh.type == TypedValue.TYPE_FRACTION) {
                h = (int) tvh.getFraction(metrics.heightPixels, metrics.heightPixels);
            } else {
                h = 0;
            }
            if (DEBUG_MEASURE) Log.d(mLogTag, "Fixed height: " + h);
            final int heightSize = MeasureSpec.getSize(heightMeasureSpec);
            if (h > 0) {
                heightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        Math.min(h, heightSize), EXACTLY);
            } else if ((mWindow.getAttributes().flags & FLAG_LAYOUT_IN_SCREEN) == 0) {
                heightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        heightSize - mFloatingInsets.top - mFloatingInsets.bottom, AT_MOST);
                mApplyFloatingVerticalInsets = true;
            }
        }
    }
    ...
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    ...
}
```

在DecorView的onMeasure方法中，会先进行SpecMode的调整，调整为EXACTLY，然后回创建出新的MeasureSpec（宽高都一样的逻辑），接着就调用了父类的onMeasure方法

接着看父类FrameLayout的onMeasure方法

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();
    // 如果测量规格有一个不是精确值，这里就为true
    final boolean measureMatchParentChildren =
            MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
            MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
    mMatchParentChildren.clear();

    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;
    //遍历所有子View
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            // 测量子view
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            //由于子布局占用的尺寸除了自身宽高之外，还包含了其距离父布局的边界的值，所以需要加上左右Margin值
            maxWidth = Math.max(maxWidth,
                    child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
            maxHeight = Math.max(maxHeight,
                    child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
            childState = combineMeasuredStates(childState, child.getMeasuredState());
            //当前的FrameLayout的MeasureSpec不都是EXACTLY,且其子View为MATCH_PARENT，
　　　　　　　　　 //则子View保存到mMatchParentChildren中，后面重新测量
　　　　　　　　   //DecorView不会走这个逻辑，因为进过了DecorView的onMeasure()流程，MeasureSpec一定都为EXACTLY
　　　　　　　　　　//会走到下面流程的情况举例：用户自布局一个FrameLayout属性为WRAP_CONTENT是，但子布局为MATCH_PARENT
            if (measureMatchParentChildren) {
                if (lp.width == LayoutParams.MATCH_PARENT ||
                        lp.height == LayoutParams.MATCH_PARENT) {
                    mMatchParentChildren.add(child);
                }
            }
        }
    }
    //最后计算得到的maxWidth和maxHeight的值需要保证能够容纳下当前Layout下所有子View，所以需要对各类情况进行处理
　　　　　//所以有以下的加上Padding值，用户设置的Mini尺寸值的对比，设置了背景图片情况的图片大小对比
    // Account for padding too
    maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
    maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

    // Check against our minimum height and width
    maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

    // Check against our foreground's minimum height and width
    final Drawable drawable = getForeground();
    if (drawable != null) {
        maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
        maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
    }

    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            resolveSizeAndState(maxHeight, heightMeasureSpec,
                    childState << MEASURED_HEIGHT_STATE_SHIFT));

    count = mMatchParentChildren.size();
    // 这里有值表明了两点：
    // 1 当前FrameLayout的宽和高的建议规格有不是精确值的
    // 2 子view有含有match_parent的地方
    if (count > 1) {
        ...
    }
}
```

FrameLayout的测量很简单，首先遍历所有子View，然后调用measureChildWithMargins进行子View的测量，这个方法是在ViewGroup中的，FrameLayout也是继承自ViewGroup。后面就是相关FrameLayout的测量了

那接着看ViewGroup的measureChildWithMargins方法

```java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

这是ViewGroup测量子View提供方法的一种，另一个是measureChildren，区别就是有没有Margins，在这里获取子View的MeasureSpec，然后调用子View自身的measure方法进行测量

取出子View的LayoutParams，然后通过getChildMeasureSpec创建子View的MeasureSpec，接着直接传递给View进行测量；**在获取子View的MeasureSepc时，是通过父布局的MeasureSpec和子View本身的LayoutParams共同来决定的**，这点可以直接从代码中直到

先看看getChildMeasureSpec方法

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    //获取父容器的SpecMode和SpecSize
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
    //父容器减去已经占用的空间，就是子View可用的空间大小
    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            //具体的大小
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

普通View根据父容器的MeasureSpec来判断是哪一种测量模式，接着同时结合View本身的LayoutParams来确定子元素的MeasureSpec，padding指父容器中已占用的空间大小，因此子View可用的大小是父容器减去padding`int size = Math.max(0, specSize - padding);`

普通View的MeasureSpec的创建规则：
| | | | |
|:-:|:-:|:-:|:-:|
| childLayoutParams\parentSpecMode | EXACTLY | AT_MOST | UNSPECIFIED |
| dp/px | EXACTLY(childSize) | EXACTLY(childSize) | EXACTLY(childSize) |
| match_parent | EXACTLY(parentSize) | AT_MOST(parentSize) | UNSPECIFIED(0) |
| wrap_content | AT_MOST(parentSize) | AT_MOST(parentSize) | UNSOECIFIED(0) |
| | | | |

回到之前，通过父容器的MeasureSpec和自身的LayoutParams确定了子View的MeasureSpec，那接着就会调用子View自己的measure方法

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...

    if (forceLayout || needsLayout) {
        ...
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } else {
            long value = mMeasureCache.valueAt(cacheIndex);
            // Casting a long to int drops the high 32 bits, no mask needed
            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
        ...
    }
    ...
}
```

measure方法是一个final方法，就意味着子类不能重写，在View的measure方法中，会去调用onMeasure方法

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

setMeasuredDimension方法会设置View的宽、高的测量值，所以我们先看getDefaultSize方法

```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

> 对于我们来说，只需要看AT_MOST和EXACTLY两种情况，实际上这个方法返回的大小就是measureSpec中的specSize，而这个specSize就是View测量后的大小，而最终的大小是由layout确定的（尽管这样，几乎所有情况，测量大小和最终大小是相等的）。
>
> UNSPECIFIED这种情况一般用于系统内部的测量过程，这种情况下，View的大小为getDefaultSize的第一个参数size，即getSuggestedMinimumWidth、getSuggestedMinimumHeight
>
> ```java
>     protected int getSuggestedMinimumWidth() {
>         return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
>     }
> 
>     protected int getSuggestedMinimumHeight() {
>          return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
>      }
> ```

> 如果View没有设置背景，那么View的宽度、高度为mMinWidth、mMinHeight，这两个对应`android:minWidth`、`android:minHeight`这个两个属性指定的值，如果不指定默认为0；如果设置了背景，则为`max(mMinWidth, mBackground.getMinimumWidth());`
>
> ```java
>     public int getMinimumWidth() {
>         final int intrinsicWidth = getIntrinsicWidth();
>         return intrinsicWidth > 0 ? intrinsicWidth : 0;
>     }
> 
>     public int getMinimumHeight() {
>         final int intrinsicHeight = getIntrinsicHeight();
>         return intrinsicHeight > 0 ? intrinsicHeight : 0;
>     }
> ```

> getMinimumWidth返回的就是Drawable的原始宽度、高度，如果Drawable没有原始值，则为0

```java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int opticalWidth  = insets.left + insets.right;
        int opticalHeight = insets.top  + insets.bottom;

        measuredWidth  += optical ? opticalWidth  : -opticalWidth;
        measuredHeight += optical ? opticalHeight : -opticalHeight;
    }
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}
```

最后将测量的宽高设置，子View的测量就结束了

同样，我们看看ViewGroup中单独提供了一个measureChildren方法

```java
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}
```

通过循环遍历所有的子View，然后调用measureChild方法

```java
protected void measureChild(View child, int parentWidthMeasureSpec,
        int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);
    //调用自身的measure方法，普通View都是通过View的measure方法进行测量
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

最后都是在View中进行自身的测量（对于普通View来说）

## layout

### ViewGroup的layout

layout过程是从ViewRootImpl中的`performLayout(lp, mWidth, mHeight);`开始的

```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
        int desiredWindowHeight) {
    mLayoutRequested = false;
    mScrollMayChange = true;
    mInLayout = true;

    final View host = mView;
    if (host == null) {
        return;
    }
    if (DEBUG_ORIENTATION || DEBUG_LAYOUT) {
        Log.v(mTag, "Laying out " + host + " to (" +
                host.getMeasuredWidth() + ", " + host.getMeasuredHeight() + ")");
    }

    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
    try {
        //对自身进行layout
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

        mInLayout = false;
        int numViewsRequestingLayout = mLayoutRequesters.size();
        if (numViewsRequestingLayout > 0) {
            // requestLayout() was called during layout.
            // If no layout-request flags are set on the requesting views, there is no problem.
            // If some requests are still pending, then we need to clear those flags and do
            // a full request/measure/layout pass to handle this situation.
            ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                    false);
            if (validLayoutRequesters != null) {
                // Set this flag to indicate that any further requests are happening during
                // the second pass, which may result in posting those requests to the next
                // frame instead
                mHandlingLayoutInLayoutRequest = true;

                // Process fresh layout requests, then measure and layout
                int numValidRequests = validLayoutRequesters.size();
                for (int i = 0; i < numValidRequests; ++i) {
                    final View view = validLayoutRequesters.get(i);
                    Log.w("View", "requestLayout() improperly called by " + view +
                            " during layout: running second layout pass");
                    view.requestLayout();
                }
                measureHierarchy(host, lp, mView.getContext().getResources(),
                        desiredWindowWidth, desiredWindowHeight);
                mInLayout = true;
                host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

                mHandlingLayoutInLayoutRequest = false;

                // Check the valid requests again, this time without checking/clearing the
                // layout flags, since requests happening during the second pass get noop'd
                validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                if (validLayoutRequesters != null) {
                    final ArrayList<View> finalRequesters = validLayoutRequesters;
                    // Post second-pass requests to the next frame
                    getRunQueue().post(new Runnable() {
                        @Override
                        public void run() {
                            int numValidRequests = finalRequesters.size();
                            for (int i = 0; i < numValidRequests; ++i) {
                                final View view = finalRequesters.get(i);
                                Log.w("View", "requestLayout() improperly called by " + view +
                                        " during second layout pass: posting in next frame");
                                view.requestLayout();
                            }
                        }
                    });
                }
            }

        }
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    mInLayout = false;
}
```

在这个方法中，通过`host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());`来对自身进行layout，host是mView，在这里也就是DecorView。`getMeasuredWidth()`和`getMeasuredHeight()`获取的是前面Measure后的值，而这个layout方法就是View中的layout方法了

```java
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);

        if (shouldDrawRoundScrollbar()) {
            if(mRoundScrollbarRenderer == null) {
                mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
            }
        } else {
            mRoundScrollbarRenderer = null;
        }

        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
        //可以通过ListenerInfo中的OnLayoutChangeListener来获取Layout修改的值
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }

    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

    if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
        mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
        notifyEnterOrExitForAutoFillIfNeeded(true);
    }
}
```

首先会通过setFrame方法来设定View的四个顶点的位置，即初始化mLeft、mRight、mTop、mBottom四个值（**需要注意的是：**这些值都是针对于父View来说的），View四个顶点确定了，那么在父容器的位置也就确定了，接着会调用onLayout方法，这个方法是父容器确定子View的位置（类似onMeasure），onLayout方法是一个空的方法，说明最后都是View自己实现的

那么我们这里是DecorView，传进的参数是`host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());`，host.getMeasuredWidth()和host.getMeasuredHeight()获取的宽高就是整个屏幕的宽高，那么也就意味着我们的DecorView就是占满了整个屏幕空间

### 子View的layout（普通View）

DecorView是一个ViewGroup，它的子View的layout过程会在onLayout方法中开始

```java
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    super.onLayout(changed, left, top, right, bottom);
    getOutsets(mOutsets);
    //当有偏移的时候就调整
    if (mOutsets.left > 0) {
        offsetLeftAndRight(-mOutsets.left);
    }
    if (mOutsets.top > 0) {
        offsetTopAndBottom(-mOutsets.top);
    }
    if (mApplyFloatingVerticalInsets) {
        offsetTopAndBottom(mFloatingInsets.top);
    }
    if (mApplyFloatingHorizontalInsets) {
        offsetLeftAndRight(mFloatingInsets.left);
    }

    // If the application changed its SystemUI metrics, we might also have to adapt
    // our shadow elevation.
    updateElevation();
    mAllowUpdateElevation = true;

    if (changed && mResizeMode == RESIZE_MODE_DOCKED_DIVIDER) {
        getViewRootImpl().requestInvalidateRootRenderNode();
    }
}
```

DecorView中的onLayout方法并没有做太多的事情，但是在方法中的第一行就调用了父类的onLayout方法。我们知道DecorView是继承自FrameLayout的，所以布局是在FrameLayout中完成的，接着看看FrameLayout的onLayout方法

```java
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    layoutChildren(left, top, right, bottom, false /* no force left gravity */);
}
```

在FrameLayout的onLayout方法直接调用了layoutChildren方法

```java
void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
    //获取子View的数量
    final int count = getChildCount();
    //获取各个方向的padding值
    final int parentLeft = getPaddingLeftWithForeground();
    final int parentRight = right - left - getPaddingRightWithForeground();

    final int parentTop = getPaddingTopWithForeground();
    final int parentBottom = bottom - top - getPaddingBottomWithForeground();
    //遍历所有的子View
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            //获取LayoutParams
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            //获取测量后的宽高
            final int width = child.getMeasuredWidth();
            final int height = child.getMeasuredHeight();

            int childLeft;
            int childTop;

            int gravity = lp.gravity;
            if (gravity == -1) {
                gravity = DEFAULT_CHILD_GRAVITY;
            }
            // 获取布局方向，可能的值为rtl、ltr,分别为从右至左、从左至右布局
            final int layoutDirection = getLayoutDirection();
            // 结合layoutDirection和View的layout_gravity计算出真正的gravity
            // Gravity.getAbsoluteGravity是一个静态方法
            final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
            // 水平方向如果为从左到右，那么，Gratity为START会被设置成LEFT，END会被设置成RIGHT
            // 水平方向如果为从右到左，那么，Gratity为START会被设置成RIGHT，END会被设置成LEFT
            final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;
            // 根据不同的水平比重计算View的水平坐标
            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                    lp.leftMargin - lp.rightMargin;
                    break;
                case Gravity.RIGHT:
                    if (!forceLeftGravity) {
                        childLeft = parentRight - width - lp.rightMargin;
                        break;
                    }
                case Gravity.LEFT:
                default:
                    childLeft = parentLeft + lp.leftMargin;
            }
            //根据垂直比重计算View的垂直坐标
            switch (verticalGravity) {
                case Gravity.TOP:
                    childTop = parentTop + lp.topMargin;
                    break;
                case Gravity.CENTER_VERTICAL:
                    childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                    lp.topMargin - lp.bottomMargin;
                    break;
                case Gravity.BOTTOM:
                    childTop = parentBottom - height - lp.bottomMargin;
                    break;
                default:
                    childTop = parentTop + lp.topMargin;
            }
            //调用子View的layout方法进行子View的layout
            child.layout(childLeft, childTop, childLeft + width, childTop + height);
        }
    }
}
```

很明显，这个方法会遍历所有的子View，计算出它的位置，最后调用子View自身的layout进行layout过程

View的layout方法

```java
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);

        ...
    }

    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

    if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
        mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
        notifyEnterOrExitForAutoFillIfNeeded(true);
    }
}
```

在View自身的layout过程中，最终就调用其onLayout方法（又是一个递归的过程）

## draw

### ViewGroup的绘制

从ViewRootImpl的performDraw方法开始

```java
private void performDraw() {
    ...
    final boolean fullRedrawNeeded = mFullRedrawNeeded;
    mFullRedrawNeeded = false;

    mIsDrawing = true;
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
    try {
        draw(fullRedrawNeeded);
    } finally {
        mIsDrawing = false;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    ...
}
```

这个方法中的重点就是draw方法

```java
private void draw(boolean fullRedrawNeeded) {
    Surface surface = mSurface;
    if (!surface.isValid()) {
        return;
    }

    if (DEBUG_FPS) {
        trackFPS();
    }
    //如果是第一次绘制，则会回调到sFirstDrawHandlers中的事件
    //在ActivityThread.attch()方法中有将回调事件加入该队列
    //回调时会执行ActivityThread.ensureJitEnable来确保即时编译相关功能
    if (!sFirstDrawComplete) {
        synchronized (sFirstDrawHandlers) {
            sFirstDrawComplete = true;
            final int count = sFirstDrawHandlers.size();
            for (int i = 0; i< count; i++) {
                mHandler.post(sFirstDrawHandlers.get(i));
            }
        }
    }
    //滚动相关处理，如果scroll发生改变，则回调dispatchOnScrollChanged()方法
    scrollToRectOrFocus(null, false);

    if (mAttachInfo.mViewScrollChanged) {
        mAttachInfo.mViewScrollChanged = false;
        mAttachInfo.mTreeObserver.dispatchOnScrollChanged();
    }
    //窗口当前是否有动画需要执行
    boolean animating = mScroller != null && mScroller.computeScrollOffset();
    ...
    final Rect dirty = mDirty;
    ...

    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        //mAttachInfo.mHardwareRenderer不为null，则表示该Window使用硬件加速进行绘制
        //执行ViewRootImpl.set()方法会判断是否使用硬件加速
        //若判断使用会调用ViewRootImpl.enableHardwareAcceleration()来初始化mHardwareRenderer
　　　　　　　//该View设置为使用硬件加速，且当前硬件加速处于可用状态
        if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
            ...
            //使用硬件加速绘制方式
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        } else {
            // If we get here with a disabled & requested hardware renderer, something went
            // wrong (an invalidate posted right before we destroyed the hardware surface
            // for instance) so we should just bail out. Locking the surface with software
            // rendering at this point would lock it forever and prevent hardware renderer
            // from doing its job when it comes back.
            // Before we request a new frame we must however attempt to reinitiliaze the
            // hardware renderer if it's in requested state. This would happen after an
            // eglTerminate() for instance.
            if (mAttachInfo.mThreadedRenderer != null &&
                    !mAttachInfo.mThreadedRenderer.isEnabled() &&
                    mAttachInfo.mThreadedRenderer.isRequested()) {

                try {
                    //尝试重新初始化当前window的硬件加速
                    mAttachInfo.mThreadedRenderer.initializeIfNeeded(
                            mWidth, mHeight, mAttachInfo, mSurface, surfaceInsets);
                } catch (OutOfResourcesException e) {
                    handleOutOfResourcesException(e);
                    return;
                }

                mFullRedrawNeeded = true;
                scheduleTraversals();
                return;
            }
            //使用软件渲染绘制方式
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                return;
            }
        }
    }
    //有动画进行重绘
    if (animating) {
        mFullRedrawNeeded = true;
        scheduleTraversals();
    }
}
```

mDirty表示的是当前需要更新的区域，即脏区域。经过一些scroll相关的处理后，如果脏区域不为空或者有动画需要执行时，便会执行重绘窗口的工作。有两种绘制方式，硬件加速绘制方式和软件渲染绘制方式，在创建窗口流程的ViewRootImpl.setView()中，会根据不同情况，来选择是否创mAttachInfo.mHardwareRenderer对象。如果该对象不为空，则会进入硬件加速绘制方式，即调用到ThreadedRenderer.draw()，这个ThreadedRenderer是不会block住UI线程，但是UI线程可以block住它，主要是在UI线程构建试图然后交给单独的线程通过OpenGL去绘制；否则则会进入软件渲染的绘制方式，调用到ViewRootImpl.drawSoftware()方法。但是无论哪种方式，都会走到mView.draw()方法，即DecorView.draw()方法。

主要看看drawSoftWare方法吧

```java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty) {

    // Draw with software renderer.
    final Canvas canvas;
    try {
        final int left = dirty.left;
        final int top = dirty.top;
        final int right = dirty.right;
        final int bottom = dirty.bottom;
        //获取canvas
        canvas = mSurface.lockCanvas(dirty);

        ...
    } catch (Surface.OutOfResourcesException e) {
        handleOutOfResourcesException(e);
        return false;
    } catch (IllegalArgumentException e) {
        ...
    }

    try {
        ...
        try {
            ...

            mView.draw(canvas);

            drawAccessibilityFocusedDrawableIfNeeded(canvas);
        } finally {
            ...
        }
    } finally {
        ...
    }
    return true;
}
```

在这个方法中，会获取canvas，然后进行一系列的操作，最终会调用mView的draw方法

在这里mView就是DecorView，实际上就是View中的draw方法

```java
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     */

    // Step 1, draw the background, if needed
    int saveCount;

    if (!dirtyOpaque) {
        //绘制背景
        drawBackground(canvas);
    }

    // skip step 2 & 5 if possible (common case)
    //判断是否需要绘制边缘渐变效果（水平方向、垂直方向）
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    //如果不需要绘制边缘渐变效果，跳过了step5
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        //绘制自己View的内容
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        //通过dispatchDraw来绘制子View
        dispatchDraw(canvas);

        drawAutofilledHighlight(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        // Step 7, draw the default focus highlight
        drawDefaultFocusHighlight(canvas);

        if (debugDraw()) {
            debugDrawFocus(canvas);
        }

        // we're done...
        return;
    }

    /*
     * Here we do the full fledged routine...
     * (this is an uncommon case where speed matters less,
     * this is why we repeat some of the tests that have been
     * done above)
     */

    boolean drawTop = false;
    boolean drawBottom = false;
    boolean drawLeft = false;
    boolean drawRight = false;

    float topFadeStrength = 0.0f;
    float bottomFadeStrength = 0.0f;
    float leftFadeStrength = 0.0f;
    float rightFadeStrength = 0.0f;

    // Step 2, save the canvas' layers
    int paddingLeft = mPaddingLeft;

    final boolean offsetRequired = isPaddingOffsetRequired();
    if (offsetRequired) {
        paddingLeft += getLeftPaddingOffset();
    }

    int left = mScrollX + paddingLeft;
    int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
    int top = mScrollY + getFadeTop(offsetRequired);
    int bottom = top + getFadeHeight(offsetRequired);

    if (offsetRequired) {
        right += getRightPaddingOffset();
        bottom += getBottomPaddingOffset();
    }

    final ScrollabilityCache scrollabilityCache = mScrollCache;
    final float fadeHeight = scrollabilityCache.fadingEdgeLength;
    int length = (int) fadeHeight;

    // clip the fade length if top and bottom fades overlap
    // overlapping fades produce odd-looking artifacts
    if (verticalEdges && (top + length > bottom - length)) {
        length = (bottom - top) / 2;
    }

    // also clip horizontal fades if necessary
    if (horizontalEdges && (left + length > right - length)) {
        length = (right - left) / 2;
    }

    if (verticalEdges) {
        topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
        drawTop = topFadeStrength * fadeHeight > 1.0f;
        bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
        drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
    }

    if (horizontalEdges) {
        leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
        drawLeft = leftFadeStrength * fadeHeight > 1.0f;
        rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
        drawRight = rightFadeStrength * fadeHeight > 1.0f;
    }

    saveCount = canvas.getSaveCount();

    int solidColor = getSolidColor();
    if (solidColor == 0) {
        final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

        if (drawTop) {
            canvas.saveLayer(left, top, right, top + length, null, flags);
        }

        if (drawBottom) {
            canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
        }

        if (drawLeft) {
            canvas.saveLayer(left, top, left + length, bottom, null, flags);
        }

        if (drawRight) {
            canvas.saveLayer(right - length, top, right, bottom, null, flags);
        }
    } else {
        scrollabilityCache.setFadeColor(solidColor);
    }

    // Step 3, draw the content
    if (!dirtyOpaque) onDraw(canvas);

    // Step 4, draw the children
    dispatchDraw(canvas);

    // Step 5, draw the fade effect and restore layers
    final Paint p = scrollabilityCache.paint;
    final Matrix matrix = scrollabilityCache.matrix;
    final Shader fade = scrollabilityCache.shader;

    if (drawTop) {
        matrix.setScale(1, fadeHeight * topFadeStrength);
        matrix.postTranslate(left, top);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(left, top, right, top + length, p);
    }

    if (drawBottom) {
        matrix.setScale(1, fadeHeight * bottomFadeStrength);
        matrix.postRotate(180);
        matrix.postTranslate(left, bottom);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(left, bottom - length, right, bottom, p);
    }

    if (drawLeft) {
        matrix.setScale(1, fadeHeight * leftFadeStrength);
        matrix.postRotate(-90);
        matrix.postTranslate(left, top);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(left, top, left + length, bottom, p);
    }

    if (drawRight) {
        matrix.setScale(1, fadeHeight * rightFadeStrength);
        matrix.postRotate(90);
        matrix.postTranslate(right, top);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(right - length, top, right, bottom, p);
    }

    canvas.restoreToCount(saveCount);

    drawAutofilledHighlight(canvas);

    // Overlay is part of the content and draws beneath Foreground
    if (mOverlay != null && !mOverlay.isEmpty()) {
        mOverlay.getOverlayView().dispatchDraw(canvas);
    }

    // Step 6, draw decorations (foreground, scrollbars)
    onDrawForeground(canvas);

    if (debugDraw()) {
        debugDrawFocus(canvas);
    }
}
```

在draw方法中，根据上面注释可以知道draw的过程分为5步

1. 画出背景background
2. 判断是否需要画边缘的渐变效果
3. 画出当前View需要显示的内容，调用onDraw()来实现
4. 调用dispatchDraw()方法，进入子视图的draw逻辑
5. 如果需要花边缘渐变效果，则在这里画
6. 绘制装饰（如滚动条）

在绘制自己的时候，是调用onDraw方法，这个方法是个空实现，是由具体的View自己实现的，我们这里是DecorView，那么就会回到DecorView的onDraw方法

```java
@Override
public void onDraw(Canvas c) {
    super.onDraw(c);

    // When we are resizing, we need the fallback background to cover the area where we have our
    // system bar background views as the navigation bar will be hidden during resizing.
    mBackgroundFallback.draw(isResizing() ? this : mContentRoot, mContentRoot, c,
            mWindow.mContentParent);
}
```

### 子View的绘制

View绘制过程的传递时通过dispatchDraw来实现的，dispatchDraw会遍历所有子元素的draw方法，这样draw事件就一层层的传递下去

dispatchDraw方法是在ViewGroup中实现的

```java
@Override
protected void dispatchDraw(Canvas canvas) {
    boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);
    final int childrenCount = mChildrenCount;
    final View[] children = mChildren;
    int flags = mGroupFlags;
    ...
    //此处不分析具体流程，需要了解的可查看相关博客
    for (int i = 0; i < childrenCount; i++) {
        while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
            final View transientChild = mTransientViews.get(transientIndex);
            if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                    transientChild.getAnimation() != null) {
                //遍历所有子View进行子View的绘制
                more |= drawChild(canvas, transientChild, drawingTime);
            }
            transientIndex++;
            if (transientIndex >= transientCount) {
                transientIndex = -1;
            }
        }

        final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
        final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
        if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
            more |= drawChild(canvas, child, drawingTime);
        }
    }
    ...
}
```

在dispatch中会遍历当前ViewGroup的子视图，然后调用drawChild()方法来依次调起子视图的绘制过程，进入ViewGroup.drawChild()代码

```java
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    return child.draw(canvas, this, drawingTime);
}
```

好了，这就又回到了View中的draw方法了

**注意**
View中有一个特殊的方法setWillNotDraw方法

```java
/**
 * If this view doesn't do any drawing on its own, set this flag to
 * allow further optimizations. By default, this flag is not set on
 * View, but could be set on some View subclasses such as ViewGroup.
 *
 * Typically, if you override {@link #onDraw(android.graphics.Canvas)}
 * you should clear this flag.
 *
 * @param willNotDraw whether or not this View draw on its own
 */
public void setWillNotDraw(boolean willNotDraw) {
    setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
}
```

从注释中可以看出，如果一个View不需要绘制任何内容，那个设置这个标记为true后，系统会进行相应的优化。默认情况下View没有启动这个优化标记位，但是ViewGroup会默认启动这个优化标记位。

## 总结

### Measure总结

顶层View向子View递归调用`View.measure()`方法，measure中又会回调`onMeasure()`方法

- MeasureSpec：测量模式和size的一个组合体，其中SpecMode只有三种

    > MeasureSpec.EXACTLY:：父容器已经检测出View所需要的精确大小，这个时候View的最终大小就是SpecSize指定的值，对应于LayoutParams中的match_parent和具体的数值这两种模式
    >
    > MeasureSpec.AT_MOST：父容器指定了一个可用大小的SpecSize，View的大小不能大于这个值，具体是什么看不同View的具体实现。对应于LayoutParams中的wrap_content
    >
    > MeasureSpec.UNSPECIFIED：父容器对View不会有任何限制，要多大给多大，一般用于系统内部，表示一种测量状态

- View的measure方法是final的，View子类只能在onMeasure方法中完成自己测量逻辑

- 最顶层DecorView测量时的MeasureSpec是由ViewRootImpl中getRootMeasureSpec方法确定的（LayoutParams宽高参数均为MATCH_PARENT，specMode是EXACTLY，specSize为物理屏幕大小）

- ViewGroup类提供了measureChild，measureChild和measureChildWithMargins方法，简化了父子View的尺寸计算

- 只要是ViewGroup的子类就必须要求LayoutParams继承子MarginLayoutParams，否则无法使用layout_margin参数

- View布局的大小由父布局和子view共同决定（父布局的MeasureSpec和子View的LayoutParams）

### Layout总结

类似Measure，都是从顶层View向子View递归调用`View.layoutt()`方法，在layout中又会回调`onLayout()`方法

- View.layout方法可被重载，ViewGroup.layout为final的不可重载，ViewGroup.onLayout为abstract的，子类必须重载实现自己的位置逻辑
- measure操作完成后得到的是对每个View经测量过的measuredWidth和measuredHeight，layout操作完成之后得到的是对每个View进行位置分配后的mLeft、mTop、mRight、mBottom，这些值都是相对于父View来说的
- 使用View的getWidth()和getHeight()方法来获取View测量的宽高，必须保证这两个方法在onLayout流程之后被调用才能返回有效值

### Draw总结

- 6个步骤：
    1. 画出背景background
    2. 判断是否需要画边缘的渐变效果
    3. 画出当前View需要显示的内容，调用onDraw()来实现
    4. 调用dispatchDraw()方法，进入子视图的draw逻辑
    5. 如果需要花边缘渐变效果，则在这里画
    6. 绘制装饰（如滚动条）
- View默认不会绘制任何内容，真正的绘制都需要自己在子类中实现。
- View默认不会绘制任何内容，真正的绘制都需要自己在子类中实现
- 默认情况下子View的ViewGroup.drawChild绘制顺序和子View被添加的顺序一致，但是你也可以重载ViewGroup.getChildDrawingOrder()方法提供不同顺序

## 特别感谢

**Android开发艺术探索**

[源码分析篇 - Android绘制流程（二）measure、layout、draw流程](https://www.cnblogs.com/tiger-wang-ms/p/6543418.html)
[Android应用层View绘制流程与源码分析](https://blog.csdn.net/yanbober/article/details/46128379)

[FrameLayout布局绘制流程解析](https://blog.csdn.net/asdgbc/article/details/48139233)

[View测量机制详解—从DecorView说起](https://blog.csdn.net/qibin0506/article/details/49245601)