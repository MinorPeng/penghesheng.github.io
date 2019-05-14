---
title: Android Fragment相关总结
tag: Android
---

# Fragment相关总结

## Fragment用法

Fragment从 Android 3.0后引入，用来替代Activity实现模块化，可以适配不同屏幕大小的手机或者平板或电视。但是Fragment不能独立存在，必须嵌入到Activity，Fragment具有自己的生命周期，接收它自己的事件，可以在Activity运行时被添加或删除。

### 静态使用Fragment

这是最简单的使用方式，将Fragment当成一个控件，直接在Activity的布局文件绑定

首先需要一个类来继承Fragment，重写onCreateView方法来指定Fragment的布局

```java
public class Fragment1 extends Fragment {

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment1, container, false);
        return view;
    }
}
```

在这里面可以像Activity处理UI、逻辑一样处理Fragment的UI、逻辑，接下来通过fragment来绑定Activity，在Activity的布局里添加并指定一个Fragment

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <fragment
        android:id="@+id/fragment1"
        android:name="com.example.a14512.activitydemo.Fragment1"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</LinearLayout>
```

当系统创建此 Activity 布局时，会实例化在布局中指定的每个Fragment，并为每个Fragment调用 onCreateView() 方法，以检索每个Fragment的布局。系统会直接插入Fragment返回的 View 来替代 \<fragment> 元素

### 动态加载Fragment

除了可以在Activity的布局指定外，还可以通过代码来实现动态添加、更新、删除等操作

首先Fragment的创建都不变，只是在Activity的布局有一点点改变

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <FrameLayout
        android:id="@+id/frameLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</LinearLayout>
```

这里只是将FrameLayout代替了fragment标签，接着我们在Activity来实例化并动态操作Fragment

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        LogUtil.d("onCreate");
        setContentView(R.layout.activity_main);
        //获取FragmentManager
        FragmentManager fragmentManager = getSupportFragmentManager();
        //获取FragmentTransaction来更改，保证一些列Fragment操作的原子性
        FragmentTransaction transaction = fragmentManager.beginTransaction();
        Fragment1 fragment1 = new Fragment1();
        //第一个参数是ViewGroup用来放置Fragment的位置，第二个参数是要放置的fragment
        transaction.add(R.id.frameLayout, fragment1);
        //提交，让更改生效
        transaction.commit();
    }
}
```

这就是刚开始加载Fragment的动态加载，如果我们需要替换另一个Fragment到这个布局容器中，同样也是获取FragmentManager、FragmentTransaction进行替换、删除等操作，最后通过FragmentTransaction来提交更改使之生效

FragmentTransaction常用的操作有如下：
| 方法名 | 含义 |
| :-: | :-: |
| add() | 向一个Activity中添加一个Fragment |
| remove() | 从Activity中移除一个Fragment |
| replace() | 使用另一个Fragment替换当前的Fragment |
| hide() | 隐藏当前的Fragment，不会调用生命周期的回调方法 |
| show() | 显示之前隐藏的Fragment，不会调用生命周期的回调方法 |
| detach() | 会将View从UI中移除，detach之后可以调用attach恢复 |
| attach() | 重建view试图，附加到UI上展示 |

- Fragment回退栈
    可以利用Fragment的回退栈（类似于Activity栈）来保存Fragment的状态，在back后回到之前的Fragment
    `transaction.addToBackStack(String name);`在commit之前，将这次的transaction添加到回退栈，这样就可以保存这次transaction

## 与ViewPager合用

Fragment的使用就像上面那样简单，但是这并不能满足我们的需求。当我们想让Fragment滑动起来，这就需要和ViewPager一起使用了

在Activity的布局中添加ViewPager

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <android.support.v4.view.ViewPager
        android:id="@+id/viewPager"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</LinearLayout>
```

这就相当于一个View容器，接着在Activity实例化

```java
//Fragment的适配器，注意FragmentPagerAdapter 和 FragmentStatePagerAdapter 的区别
public class FragmentAdapter extends FragmentPagerAdapter {
    private ArrayList<Fragment> mFragments = new ArrayList<>();

    public FragmentAdapter(FragmentManager fm, ArrayList<Fragment> fragments) {
        super(fm);
        this.mFragments = fragments;
    }

    @Override
    public Fragment getItem(int position) {
        return mFragments.get(position);
    }

    @Override
    public int getCount() {
        return mFragments.size();
    }
}

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        LogUtil.d("onCreate");
        setContentView(R.layout.activity_main);
        ViewPager viewPager = findViewById(R.id.viewPager);
        ArrayList<Fragment> fragments = new ArrayList<>();
        Fragment1 fragment1 = new Fragment1();
        Fragment2 fragment2 = new Fragment2();
        Fragment3 fragment3 = new Fragment3();
        fragments.add(fragment1);
        fragments.add(fragment2);
        fragments.add(fragment3);
        FragmentAdapter adapter = new FragmentAdapter(getSupportFragmentManager(), fragments);
        viewPager.setAdapter(adapter);
        viewPager.setCurrentItem(0);
        //预加载
        viewPager.setOffscreenPageLimit(4);
    }
}
```

