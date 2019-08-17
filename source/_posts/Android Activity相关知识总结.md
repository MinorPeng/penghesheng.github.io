---
title: Android Activity相关知识总结
tag: Android 

---

<meta name="referrer" content="no-referrer" />



[TOC]

# Activity相关知识总结

## Activity的用法

Activity主要用于和用户进行交互，负责UI的加载与页面之间的跳转

应用启动的时候都会加载一个默认的Activity，这就是我们通常的主活动，在AndroidManifest里面设置intent-filter

```xml
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
</intent-filter>
```

通过给Activity添加如此拦截器，设置为APP的主活动，也就是整个APP的入口（表面上的入口），同时在AndroidManifest里面，标签\<activity>就标明了这是一个活动

直接新建一个应用，默认的Activity会是这样

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

代码中可以看出，Activity就是一个继承AppCompatActivity的类，继承这个类后，就成了一个活动，当然也可以继承Activity这个类，两者的区别在于AppCompatActivity是v7的，也就是兼容性更好，Activity兼容性相比之下比较差一点，当然，现在都一般用AppCompatActivity。

由此可见，创建一个活动很简单，继承了AppCompatActivity就行了，然后你就可以去操作你的UI加载和布局了

## Activity的启动模式

每一个程序都有一个或很多个Activity组成，因此Android内部使用通过回退栈开管理Activity实例，这就是hiActivity栈。对于Android来说，处于栈顶的Activity就是当前显示的页面，通过返回键来销毁这个栈顶的Activity回到上一个Activity
[![网上找的图](http://upload-images.jianshu.io/upload_images/3985563-83df2c2b5d3afafd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](http://upload-images.jianshu.io/upload_images/3985563-83df2c2b5d3afafd.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)网上找的图

通过AndroidManifest的\<activity>标签来指定`android:launchMode`来选择启动模式（或者通过Intent设置标志位来指定启动模式`intent.addFlags(Intent,FLAG_ACTIVITY_NEW_TASK);`），进而控制Activity栈的出栈规则。Intent则是Android程序中各组件之间进行交互的一种重要方式，不仅可以指明当前组件要执行的动作，还可以在不同组件中传递数据。因此，Activity的启动一般是通过它来实现的。

|               标志位                |                             含义                             |
| :---------------------------------: | :----------------------------------------------------------: |
|       FLAG_ACTIVITY_NEW_TASK        |           新活动放置在新建的一个栈，对应singleTask           |
|      FLAG_ACTIVITY_SINGLE_TOP       |               singleTop，处于栈顶不会新建实例                |
|       FLAG_ACTIVITY_CLEAR_TOP       | 如果设置，并且这个Activity已经在当前的Task中运行，因此，不再是重新启动一个这个Activity的实例，而是在这个Activity上方的所有Activity都将关闭，然后这个Intent会作为一个新的Intent投递到老的Activity（现在位于顶端）中 |
| FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS  | 如果设置，新活动不保存在最近启动的活动列表中，等同于指定`android:excludeFromRecents="true"` |
|      FLAG_ACTIVITY_NO_HISTORY       | 如果新活动不是在历史栈中，当用户离开这个活动（不管是新启动一个活动还是结束这个活动）就直接结束，设置此标志activity将不添加到回退栈（backStack） |
|     FLAG_ACTIVITY_MULTIPLE_TASK     | 新建一个任务栈，用于放置这个活动，这个标志总是和FLAG_ACTIVITY_NEW_DOCUMENT或者FLAG_ACTIVITY_NEW_TASK联合使用 |
|   FLAG_ACTIVITY_BROUGHT_TO_FRONT    | 这个flag不能正常地被应用程序代码设置，而是系统为你设置由于在 launchMode 设置为singleTask模式 |
|      FLAG_ACTIVITY_CLEAR_TASK       | 如果通过 Context.startactivity()去设置/启动一个Intent，这个flag将导致任何存在的task，将与活动开始前清除的活动相关 |
|    FLAG_ACTIVITY_FORWARD_RESULT     | 如果设置这个intent是被用来从一个现有的acitivity启动到新的acitivity，现有activity的回复目标将被转移到新的activity |
| FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY | 这个flag不能正常地被应用程序代码设置，而是系统为你设置，如果这个活动正在展开的历史堆栈（长按 Home键） |
|     FLAG_ACTIVITY_NEW_DOCUMENT      | 此标志用于将文档打开到一个新的任务中，该任务源于intent启动的活动 |
|     FLAG_ACTIVITY_NO_ANIMATION      | 如果通过 Context.startactivity()去设置/启动一个Intent，这个标志将阻止系统执行一个活动去下一个活动的过渡动画 |
|    FLAG_ACTIVITY_NO_USER_ACTION     | 设置此标志，将阻止onuserleavehint()正常回调发生在当前最前的活动，在它被停下来作为新启动活动被带到前面 |
|    FLAG_ACTIVITY_PREVIOUS_IS_TOP    | 如果设置并使用此意图从现有的一个activity a启动到新activity b，新avitivity b将不会被视为栈顶而是activity a，而是决定是否新意图传递到顶部而不是启动新的活动 |
| FLAG_ACTIVITY_RESET_TASK_IF_NEEDED  | 设置此标志使这个活动要么开始在一个新的任务或带到现有的任务的顶部，那么它将被启动作为任务的前门 |
|   FLAG_ACTIVITY_REORDER_TO_FRONT    | 如果在通过 Context.startactivity()去设置/启动一个Intent，如果需要启动的activity已经运行，此标志使被启动的活动被带到任务的历史堆栈的前面 |
|     FLAG_ACTIVITY_TASK_ON_HOME      | 如果在通过 Context.startactivity()去设置/启动一个Intent，此flag将使新启动任务置于当前活动任务的顶部（如果只有一个task时） |
| FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET | 这个常数是在API级别21废弃掉。在API 21执用 flag_activity_new_document 替代 |
|                                     |                                                              |

```java
Intent intent = new Intent(this, TargetActivity.class);
intent.addFlags(Intent,FLAG_ACTIVITY_NEW_TASK);
startActivity(intent);
```

这就是简单活动的启动

Activity可以通过Intent来实现启动各组件之间的通信。首先我们先见识了显示启动Activity的用法，现在看看隐式使用Intent。

首先在AndroidManifest中，通过\<activtiy>标签被指\<intent-filter>来指定活动响应的action和category

```java
<activity android:name=".SecondActivity">
    <intent-filter>
        <action android:name="com.example.a14512.activitydemo.ACTION_START"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="com.example.a14512.activitydemo.MY_CATEGORY"/>
    </intent-filter>
</activity>

//启动活动，传入action
Intent intent = new Intent("com.example.a14512.activitydemo.ACTION_START");
//添加category
intent.addCategory("com.example.a14512.activitydemo.MY_CATEGORY");
startActivity(intent);
```

每个Intent只能指定一个action，但是可以指定多个category，通过addCategory来添加（启动活动时一定要注意action和category要与AndroidManifest中的匹配对应，否则出现找不到Activity的崩溃）；除了action和category外，还有一个data，一旦在\<intent-filter>设置了这三个过滤类别，当启动activity的时候，Intent必须完全匹配这些才能成功启动Activity。另外，一个intent-filter可以有有多个action、category、data，而一个Activity可以有多个intent-filter，一个Intent只要能匹配其中一个组就可以启动Activity

- Intent-filter匹配规则

    1. action
        是一个字符串，系统预定义了一些action，同时我们也可以定义自己的action。Intent中的action必须和intent-filter中的action匹配（字符串值完全一样）。 action字符串区分大小写

    2. category
        是一个字符串，系统预定义了一些category，也可以自定义。Intent中可以不用设置category（系统会默认添加默认的default，intent-filter必须添加一个default的category），但一旦设置了，就必须和intent-filter中的category中的一个匹配

    3. data
        和action类似，如果intent-filter中定义了data，Intent必须也要定义可匹配的data
        data语法：

        ```java
        <data android:scheme="string"
                    android:host="string"
                    android:port="string"
                    android:path="string"
                    android:pathPattern="string"
                    android:pathPrefix="string"
                    android:mimeType="string" />
        ```

        data由mimeType和URI两部分组成，mimeType指媒体类型，比如image/jpeg、video/*等，可以表示图片、文本、视频等不同的媒体格式。

        URI的结构：
        `<scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>]`

        > scheme：URI的模式，比如http、file、content；如果没有指定scheme，则整个URI无效
        > host：URI的主机名
        > port：URI中的端口号
        > path、pathPattern、pathPrefix：表述路径信息，path表示完整路径，pathPattern也表示完整路径，但是可以包含*统配符，pathPrefix表示路径的前缀信息

        一个小例子：

        ```java
        <intent-filter>
                ...
                <data android:scheme="image/*"/>
        </intent-filter>
        
        //intent使用，如果传入的uri参数与intent-filter中不匹配就会报错
        //不能先调用setData，再调用setType
        intent.setDataAndType(Uri.parse("file://abc"), "image/png");
        ```

在每次使用隐式启动Activity时，可以通过PackageManager的resolveActivity或Intent的resolveActivity方法先检测是否有Activity匹配，放止找不到Activity。

- 隐式Intent的其他用法

    1. 打开网页

        ```java
        //Android内置的动作
        Intent intent = new Intent(Intent.ACTION_VIEW);
        //setData接受一个Uri对象，主要用于指定正在操作的数据
        intent.setData(Uri.parse("http://www.baidu.com"));
        startActivity(intent);
        ```

    2. 调用拨号

        ```java
        //内置的动作
        Intent intent = new Intent(Intent.ACTION_DIAL);
        //同样一个Uri对象
        intent.setData(Uri.parse("tel:10086"));
        startActivity(intent);
        ```

下面介绍四种启动模式

### **standard**

Activity的标准启动模式，当你没有通过`android:launchMode`来指定启动模式的时候，Activity的启动模式就是这个了。

在这种情况下，Activity的实例可以有多个，Activity可以被多次实例化。每启动一个Activity，Activity栈中就会有这么一个实例，Activity栈不会管之前创建没有，会重复实例化。
[![借用郭神的示例图](https://upload-images.jianshu.io/upload_images/4061843-8ce1962079fe22e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-8ce1962079fe22e0.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)借用郭神的示例图

在MainActivity的布局添加一个Button，用来启动Activity，打印日志
[![standard](https://upload-images.jianshu.io/upload_images/4061843-909953040b9cc7ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-909953040b9cc7ad.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)standard由此可见，每启动一个Activity，就会新建一个Activity的实例，依次放入栈中

### **singleTop**

栈顶复用模式，在这中模式下，如果Activity已经处于栈顶，重复启动这个Activity的话，不会新建这个Activity的实例，会重用这个已经在栈顶的Activity，并会调用该实例的onNewIntent函数将Intent对象传递到这个实例中；但是如果不在栈顶的话，启动一个Activity就会新建一个Activity的实例，即使以前创建了一次，这也就意味着，不是栈顶的Activity仍然可以重复实例化
[![处于栈顶](https://upload-images.jianshu.io/upload_images/4061843-0718c07f0a8a513a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-0718c07f0a8a513a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)处于栈顶
[![没有处于栈顶](https://upload-images.jianshu.io/upload_images/4061843-674b6ab8f33a39c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-674b6ab8f33a39c2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)没有处于栈顶

新建一个SecondActivity，将SecondActivity和MainActivity的启动模式设为singleTop，先启动SecondActivity，再启动一次SecondActivity，打印日志
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-1721e603d9e0de8c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-1721e603d9e0de8c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png由此可见，不管你在SecondActivity启动几次SecondActivity，它总是不会新建实例，只会重新调用onNewIntent这个方法来接受传递过来的Intent

然后再试一下，启动MainActivity，再启动SecondActivity，最后再启动MainActivity
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-08e44548d89d4933.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-08e44548d89d4933.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png可以看见，当MainActivity和SecondActivity不是处于栈顶的时候，就可以再次创建自己的实例

### **singleTask**

栈内复用模式，是一种单实例模式，在一个Activity栈中只能存在一个实例，当启动一个Activity后，系统会首先查找是由存在A想要的任务栈，如果不存在，就要重新创建一个，然后在把这个Activity的实例放到任务栈中；如果已经存在这个Activity要的任务栈，就查看栈中是否有这个Activity的实例，如果已存在这个Activity的实例，当再一次启动的时候，如果上面有其他的Activity，就会销毁该Activity上的所有Activity，最终让这个Activity处于栈顶，然后调用onNewIntent函数

- 如果要启动的Activity需要的栈就是跟MainActivity一个栈
    [![同一个栈](https://upload-images.jianshu.io/upload_images/4061843-f537a3dae37e9fd8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-f537a3dae37e9fd8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)同一个栈

- 要启动的Activity跟MainActivity不是同一个栈（开发艺术探索）
    [![不同栈，且处于栈顶](https://upload-images.jianshu.io/upload_images/4061843-2deca2228ccf3d3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-2deca2228ccf3d3a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)不同栈，且处于栈顶

    [![不同栈，但不是栈顶](https://upload-images.jianshu.io/upload_images/4061843-048db956f587adc5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-048db956f587adc5.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)不同栈，但不是栈顶

创建ThirdActivity，启动模式设为singlTask。其余默认
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-b3a68a0130c09228.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-b3a68a0130c09228.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png可以看到，先启动了ThirdActivity，接着启动了SecondActivity，然后又启动ThirdActivity，ThirdActivity并没有重新实例化，而是restart了之前的，并销毁了SecondActivity。

将ThirdActivity单独指定一个栈，启动模式为singleTask，其余为默认，然后一次启动MainActivity、SecondActivity、ThirdActivity，再次启动ThirdActivity
[![不同栈，且处于栈顶](https://upload-images.jianshu.io/upload_images/4061843-c2ef318fad9ba1f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-c2ef318fad9ba1f1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)不同栈，且处于栈顶可以看出，单独一个栈的ThirdActivity已经处于栈顶，不管你重复启动多少次，都不会变

将FourthActivity和ThirdActivity单独指定一个栈，利用`android:taskAffinity`指定，ThirdActivity设置启动模式为singleTask，其他默认，然后先启动SecondActivity，接着启动ThirdActivity，再启动FourthActivity，再启动ThirdActivity，打印日志
[![不同栈，不是栈顶](https://upload-images.jianshu.io/upload_images/4061843-3d43ba6987c31fe2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-3d43ba6987c31fe2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)不同栈，不是栈顶可以看出，当单独一个栈的ThirdActivity和FourthActivity，重新启动ThirdActivity，会销毁FourthActivity，但不会影响MainActivity这个栈的活动

### **singleInstance**

这是一种加强的单实例模式，在一个Activity栈中只能存在一个实例，且这个栈是独立的一个栈，保证整
个系统中只有一个singleInstance的栈和一个Activity实例
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-efb2d9b75211992e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-efb2d9b75211992e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png

将SecondActivity的启动模式改为singleInstance，其余默认，然后依次启动MainActivity、SecondActivity、FourthActivity
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-c29d1feeb1430261.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-c29d1feeb1430261.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png可以看出，SecondActivity和MainActivity的TaskId并不一致，说明它们不处于同一个栈

## Activity和其他组件的通信

### Activity和Activity

首先，我们可以通过Intent来显示启动活动，那么与此同时，我们可以通过Intent来向启动的活动传递数据，进而进行通信

#### 使用Intent

- intent向下一个活动传递数据：
    先来个简单的例子

    ```java
    Intent intent = new Intent(MainActivity.this, SecondActivity.class);
    //传递一个String数据
    intent.putExtra("data_key", "MainActivity");
    startActivity(intent);
    
    Intent intent = getIntent();
    //根据key值接收String数据
    String data = intent.getStringExtra("data_key");
    ```

    这就是一个简单的数据传递

    Intent传递的数据也是有限的，只能传递基本类型、String、Parcelable、Serializable以及它们相关的数组类型，这是Intent可以直接传递的类型，当然，还可以将数据装在Bundle里面，通过Intent来传递，同样在传递时，都需要指定一个String的name值，方便在接收的时候准确获取对应的数据。

- intent返回数据给上一个活动
    一个例子

    ```java
    //启动活动
    Intent intent = new Intent(MainActivity.this, SecondActivity.class);
    //requestCode设置为1
    startActivityForResult(intent, 1);
    
    //重写方法，requestCode一般使用系统内置的，也可以自己自定义，resultCode时我们前面设置的
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        if (resultCode == RESULT_OK) {
            switch (requestCode) {
                //判断数据来源
                case 1:
                    //获取数据
                    String resultData = data.getStringExtra("data_key");
                    break;
                default:
                    break;
            }
        }
    }
    
    //在结束活动的时候
    Intent intent = new Intent();
    intent.putExtra("data_key", "result");
    //第一个参数用于向上一个活动返回处理结果，一般为RESULT_OK或RESULT_CANCELED，第二个参数用于返回带有数据的Intent
    setResult(RESULT_OK, intent);
    ```

#### 借助类的静态变量

由于类的静态成员可以通过“className.fileName”进行访问，所以可以在Activity中生命一个静态变量从而实现Activity之间的数据通信

例如：

```java
//MainActivity中
//先查看值
LogUtil.d(SecondActivity.name);
SecondActivity.name = "MainActivity change";
//查看修改后的值
LogUtil.d(SecondActivity.name);
Button button = findViewById(R.id.btnStartActivity);
button.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        Intent intent = new Intent(MainActivity.this, SecondActivity.class);
        startActivity(intent);
    }
});

//SecondActivity中，声明一个静态变量并赋予初值
public static String name = "SecondActivity";
//启动SecondActivity后查看值是否变更
LogUtil.d(name);
```

结果：
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-daa46052a40b37e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-daa46052a40b37e3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
可以看出，我们可以访问SecondActivity的静态变量，通过这个例子，我们也可以实现两个Activity之间的通信

#### 通过全局变量/Application

跟在Activity中生命一个静态变量类似，只不过单独一个类生命的静态变量就是全局访问的，这就跟我们平时会写一个Config类来管理全局静态常量类似。就是访问同一块内存区（内存共享）

小例子：

```java
//第三方类
public class Config {
    public static String name = "Config";
}

//MainActivity中
LogUtil.d(Config.name);
Config.name = "MainActivity change";
LogUtil.d(Config.name);

//SecondActivity中
LogUtil.d(Config.name);
```

结果：
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-fce165e51ca8bef8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-fce165e51ca8bef8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
从结果中就可以看出，效果跟前面一个是一样的，原理其实也差不多

#### Broadcast或者LocalBroadcast

在一个Activity发送广播，在另一个接受并解析数据

#### 外部存储

外部存储就是通过访问第三方，来实现数据通信（这个需要注意同步并发的问题），一下几个基本上都是数据的持久化

- SharedPreference
- SQLite
- File
- Service
    后台服务，数据存取，直连的Activity就可以使用这些数据

#### EventBus

适用于页面之间的信息传递和同步，EventBus是一个第三方库，不仅可以用于Activity和Activity，其余的组件间都可以，这个开源库就是用来进行组件间的通信的

#### RxBus

类似于EventBus，就是EventBus的一种替代

#### otto

OTTO是Square推出的库，原理与EventBus相同，实现方式也非常类似。不同的是接收方的回调方法不是固定的那四个，而是通过@Subscribe注解标志来识别接收方的回调方法的。

### Activity和Fragment

1. 如果Activity中有Fragment实例，可以通过直接调用Fragment中的public方法

    ```java
    public class Fragment1 extends Fragment {
        public String getFun() {
            return "fragment";
        }
    }
    
    //Activity中使用
    Fragment1 fragment1 = new Fragment1();
    fragment1.getFun();
    ```

2. 如果Activity中没有Fragment实例，每个Fragment都有唯一的TAG或者ID,可以通过getFragmentManager.findFragmentByTag()或者findFragmentById()获得任何Fragment实例，然后进行1中的操作

    ```java
    //通过findFragmentById方法获取Fragment
    Fragment1 fragment1 = (Fragment1) getSupportFragmentManager().findFragmentById(R.id.fragment1);
    
    //通过findFragmentByTag
    Fragment1 fragment1 = (Fragment1) getSupportFragmentManager().findFragmentByTag("Fragment1");
    ```

3. 如果Activity向Fragment传递参数，在Activity通过setArguments传递，在Fragment通过getArguments获取
    官方文档推荐，在每个Frament里必须有一个空的构造函数，以便其可以实例化，但是不推荐含有带参的构造方法，用来传递参数，因为这些构造函数，并不能在调用Fragment时，将Fragment重新实例化，如果需要传递参数，可以通过setArguments传递，在Fragment通过getArguments。

    ```java
    Fragment1 fragment1 = new Fragment1();
    //通过setArguments和Bundle传递数据
    fragment1.setArguments(new Bundle());
    
    //获取Activity传递过来的数据
    Bundle bundle = getArguments();
    ```

4. 通过回调
    （1）在Activity中写接口，Fragment实现接口，在Activity回调

    ```java
    //Activity中
    public interface MainListener {
        void getFun1();
    }
    
    private MainListener mMainListener;
    Fragment1 fragment1 = new Fragment1();
    mMainListener = fragment1;
    //回调
    mMainListener.getFun1();
    
    //Fragment中
    public class Fragment1 extends Fragment implements MainActivity.MainListener {
        ...
        @Override
        public void getFun1() {
            LogUtil.d("main interface");
        }
    }
    ```

    （2）在Fragment中写接口，Activity实现接口，在Fragment中回调

    ```java
    //Fragment中
    public class Fragment1 extends Fragment {
        private FragmentListener mListener;
    
        @Nullable
        @Override
        public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
            mListener = (FragmentListener) getActivity();
            //回调
            mListener.getFun2();
            View view = inflater.inflate(R.layout.fragment1, container, false);
            return view;
        }
    
        public interface FragmentListener {
            void getFun2();
        }
    }
    
    //Activity中实现接口
    public class MainActivity extends AppCompatActivity implements Fragment1.FragmentListener {
        ...
        @Override
        public void getFun2() {
            LogUtil.d("fragment interface");
        }
    }
    ```

    由此看来，回调的方法是可以行的，不管你在Activity中回调还是在Fragment中回调都是可以的

5. Handler
    也可以使用Handler实现组件通信

    ```java
    //Activity
    @SuppressLint("HandlerLeak")
    public Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };
    
    //Fragment
    @Override
    public void onAttach(Activity activity) {
        super.onAttach(activity);
        if (activity instanceof MainActivity) {
            Handler handler = ((MainActivity) activity).mHandler;
        }
    }
    ```

6. 广播

7. EventBus

8. RxBus

### Activity和Service

- Activity启动Service的两种方式:

    ```java
    //CustomService 是自定义Service，完成一些后台操作
    Intent intent = new Intent(MainActivity.this，CustomService.class)；
    //start和stop Service的时候都可以进行数据传递
    startService(intent);
    stopService(intent);
    
    //可以通过绑定服务启动，来进行一些操作
    bindService(new Intent(MainActivity.this，CustomService.class)), new ServiceConnection() {
            @Override
            public void onServiceConnected(ComponentName name, IBinder service) {
                //当前启动的service 一些数据就会回调回这里，我们在Activity中操作这些数据即可
            }
    
            @Override
            public void onServiceDisconnected(ComponentName name) {
    
            }
        }, flags);
    ```

    从启动方式就可以看出，通过Bundle对象的形式存储，通过Intent传输，来完成Activity向Service传递数据的操作

- 接口回调
    Service设置接口实例，Activity实现

### Activity和Broadcast

通过发送广播，利用Intent传递数据，实现Activity和Broadcast的通信

```java
Intent intent = new Intent();
sendBroadcast(intent);
```

Activity与BroadcastReceiver通信时，用的也是Intent传递，Bundle存储

## Activity的生命周期

[![经典周期图](http://upload-images.jianshu.io/upload_images/3985563-a2caa08d3cbca003.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](http://upload-images.jianshu.io/upload_images/3985563-a2caa08d3cbca003.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)经典周期图

### 典型情况下的生命周期

1. onCreat
    表示Activity正在被创建，生命周期的第一个方法。一般做一些初始化的工作
2. onStart
    表示Activity正在启动，即将开始，Activity已经可见，但是没有出现在前台（在后台），还无法和用户交互（我们无法看到，但是Activity已经显示出来）
3. onResume
    表示Activity已经可见了，并且出现在前台并开始活动。
4. onPause
    表示Activity正在停止，正常情况onStop会接着调用。特殊情况下，如果快速地返回当前Activity，会调用onResume
5. onStop
    表示Activity即将停止，可以做一些稍微重量级的回收工作，不能耗时操作
6. onDestroy
    表示Activity即将被销毁，做一些回收工作和最终的资源释放
7. onRestart
    表示Activity正在重新启动。一般情况下，Activity由不可见重新变为可见状态时调用

除了onRestart方法，其余的都是两两相对的，从而可以分为3中生存期

- 完整生存期
    在onCreat和onDestroy方法之间所经历的就是一个正常的完整的周期
- 可见生存周期
    在onStart和onStop方法之间所经历的就是可见生存期，在这个周期内，活动对于用户总是可见的，即使可能无法交互。
- 前台生存期
    在onResume和onPause方法之间所经历的就是前台生存期，在这个期间，活动总是处于运行状态的，可以和用户交互

小例子：
当启动一个正常的活动时，观察MainActivity的生命周期，当启动MainActivity时，onCreat、onStart、onResume先后调用
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-a75c83eefd1005a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-a75c83eefd1005a0.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png

接着启动SecondActivity，观察MainActivity的周期，发现，onPause、onStop被调用了，此时MainActivity被SecondActivity遮挡了。从结果中同样可以观察到，先是MainActivity的onPause调用，然后是SecondActivity的onCreate、onStart、onResume被调用（此时，MainActivity变为完全不可见了），然后MainActivity的onSavaInstanceState、onStop被调用
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-8d2799aab84769f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-8d2799aab84769f6.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png

接着按下back键，返回MainActivity，此时onRestart、onStart、onResume调用（此前MainActivity是停止状态，所以不会重建调用onCreate）。从结果中还以知道，先是SecondActivity的onPause被调用，接着MainActivity的onRestart、onStart、onResume被调用，然后SecondActivity的onStop、onDestroy被调用。说明SecondActivity的回收需要时间，会先onPause，然后展示MainActivity让它置于前台，最后才慢慢收回SecondActivity
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-f5931b493ab0279b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-f5931b493ab0279b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png

当点击按钮，启动DialogActivity（主题设置为Dialog，对话框式的活动），发现MainActivity只有onPause调用了，onStop并没有调用（因为Dialog并没有完全遮挡MainActivity），按下back键，MainActivity的onResume调用。
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-ddde0b9f57d12d84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-ddde0b9f57d12d84.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-1be8fe28f84d8dbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-1be8fe28f84d8dbd.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png

同理，如果启动一个Activity采用了透明主题，那么也会只调用onPause而不会调用onStop
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-3e81ecf46caf577f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-3e81ecf46caf577f.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
按下back键后，MainActivity不会调用onRestart、onStart方法，直接调用onResume
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-4e875ba6e5047470.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-4e875ba6e5047470.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png

最后back键退出MainActivity，onPause、onStop、onDestroy依次调用
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-903fe3056dc8a5fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-903fe3056dc8a5fd.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png

onSaveInstanceState方法用于数据临时保存，在活动被系统回收之前一定会被调用，主动销毁活动不会调用这个方法。

当targetSdkVersion小于等于11（也就是2.3及其之前）时onSaveInstanceState是在onPause方法中调用的，而大于11（3.0开始）时是在onStop方法中调用的。
而onRestoreInstanceState是在onStart之后、onResume之前调用的

### 异常情况下的生命周期

- 资源相关的系统配置发生改变导致Activity被杀死并重新创建
    例如横竖屏切换：targetSdkVersion小于等于10时[![image.png](https://upload-images.jianshu.io/upload_images/4061843-c63dfd46457150c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-c63dfd46457150c5.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
    targetSdkVersion大于10时
    [![image.png](https://upload-images.jianshu.io/upload_images/4061843-1b761d4a24ebdf42.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-1b761d4a24ebdf42.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
    这两者的区别就是onSaveInstanceState和onPause调用的先后顺序不同，<=10的时候式onSaveInstanceState先于onPause被调用，>10的的时候式onPause先于onSaveInstanceState被调用

- 资源内存不足导致低优先级的Activity被杀死
    优先级：

    > (1) 前台Activity——正在和用户交互的Activity，优先级最高
    > (2) 可见但非前台Activity——比如Activity中弹出了一个对话框，导致Activity可见但是位于后台无法和用户交互
    > (3) 后台Activity——已经被暂停的Activity，比如执行了onStop，优先级最低

    当系统内存不足时，会按照上述优先级从低到高去杀死目标Activity所在的进程。我们在平常使用手机时，能经常感受到这一现象。这种情况下数组存储和恢复过程和上述情况一致，生命周期情况也一样

    对应的三种运行状态：

    > (1)Resumed（活动状态） 又叫Running状态，这个Activity正在屏幕上显示，并且有用户焦点。这个很好理解，就是用户正在操作的那个界面
    >
    > ②Paused（暂停状态） 这是一个比较不常见的状态。这个Activity在屏幕上是可见的，但是并不是在屏幕最前端的那个Activity。比如有另一个非全屏或者透明的Activity是Resumed状态，没有完全遮盖这个Activity
    >
    > ③Stopped（停止状态） 当Activity完全不可见时，此时Activity还在后台运行，仍然在内存中保留Activity的状态，并不是完全销毁。这个也很好理解，当跳转的另外一个界面，之前的界面还在后台，按回退按钮还会恢复原来的状态，大部分软件在打开的时候，直接按Home键，并不会关闭它，此时的Activity就是Stopped状态

可以通过在AndroidManifest文件的Activity中指定如下属性：`android:configChanges = "orientation| screenSize"`来避免横竖屏切换时，Activity的销毁和重建，而是回调了下面的方法：

```java
@Override
public void onConfigurationChanged(Configuration newConfig) {
    super.onConfigurationChanged(newConfig);
}
```

| configChanges的项目 |                            含义                            |
| :-----------------: | :--------------------------------------------------------: |
|       locale        |         设备的本地位置发生了改变，一般指切换了语言         |
|     orientation     |              屏幕方向发生了改变，比如旋转屏幕              |
|   keyboardHidden    |            键盘的访问性发生了改变，比如调出键盘            |
|         mcc         |       SIM卡唯一标识的国家代码，标识mcc代码发生了改变       |
|         mnc         |              SIM的运营商代码。标识mnc发生改变              |
|     touchscreen     |             触摸屏发生了改变，正常情况不会发生             |
|      keyboard       |           键盘类型发生了改变，比如使用了外接键盘           |
|     navigation      |       系统导航方式发生了改变，比如是否开启了夜间模式       |
|    screenLayout     |   屏幕布局发生了改变，很可能是用户激活了另外一个显示设备   |
|      fontScale      |       系统字体缩放比例发生了改变，比如选择一个新字号       |
|       uiMode        |          用户界面模式发生了改变，比如开启夜间模式          |
|     screenSize      | 屏幕的尺寸信息发生了改变，当旋转屏幕时，屏幕尺寸会发生变化 |
| smallestScreenSize  |                 设备的物理屏幕尺寸发生改变                 |
|   layoutDirection   |                      布局方向发生改变                      |
|                     |                                                            |

只有前三个经常用到，其他很少使用

## Activity的启动过程（源码分析）

## 相关面试题

1. Activity与Service通信的方式
    启动服务的时候可以通信
    bindService的时候，进行通信
    回调接口，在Service中实例回调，Activity中实现

2. 前台切换到后台，然后再回到前台，Activity生命周期回调方法。弹出Dialog，生命值周期回调方法。
    先是onPasuse，然后新活动的onCreate、onStart、onResume，再是旧的onSaveInstanceState用于保存数据，接着是onStop

    弹出Dialog，activity的生命周期不变，Dialog和Toast时通过Window添加的View，不会对Activity的生命周期有影响

3. Activity 上有 Dialog 的时候按 home 键时的生命周期
    onPause、onSaveInstanceState、onStop

4. 横竖屏切换的时候，Activity 各种情况下的生命周期
    当targetSdkVersion大于等于10时：
    onPase、onSavInstanceState、onStop、onDestroy、onCreate、onStart、onRestoreInstanceState、onResume

    当targetSdkVersion小于10时：
    onSavInstanceState、onPase、onStop、onDestroy、onCreate、onStart、onRestoreInstanceState、onResume

5. 四大组件
    Activity：一个界面、活动
    Service：服务
    Broadcast：你的应用可以使用它对外部事件进行过滤只对感兴趣的外部事件(如当电话呼入时，或者数据网络可用时)进行接收并做出响应
    ContentProvider：使一个应用程序的指定数据集提供给其他应用程序

6. 四大组件是在主线程运行的吗
    都是的

7. 怎么启动activity
    Intent隐式启动或显示启动
    隐式启动：

    ```java
    <activity android:name=".SecondActivity">
        <intent-filter>
            <action android:name="com.example.a14512.activitydemo.ACTION_START"/>
            <category android:name="android.intent.category.DEFAULT"/>
            <category android:name="com.example.a14512.activitydemo.MY_CATEGORY"/>
        </intent-filter>
    </activity>
    
    //启动活动，传入action
    Intent intent = new Intent("com.example.a14512.activitydemo.ACTION_START");
    //添加category
    intent.addCategory("com.example.a14512.activitydemo.MY_CATEGORY");
    startActivity(intent);
    ```

    显示启动：

    ```java
    Intent intent = new Intent(this, TargetActivity.class);
    intent.addFlags(Intent,FLAG_ACTIVITY_NEW_TASK);
    startActivity(intent);
    ```

8. 描述一下Activity栈
    Activity的管理是采用任务栈的形式，任务栈采用“后进先出”的栈结构

9. 保存Activity状态
    onSaveInstanceState、onRestoreInstanceState

10. 下拉状态栏是不是影响activity的生命周期，如果在onStop的时候做了网络请求，onResume的时候怎么恢复
    下拉状态栏不影响生命周期

11. 两个Activity 之间跳转时必然会执行的是哪几个方法？
    旧Activity的onPause，新Activity的onCreate、onStart、onResume，旧Activity的onStop、onSaveInstanceState

12. Activity之间的通信方式
    Intent、广播、EventBus、RxBus、otto、静态变量、全局变量、外部存储

13. Activity的启动模式，每种启动模式的使用场景，singletop中回调onNewIntent和finish掉之后onCreate()有什么不同？
    singleTop的模式下，如果Activity已经处于栈顶，那么接着再次启动这个Activity，就会调用onNewIntent来接受Intent实例，因为这个Activity没有发生改变，而不是重新新建一个Activity的实例放于任务栈中。如果不是栈顶，那么还是会新建要给实例，会调用onCreate

    而finish掉后的onCreate就是新建了一个Activity的实例，然后放于任务栈中

14. singleTask启动模式、Activity的启动模式以及使用场景
    栈内复用模式，是一种单实例模式，在一个Activity栈中只能存在一个实例，当启动一个Activity后，系统会首先查找是由存在A想要的任务栈，如果不存在，就要重新创建一个，然后在把这个Activity的实例放到任务栈中；如果已经存在这个Activity要的任务栈，就查看栈中是否有这个Activity的实例，如果已存在这个Activity的实例，当再一次启动的时候，如果上面有其他的Activity，就会销毁该Activity上的所有Activity，最终让这个Activity处于栈顶，然后调用onNewIntent函数

15. Activity的生命周期和缓存
    正常的生命周期：onCreate、onStart、onResume、onPause、onStop、onDestroy（可能会出现的onRestart）

    缓存：Activity 由于异常终止时，系统会调用 onSaveInstanceState()来保存 Activity 状态(onStop()之前和onPause()没有既定的时序关系)。当重建时，会调用 onRestoreInstanceState()，并且把 Activity 销毁时 onSaveInstanceState()方法所保存的 Bundle 对象参数同时传递给 onSaveInstanceState()和onCreate()方法。因此，可通过 onRestoreInstanceState()方法来恢复 Activity 的状态，该方法的调用时机是在 onStart()之后。onCreate()和 onRestoreInstanceState()的区别：onRestoreInstanceState()回调则表明其中Bundle对象非空，不用加非空判断。onCreate()需要非空判断。建议使用onRestoreInstanceState().

16. 切换activity执行顺序
    旧Activity的onPause，新Activity的onCreate、onStart、onResume，旧Activity的onStop、onSaveInstanceState

17. 如何安全的退出一个已经开启多个activity的APP
    使用广播关闭所有的Activity或者通过Activity栈来管理所有的活动，在退出时安全结束所有的活动

18. Activity启动模式，intent匹配规则
    standard、singleTop、singleTask、singleInstance
    匹配规则：Intent指定的唯一action、多个category、多个data都要在intent-filter中匹配（找到字符串值相同的）才算匹配成功，如果匹配一个没有在intent-filter中声明的活动会直接报错

19. 如何判断一个activity（启动方式是singtonTop）是正常启动还是复用启动
    生命周期可以监测到，正常启动的Activity会经历onCreate、onStart、onResume方法，而复用启动的Activity不会onCreate、onStart

    还可以查看onNewIntent方法是否被调用，只有复用启动的情况下，onNewIntent方法会调用

20. Android里的Intent传递的数据有大小限制吗，如何解决？
    不仅有大小限制（1MB左右，不同机型不一样），而且还有类型限制，最好的方式就是通过Bundle这个容器来装数据（也不建议传递大量数据），然后进行数据传递

    数据持久化、ContentProvider、EventBus

21. Intent的使用方法，可以传递哪些数据类型
    显示或隐式启动一个活动，传递数据
    可以传递基本类型、String、Parcelable、Serializable及其数组类型，还有就是Bundle容器

22. 四种LaunchMode及其使用场景
    standard：绝大多数Activity。如果以这种方式启动的Activity被跨进程调用，在5.0之前新启动的Activity实例会放入发送Intent的Task的栈的顶部，尽管它们属于不同的程序，这似乎有点费解看起来也不是那么合理，所以在5.0之后，上述情景会创建一个新的Task，新启动的Activity就会放入刚创建的Task中，这样就合理的多了

    singleTop：在通知栏点击收到的通知，然后需要启动一个Activity，这个Activity就可以用singleTop，否则每次点击都会新建一个Activity。当然实际的开发过程中，测试妹纸没准给你提过这样的bug：某个场景下连续快速点击，启动了两个Activity。如果这个时候待启动的Activity使用 singleTop模式也是可以避免这个Bug的。同standard模式，如果是外部程序启动singleTop的Activity，在Android 5.0之前新创建的Activity会位于调用者的Task中，5.0及以后会放入新的Task中

    singleTask：大多数App的主页。对于大部分应用，当我们在主界面点击回退按钮的时候都是退出应用，那么当我们第一次进入主界面之后，主界面位于栈底，以后不管我们打开了多少个Activity，只要我们再次回到主界面，都应该使用将主界面Activity上所有的Activity移除的方式来让主界面Activity处于栈顶，而不是往栈顶新加一个主界面Activity的实例，通过这种方式能够保证退出应用时所有的Activity都能报销毁。在跨应用Intent传递时，如果系统中不存在singleTask Activity的实例，那么将创建一个新的Task，然后创建SingleTask Activity的实例，将其放入新的Task中

    singleInstance： 呼叫来电界面。这种模式的使用情况比较罕见，在Launcher中可能使用。或者你确定你需要使Activity只有一个实例。建议谨慎使用

23. Android中main方法入口在哪里
    ActivityThread里面，有一个main函数

24. activity的startActivity和context的startActivity区别
    最终调用的都是 Activity 类实现的 startActivity 方法
    activity的startActivity是直接调用的Activity的startActivity
    context的startActivity是Context的抽象方法，而Context的一个子类ContextWrapper简单实现了这个方法，但是需要实例化，接着是ContextWrapper的一个子类ContextThemeWrapper并没有实现这个方法，而Activity是继承ContextThemeWrapper的，所以自然而然startActivity方法最终回到了Activity中

## 相关参考