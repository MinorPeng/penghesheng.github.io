---
title: Android View系列(二)——RelativeLayout源码分析
tag: Android
date: 2019-05-01
---

[TOC]

# RelativeLayout源码

## 简介

RelativeLayout是我们日常布局中除了LinearLayout外，基本使用的就是它了；RelativeLayout就是相对布局，可以根据某一个view的位置，将新的View放置到某View的周围

RelativeLayout和LinearLayout一样，都是直接继承自ViewGroup，下面依旧从measure、layout、draw进行一个源码的查看

## onMeasure

RelativeLayout的onMeasure方法代码也是很多，我们一点一点看

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
       if (mDirtyHierarchy) {
           mDirtyHierarchy = false;
           sortChildren();
       }
       ...
}
```

首先有一个脏布局的判断，如果是脏布局的话，需要先对子View进行一个排序

```java
public void requestLayout() {
    super.requestLayout();
    mDirtyHierarchy = true;
}
```

根据mDirtyHierarchy的赋值，可以看到，只有在requestLayout的时候，才会被判定为脏布局，进行重新测量重新布局

### 排序

那先看看排序吧

```java
private void sortChildren() {
       final int count = getChildCount();
       //初始化
       if (mSortedVerticalChildren == null || mSortedVerticalChildren.length != count) {
           mSortedVerticalChildren = new View[count];
       }

       if (mSortedHorizontalChildren == null || mSortedHorizontalChildren.length != count) {
           mSortedHorizontalChildren = new View[count];
       }
       final DependencyGraph graph = mGraph;
       //清空
       graph.clear();
	//添加到graph中
       for (int i = 0; i < count; i++) {
           graph.add(getChildAt(i));
       }
	//根据水平和垂直两个方向进行排序，然后放入前面初始化的两个容器
       graph.getSortedViews(mSortedVerticalChildren, RULES_VERTICAL);
       graph.getSortedViews(mSortedHorizontalChildren, RULES_HORIZONTAL);
   }
```

这个方法中获取了子View的个数，然后初始化了两个容器用来存放水平和垂直方向排好序的子View，将子View的关系依赖graph进行初始化、清空，然后将所有的子View添加到关系依赖中

先看看这个DependencyGraph吧，后面的排序也是通过它来进行的

```java
...
private static class DependencyGraph {
       //存放所有的一级子View
       private ArrayList<Node> mNodes = new ArrayList<Node>();
	//存放有id的view
       private SparseArray<Node> mKeyNodes = new SparseArray<Node>();
	//临时数据结构，用于有序的放置目标的View数组
       private ArrayDeque<Node> mRoots = new ArrayDeque<Node>();
	...
   }
```

在DependencyGraph中有三个容器，mNodes用来存放所有的一级子View；mKeyNodes用来存放所有有id的子View，并且在后面的`addView()`方法中可以知道，viewId是作为key，view作为value；mRoots算是一个临时的容器，用来存放排好序的view

接着先看看这个Node吧

```java
static class Node {
          View view;
          final ArrayMap<Node, DependencyGraph> dependents =
                  new ArrayMap<Node, DependencyGraph>();

          final SparseArray<Node> dependencies = new SparseArray<Node>();
          
          private static final int POOL_LIMIT = 100;
          private static final SynchronizedPool<Node> sPool =
                  new SynchronizedPool<Node>(POOL_LIMIT);

          static Node acquire(View view) {
              Node node = sPool.acquire();
              if (node == null) {
                  node = new Node();
              }
              node.view = view;
              return node;
          }

          void release() {
              view = null;
              dependents.clear();
              dependencies.clear();

              sPool.release(this);
          }
      }
