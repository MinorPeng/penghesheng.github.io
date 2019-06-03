---
title: Android View系列(三)——LinearLayout源码分析
tag: Android
date: 2019-04-29
---

[TOC]

# LinearLayout 源码分析

## 简介

LinearLayout是我们开发中最常用的布局之一了，就是我们所说的线程布局，其分水平（HORIZONTAL）和垂直（VERTICAL）两个方向进行布局，接下来我们就慢慢看看这个LinearLayout

首先看看类的继承关系

```java
public class LinearLayout extends ViewGroup {
	...
}
```

很明显，直接继承自ViewGroup，当然，其他常用的布局RealativeLayout、FrameLayout等都是直接继承ViewGroup的

接着简单看看LinearLayout中的一些属性吧

```java
//水平方向和垂直方向
   public static final int HORIZONTAL = 0;
   public static final int VERTICAL = 1;
//几种分割线的状态
   //不展示dividers
   public static final int SHOW_DIVIDER_NONE = 0;
   //展示dividers在group的前面
   public static final int SHOW_DIVIDER_BEGINNING = 1;
   //给viewgroup的每一个item之间都展示divider
   public static final int SHOW_DIVIDER_MIDDLE = 2;
   //展示在ViewGroup的后面
   public static final int SHOW_DIVIDER_END = 4;
...
   //基线对齐变量
   @ViewDebug.ExportedProperty(category = "layout")
   private boolean mBaselineAligned = true;
//基线对齐对象index
   @ViewDebug.ExportedProperty(category = "layout")
   private int mBaselineAlignedChildIndex = -1;
//基线的额外偏移量
   @ViewDebug.ExportedProperty(category = "measurement")
   private int mBaselineChildTop = 0;
//方向
   @ViewDebug.ExportedProperty(category = "measurement")
   private int mOrientation;
...
   //对齐方式
   private int mGravity = Gravity.START | Gravity.TOP;
//整个LinearLayout子View的高度和（Vertical）或者宽度和（Historical）
   @ViewDebug.ExportedProperty(category = "measurement")
   private int mTotalLength;
//权重和
   @ViewDebug.ExportedProperty(category = "layout")
   private float mWeightSum;
//权重最小尺寸对象
   @ViewDebug.ExportedProperty(category = "layout")
   private boolean mUseLargestChild;
//基线对齐相关
   private int[] mMaxAscent;
   private int[] mMaxDescent;

   private static final int VERTICAL_GRAVITY_COUNT = 4;
//几种index
   private static final int INDEX_CENTER_VERTICAL = 0;
   private static final int INDEX_TOP = 1;
   private static final int INDEX_BOTTOM = 2;
   private static final int INDEX_FILL = 3;
//分割线相关
   private Drawable mDivider;
   private int mDividerWidth;
   private int mDividerHeight;
   private int mShowDividers;
   private int mDividerPadding;

   private int mLayoutDirection = View.LAYOUT_DIRECTION_UNDEFINED;
```

属性简单就看这么多吧，接着进入重点吧，看看LinearLayout是怎么测量、布局、绘制的

## onMeasure

```java
@Override
   protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
       if (mOrientation == VERTICAL) {
           measureVertical(widthMeasureSpec, heightMeasureSpec);
       } else {
           measureHorizontal(widthMeasureSpec, heightMeasureSpec);
       }
   }
```

在onMeasure方法中，对方向进行了判断，来决定是垂直测量还是横向测量，我们就看垂直方向的吧

由于方法有点长，分几个部分一点一点看吧，先看这个函数内的一些变量吧

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
       //每次测量都会将ziView的高度和进行初始化
       mTotalLength = 0;
       //子View最大宽度，不包括含有layout_weight权重的子View
       int maxWidth = 0;
       //测量状态
       int childState = 0;
       //子View中layout_weight <= 0的最大高度
       int alternativeMaxWidth = 0;
       //子View中layout_weight > 0的最大高度
       int weightedMaxWidth = 0;
       //是否全是match_parent
       boolean allFillParent = true;
       //子View权重和
       float totalWeight = 0;
	//子View个数
       final int count = getVirtualChildCount();
	//测量模式
       final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
       final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
	//为match_parent时未true
       boolean matchWidth = false;
       boolean skippedMeasure = false;
	//基线对齐的对象index
       final int baselineChildIndex = mBaselineAlignedChildIndex;
       //权重最小尺寸对象
       final boolean useLargestChild = mUseLargestChild;
       //子View中最高高度
       int largestChildHeight = Integer.MIN_VALUE;
       int consumedExcessSpace = 0;
       //能够显示的View个数
       int nonSkippedChildCount = 0;
       ...
   }
