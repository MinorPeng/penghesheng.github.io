---
title: Android实现滑动返回的几种方法总结
tag: Android
---

# 关于Android实现滑动返回的几种方法总结

标签（空格分隔）： 此博客若要转载，请注明出处

------

关于Android实现滑动返回的方法，网上有很多种，实现的方式也都各不一样。有用SwipeBackLayout开源库的，有用SlidingPaneLayout控件的，有通过使用GestureDetector手势识别的类的，也有写一个基类的，还有一些其他的实现方法。总之，实现滑动返回的方法各种各样，但同样也各有千秋。在这里，我主要对以下几种方法进行了学习，并一一实现。（注意：我这次Demo是在MaterialsDesign的基础上进行编写代码的，不过这并不影响这几种方法的实现，你可以到这个地址下载源代码进行查看学习：<https://github.com/PengHesheng/ToolBar-BackActivity>）

- SwipeBackLayout开源库
- GestureDetector
- SlidingPaneLayout

------

在这里先声明一下，由于有些方法需要重新设置style，对theme的要求也不尽相同，我在Demo中统一使用了下面一个Theme，所以讲解方法的开始，我先把要新建的style/theme的代码贴出来，还有滑动返回的其中一种动画的设置也贴出来，当然，这主题的设置和动画的设置是在网上找的，在博客的最后我会贴出相关的博客

style

```xml
<resources xmlns:tools="http://schemas.android.com/tools">

    <!-- Base application theme. -->
    <style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
         <!--Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
    </style>
    
    <!--新建一个主题，设置为透明样式，保证滑动的时候能看到下面的Activity-->
    <style name="JK.SwipeBack.Transparent.Theme" parent="AppTheme">
        <item name="android:windowBackground">@android:color/transparent</item>
        <item name="android:windowIsTranslucent">true</item>
        <item name="android:windowAnimationStyle">@style/JK.Animation.SlidingBack</item>
        <item name="android:actionBarStyle">@style/JKActionBar.Custom</item>
    </style>

    <style name="JKActionBar.Custom" parent="@style/Widget.AppCompat.Light.ActionBar.Solid.Inverse">
        <item name="displayOptions">showCustom</item>
        <item name="android:background">@android:color/transparent</item>
        <item name="background">@android:color/transparent</item>
        <item name="android:displayOptions" tools:ignore="NewApi">showCustom</item>
        <item name="android:height">?actionBarSize</item>
    </style>

    <style name="JK.Animation.SlidingBack" parent="@android:style/Animation.Activity">
        <item name="android:activityOpenEnterAnimation">@anim/in_from_right</item>
        <item name="android:activityOpenExitAnimation">@anim/out_to_right</item>
        <item name="android:activityCloseEnterAnimation">@anim/in_from_right</item>
        <item name="android:activityCloseExitAnimation">@anim/out_to_right</item>
        <item name="android:wallpaperOpenEnterAnimation">@anim/in_from_right</item>
        <item name="android:wallpaperOpenExitAnimation">@anim/out_to_right</item>
        <item name="android:wallpaperCloseEnterAnimation">@anim/in_from_right</item>
        <item name="android:wallpaperCloseExitAnimation">@anim/out_to_right</item>
        <item name="android:wallpaperIntraOpenEnterAnimation">@anim/in_from_right</item>
        <item name="android:wallpaperIntraOpenExitAnimation">@anim/out_to_right</item>
        <item name="android:wallpaperIntraCloseEnterAnimation">@anim/in_from_right</item>
        <item name="android:wallpaperIntraCloseExitAnimation">@anim/out_to_right</item>
    </style>
</resources>
```

滑动返回的动画设置

in_form_left：

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">

<translate
    android:duration="400"
    android:fromXDelta="100%p"
    android:toXDelta="0%p" />
</set>
```

in_from_right：

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:shareInterpolator="false"
     android:zAdjustment="top">

    <translate
        android:duration="200"
        android:fromXDelta="100.0%p"
        android:toXDelta="0.0" />
</set>
```

out_to_left：

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">

    <translate
        android:duration="400"
        android:fromXDelta="0%p"
        android:toXDelta="-100%p" />