```

Node是用来方便我们处理RelativeLayout中的依赖关系的结点，一个Node就是一个View；在Node中会记录该View对应的依赖关系，以及它的依赖从属关系；如果一个Node没有依赖关系，那么它就是一个root，根结点

view就不用说了，就是我们的子View；dependents就是记录该View的从属关系的；dependencies记录依赖关系；sPool是SynchronizedPool（继承自SimplePool，在Pools中），就是用来存放Node来实现共享的，同时这个pool也是线程安全的

Node中通过acquire来初始化一个Node，通过release来释放，这些都很明显了

接着看看DependencyGraph的几个方法，先看一下`clear()`和`add()`方法吧

```java
private static class DependencyGraph {
	...
	//清空操作，清空容器，释放对象
       void clear() {
           final ArrayList<Node> nodes = mNodes;
           final int count = nodes.size();
           for (int i = 0; i < count; i++) {
               nodes.get(i).release();
           }
           nodes.clear();
           mKeyNodes.clear();
           mRoots.clear();
       }

       void add(View view) {
           final int id = view.getId();
           //将view包装成Node
           final Node node = Node.acquire(view);
		//mKeyNodes只存放有id的View
           if (id != View.NO_ID) {
               mKeyNodes.put(id, node);
           }
           mNodes.add(node);
       }
       ...
   }
```

`cleae()`方法很简单，就是释放Node对象，清空之前说的几个容器

`add()`方法也不难，获取到子View的id，把View生成Node，然后放入对应的容器

接着看看最重要的方法，排序把`getSortedViews()`

```java
private static class DependencyGraph {
       ...
       void getSortedViews(View[] sorted, int... rules) {
           final ArrayDeque<Node> roots = findRoots(rules);
           int index = 0;
           Node node;
           while ((node = roots.pollLast()) != null) {
               final View view = node.view;
               final int key = view.getId();
               //将符合规则的View加到 sorted中
               sorted[index++] = view;
               final ArrayMap<Node, DependencyGraph> dependents = node.dependents;
               final int count = dependents.size();
               //遍历所有依赖自己的node
               for (int i = 0; i < count; i++) {
                   final Node dependent = dependents.keyAt(i);
                   final SparseArray<Node> dependencies = dependent.dependencies;

                   dependencies.remove(key);
                   //如果解除依赖后没有其它依赖 则将该node也视为rootNode
                   if (dependencies.size() == 0) {
                       roots.add(dependent);
                   }
               }
           }

           if (index < sorted.length) {
               throw new IllegalStateException("Circular dependencies cannot exist"
                       + " in RelativeLayout");
           }
       }
	//找到root，就是不依赖于任何一个View和是任何一个View是从属关系的
       private ArrayDeque<Node> findRoots(int[] rulesFilter) {
           final SparseArray<Node> keyNodes = mKeyNodes;
           final ArrayList<Node> nodes = mNodes;
           final int count = nodes.size();
		//清除所有的关系
           for (int i = 0; i < count; i++) {
               final Node node = nodes.get(i);
               node.dependents.clear();
               node.dependencies.clear();
           }
           // 建立关系 遍历所有node  存入当前view和他所依赖的关系
           for (int i = 0; i < count; i++) {
               final Node node = nodes.get(i);
               final LayoutParams layoutParams = (LayoutParams) node.view.getLayoutParams();
               //根据LayoutParams获取其关系
               final int[] rules = layoutParams.mRules;
               final int rulesCount = rulesFilter.length;
               //遍历所有的关系
               for (int j = 0; j < rulesCount; j++) {
                   //根据依赖关系找到对应的ViewId
                   final int rule = rules[rulesFilter[j]];
                   if (rule > 0) {
                       // 找到有依赖关系的view
                       final Node dependency = keyNodes.get(rule);
                       // 跳过自己和空View
                       if (dependency == null || dependency == node) {
                           continue;
                       }
                       // 添加依赖
                       dependency.dependents.put(node, this);
                       // 设置依赖关系（从属关系）
                       node.dependencies.put(rule, dependency);
                   }
               }
           }
		//从这里就可以知道mRoots就是一个临时的存储
           final ArrayDeque<Node> roots = mRoots;
           roots.clear();

           // 将没有依赖关系的node添加到roots中
           for (int i = 0; i < count; i++) {
               final Node node = nodes.get(i);
               if (node.dependencies.size() == 0) roots.addLast(node);
           }
           return roots;
       }
       ...
   }