就这样简单就实现了Fragment和ViewPager的联合使用

## Fragment与其他组件的通信

### Fragment和Fragment

- Fragment是同一层级的，Fragment都处于Activity中，没有嵌套
    1. 直接在一个Fragment中调用另一个Fragment的方法（这种方法对于Fragment有着一定的要求）
        这种就是通过获取到另一个Fragment的实例，就可以直接操作另一个Fragment的方法或属性。一般是通过getActivity().getFragmentManager()来获取FragmentManager，这样就可以通过findFragmentById或者findFragmentByTag方法来获取Fragment的实例，这就跟Activity于Fragment的通信一样，因为在Fragment中可以获取到所依赖的Activity
    2. 接口回调，原理跟Activity和Activity的通信差不多
    3. 广播
- Fragment不是同一层级的，Fragment嵌套Fragment
    在宿主Fragment中可以通过getChildFragmentManager来获取FragmentManager类管理子Fragment
    对于子Fragment来说，可以通过getParentFragment() 来获取宿主的FragmentManager

### Fragment和Activity

- 如果你Activity中包含自己管理的Fragment的引用，可以通过引用直接访问所有的Fragment的public方法
- 如果Activity中未保存任何Fragment的引用，那么没关系，每个Fragment都有一个唯一的TAG或者ID,可以通过getFragmentManager.findFragmentByTag()或者findFragmentById()获得任何Fragment实例，然后进行操作。
- 在Fragment中可以通过getActivity得到当前绑定的Activity的实例，然后进行操作，直接调用Activity中的public方法
- Hanlder
- EventBus
- 广播
- getArguments获取Activity传递的参数

## Fragment的生命周期