```

这些变量大致的意思都写在代码注释里了，简单过一下就ok了

接着看看方法内的后面一部分，测量所有的子View

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
	...
	// See how tall everyone is. Also remember max width.
       for (int i = 0; i < count; ++i) {
           final View child = getVirtualChildAt(i);
           if (child == null) {
               //为null，+0
               mTotalLength += measureNullChild(i);
               continue;
           }
		//View.GONE的子View也不会计算在里面
           if (child.getVisibility() == View.GONE) {
              i += getChildrenSkipCount(child, i);
              continue;
           }
           
           nonSkippedChildCount++;
           //如果有分割线，则要加上分割线的高度
           if (hasDividerBeforeChildAt(i)) {
               mTotalLength += mDividerHeight;
           }
		//获取子View的LayoutParams并且强转为父布局的LayoutParams
           final LayoutParams lp = (LayoutParams) child.getLayoutParams();
		//累加权重
           totalWeight += lp.weight;
		//是否使用多余的空间，如果设置了权重，并且height==0，表示希望使用剩下的空间，则跳过此次测量，稍后进行测量
           final boolean useExcessSpace = lp.height == 0 && lp.weight > 0;
           //有权重的情况下且LinearLayout高度确定
           if (heightMode == MeasureSpec.EXACTLY && useExcessSpace) {
               final int totalLength = mTotalLength;
               mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
               skippedMeasure = true;
           } else {
               if (useExcessSpace) {
                   //有权重且LinearLayout高度不确定的情况，则重置子View高度为wrap_content
                   lp.height = LayoutParams.WRAP_CONTENT;
               }
               //正常测量过程
               final int usedHeight = totalWeight == 0 ? mTotalLength : 0;
               //进行子View的测量
               measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                       heightMeasureSpec, usedHeight);

               final int childHeight = child.getMeasuredHeight();
               if (useExcessSpace) {
                   //重置子View高度
                   lp.height = 0;
                   consumedExcessSpace += childHeight;
               }

               final int totalLength = mTotalLength;
               //计算较大的totalLength
               mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                      lp.bottomMargin + getNextLocationOffset(child));
			//设置最高子View大小
               if (useLargestChild) {
                   largestChildHeight = Math.max(childHeight, largestChildHeight);
               }
           }
		//mBaselineChildTop 表示指定的 baseline 的子视图的顶部高度
           if ((baselineChildIndex >= 0) && (baselineChildIndex == i + 1)) {
              mBaselineChildTop = mTotalLength;
           }
		//为baseline的子View不允许有weight属性
           if (i < baselineChildIndex && lp.weight > 0) {
               throw new RuntimeException("A child of LinearLayout with index "
                       + "less than mBaselineAlignedChildIndex has weight > 0, which "
                       + "won't work.  Either remove the weight, or don't set "
                       + "mBaselineAlignedChildIndex.");
           }
		//当LinearLayout非EXACTLY模式 并且自View为MATCH_PARENT时
     		//设置matchWidth和matchWidthLocally为true
     		//该子View占据LinearLayout水平方向上所有空间
           boolean matchWidthLocally = false;
           if (widthMode != MeasureSpec.EXACTLY && lp.width == LayoutParams.MATCH_PARENT) {
               matchWidth = true;
               matchWidthLocally = true;
           }
		//进行变量的赋值
           final int margin = lp.leftMargin + lp.rightMargin;
           final int measuredWidth = child.getMeasuredWidth() + margin;
           maxWidth = Math.max(maxWidth, measuredWidth);
           childState = combineMeasuredStates(childState, child.getMeasuredState());
           allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;
           if (lp.weight > 0) {
               weightedMaxWidth = Math.max(weightedMaxWidth,
                       matchWidthLocally ? margin : measuredWidth);
           } else {
               alternativeMaxWidth = Math.max(alternativeMaxWidth,
                       matchWidthLocally ? margin : measuredWidth);
           }
           i += getChildrenSkipCount(child, i);
       }
       ...
   }
```