```

通过`getSortedViews()`的注释，大概了解到排序的方式，根据依赖关系进行一个排序；例如View C依赖于View A，View A又依赖于View B，那么最后的排序结果就是B->A->C（我们这里的依赖就是RelativeLayout中的依赖）

根据rules（从传入的参数知道就是方向——水平、垂直）找到roots（没有任何依赖关系和从属关系的Node）

首先通过`findRoots()`获取所有得roots，在`findRoots()`中，通过LayoutParams来获取view对应的依赖关系，然后给对应Node设置对应的依赖关系，而找到对应的有依赖关系的View则是通过ViewId来的；举个例子，View A（id:viewA）依赖于View B(id:viewB），在布局文件xml中我们假设是这样写的：

```xml
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <Button
        android:id="@+id/viewA"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="A"/>
    
    <Button
        android:id="@+id/viewB"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" 
        android:text="B"
        android:layout_toEndOf="@+id/viewA"
        android:layout_toRightOf="@+id/viewA" />
</RelativeLayout>
```

我们写了两个Button来解释，这个布局的结果就是viewB依赖于viewA，在viewA的右边；那么在findRoots的过程中，View A就是一个root，那么View B就不是，在遍历到View B时，就会根据依赖关系`layout_toEndOf`、`layout_toRightOf`来获取到View A的id`viewA`，这样就可以在mKeyNodes中获取到View A，然后将这个从属关系添加到View A的Node中（B是依赖于A的），而在View B这个Node中，它的dependency是依赖于View A的；最后遍历完所有的结点，所有root都会添加到mRoots中并返回，所有非root的Node都会与其他Node建立相应的关系

通过`findRoots()`不仅找到了root，同时将所有的node都建立了对应得关系

在`getSortedViews()`后面的遍历过程中，就主要是个排序的过程了，就是将View放置到sorted这个`View[]`中，这是前面传进来用于存放排好序的View的

### 排序后初始化

接着回到我们的onMeasure方法，先看看这个方法里的一些变量吧

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	...
	int myWidth = -1;
       int myHeight = -1;
       int width = 0;
       int height = 0;
	//获取宽度、高度的测量模式以及size
       final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
       final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
       final int widthSize = MeasureSpec.getSize(widthMeasureSpec);
       final int heightSize = MeasureSpec.getSize(heightMeasureSpec);
       //如果不是UNSPECIFIED模式，那就把widthSize的值赋给myWidth 
       if (widthMode != MeasureSpec.UNSPECIFIED) {
           myWidth = widthSize;
       }
	//如果不是UNSPECIFIED模式 则将heightSize赋值于myHeight
       if (heightMode != MeasureSpec.UNSPECIFIED) {
           myHeight = heightSize;
       }
	//如果是EXACTLY模式 则将myWidth和myHeight记录
       if (widthMode == MeasureSpec.EXACTLY) {
           width = myWidth;
       }
	//与宽度一样
       if (heightMode == MeasureSpec.EXACTLY) {
           height = myHeight;
       }

       View ignore = null;
       //判断是否为Start 和  top 确定左上角坐标
       int gravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;
       final boolean horizontalGravity = gravity != Gravity.START && gravity != 0;
       gravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
       final boolean verticalGravity = gravity != Gravity.TOP && gravity != 0;

       int left = Integer.MAX_VALUE;
       int top = Integer.MAX_VALUE;
       int right = Integer.MIN_VALUE;
       int bottom = Integer.MIN_VALUE;
	
       boolean offsetHorizontalAxis = false;
       boolean offsetVerticalAxis = false;
	//记录ignore的view
       if ((horizontalGravity || verticalGravity) && mIgnoreGravity != View.NO_ID) {
           ignore = findViewById(mIgnoreGravity);
       }
	//宽度和高度是否是wrap模式
       final boolean isWrapContentWidth = widthMode != MeasureSpec.EXACTLY;
       final boolean isWrapContentHeight = heightMode != MeasureSpec.EXACTLY;
       //在计算和分配的子View的坐标的时候 需要用到父VIew的尺寸 但是暂时无法拿到准确值(待完成下面操作)，先使用默认值代替 在计算后 用偏移量更新真是坐标
       final int layoutDirection = getLayoutDirection();
       if (isLayoutRtl() && myWidth == -1) {
           myWidth = DEFAULT_WIDTH;
       }
       ...
}
```

