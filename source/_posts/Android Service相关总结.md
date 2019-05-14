---
title: Android Service相关总结
tag: Android
---

# Service相关总结

## Service的用法

### 简介

1. Service是Android中实现程序后台运行的解决方案，适合执行那些不需要和用户进行交互而且要求长期运行的任务
2. 服务不依赖于任何用户界面
3. 服务不是运行在一个独立的进程，而是依赖于创建服务时所在的应用程序进程，当应用程序被杀掉，服务也会停止运行
4. 服务默认运行在主线程

### 基本用法

首先新建一个MyService继承自Service，重写父类的onCreate、onStartCommand、onDestroy

## Service的生命周期

## Service的源码分析

## 相关面试题

1. service生命周期
    [![两种启动方式的比较](https://upload-images.jianshu.io/upload_images/4061843-bdf0da877481143a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-bdf0da877481143a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)两种启动方式的比较

    不管哪种方法，onCreate只执行一次；在单独的startService的时候，onStartCommand可以执行多次（每次starService都会调用）；在stopService的时候，onDestroy会直接调用，（但是如果使用了bindService，必须先取消绑定unbindService，否则无法销毁服务）；在单独的bindService的时候，bindService会调用onBind，onBind只会调用一次，且onBind会返回一个IBind接口实例，用于操作服务；在unbindService的时候，onDestory也会调用

    如果一个服务同时使用了startService和bindService，那么就必须同时调用stopService、unbindService，onDestroy才会调用。此时是startService、onCreate、onStartCommand、bindService、onBind、unbindService、onUnbind、stopService、onDestroy

    在Service每一次的开启关闭过程中，只有onStart可被多次调用(通过多次startService调用)，其他onCreate，onBind，onUnbind，onDestory在一个生命周期中只能被调用一次

2. 说说Activity、Intent、Service 是什么关系
    Activity用于和用户交互的界面，是看得见的，每一个Activity都被实现为一个单独的类，这些类都是从Activity基类中继承来的，Activity类会显示由视图控件组成的用户接口，并对视图控件的事件做出响应。

    Intent的调用是用来进行架构屏幕之间的切换的。Intent是描述应用想要做什么。Intent数据结构中两个最重要的部分是动作和动作对应的数据，一个动作对应一个动作数据，Intent主要用于组件之间的通信和数据传递

    Service用于后台服务和一些前台的服务，是运行在后台的代码，不能与用户交互，可以运行在自己的进程，也可以运行在其他应用程序进程的上下文里。需要通过某一个Activity或者其他Context对象来调用。

    Activity跳转到Activity，Activity启动Service，Service打开Activity都需要Intent表明跳转的意图，以及传递参数，Intent是这些组件间信号传递的承载，Activity和Service都是Conttext类的子类ContextWrapper的子类，都不能进行耗时操作，都运行在主线程之间。

3. IntentService原理以及应用场景
    IntentService相比父类Service而言，最大特点是其回调函数onHandleIntent中可以直接进行耗时操作，不必再开线程。在onCreate的时候，初始化了一个HandlerThread（工作线程或子线程），通过HandlerThread的消息队列Looper来初始化ServiceHandler（就是一个Handler），之后handleMessage，包括onHandleIntent等函数都运行在工作线程中，每次启动的时候都会执行onStartCommand方法，而onStartCommand又会调用onStart，发送消息到ServiceHandler中处理（ServiceHandler是拥有HandlerThread的Looper的），handleMessage中onHandleIntent处理完了后，就会自我销毁，回调onDestroy。如果多次调用onHandleIntent函数（也就是有多个耗时任务要执行），多个耗时任务会按顺序依次执行。

    ```java
    public abstract class IntentService extends Service {
        private volatile Looper mServiceLooper;
        private volatile ServiceHandler mServiceHandler;
        private String mName;
        private boolean mRedelivery;
    
        private final class ServiceHandler extends Handler {
            public ServiceHandler(Looper looper) {
                super(looper);
            }
    
            @Override
            public void handleMessage(Message msg) {
                //在子线程中执行
                onHandleIntent((Intent)msg.obj);
                //然后自我结束服务
                stopSelf(msg.arg1);
            }
        }
    
        public IntentService(String name) {
            super();
            mName = name;
        }
        ...
        @Override
        public void onCreate() {
            super.onCreate();
            //会创建一个HandlerThread，持有Looper的线程，初始化的时候指定线程名
            HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
            thread.start();
    
            mServiceLooper = thread.getLooper();
            //用线程的Looper来初始化ServiceHandler
            mServiceHandler = new ServiceHandler(mServiceLooper);
        }
    
        @Override
        public void onStart(@Nullable Intent intent, int startId) {
            //每次onStart的时候就会发送一条消息到Handler的Looper中，这样就保证了有序执行
            Message msg = mServiceHandler.obtainMessage();
            msg.arg1 = startId;
            msg.obj = intent;
            mServiceHandler.sendMessage(msg);
        }
        ...
        @Override
        public void onDestroy() {
            //销毁的时候退出队列
            mServiceLooper.quit();
        }
        ...
    }
    ```

    IntentService是一种无状态的Service，不支持binService（支持的话就违背了IntentService的设计规范）。例如在之前做的app中，有一个需求是下载某段时间用户保存的图片，下载完成后显示在imageView中；一般需要下载的图片有很多，每下一个图片就是一个线程，下载完后立即显示出来，所以肯定希望下载是按顺序依次下载，这样用户体验就比较好。当时的处理方式欠妥，直接开多个线程去下载，在前一个图片下载完成之前，其他线程必须等待；这样虽然也可以实现功能，但效率上不高，甚至可能出现ANR（具体代码逻辑是，如果后面的线程获得了cpu,而前面的图片还没下完，则等待；假设前一张图片没下完，而后面的线程一直获得cpu，就有问题）。

4. Service的开启方式 （1）startService()，（2）bindService()
    startService和bindService是启动Service的两种方式，不同之处在于startService只能通知服务启动，只能通过stopService来停止服务，但是bindService可以通过IBind接口获取Service实例，来进行Activity和Service的通信，以及对Service的一些管理操作，可以多个Activity绑定Service。当没有Activity绑定才会自动销毁。两者都只会调用一次onCreate，但两者的模式都是独立的。

5. 如何保证Service不被杀死？比较省电的方式是什么
    （1）提升Service的优先级

    ​			intent-filter中可以设置`android:priority="1000"`来提升优先级，1000是最高值  

    （2）提升Service的进程优先级，提升进程优先级，来防止进程的回收  

    （3）在onDestroy中重启Service
    （4）onStartCommand中手动返回START_STICKY，进程被杀死的时候没有效果  

    （5）Application加上Persistent属性
    （6）监听系统广播判断Service状态
    （7）修改权限为系统应用（流氓）
    （8）双进程Service： 让2个进程互相保护**，其中一个Service被清理后，另外没被清理的进程可以立即重启进程
    （9）加上两个类似于守护进程的Service， 分别检查Service的运行状态，注册响应的广播，对其进行守护,一旦发现没有运行就将其启动.
    （10）QQ黑科技: 在应用退到后台后，另起一个只有 1 像素的页面停留在桌面上，让自己保持前台状态，保护自己不被后台清理工具杀死

    省电方式：不知道

6. 怎么在Service中创建Dialog对话框
    以系统对话框的形式弹出

    `dialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);`

    提升为系统级的对话框

7. Service 与 IntentService 的区别
    Service既不是线程，也不是进程，而是依附于主线程的一个组件。Service用于在后台处理一些逻辑业务，不需要和用户进行交互。

    IntentService继承于Service，用于处理异步请求，调用startService（intent）方法后，将请求通过intent传递给intentService，intentService在onCreate方法中构建一个HandlerThread的子线程，用于处理传递过来的请求。通过HandlerThread单独开启一个线程来依次处理所有Intent请求对象所对应的任务。这样以免事务处理阻塞主线程（ＡＮＲ）。执行完所一个Intent请求对象所对应的工作之后，如果没有新的Intent请求达到，则自动停止Service；否则执行下一个Intent请求所对应的任务。

    IntentService在处理事务时，还是采用的Handler方式，创建一个名叫ServiceHandler的内部Handler，并把它直接绑定到HandlerThread所对应的子线程。 ServiceHandler把处理一个intent所对应的事务都封装到叫做onHandleIntent的虚函数；因此我们直接实现虚函数onHandleIntent，再在里面根据Intent的不同进行不同的事务处理就可以了。另外，IntentService默认实现了Onbind（）方法，返回值为null

8. 简单说说 CompletionService 的作用和使用场景及原理
    CompletionService 是一个泛型接口，其实现类是 ExecutorCompletionService，其主要解决的场景是：主线程提交多个任务然后希望有任务完成就处理结果，并且按照任务完成顺序逐个处理（譬如并发请求返回刷新UI的操作，就可以谁请求成功就开始刷而不用等待所有OK才刷等）

    ExecutorCompletionService 实现类依赖于 Executor 完成实际的任务提交执行，自己主要负责结果的排队处理，AbstractExecutorService 的 invokAny 实现就依赖此类，ExecutorCompletionService 内部有一个额外的队列，每个提交给 Executor 的任务都是通过继承 FutureTask 封装过的，FutureTask 在任务结束后会回调 done 方法，所以 ExecutorCompletionService 就在继承 FutureTask 封装重写的 done 方法中将当前 FutureTask 加入额外队列，然后我们通过其 take 或者 poll 方法获取的实质就是从这个额外队列中取数据

9. Service有没有隐式启动方式，（从某一个版本开始）为什么没有
    现在没有，5.0之前有

    可能会引发安全性问题，如果是隐式启动service，可能会导致系统进程挂掉，出现不断重启的现象。所以在5.0之后，Google在隐式启动Service的时候抛出了异常

## 相关参考