遍历所有子View，根据index获取一个子View，在这里我们看到为null的时候mTotalLength加的是0，当子View是View.GONE时，也是加0，所以从这里可以看出，当子View为View.GONE是不计算在内的；这里简单提一下，内部获取子View的LayoutParams是要强转为父View的LayoutParams的，所以通常在代码来控制子View的位置是要获取父View的LayoutParams

接着判断当LinearLayout为EXACTLY时且子View设置了权重，表示这个子View要使用剩下的空间，那么就会先跳过这次正常的测量，会在后面有权重的时候进行单独的测量

接着正常子View的测量，mTotalLength累加子View的高度，测量完成重置子View的高度等；然后接着测量View宽度

接着看后面的

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
	...
       //有divider时加上divider的高度
	if (nonSkippedChildCount > 0 && hasDividerBeforeChildAt(count)) {
           mTotalLength += mDividerHeight;
       }
       //如果高度测量模式为AT_MOST或者UNSPECIFIED 则进行二次测量 useLargestChild
	if (useLargestChild &&
               (heightMode == MeasureSpec.AT_MOST || heightMode == MeasureSpec.UNSPECIFIED)) {
           mTotalLength = 0;
           for (int i = 0; i < count; ++i) {
               final View child = getVirtualChildAt(i);
               if (child == null) {
                   mTotalLength += measureNullChild(i);
                   continue;
               }

               if (child.getVisibility() == GONE) {
                   i += getChildrenSkipCount(child, i);
                   continue;
               }

               final LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams)
                       child.getLayoutParams();
               final int totalLength = mTotalLength;
               mTotalLength = Math.max(totalLength, totalLength + largestChildHeight +
                       lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
           }
       }
       //加上paddingTo、paddingBottom
       mTotalLength += mPaddingTop + mPaddingBottom;
       int heightSize = mTotalLength;
       // Check against our minimum height
       heightSize = Math.max(heightSize, getSuggestedMinimumHeight());
       // Reconcile our calculated size with the heightMeasureSpec
       int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
       heightSize = heightSizeAndState & MEASURED_SIZE_MASK;
       ...
}
```

这次就是判断是否需要第二次测量，如果高度测量模式为AT_MOST或者UNSPECIFIED 则进行二次测量 且设置了useLargestChild，而useLargestChild是mUseLargestChild，而mUseLargestChild是在LinearLayout初始化时，在布局中设置，`mUseLargestChild = a.getBoolean(R.styleable.LinearLayout_measureWithLargestChild, false);`也就是说只有设置了measureWithLargestChild为true（默认false）时才可能进行二次测量

然后后面就是mTotalLength加上padding，重新计算heightSize

接着看后面

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
	...
       //额外空间的计算
	int remainingExcess = heightSize - mTotalLength
               + (mAllowInconsistentMeasurement ? 0 : consumedExcessSpace);
       if (skippedMeasure || remainingExcess != 0 && totalWeight > 0.0f) {
           //如果有总的权重和，就使用设置的，否则使用子View计算来的
           float remainingWeightSum = mWeightSum > 0.0f ? mWeightSum : totalWeight;
           mTotalLength = 0;
           //有权重，需要进行二次测量有权重的子View
           for (int i = 0; i < count; ++i) {
               final View child = getVirtualChildAt(i);
               if (child == null || child.getVisibility() == View.GONE) {
                   continue;
               }

               final LayoutParams lp = (LayoutParams) child.getLayoutParams();
               final float childWeight = lp.weight;
               if (childWeight > 0) {
                   final int share = (int) (childWeight * remainingExcess / remainingWeightSum);
                   remainingExcess -= share;
                   remainingWeightSum -= childWeight;

                   final int childHeight;
                   if (mUseLargestChild && heightMode != MeasureSpec.EXACTLY) {
                       childHeight = largestChildHeight;
                   } else if (lp.height == 0 && (!mAllowInconsistentMeasurement
                           || heightMode == MeasureSpec.EXACTLY)) {
                       childHeight = share;
                   } else {
                       //定义权重子控件重新测量，这时候childWidth是子控件本身的高度加上通过权重计算的额外高度
                   	childHeight = child.getMeasuredHeight() + share;
                   }

                   final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                           Math.max(0, childHeight), MeasureSpec.EXACTLY);
                   final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                           mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin,
                           lp.width);
                   child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
                   childState = combineMeasuredStates(childState, child.getMeasuredState()
                           & (MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
               }

               final int margin =  lp.leftMargin + lp.rightMargin;
               final int measuredWidth = child.getMeasuredWidth() + margin;
               maxWidth = Math.max(maxWidth, measuredWidth);

               boolean matchWidthLocally = widthMode != MeasureSpec.EXACTLY &&
                       lp.width == LayoutParams.MATCH_PARENT;

               alternativeMaxWidth = Math.max(alternativeMaxWidth,
                       matchWidthLocally ? margin : measuredWidth);

               allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;

               final int totalLength = mTotalLength;
               mTotalLength = Math.max(totalLength, totalLength + child.getMeasuredHeight() +
                       lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
           }
           // Add in our padding
           mTotalLength += mPaddingTop + mPaddingBottom;
           // TODO: Should we recompute the heightSpec based on the new total length?
       } else {
           alternativeMaxWidth = Math.max(alternativeMaxWidth,
                                          weightedMaxWidth);
          //当设置了权重最小尺寸
           if (useLargestChild && heightMode != MeasureSpec.EXACTLY) {
               for (int i = 0; i < count; i++) {
                   final View child = getVirtualChildAt(i);
                   if (child == null || child.getVisibility() == View.GONE) {
                       continue;
                   }

                   final LinearLayout.LayoutParams lp =
                           (LinearLayout.LayoutParams) child.getLayoutParams();

                   float childExtra = lp.weight;
                   if (childExtra > 0) {
                       child.measure(
                               MeasureSpec.makeMeasureSpec(child.getMeasuredWidth(),
                                       MeasureSpec.EXACTLY),
                               MeasureSpec.makeMeasureSpec(largestChildHeight,
                                       MeasureSpec.EXACTLY));
                   }
               }
           }
       }

       if (!allFillParent && widthMode != MeasureSpec.EXACTLY) {
           maxWidth = alternativeMaxWidth;
       }

       maxWidth += mPaddingLeft + mPaddingRight;
       // Check against our minimum width
       maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

       setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
               heightSizeAndState);

       if (matchWidth) {
           forceUniformWidth(count, heightMeasureSpec);
       }
}
```