就是一些记录变量的初始化、获取测量模式和size以及一些相应的调整（如果不是UNSPECIFIED模式，那就把widthSize的值赋给myWidth等）

接着看后面的吧

### 水平方向

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	...
	View[] views = mSortedHorizontalChildren;
       int count = views.length;

       for (int i = 0; i < count; i++) {
           View child = views[i];
           if (child.getVisibility() != GONE) {
               LayoutParams params = (LayoutParams) child.getLayoutParams();
               //根据方向获得子view中设置的规则
               int[] rules = params.getRules(layoutDirection);
			//将左右方向规则转换为左右的坐标
               applyHorizontalSizeRules(params, myWidth, rules);
               //测算水平方向的子View的尺寸
               measureChildHorizontal(child, params, myWidth, myHeight);
			//确定水平方向子View的位置
               if (positionChildHorizontal(child, params, myWidth, isWrapContentWidth)) {
                   offsetHorizontalAxis = true;
               }
           }
       }
       ...
   }
```

这段代码处理的是水平方向上的关系，通过遍历水平方向关系的view，同样不处理是View.Gone的View

根据layoutDirection从LayoutParams中获取规则关系，通过`applyHorizontalSizeRules()`将规则转换为坐标

```java
private void applyHorizontalSizeRules(LayoutParams childParams, int myWidth, int[] rules) {
       RelativeLayout.LayoutParams anchorParams;
       childParams.mLeft = VALUE_NOT_SET;
       childParams.mRight = VALUE_NOT_SET;
       //得到当前子View的layout_toLeftOf属性对应的View
       anchorParams = getRelatedViewParams(rules, LEFT_OF);
       if (anchorParams != null) {
           //如果这个属性存在 则当前子View的右坐标是layout_toLeftOf对应的view的左坐标减去对应view的marginLeft的值和自身marginRight的值
           childParams.mRight = anchorParams.mLeft - (anchorParams.leftMargin +
                   childParams.rightMargin);
       } else if (childParams.alignWithParent && rules[LEFT_OF] != 0) {
           //如果alignWithParent为true alignWithParent取alignWithParentIfMissing
       	//如果layout_toLeftOf的view为空 或者gone 则将RelativeLayout当做被依赖的对象
           if (myWidth >= 0) {
               //如果父容器RelativeLayout的宽度大于0
           	//则子View的右坐标为 父RelativeLayout的宽度减去 mPaddingRight 和自身的marginRight
               childParams.mRight = myWidth - mPaddingRight - childParams.rightMargin;
           }
       }
	//同样的方式，得到当前子View的layout_toRightOf属性对应的View
       anchorParams = getRelatedViewParams(rules, RIGHT_OF);
       if (anchorParams != null) {
           childParams.mLeft = anchorParams.mRight + (anchorParams.rightMargin +
                   childParams.leftMargin);
       } else if (childParams.alignWithParent && rules[RIGHT_OF] != 0) {
           childParams.mLeft = mPaddingLeft + childParams.leftMargin;
       }
	//同样的方式，得到当前子View的layout_toAlignLeftOf属性对应的View
       anchorParams = getRelatedViewParams(rules, ALIGN_LEFT);
       if (anchorParams != null) {
           childParams.mLeft = anchorParams.mLeft + childParams.leftMargin;
       } else if (childParams.alignWithParent && rules[ALIGN_LEFT] != 0) {
           childParams.mLeft = mPaddingLeft + childParams.leftMargin;
       }
	//同样的方式，得到当前子View的layout_toAlignRightOf属性对应的View
       anchorParams = getRelatedViewParams(rules, ALIGN_RIGHT);
       if (anchorParams != null) {
           childParams.mRight = anchorParams.mRight - childParams.rightMargin;
       } else if (childParams.alignWithParent && rules[ALIGN_RIGHT] != 0) {
           if (myWidth >= 0) {
               childParams.mRight = myWidth - mPaddingRight - childParams.rightMargin;
           }
       }

       if (0 != rules[ALIGN_PARENT_LEFT]) {
           childParams.mLeft = mPaddingLeft + childParams.leftMargin;
       }

       if (0 != rules[ALIGN_PARENT_RIGHT]) {
           if (myWidth >= 0) {
               childParams.mRight = myWidth - mPaddingRight - childParams.rightMargin;
           }
       }
   }
