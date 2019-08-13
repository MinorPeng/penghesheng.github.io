---
title: Android View系列(四)——FrameLayout源码分析
tag: Android
date: 2019-05-04

---

<meta name="referrer" content="no-referrer" />



[TOC]

# FrameLayout源码分析

## 简介

FrameLayout是层级布局，即所有的子View都是从底层向上层开始，默认不指定margin或者layout_gravity的话，每个子view的坐标其实坐标都是Framelayout的其实坐标

```java
public class FrameLayout extends ViewGroup {
	...
}
```

FrameLayout就是直接继承自ViewGroup的

接着看看measure过程吧

## onMeasure

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
       int count = getChildCount();
	//判断FrameLayout 宽 高 是否同时设置了Match_parent或者 设置了指定大小的宽高
   	// measureMatchParentChildren 为 true，则表示没有设置了Match_parent或者 设置了指定大小的宽高
       final boolean measureMatchParentChildren =
               MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
               MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
       mMatchParentChildren.clear();

       int maxHeight = 0;
       int maxWidth = 0;
       int childState = 0;
	//遍历所有子View计算出所有的子View中的最大宽度 maxWidth 和最大高度 maxHeigh
       for (int i = 0; i < count; i++) {
           final View child = getChildAt(i);
           if (mMeasureAllChildren || child.getVisibility() != GONE) {
               //计算子View的宽高，包含了子View的左右上下Margin，直接调用ViewGroup中的方法
               measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
               final LayoutParams lp = (LayoutParams) child.getLayoutParams();
               maxWidth = Math.max(maxWidth,
                       child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
               maxHeight = Math.max(maxHeight,
                       child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
               childState = combineMeasuredStates(childState, child.getMeasuredState());
               // FrameLayout没有设置固定大小的宽高或Match_parent高宽，则收集所有设置了Match_parent的子View 
               if (measureMatchParentChildren) {
                   if (lp.width == LayoutParams.MATCH_PARENT ||
                           lp.height == LayoutParams.MATCH_PARENT) {
                       mMatchParentChildren.add(child);
                   }
               }
           }
       }
       // 加上padding
       maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
       maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

       maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
       maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

       final Drawable drawable = getForeground();
       if (drawable != null) {
           maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
           maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
       }
	//设置自身的宽高
       setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
               resolveSizeAndState(maxHeight, heightMeasureSpec,
                       childState << MEASURED_HEIGHT_STATE_SHIFT));
	...
   }
```

先看前部分，根据FrameLayout 宽 高 是否同时设置了Match_parent或者 设置了指定大小来设置一个Boolean值，然后遍历所有的子View，计算出最大宽度和最大高度，同时调用ViewGroup中的方法进行子View的测量；如果FrameLayout没有设置为match_parent或者固定的值，则会存储是match_parent的所有子View

接着就是加上padding值，然后设置自己的宽高

接着看后面部分

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
       ...
       // 如果FrameLayout没有同时设置固定大小或者Match_parent宽高，且有子View设置了Match_parent宽高，则重新测量 设置了Match_Parent宽高的子View
       count = mMatchParentChildren.size();
       if (count > 1) {
           for (int i = 0; i < count; i++) {
               final View child = mMatchParentChildren.get(i);
               final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

               final int childWidthMeasureSpec;
               if (lp.width == LayoutParams.MATCH_PARENT) {
                   final int width = Math.max(0, getMeasuredWidth()
                           - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                           - lp.leftMargin - lp.rightMargin);
                   childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                           width, MeasureSpec.EXACTLY);
               } else {
                   childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                           getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                           lp.leftMargin + lp.rightMargin,
                           lp.width);
               }

               final int childHeightMeasureSpec;
               if (lp.height == LayoutParams.MATCH_PARENT) {
                   final int height = Math.max(0, getMeasuredHeight()
                           - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                           - lp.topMargin - lp.bottomMargin);
                   childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                           height, MeasureSpec.EXACTLY);
               } else {
                   childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                           getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                           lp.topMargin + lp.bottomMargin,
                           lp.height);
               }
			
               child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
           }
       }
}
```