[![经典图](http://i.imgur.com/fjGYjRN.png)](http://i.imgur.com/fjGYjRN.png)经典图

- onAttach()
    关联Activity时调用
- onCreate()
    创建Fragment时调用，在这里必须初始化Fragment的基础组件
- onCreateView()
    Fragment要绘制自己的界面时调用，这个方法必须返回Fragment的layout，也可以返回null(表示没有界面)
- onActivityCreated()
    当Activity对象完成自己的onCreate方法时调用
- onStart()
    Fragment的UI可见时调用
- onResume()
    Fragment的UI可交互时调用
- onPause()
    Fragment 可见但不可交互时调用
- onStop()
    Fragment 完全不可见时调用
- onDestroyView()
    Fragment 移除视图时调用
- onDestroy()
    清理View资源时调用
- onDetach()
    失去Activity关联时调用

### Fragment和Activity生命周期关系

[![Fragment和Activity](http://i.imgur.com/xglLE0e.png)](http://i.imgur.com/xglLE0e.png)Fragment和Activity

首先加载Fragment，看看生命周期
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-06ee9f0101301e4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-06ee9f0101301e4a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
Fragment在宿主onCreate后相应的生命周期就开始执行了，跟一个View有点类似，在宿主onResume后才调用onResume

如果是通过按钮来添加就有所区别，因为这时Activity已经完成了onResume的调用
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-3637646cf201c4b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-3637646cf201c4b3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
先是宿主Activity的onCreate、onStart、onResume调用，接着加载Fragment，onAttach、onCreate、onCreateView、onActivityCreated、onStart、onResume依次调用

跳转到一个新的活动
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-9506ff5f453c1de3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-9506ff5f453c1de3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
当在Fragment中启动一个新的活动的时候，先是Fragment的onPause调用，接着是宿主Activity的onPause调用，然后就是新活动的生命周期，当Fragment和宿主Activity完全不可见后，就先调用Fragment的onStop，接着是宿主Activity的onStop

如果新启动的Activity是透明主题或者是一些其他的view不能遮挡Fragment，那么就不会执行onStop
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-caad0df0c1a6ac4e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-caad0df0c1a6ac4e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
这个跟Activity被遮挡类似，不会有onStop调用

接着按下back键结束活动
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-a570c13e13d1dbbb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-a570c13e13d1dbbb.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
当结束一个活动后，发现是宿主Activity先onRestart，然后再是Fragment的onStart，接着宿主Activity的onStart、onResume调用，最后才是Fragment的onResume调用（这里就可以看出Fragment是依赖于宿主Activity的，只有当Activity完全可见的时候，Fragment的onResume才会调用）

再back键结束宿主活动
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-a6894843ce0e1288.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-a6894843ce0e1288.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
结束宿主活动时，先是Fragment的onPause调用，接着是宿主Activity的onPause，Fragment的onStop，宿主的onStop，然后Fragment的onDestroyView、onDestroy、onDetach，最后是宿主活动的onDestroy

试试replace后的生命周期
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-6f5deb44aab68d49.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-6f5deb44aab68d49.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
replace对Fragment的生命周期有影响，就是Fragment结束的生命周期

hide之前
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-7a32739f3657bcb3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-7a32739f3657bcb3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
hide后的生命周期
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-7a32739f3657bcb3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-7a32739f3657bcb3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
show后的生命周期
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-7a32739f3657bcb3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-7a32739f3657bcb3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
发现，不管是hide还是show，都不会影响Fragment的生命周期

remove后的生命周期
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-cd0a8865d10f442e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-cd0a8865d10f442e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
当remove的时候，就相当于将Fragment从宿主Activity移除，所以就会结束Fragment

### 异常情况的Fragment生命周期

例如当横竖屏切换时
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-b6709b8d62760553.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-b6709b8d62760553.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
Fragment依次调用onPause、onSaveInstanceState、onStop、onDestoryView、onDestroy、onDetach，销毁时候的生命周期调用Fragment总是先于宿主Activity；接着重新构建的时候，却是Fragment先调用onAttach、onCreate，然后才是宿主Activity的onCreate，再然后又是Fragment的onCreateView、onActivityCreated、onViewStateRestored、onStart，接着宿主Activity的onStart、onRestoreInstanceState、onResume，最后时Fragment的onResume。可以看出，销毁时先保存Fragment、先销毁Fragment，重建时，先onCreateFragment，然后在宿主Activity onCreate之后就先恢复Fragment，当宿主resume后，Fragment才resume直接展示了。

## 相关面试题

1. fragment 各种情况下的生命周期
    切换到Fragment(第一次)

    > onAttach
    > onCreate
    > onCreateView
    > onActivityCreated
    > onStart
    > onResume

    屏幕熄灭

    > onPause
    > onSaveInstanceState
    > onStop

    屏幕解锁

    > onStart
    > onResume

    切换到其他Fragment

    > onPause
    > onStop
    > onDestroyView

    切换回本身

    > onCreateView
    > onActivityCreated
    > onStart
    > onResume

    回到桌面

    > onPause
    > onSaveInstanceState
    > onStop

    回到应用

    > onStart
    > onResume

    退出应用

    > onPause
    > onStop
    > onDestroyView
    > onDestroy
    > onDetach

2. 遇到过哪些关于Fragment的问题，如何处理的？
    Fragment嵌套Fragment时通信的问题，还有就是一些控制操作的问题

    在宿主Fragment中可以通过getChildFragmentManager来获取FragmentManager类管理子Fragment
    对于子Fragment来说，可以通过getParentFragment() 来获取宿主的FragmentManager

3. 如何实现Fragment的滑动
    ViewPager的联合使用，通过ViewPager来操作

4. ViewPager使用细节，如何设置成每次只初始化当前的Fragment，其他的不初始化
    viewPager有一个setOffscreenPageLimit(int);默认值是1，这个就是每次左右Fragment的初始化个数

5. fragment之间传递数据的方式？
    重复问题

6. Fragment的生命周期，栈管理，和commit()类似的方法还有哪个
    [Fragment笔记整理](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0214/2481.html)

7. Fragment状态保存startActivityForResult是哪个类的方法，在什么情况下使用，如果在Adapter中使用应该如何解耦

8. Fragment的懒加载：
    在Fragment中有一个setUserVisibleHint这个方法，而且这个方法是优于onCreate()方法的，所以也可以作为Fragment的一个生命周期来看待，它会通过isVisibleToUser告诉我们当前Fragment我们是否可见，getUserVisibleHint()，它就是用来判断当前Fragment是否可见

9. Fragment如何去调用Activity的方法
    getActivity、getContext

10. 如何切换 fragement,不重新实例化
    适用Fragment适配器，和ViewPager联合适用，基本上都是提前实例化好，切换时使用

11. Fragment与Fragment、Activity通信的方式
    Fragment与Fragment：
    （1）在宿主Fragment中可以通过getChildFragmentManager来获取FragmentManager类管理子Fragment
    （2）对于子Fragment来说，可以通过getParentFragment() 来获取宿主的FragmentManager
    （3）如果能直接获取对应的实例可以直接通信
    （4）接口，同样适用和Activity
    （5）广播，同样适用和Activity
    （6）EventBus

    Fragment与Activity：
    和前面Fragment之间有些重复
    不同的就是在Fragment中可以通过getActivity来获取Activity的实例

12. Activity与Fragment之间生命周期比较
    如前面的图

## 相关参考

[Fragment 用法总结（一）](https://blog.csdn.net/handsome_926/article/details/50736024)
[Fragment 用法总结（二）](https://blog.csdn.net/handsome_926/article/details/50771239)
[Android Fragment 真正的完全解析（下）](https://blog.csdn.net/lmj623565791/article/details/37992017)
[Android Fragment使用(二) 嵌套Fragments (Nested Fragments) 的使用及常见错误](https://www.cnblogs.com/mengdd/p/5552721.html)
[Fragment生命周期详解](http://seniorzhai.github.io/2015/01/05/Fragment生命周期详解/)