```

这个方法就是将水平方向的关系（比如toLeftOf、toRightOf）转化为对应的坐标，就没啥看的了，很简单

回到之前的循环，`applyHorizontalSizeRules()`结束后，`measureChildHorizontal()`测量水平方向的View

```java
private void measureChildHorizontal(
           View child, LayoutParams params, int myWidth, int myHeight) {
       //获得child的宽度MeasureSpec
       final int childWidthMeasureSpec = getChildMeasureSpec(params.mLeft, params.mRight,
               params.width, params.leftMargin, params.rightMargin, mPaddingLeft, mPaddingRight,
               myWidth);

       final int childHeightMeasureSpec;
       //在低于4.2的时候 mAllowBrokenMeasureSpecs为true
   	//当myHeight < 0 时 则根据父RelativeLayout设置其MeasureSpec模式
       if (myHeight < 0 && !mAllowBrokenMeasureSpecs) {
           //如果父RelativeLayout的height大于0  则设置子view的MeasureSpec模式为EXACTLY
       	if (params.height >= 0) {
               childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                       params.height, MeasureSpec.EXACTLY);
           } else {
               //如果其小于0  则设置子View的MeasureSpec为UNSPECIFIED
               childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED);
           }
       } else {
           //当当前myHeight >= 0
     		//判断当前高度是否与父RelativeLayout高度相同 设置heightMode
     		//根据maxHeight 和heightMode设置子View的MeasureSpec模式
           final int maxHeight;
           if (mMeasureVerticalWithPaddingMargin) {
               maxHeight = Math.max(0, myHeight - mPaddingTop - mPaddingBottom
                       - params.topMargin - params.bottomMargin);
           } else {
               maxHeight = Math.max(0, myHeight);
           }

           final int heightMode;
           if (params.height == LayoutParams.MATCH_PARENT) {
               heightMode = MeasureSpec.EXACTLY;
           } else {
               heightMode = MeasureSpec.AT_MOST;
           }
           childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(maxHeight, heightMode);
       }
	//测量子View自身
       child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
   }