后面部分就是判断是否需要重新测量，如果FrameLayout没有同时设置固定大小或者Match_parent宽高，且有子View设置了Match_parent宽高，则重新测量 设置了Match_Parent宽高的子View

onMeasure过程就这么多，毕竟FrameLayout不算难

## onLayout

```java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
       layoutChildren(left, top, right, bottom, false /* no force left gravity */);
   }

   void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
       final int count = getChildCount();

       final int parentLeft = getPaddingLeftWithForeground();
       final int parentRight = right - left - getPaddingRightWithForeground();

       final int parentTop = getPaddingTopWithForeground();
       final int parentBottom = bottom - top - getPaddingBottomWithForeground();
	//遍历子View
       for (int i = 0; i < count; i++) {
           final View child = getChildAt(i);
           //同样，View的可见性不为GONE才会计算在内
           if (child.getVisibility() != GONE) {
               final LayoutParams lp = (LayoutParams) child.getLayoutParams();
               final int width = child.getMeasuredWidth();
               final int height = child.getMeasuredHeight();
               int childLeft;
               int childTop;
               int gravity = lp.gravity;
               if (gravity == -1) {
                   gravity = DEFAULT_CHILD_GRAVITY;
               }

               final int layoutDirection = getLayoutDirection();
               final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
               final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;
			// 根据水平方向Gravity，计算View的左右，坐标
               switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                   case Gravity.CENTER_HORIZONTAL:
                       childLeft = parentLeft + (parentRight - parentLeft - width) / 2 + lp.leftMargin - lp.rightMargin;
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
			// 根据垂直方向Gravity，计算View的上下，坐标
               switch (verticalGravity) {
                   case Gravity.TOP:
                       childTop = parentTop + lp.topMargin;
                       break;
                   case Gravity.CENTER_VERTICAL:
                       childTop = parentTop + (parentBottom - parentTop - height) / 2 + lp.topMargin - lp.bottomMargin;
                       break;
                   case Gravity.BOTTOM:
                       childTop = parentBottom - height - lp.bottomMargin;
                       break;
                   default:
                       childTop = parentTop + lp.topMargin;
               }

               child.layout(childLeft, childTop, childLeft + width, childTop + height);
           }
       }
   }
```

onLayout方法主要就是对子View进行layout

在对子View进行layout的过程中，主要就是通过遍历所有的子View，然后根据水平和垂直两个方向计算view具体的坐标，最后通过子View本身的layout方法进行放置。因为每个view在不同的层级，所以只需要计算每个View的Layout_gravity，默认的`Layout_gravity 为Gravity_left | GravityTop`, 在根据Gravity计算时，分别通过水平方向的Gravity计算出 左右 坐标，然后根据垂直方向的Gravity计算出上下坐标。

## onDraw

FrameLayout没有自己实现onDraw方法

## FrameLayout、LinearLayout和RelativeLayout的性能对比

当RelativeLayout和LinearLayout作为ViewGroup表达相同的布局的时候，谁的绘制更快一些，性能相对更好一些？

通过网上的很多实验结果我们得之，两者绘制同样的界面时layout和draw的过程时间消耗相差无几，关键在于measure过程RelativeLayout比LinearLayout慢了一些。我们知道ViewGroup是没有onMeasure方法的，这个方法是交给子类自己实现的。因为不同的ViewGroup子类布局都不一样，那么onMeasure索性就全部交给他们自己实现好了

LinearLayout：在没有权重的情况下，就只会单纯的遍历一个方向，遍历一次所有的View；如果View设置了权重 ，那么在第一次遍历的时候这个View是不会进行测量的，在第二次测量（专门用于测量权weight重的）；所以无权重一次遍历，有权重两次遍历

RelativeLayout：因为依赖关系，所以在进行排序后，分别会对水平、垂直方向进行遍历，所以两次遍历

FrameLayout：某种情况上来说，FrameLayout也可能导致二次测量，不过FrameLayout的二次测量就只针对View为match_parent的了

## 特别鸣谢

[FrameLayout 源码分析](https://blog.csdn.net/m0_37778101/article/details/85061171)