整个垂直方向的Measure流程大致就这样了，水平方向是类似的

## onLayout

layout过程其实逻辑基本和Measure是类似的

```java
protected void onLayout(boolean changed, int l, int t, int r, int b) {
       if (mOrientation == VERTICAL) {
           layoutVertical(l, t, r, b);
       } else {
           layoutHorizontal(l, t, r, b);
       }
   }
```

同样，不同的方向进行不同的layout，这里还是以Vertical方向为例

```java
void layoutVertical(int left, int top, int right, int bottom) {
       final int paddingLeft = mPaddingLeft;

       int childTop;
       int childLeft;

       // 子View默认右侧位置
       final int width = right - left;
       int childRight = width - mPaddingRight;

       // 子View可用空间大小
       int childSpace = width - paddingLeft - mPaddingRight;

       final int count = getVirtualChildCount();

       final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
       final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;
	//根据LinearLayout设置的对其方式 设置第一个子View的Top值
       switch (majorGravity) {
          case Gravity.BOTTOM:
              // mTotalLength contains the padding already
              childTop = mPaddingTop + bottom - top - mTotalLength;
              break;

              // mTotalLength contains the padding already
          case Gravity.CENTER_VERTICAL:
              childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
              break;

          case Gravity.TOP:
          default:
              childTop = mPaddingTop;
              break;
       }
	//遍历子View
       for (int i = 0; i < count; i++) {
           final View child = getVirtualChildAt(i);
           if (child == null) {
               childTop += measureNullChild(i);
           } else if (child.getVisibility() != GONE) {
               //LinearLayout中子View的宽和高有measure过程决定
               final int childWidth = child.getMeasuredWidth();
               final int childHeight = child.getMeasuredHeight();

               final LinearLayout.LayoutParams lp =
                       (LinearLayout.LayoutParams) child.getLayoutParams();

               int gravity = lp.gravity;
               if (gravity < 0) {
                   gravity = minorGravity;
               }
               final int layoutDirection = getLayoutDirection();
               final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
               switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                   case Gravity.CENTER_HORIZONTAL:
                       childLeft = paddingLeft + ((childSpace - childWidth) / 2)
                               + lp.leftMargin - lp.rightMargin;
                       break;

                   case Gravity.RIGHT:
                       childLeft = childRight - childWidth - lp.rightMargin;
                       break;

                   case Gravity.LEFT:
                   default:
                       childLeft = paddingLeft + lp.leftMargin;
                       break;
               }

               if (hasDividerBeforeChildAt(i)) {
                   childTop += mDividerHeight;
               }

               childTop += lp.topMargin;
               //用setChildFrame()方法设置子控件控件的在父控件上的坐标轴
               setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                       childWidth, childHeight);
               childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

               i += getChildrenSkipCount(child, i);
           }
       }
   }
```