```

然后看看是如何获取子View的宽度测量模式的

```java
getChildMeasureSpec(int childStart, int childEnd,
           int childSize, int startMargin, int endMargin, int startPadding,
           int endPadding, int mySize) {
       int childSpecMode = 0;
       int childSpecSize = 0;

       final boolean isUnspecified = mySize < 0;
       if (isUnspecified && !mAllowBrokenMeasureSpecs) {
           if (childStart != VALUE_NOT_SET && childEnd != VALUE_NOT_SET) {
               //如果子View的左边距和右边距都不为VALUE_NOT_SET
           	//且右边距坐标大于左边距坐标 则将其差当做宽度赋予View 设置模式为EXACTLY
               childSpecSize = Math.max(0, childEnd - childStart);
               childSpecMode = MeasureSpec.EXACTLY;
           } else if (childSize >= 0) {
               //如果childSpecSize >= 0 则赋值于childSpecSize，同样设置模式为EXACTLY
               childSpecSize = childSize;
               childSpecMode = MeasureSpec.EXACTLY;
           } else {
               // 都不满足则设置模式为UNSPECIFIED
               childSpecSize = 0;
               childSpecMode = MeasureSpec.UNSPECIFIED;
           }
           return MeasureSpec.makeMeasureSpec(childSpecSize, childSpecMode);
       }
       
       int tempStart = childStart;
       int tempEnd = childEnd;
       //如果没有指定start值 则默认赋予 padding和merage的值
       if (tempStart == VALUE_NOT_SET) {
           tempStart = startPadding + startMargin;
       }
       if (tempEnd == VALUE_NOT_SET) {
           tempEnd = mySize - endPadding - endMargin;
       }
       //指定最大可提供的大小
       final int maxAvailable = tempEnd - tempStart;

       if (childStart != VALUE_NOT_SET && childEnd != VALUE_NOT_SET) {
           //如果Start和End都是有效值 根据isUnspecified设置specMode为UNSPECIFIED或EXACTLY，并将设置对应的size
           childSpecMode = isUnspecified ? MeasureSpec.UNSPECIFIED : MeasureSpec.EXACTLY;
           childSpecSize = Math.max(0, maxAvailable);
       } else {
           if (childSize >= 0) {
               //设置模式为EXACTLY，判断maxAvailable和childSize情况 取较大值设置为childSpecSize
               childSpecMode = MeasureSpec.EXACTLY;
               if (maxAvailable >= 0) {
                   // We have a maximum size in this dimension.
                   childSpecSize = Math.min(maxAvailable, childSize);
               } else {
                   // We can grow in this dimension.
                   childSpecSize = childSize;
               }
           } else if (childSize == LayoutParams.MATCH_PARENT) {
               //如果子View是match模式 参照isUnspecified设置相关
               childSpecMode = isUnspecified ? MeasureSpec.UNSPECIFIED : MeasureSpec.EXACTLY;
               childSpecSize = Math.max(0, maxAvailable);
           } else if (childSize == LayoutParams.WRAP_CONTENT) {
               // Child wants to wrap content. Use AT_MOST to communicate
               // available space if we know our max size.
               if (maxAvailable >= 0) {
                   // We have a maximum size in this dimension.
                   childSpecMode = MeasureSpec.AT_MOST;
                   childSpecSize = maxAvailable;
               } else {
                   // We can grow in this dimension. Child can be as big as it
                   // wants.
                   childSpecMode = MeasureSpec.UNSPECIFIED;
                   childSpecSize = 0;
               }
           }
       }

       return MeasureSpec.makeMeasureSpec(childSpecSize, childSpecMode);
   }
```

至此就水平方向的View就第一次测量完了，接着就是计算位置了

```java
private boolean positionChildHorizontal(View child, LayoutParams params, int myWidth,
           boolean wrapContent) {
	//获取RelativeLayout的布局方向
       final int layoutDirection = getLayoutDirection();
       int[] rules = params.getRules(layoutDirection);

       if (params.mLeft == VALUE_NOT_SET && params.mRight != VALUE_NOT_SET) {
           // 如果右边界有效 左边界无效 根据右边界计算出左边界
           params.mLeft = params.mRight - child.getMeasuredWidth();
       } else if (params.mLeft != VALUE_NOT_SET && params.mRight == VALUE_NOT_SET) {
           // 与上面相反
           params.mRight = params.mLeft + child.getMeasuredWidth();
       } else if (params.mLeft == VALUE_NOT_SET && params.mRight == VALUE_NOT_SET) {
          //都无效的时候
           if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_HORIZONTAL] != 0) {
               if (!wrapContent) {
                   //非wrap情况下，把子View水平中心固定在RelativeLayout的中心
                   centerHorizontal(child, params, myWidth);
               } else {
                   //wrap情况下，左边距为padding+margin，右边距为左边距加上测量宽度
                   positionAtEdge(child, params, myWidth);
               }
               return true;
           } else {
               // This is the default case. For RTL we start from the right and for LTR we start
               // from the left. This will give LEFT/TOP for LTR and RIGHT/TOP for RTL.
               positionAtEdge(child, params, myWidth);
           }
       }
       return rules[ALIGN_PARENT_END] != 0;
   }