</set>
```

out_to_right：

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
     android:shareInterpolator="false"
     android:zAdjustment="top">

    <translate
        android:duration="200"
        android:fromXDelta="0.0"
        android:toXDelta="100.0%p" />
</set>
```

------

## [SwipeBackLayout](https://github.com/liuguangqiang/SwipeBack)

效果图：（为设置为透明和没有设置为透明）

[![此处输入图片的描述](http://ojplrudb4.bkt.clouddn.com/SwipeBackLayout.gif)](http://ojplrudb4.bkt.clouddn.com/SwipeBackLayout.gif)此处输入图片的描述

这个开源库的讲解我就不必要过多的陈述，在这里我只提一部分实现时需要注意的事项以及我踩到的坑，想详细了解的，直接到[SwipeBackLayout](https://github.com/liuguangqiang/SwipeBack)开源库查看代码和详解。也可看一下这篇博客[SwipeBackLayout源代码分析](http://skykai521.github.io/2016/03/04/SwipeBackLayout源代码分析/)。

**原理：**这种实现的重点在于将Activity的属性设置为透明的，然后上方的Activity就可以在跟随手指移动时候放一个半透明的层表示那种渐变的阴影效果，全部滑动完成后再把上方Activity销毁掉。
向右滑动销毁（finish）Activity。

**注意：**由于设置为了全透明，所以当我使用4.0.3版本进行开发的时候，由于活动默认的是白色的，所以当我继承这个类时，运行后的结果效果不太好，在新建的活动中能看见上一个活动的布局，这个体验感非常差，在后面的调试中，发现只要把新建的活动设置一个其他颜色的背景就行了，比如我设置为了gray，这样就没有了那个BUG。我们现在主要讨论的是向右滑动返回，所以在继承该类的时候，一定要有setDragEdge(SwipeBackLayout.DragEdge.LEFT); 这一行代码，原因代码中也说了。

在这里，我们需要先导入[SwipeBackLayout](https://github.com/liuguangqiang/SwipeBack)开源库，最新版本在开源库上自行查看，我得Demo用的是`compile 'com.github.liuguangqiang.swipeback:library:1.0.2@aar'`，导入成功后直接在你需要用到滑动返回的Activity继承SwipeBackActivity就行了，在这里还需要注意的是Activity的属性设置为透明的，就跟前面的原理一样，这里是需要着重注意的，在后面几个方法中，这一点同样很重要，几乎所有的方法都需要设置一下Activity的属性。

实现时如代码

```java
package com.example.hp.toolbartest.Activity;

import android.content.Intent;
import android.os.Bundle;
import android.support.design.widget.CollapsingToolbarLayout;
import android.support.v7.app.ActionBar;
import android.support.v7.widget.Toolbar;
import android.view.MenuItem;
import android.widget.ImageView;
import android.widget.TextView;

import com.bumptech.glide.Glide;
import com.example.hp.toolbartest.R;
import com.liuguangqiang.swipeback.SwipeBackActivity;
import com.liuguangqiang.swipeback.SwipeBackLayout;

public class FruitActivity extends SwipeBackActivity {

    public static final String FRUIT_NAME = "fruit_name";
    public static final String FRUIT_IMAGE_ID = "fruit_image_id";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_fruit);

        //用于设置向右滑动为返回，该库默认是向上滑动为返回
        setDragEdge(SwipeBackLayout.DragEdge.LEFT);

        Intent intent = getIntent();
        String fruitName = intent.getStringExtra(FRUIT_NAME);
        int fruitImageId = intent.getIntExtra(FRUIT_IMAGE_ID, 0);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar1);
        CollapsingToolbarLayout collapsingToolbarLayout = (CollapsingToolbarLayout)
                findViewById(R.id.collapsing_toolbar);
        ImageView fruitImageView = (ImageView) findViewById(R.id.fruit_image_view);
        TextView fruitContentText = (TextView) findViewById(R.id.fruit_content_text);
        setSupportActionBar(toolbar);
        ActionBar actionBar = getSupportActionBar();
        if (actionBar != null) {
            actionBar.setDisplayHomeAsUpEnabled(true);
        }
        collapsingToolbarLayout.setTitle(fruitName);
        Glide.with(this).load(fruitImageId).into(fruitImageView);
        String fruitContent = generateFruitContent(fruitName);
        fruitContentText.setText(fruitContent);
    }
    //循环生成一个字符串作为内容详解
    private String generateFruitContent(String fruitName) {
        StringBuilder fruitContent = new StringBuilder();
        for (int i=0; i<500; i++) {
            fruitContent.append(fruitName);
        }
        return fruitContent.toString();
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()) {
            case android.R.id.home:
                finish();
                return true;
        }
        return super.onOptionsItemSelected(item);
    }
}
```

代码的关键就只有继承该类和设置一下滑动返回的方向，其他的是我ToolBar的代码的实现，与此次讨论无关。

------

## 利用[GestureDetector](https://developer.android.google.cn/reference/android/view/GestureDetector.html)类

效果图：

[![此处输入图片的描述](http://ojplrudb4.bkt.clouddn.com/GestureBackActivity.gif)](http://ojplrudb4.bkt.clouddn.com/GestureBackActivity.gif)此处输入图片的描述

对于GestureDetector这个类，涉及的内容相对比较复杂一点，详细的了解就看官方文档[GestureDetector](https://developer.android.google.cn/reference/android/view/GestureDetector.html)。对于我们要如何实现滑动返回，我们首先需要建一个Activity的管理类AppManager，这样方便我们对Activity的生命周期进行管理，并安全退出，所以我们需要在主活动里对每启动一个Activity，就往AppManager里添加。
为了方便，同样构建了一个GestureBackActivity的基类，继承AppCompatActivity，重写AppCompatActivity的一些基本方法，然后重写dispatchTouchEvent事件机制，随着又写了手势监听器，用来监听滑动的手势，并重写GestureDetector.OnGestureListener，在重写的方法里面加入相应的逻辑处理，详细的解释我会写在代码里。多余的话就不用说了，直接给出代码。

MainActivity类

```java
//启动活动时，添加到AppManager，并设置返回时的动画
public void startActivity(Class<?> cls) {
    Intent intent = new Intent();
    intent.setClass(this, cls);
    startActivity(intent);
    overridePendingTransition(R.anim.in_from_left, R.anim.out_to_left);
}
```

AppManager类

```java
/**
 * 一个Activity管理类，对所有的Activity的生命周期进行管理
 */

public class AppManager {

    private static Stack<Activity> activityStack;
    private static AppManager instance;

    private AppManager() {
    }
    /**
    * 单一实例
    * */
    public static AppManager getAppManager() {
        if (instance == null) {
            instance = new AppManager();
        }
        return instance;
    }
    /**
    * 添加Activity到堆栈中
    * */
    public void addActivity(Activity activity) {
        if (activityStack == null) {
            activityStack = new Stack<Activity>();
        }
        activityStack.add(activity);
    }
    /**
    * 获取当前Activity（堆栈中最后压入的）
    * */
    public Activity currentActivity() {
        Activity activity = activityStack.lastElement();
        return activity;
    }
    /**
     * 结束当前Activity（堆栈中最后压入的）
     */
    public void finishActivity() {
        Activity activity = activityStack.lastElement();
        finishActivity(activity);
    }
    /**
    * 结束指定的Activity
    * */
    public void finishActivity(Activity activity) {
        if (activity != null) {
            activityStack.remove(activity);
            activity.finish();
            activity.overridePendingTransition(R.anim.in_from_right,
                    R.anim.out_to_right);
            activity = null;
        }
    }
    /**
    * 结束指定类名的Activity
    * */
    public void finishActivity(Class<?> cls) {
        for (Activity activity : activityStack) {
            if (activity.getClass().equals(cls)) {
                finishActivity(activity);
            }
        }
    }
    /**
    * 结束所有Activity
    * */
    public void finishAllActivity() {
        for (int i=0, size = activityStack.size(); i<size; i++) {
            if (null != activityStack.get(i)) {
                activityStack.get(i).finish();
            }
        }
        activityStack.clear();
    }
    /**
    * 彻底退出应用程序，安全退出
    * */
    public void AppExit(Context context) {
        try {
            finishAllActivity();
            ActivityManager activityManager = (ActivityManager) context
                    .getSystemService(Context.ACTIVITY_SERVICE);
            activityManager.restartPackage(context.getPackageName());
            System.exit(0);
            android.os.Process.killProcess(android.os.Process.myPid());
        } catch (Exception e) {
        }
    }
}
```

GestureBackActivity基类

```java
/**
 * 构造一个手势滑动返回的基类，需要使用滑动返回的Activity只需继承该类就行
 */

public class GestureBackActivity extends AppCompatActivity {

    private GestureDetector myDectector;
    private static final String TAG = "GestureBackActivity";
    boolean flingFinishEnabled = true;

    @Override
    protected void onCreate(Bundle savedInstanceState) {

        super.onCreate(savedInstanceState);
        initGestureDetector();
        AppManager.getAppManager().addActivity(this);
    }

    private void initGestureDetector() {
        if (myDectector == null) {
            myDectector = new GestureDetector(this, (GestureDetector.OnGestureListener) new MyGestureListener());
        }

    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {

        if (flingFinishEnabled) {
            return myDectector.onTouchEvent(ev) || super.dispatchTouchEvent(ev);
        }
        return super.dispatchTouchEvent(ev);
    }

    /**
     * 手势监听器
     *
     */
    public class MyGestureListener implements GestureDetector.OnGestureListener {

        @Override
        public boolean onDown(MotionEvent e) {
            // Toast.makeText(getApplicationContext(),"down",Toast.LENGTH_SHORT).show();
            return true;
        }

        @Override
        public void onShowPress(MotionEvent e) {
            // TODO Auto-generated method stub

        }

        @Override
        public boolean onSingleTapUp(MotionEvent e) {
            // Toast.makeText(getApplicationContext(),"onSingleTapUp",Toast.LENGTH_SHORT).show();
            return true;
        }

        @Override
        public boolean onScroll(MotionEvent e1, MotionEvent e2,
                                float distanceX, float distanceY) {
            // TODO Auto-generated method stub
            return false;
        }

        @Override
        public void onLongPress(MotionEvent e) {
            // TODO Auto-generated method stub

        }

        @Override
        public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX,
                               float velocityY) {
            //对滑动距离的监听及判断
            if (e1.getX() - e2.getX() > 100 && Math.abs(velocityX) > 0) {
                Log.d(TAG, "向左滑动");
            } else if (e2.getX() - e1.getX() > 100 && Math.abs(velocityX) > 0) {
                Log.d(TAG, "向右滑动");
                AppManager.getAppManager().finishActivity();
            }
            return false;
        }

    }
}
```

这种方式相对于SwipeBack来说相对较好一点，不需要更改布局的背景，不过我在一开始出现了闪屏的问题，原因目前我还没找到，有懂得大佬欢迎指教。

------

## [SlidingPaneLayout](https://developer.android.google.cn/reference/android/support/v4/widget/SlidingPaneLayout.html)

效果图：

[![此处输入图片的描述](http://ojplrudb4.bkt.clouddn.com/SlidingPaneLayout.gif)](http://ojplrudb4.bkt.clouddn.com/SlidingPaneLayout.gif)此处输入图片的描述

相对于前面两种实现方式，我个人比较喜欢这一种，因为这一种方式并不是很复杂，而且效果相对是最好的一个，但是理解的难度相对要大一点，在我看来的话，不过也不是很难，都还是比较容易学的,先给出官方文档[SlidingPaneLayout](https://developer.android.google.cn/reference/android/support/v4/widget/SlidingPaneLayout.html)。
这个基类BaseActivity是根据SlidingPaneLayout来构造的，当然也需要继承AppCompatActivity，这是当我们直接继承该类时所必须的。在initSwipeBackFinish()初始化时，通过反射机制来改变通过反射改变mOverhangSize的值为0，这个mOverhangSize值为菜单到右边屏幕的最短距离，默认是32dp，现在给它改成0，然后通过slidingPaneLayout来设置视图，下面直接上代码。

BaseActivity类

```java
/**
 * Created by HP on 2017/3/21.
 * 利用slidingPaneLayout构造一个基类，来实现滑动返回，继承该类，便可实现滑动返回
 */

public abstract class BaseActivity extends AppCompatActivity implements SlidingPaneLayout.PanelSlideListener {

    public final static String TAG = BaseActivity.class.getCanonicalName();


    FrameLayout mContainerFl;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        initSwipeBackFinish();
        super.onCreate(savedInstanceState);

    }

    /**
     * 初始化滑动返回
     */
    private void initSwipeBackFinish() {
        if (isSupportSwipeBack()) {
            SlidingPaneLayout slidingPaneLayout = new SlidingPaneLayout(this);
            //通过反射改变mOverhangSize的值为0，这个mOverhangSize值为菜单到右边屏幕的最短距离，默认
            //是32dp，现在给它改成0
            try {
                //mOverhangSize属性，意思就是左菜单离右边屏幕边缘的距离
                Field f_overHang = SlidingPaneLayout.class.getDeclaredField("mOverhangSize");
                f_overHang.setAccessible(true);
                //设置左菜单离右边屏幕边缘的距离为0，设置全屏
                f_overHang.set(slidingPaneLayout, 0);
            } catch (Exception e) {
                e.printStackTrace();
            }
            slidingPaneLayout.setPanelSlideListener(this);
            slidingPaneLayout.setSliderFadeColor(getResources().getColor(android.R.color.transparent));
            // 左侧的透明视图
            View leftView = new View(this);
            leftView.setLayoutParams(new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
            slidingPaneLayout.addView(leftView, 0);  //添加到SlidingPaneLayout中
            // 右侧的内容视图
            ViewGroup decor = (ViewGroup) getWindow().getDecorView();
            ViewGroup decorChild = (ViewGroup) decor.getChildAt(0);
            decorChild.setBackgroundColor(getResources().getColor(android.R.color.white));
            decor.removeView(decorChild);
            decor.addView(slidingPaneLayout);
            // 为 SlidingPaneLayout 添加内容视图
            slidingPaneLayout.addView(decorChild, 1);
        }
    }

    /**
     * 是否支持滑动返回
     *
     * @return
     */
    protected boolean isSupportSwipeBack() {
        return true;
    }

    @Override
    public void onPanelClosed(View view) {

    }

    @Override
    public void onPanelOpened(View view) {
        finish();
        this.overridePendingTransition(0, R.anim.out_to_right);
    }

    @Override
    public void onPanelSlide(View view, float v) {
    }
}
```

这种方法算是最方便的，没有前面两种方法的BUG，只要你看懂了SlidingPaneLayout，这种方法你基本上就算会了。

------

还有一种滑动返回的实现是直接在Activity里面实现的，在中间Activity里通过手势监听来实现的，但我觉得这样并没有什么兼容性，没有上面的三种方法扩展性强，所以我这里就不阐述了，给出一篇博客，有兴趣的自己看看[android实现向右滑动返回功能](http://blog.csdn.net/ff20081528/article/details/17845753)。
还有看见网上有提到用ViewDragHelper来实现的，暂时我还没不太了解，我会在以后的博客中写出自己实现后的一些想法，这里就暂时先搁着。

关于滑动返回的还有其他我没怎么了解的方法，但肯定还有，知道的人欢迎推荐！谢谢！

------

## 推荐博客区

下面先给出与本博客相关的额博客并感谢这些博主：

[仿手机QQ聊天列表滑动菜单删除和手势滑动返回的两种方式](http://blog.csdn.net/finddreams/article/details/44678171)
[SlidingPaneLayout详解](http://www.jianshu.com/p/67ce59c9e747)

给出一些博客自行参考：

[Activity滑动返回操作](http://www.jianshu.com/p/5104a6a53716#)
[一步一步教你150行代码实现简书滑动返回效果](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0729/3241.html)
[BGASwipeBackLayout-Android](https://github.com/bingoogolapple/BGASwipeBackLayout-Android)

------

若要转载，请注明[出处](http://penghesheng.cn/)

最后，欢迎到[代码源地址](https://github.com/PengHesheng/ToolBar-BackActivity)进行下载学习，也欢迎大佬们指教！