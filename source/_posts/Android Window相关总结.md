---
title: Android Window相关总结
tag: Android

---

<meta name="referrer" content="no-referrer" />



# Window相关总结

## Window概念

## 几种Window的用法

## Window的原理及源码分析

## 相关面试题

1. AlertDialog,popupWindow,Activity区别
    AlertDialog builder：用来提示用户一些信息,用起来也比较简单,设置标题类容和按钮即可,如果是加载的自定义的view ,调用 dialog.setView(layout);加载布局即可
    popupWindow：就是一个悬浮在Activity之上的窗口，可以用展示任意布局文件。
    activity：Activity是Android系统中的四大组件之一，可以用于显示View。Activity是一个与用记交互的系统模块，几乎所有的Activity都是和用户进行交互的

    区别：AlertDialog是非阻塞式对话框：AlertDialog弹出时，后台还可以做事情；而PopupWindow是阻塞式对话框：PopupWindow弹出时，程序会等待，在PopupWindow退出前，程序一直等待，只有当我们调用了dismiss方法的后，PopupWindow退出，程序才会向下执行。这两种区别的表现是：AlertDialog弹出时，背景是黑色的，但是当我们点击背景，AlertDialog会消失，证明程序不仅响应AlertDialog的操作，还响应其他操作，其他程序没有被阻塞，这说明了AlertDialog是非阻塞式对话框；PopupWindow弹出时，背景没有什么变化，但是当我们点击背景的时候，程序没有响应，只允许我们操作PopupWindow，其他操作被阻塞。我们在写程序的过程中可以根据自己的需要选择使用Popupwindow或者是Dialog

    （1）Popupwindow在显示之前一定要设置宽高，Dialog无此限制。
    （2）Popupwindow默认不会响应物理键盘的back，除非显示设置了popup.setFocusable(true);而在点击back的时候，Dialog会消失。
    （3）Popupwindow不会给页面其他的部分添加蒙层，而Dialog会。
    （4）Popupwindow没有标题，Dialog默认有标题，可以通过dialog.requestWindowFeature(Window.FEATURE_NO_TITLE);取消标题
    （5）二者显示的时候都要设置Gravity。如果不设置，Dialog默认是Gravity.CENTER。
    （6）二者都有默认的背景，都可以通过setBackgroundDrawable(new ColorDrawable(android.R.color.transparent));去掉

2. Activity-Window-View三者的差别
    Activity像一个工匠（控制单元），Window像窗户（承载模型），View像窗花（显示视图） LayoutInflater像剪刀，Xml配置像窗花图纸。
    1.在Activity中调用attach，创建了一个Window
    2.创建的window是其子类PhoneWindow，在attach中创建PhoneWindow
    3.在Activity中调用setContentView(R.layout.xxx)
    4.其中实际上是调用的getWindow().setContentView()
    5.调用PhoneWindow中的setContentView方法
    6.创建ParentView： 作为ViewGroup的子类，实际是创建的DecorView(作为FramLayout的子类）
    7.将指定的R.layout.xxx进行填充 通过布局填充器进行填充【其中的parent指的就是DecorView】
    8.调用到ViewGroup
    9.调用ViewGroup的removeAllView()，先将所有的view移除掉添加新的view：addView()

3. Window的添加过程
    [Android解析WindowManager（三）Window的添加过程](https://blog.csdn.net/itachi85/article/details/78025485)
    [【源码学习】window 添加 view](https://blog.csdn.net/u011494285/article/details/80979047)

4. Window的删除过程
    [Android解析WindowManagerService（三）Window的删除过程](https://blog.csdn.net/itachi85/article/details/79134490)

5. Window的更新过程
    [【源码学习】window 的删除及更新过程](https://blog.csdn.net/u011494285/article/details/81091213)

    [理解Window的添加，删除，刷新内部机制](https://juejin.im/post/5a5c440cf265da3e5033bcdf)

6. Activity、Window、DecorView、ViewRootImpl之间的区别与联系
    Activity：
    Activity并不负责视图控制，它只是控制生命周期和处理事件。真正控制视图的是Window。一个Activity包含了一个Window，Window才是真正代表一个窗口。Activity就像一个控制器，统筹视图的添加与显示，以及通过其他回调方法，来与Window、以及View进行交互。

    Window：
    Window是视图的承载器，内部持有一个 DecorView，而这个DecorView才是 view的根布局。Window是一个抽象类，实际在Activity中持有的是其子类PhoneWindow。PhoneWindow中有个内部类DecorView，通过创建DecorView来加载Activity中设置的布局 R.layout.activity_main 。Window 通过WindowManager将DecorView加载其中，并将DecorView交给ViewRoot，进行视图绘制以及其他交互。

    DecorView：
    DecorView是FrameLayout的子类，它可以被认为是Android视图树的根节点视图。DecorView作为顶级View，一般情况下它内部包含一个竖直方向的LinearLayout，在这个LinearLayout里面有上下三个部分，上面是个ViewStub,延迟加载的视图（应该是设置ActionBar,根据Theme设置），中间的是标题栏(根据Theme设置，有的布局没有)，下面的是内容栏。

    ViewRoot：
    ViewRoot可能比较陌生，但是其作用非常重大。所有View的绘制以及事件分发等交互都是通过它来执行或传递的。ViewRoot对应ViewRootImpl类，它是连接WindowManagerService和DecorView的纽带，View的三大流程(测量（measure），布局（layout），绘制（draw）)均通过ViewRoot来完成。ViewRoot并不属于View树的一份子。从源码实现上来看，它既非View的子类，也非View的父类，但是，它实现了ViewParent接口，这让它可以作为View的名义上的父视图。RootView继承了Handler类，可以接收事件并分发，Android的所有触屏事件、按键事件、界面刷新等事件都是通过ViewRoot进行分发的。

    [![比较](https://img-blog.csdn.net/20171122172245272?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxNDczMjEwMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](https://img-blog.csdn.net/20171122172245272?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxNDczMjEwMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)比较