```

到此，一个子View的位置就算确定了

### 垂直方向

接着看垂直方向

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	...	
	views = mSortedVerticalChildren;
       count = views.length;
       final int targetSdkVersion = getContext().getApplicationInfo().targetSdkVersion;

       for (int i = 0; i < count; i++) {
           final View child = views[i];
           if (child.getVisibility() != GONE) {
               final LayoutParams params = (LayoutParams) child.getLayoutParams();
			//和水平方向是同样的方法
               applyVerticalSizeRules(params, myHeight, child.getBaseline());
               measureChild(child, params, myWidth, myHeight);
               if (positionChildVertical(child, params, myHeight, isWrapContentHeight)) {
                   offsetVerticalAxis = true;
               }

               if (isWrapContentWidth) {
                   if (isLayoutRtl()) {
                       if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
                           width = Math.max(width, myWidth - params.mLeft);
                       } else {
                           width = Math.max(width, myWidth - params.mLeft + params.leftMargin);
                       }
                   } else {
                       if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
                           width = Math.max(width, params.mRight);
                       } else {
                           width = Math.max(width, params.mRight + params.rightMargin);
                       }
                   }
               }

               if (isWrapContentHeight) {
                   if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
                       height = Math.max(height, params.mBottom);
                   } else {
                       height = Math.max(height, params.mBottom + params.bottomMargin);
                   }
               }

               if (child != ignore || verticalGravity) {
                   left = Math.min(left, params.mLeft - params.leftMargin);
                   top = Math.min(top, params.mTop - params.topMargin);
               }

               if (child != ignore || horizontalGravity) {
                   right = Math.max(right, params.mRight + params.rightMargin);
                   bottom = Math.max(bottom, params.mBottom + params.bottomMargin);
               }
           }
       }
       ...
   }
```

整体的逻辑基本和水平方向的一致，所以就不细细再分析了

### Baseline的计算

接着看后面的

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	...
	View baselineView = null;
       LayoutParams baselineParams = null;
       for (int i = 0; i < count; i++) {
           final View child = views[i];
           if (child.getVisibility() != GONE) {
               final LayoutParams childParams = (LayoutParams) child.getLayoutParams();
               if (baselineView == null || baselineParams == null
                       || compareLayoutPosition(childParams, baselineParams) < 0) {
                   baselineView = child;
                   baselineParams = childParams;
               }
           }
       }
       mBaselineView = baselineView;
       ...
}
```

就是重新计算了Baseline

### 一些修正

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	...
       //宽度是wrap_content
	if (isWrapContentWidth) {
           width += mPaddingRight;
           if (mLayoutParams != null && mLayoutParams.width >= 0) {
               width = Math.max(width, mLayoutParams.width);
           }

           width = Math.max(width, getSuggestedMinimumWidth());
           width = resolveSize(width, widthMeasureSpec);

           if (offsetHorizontalAxis) {
               for (int i = 0; i < count; i++) {
                   final View child = views[i];
                   if (child.getVisibility() != GONE) {
                       final LayoutParams params = (LayoutParams) child.getLayoutParams();
                       final int[] rules = params.getRules(layoutDirection);
                       if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_HORIZONTAL] != 0) {
                           centerHorizontal(child, params, width);
                       } else if (rules[ALIGN_PARENT_RIGHT] != 0) {
                           final int childWidth = child.getMeasuredWidth();
                           params.mLeft = width - mPaddingRight - childWidth;
                           params.mRight = params.mLeft + childWidth;
                       }
                   }
               }
           }
       }
	//高度是wrap_content
       if (isWrapContentHeight) {
           height += mPaddingBottom;
           if (mLayoutParams != null && mLayoutParams.height >= 0) {
               height = Math.max(height, mLayoutParams.height);
           }

           height = Math.max(height, getSuggestedMinimumHeight());
           height = resolveSize(height, heightMeasureSpec);

           if (offsetVerticalAxis) {
               for (int i = 0; i < count; i++) {
                   final View child = views[i];
                   if (child.getVisibility() != GONE) {
                       final LayoutParams params = (LayoutParams) child.getLayoutParams();
                       final int[] rules = params.getRules(layoutDirection);
                       if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_VERTICAL] != 0) {
                           centerVertical(child, params, height);
                       } else if (rules[ALIGN_PARENT_BOTTOM] != 0) {
                           final int childHeight = child.getMeasuredHeight();
                           params.mTop = height - mPaddingBottom - childHeight;
                           params.mBottom = params.mTop + childHeight;
                       }
                   }
               }
           }
       }
       //根据gravity再次修正
       if (horizontalGravity || verticalGravity) {
           final Rect selfBounds = mSelfBounds;
           selfBounds.set(mPaddingLeft, mPaddingTop, width - mPaddingRight,
                   height - mPaddingBottom);

           final Rect contentBounds = mContentBounds;
           Gravity.apply(mGravity, right - left, bottom - top, selfBounds, contentBounds,
                   layoutDirection);

           final int horizontalOffset = contentBounds.left - left;
           final int verticalOffset = contentBounds.top - top;
           if (horizontalOffset != 0 || verticalOffset != 0) {
               for (int i = 0; i < count; i++) {
                   final View child = views[i];
                   if (child.getVisibility() != GONE && child != ignore) {
                       final LayoutParams params = (LayoutParams) child.getLayoutParams();
                       if (horizontalGravity) {
                           params.mLeft += horizontalOffset;
                           params.mRight += horizontalOffset;
                       }
                       if (verticalGravity) {
                           params.mTop += verticalOffset;
                           params.mBottom += verticalOffset;
                       }
                   }
               }
           }
       }
	 //如果是RTL(右到左显示)则再次修改
       if (isLayoutRtl()) {
           final int offsetWidth = myWidth - width;
           for (int i = 0; i < count; i++) {
               final View child = views[i];
               if (child.getVisibility() != GONE) {
                   final LayoutParams params = (LayoutParams) child.getLayoutParams();
                   params.mLeft -= offsetWidth;
                   params.mRight -= offsetWidth;
               }
           }
       }
	//设置测量后的Dimension
       setMeasuredDimension(width, height);
}
```

