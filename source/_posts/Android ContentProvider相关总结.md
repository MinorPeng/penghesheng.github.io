---
title: Android ContentProvider相关总结
tag: Android、面试
---

# ContentProvider相关总结

## ContentProvider用法

## ContentProvider与其他数据库相关

## ContentProvider源码分析

## 相关面试题

1. Android系统为什么会设计ContentProvider，进程共享和线程安全问题
    （1）封装

    ​		对数据进行封装，提供统一的接口，使用者完全不必关心这些数据是在DB，XML、Preferences或者网络请求来的。当项目需求要改变数据来源时，使用我们的地方完全不需要修改。  

    （2）提供一种跨进程数据共享的方式

    ​		由系统来管理ContentProvider的创建、生命周期及访问的线程分配，简化我们在应用间共享数据（进程间通信）的方式。我们只管通过ContentResolver访问ContentProvider所提示的数据接口，而不需要担心它所在进程是启动还是未启动  

    （3）更好的数据访问权限管理

    ​		ContentProvider可以对开发的数据进行权限设置，不同的URI可以对应不同的权限，只有符合权限要求的组件才能访问到ContentProvider的具体操作。

    

    ContentProvider是一个APP间共享数据的接口。一个程序可以通过实现一个Content provider的抽象接口将自己的数据完全暴露出去，例如在A APP中实现建ContentProvider，并在Manifest中生命它的Uri和权限，在B APP中注册权限，并通过ContentResolver和Uri进行增删改查；
    ContentProvider也是通过Binder机制实现跨进程的通信，通过匿名共享内存的方式进行数据的传输，一个应用进程有16个Binder线程去和远程线程进行交互，每个线程可占用的缓存空间为128KB，超出会报异常

    ContentProvider的线程安全是跨进程的，不管Provider使用方是同一个进程的不同线程，还是不同的进程，Provider方实际上是同一个Provider对象实例，并发访问时，Provider方query()方法运行在不同的线程，实际上是运行在Provider方的进程的Binder线程池中。在AMS中，获取Provider相关的方法都有同步锁，所以这个Provider远程对象实际上是同一个

2. ContentProvider、ContentResolver与ContentObserver之间的关系是什么？
    ContentProvider：管理数据，提供数据的增删改查操作，数据源可以是数据库、文件、XML、网络等，ContentProvider提供了统一的接口，可以用来做进程间数据共享

    ContentResolver：可以不同的URI操作不同的ContentProvider中的数据，外部进程可以通过ContentResolver与ContentProvider进行交互，用于获取内容提供器提供的数据

    ContentObserver：监听、观察ContentProvider中的数据变化，并将变化通知给外界

3. 批量插入50条联系人，比较高效的方法，ContentProvider是否了解原理
    SQL语句采用“insert into tb (…) values(…),(…)…;”方式，，也就是不要重复去使用一个SQL语句，重复执行相同的操作，直接一次性添加多条数据进去

    [理解ContentProvider原理](http://gityuan.com/2016/07/30/content-provider/)

4. 请介绍下ContentProvider 是如何实现数据共享的
    ContentProvider 是应用程序之间共享数据的接口. 使用的时候首先自定义一个类继承 ContentProvider, 然后覆写 query、insert、update、delete 等方法. 因为其是四大组件之一因此必须在 AndroidManifest 文件中进行注册. 把自己的数据通过 uri 的形式共享出去

    第三方可以通过 ContentResolver 来访问该 Provider

5. ContentProvider的权限管理(读写分离，权限控制-精确到表级，URL控制)
    （1）Provider可以提供读权限android:readPermission，写权限android:writePermission，或者权限android:permission（读写都设置）

    我们可以针对其中某个或某部分URI，单独进行权限设置。除了android:pathPrefix，还可以有android:path和android:pathPatten，例如 android:pathPattern=”/hello/.*“（注意，通配符*之前有个‘.’）
    例如，我们可以只开发 content://com.robert.propermission.PrivProvider/hello路径下权限，不允许 访问其他路径，如下声明：

    ```xml
    <provider android:name=".PrivProvider"
        android:authorities="com.robert.propermission.PrivProvider"
        android:readPermission="com.robert.READ_CONTENTPROVIDER"
        android:exported="true" >
        <path-permission android:pathPrefix="/hello" android:readPermission="READ_HELLO_CONTENTPROVIDER" />
    </provider>
    ```

    Provider的granting
    全局granting
    ProPermissionClient具有读取 content provider的权限，它去调用另一个应用C的activity，例子中这个另一个应用C为ProPermissionGrant，但是这个例子没有读 取content provider的权限，ProPermissionClient可以将自己的权限通过intent传递给应用C，让其也具有访问content provider的权限。
    对于应用A，相关的content provider为：

    ```xml
    <provider android:name=".PrivProvider"
        android:authorities="com.robert.propermission.PrivProvider"
        android:readPermission="com.robert.READ_CONTENTPROVIDER"
        android:grantUriPermissions="true"
        android:exported="true" />
    ```

    对于应用B，其具有com.robert.permission.READ_CONTENTPROVIDER的权限，而应用C不具备，应用B通过intent调用应用C的代码如下：

    ```java
    Intent intent = new Intent(this,ReadProvider.class);
    intent.setClassName("com.robert.example.propermissiongrant", "com.robert.example.propermissiongrant.MainActivity");
    intent.setData(Uri.parse("content://com.robert.propermission.PrivProvider/world/1"));
    intent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);  //传递权限
    startActivity(intent);
    ```

    所调用的C的activity具备访问content provider的权限。
    如果我们将provider的属性android:grantUriPermissions设置为false，则不允许通过接受传递的权限方式进行访问，即B所调用的C的activity不能读content provider，就会报错
    部分URI的granting

    我们只希望部分的URI允许grant权限访问，而不是开放整个provider，如下：

    ```xml
    <provider android:name=".PrivProvider"
        android:authorities="com.robert.propermission.PrivProvider"
        android:readPermission="com.robert.permission.READ_CONTENTPROVIDER"
        android:exported="true" >
        <grant-uri-permission android:pathPrefix="/hello" />
    </provider>
    ```

    我们将之允许前缀为hello的部分URI访问。一旦 我们设置了grant-uri-permission，则全局的android:grantUriPermissions属性将无效，无论设置true还 是flase，也都是只允许grant-uri-permission是声明的部分uri可以被grant权限访问。

    > android:grantUriPermssions:临时许可标志。
    > android:permission:Provider读写权限。
    > android:readPermission:Provider的读权限。
    > android:writePermission:Provider的写权限。
    > android:enabled:标记允许系统启动Provider。
    > android:exported:标记允许其他应用程序使用这个Provider。
    > android:multiProcess:标记允许系统启动Provider相同的进程中调用客户端。
    >
    > \<path-permission>：只针对某一个路径开通权限，不允许访问其他路径
    > android:pathPattern：具体的路径
    > android:permission

    [ContentProvider权限设置](https://blog.csdn.net/robertcpp/article/details/51337891)
    [Content Provider的权限](https://www.cnblogs.com/622698abc/p/6033080.html)

## 相关参考