layout也十分简单，就这么多东西

## onDraw

draw也是一样的逻辑

```java
protected void onDraw(Canvas canvas) {
       if (mDivider == null) {
           return;
       }

       if (mOrientation == VERTICAL) {
           drawDividersVertical(canvas);
       } else {
           drawDividersHorizontal(canvas);
       }
   }
```

直接看drawDividersVertical吧

```java
void drawDividersVertical(Canvas canvas) {
       final int count = getVirtualChildCount();
       for (int i = 0; i < count; i++) {
           final View child = getVirtualChildAt(i);
           if (child != null && child.getVisibility() != GONE) {
               if (hasDividerBeforeChildAt(i)) {
                   final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                   final int top = child.getTop() - lp.topMargin - mDividerHeight;
                   drawHorizontalDivider(canvas, top);
               }
           }
       }

       if (hasDividerBeforeChildAt(count)) {
           final View child = getLastNonGoneChild();
           int bottom = 0;
           if (child == null) {
               bottom = getHeight() - getPaddingBottom() - mDividerHeight;
           } else {
               final LayoutParams lp = (LayoutParams) child.getLayoutParams();
               bottom = child.getBottom() + lp.bottomMargin;
           }
           drawHorizontalDivider(canvas, bottom);
       }
   }
```

绘制里面就主要是分割线的绘制，明显易懂，就不再多说了

## FrameLayout、LinearLayout和RelativeLayout的性能对比

当RelativeLayout和LinearLayout作为ViewGroup表达相同的布局的时候，谁的绘制更快一些，性能相对更好一些？

通过网上的很多实验结果我们得之，两者绘制同样的界面时layout和draw的过程时间消耗相差无几，关键在于measure过程RelativeLayout比LinearLayout慢了一些。我们知道ViewGroup是没有onMeasure方法的，这个方法是交给子类自己实现的。因为不同的ViewGroup子类布局都不一样，那么onMeasure索性就全部交给他们自己实现好了

LinearLayout：在没有权重的情况下，就只会单纯的遍历一个方向，遍历一次所有的View；如果View设置了权重 ，那么在第一次遍历的时候这个View是不会进行测量的，在第二次测量（专门用于测量权weight重的）；所以无权重一次遍历，有权重两次遍历

RelativeLayout：因为依赖关系，所以在进行排序后，分别会对水平、垂直方向进行遍历，所以两次遍历

FrameLayout：某种情况上来说，FrameLayout也可能导致二次测量，不过FrameLayout的二次测量就只针对View为match_parent的了

## 特别鸣谢

[Android View 相关源码分析之四 LinearLayout源码分析](https://www.jianshu.com/p/3b52e6856b72)

[你对LinearLayout到底有多少了解？（二）-源码篇](https://www.jianshu.com/p/43d49b7d2ad2)