这些修正也十分简单，在满足条件下，进行params上的一些重新计算和修正

最后设置测量后的Dimension，至此，onMeasure结束

## onLayout

```java
protected void onLayout(boolean changed, int l, int t, int r, int b) {
       //  The layout has actually already been performed and the positions
       //  cached.  Apply the cached values to the children.
       final int count = getChildCount();

       for (int i = 0; i < count; i++) {
           View child = getChildAt(i);
           if (child.getVisibility() != GONE) {
               RelativeLayout.LayoutParams st =
                       (RelativeLayout.LayoutParams) child.getLayoutParams();
               child.layout(st.mLeft, st.mTop, st.mRight, st.mBottom);
           }
       }
   }
```

onLayout过程基本没啥逻辑了，就是根据left、top、right、bottom来放置子View

## onDraw

RelativeLayout没有重写onDraw方法，就是ViewGroup的默认实现

## 总结

主要就是onMeasure过程

1. 先把内部子view根据纵向关系和横向关系排序（排序的过程，会遍历所有的view，并记录他们的依赖关系）
2. 初始化一些变量值
3. 遍历水平关系的view
4. 遍历竖直关系的view
5. baseline计算
6. 宽度和高度修正

## FrameLayout、LinearLayout和RelativeLayout的性能对比

当RelativeLayout和LinearLayout作为ViewGroup表达相同的布局的时候，谁的绘制更快一些，性能相对更好一些？

通过网上的很多实验结果我们得之，两者绘制同样的界面时layout和draw的过程时间消耗相差无几，关键在于measure过程RelativeLayout比LinearLayout慢了一些。我们知道ViewGroup是没有onMeasure方法的，这个方法是交给子类自己实现的。因为不同的ViewGroup子类布局都不一样，那么onMeasure索性就全部交给他们自己实现好了

LinearLayout：在没有权重的情况下，就只会单纯的遍历一个方向，遍历一次所有的View；如果View设置了权重 ，那么在第一次遍历的时候这个View是不会进行测量的，在第二次测量（专门用于测量权weight重的）；所以无权重一次遍历，有权重两次遍历

RelativeLayout：因为依赖关系，所以在进行排序后，分别会对水平、垂直方向进行遍历，所以两次遍历

FrameLayout：某种情况上来说，FrameLayout也可能导致二次测量，不过FrameLayout的二次测量就只针对View为match_parent的了

## 特别鸣谢

[Android View 相关源码分析之五 RelativeLayout 源码分析](https://www.jianshu.com/p/540342ea3f75)

[RelativeLayout源码解析](https://blog.csdn.net/wz249863091/article/details/51757069)