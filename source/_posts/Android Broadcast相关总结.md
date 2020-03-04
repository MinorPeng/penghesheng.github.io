---
title: Android Broadcast相关总结
tag: Android
category: Android

---

<meta name="referrer" content="no-referrer" />



# Broadcast相关总结

## Broadcast用法

## Broadcast其他注意事项

## Broadcast源码分析

## 相关面试题

1. 广播传输的数据是否有限制，是多少，为什么要限制？
    （1）广播是通过Intent携带需要传递的数据的
    （2）Intent是通过Binder机制实现的
    （3）Binder对数据大小有限制，不同room不一样，一般为1M

2. 广播的分类？
    标准广播：通过context. sendBroadcast或者context. sendBroadcastAsUser发送给当前系统中所有注册的接受者，也就是只要注册了就会接收到。应用在需要通知各个广播接收者的情况下使用，如开机启动

    有序广播：接收者按照优先级处理广播，并且前面处理广播的接受者可以中止广播的传递，一般通过context. sendOrderedBroadcast或者context.sendOrderedBroadcastAsUser，在需要有特定拦截的场景下使用，如黑名单短信、电话拦截

    粘性广播：可以发送给以后注册的接受者，意思是系统会将前面的粘性广播保存在AMS中，一旦注册了与以保存的粘性广播符合的广播，在注册结束后会立即收到广播，一般通过context. sendStickyBroadcast或context.sendStickyOrderedBroadcast来发送，从字面上看，可以看出来粘性广播也分为普通粘性广播和有序粘性广播

    本地广播：发出的广播只能在应用程序内部进行传递，广播接收器也只能接受来自本应用程序的广播

    全局广播：系统和广播，发出的广播可以被其他任何应用程序接收到，并且也可以接受到其他任何应用程序的广播

3. 广播的使用场景，使用方式
    广播是一种广泛运用的在应用程序之间传输信息的机制，主要用来监听系统或者应用发出的广播信息，然后根据广播信息作为相应的逻辑处理，也可以用来传输少量、频率低的数据。
    在实现开机启动服务和网络状态改变、电量变化、短信和来电时通过接收系统的广播让应用程序作出相应的处理。

    使用：

    ```java
    //在AndroidManifest中静态注册
    <receiver
        android:name=".MyBroadcastReceiver"
        android:enabled="true"
        android:exported="true">
        <intent-filter android:priority="100">
            <action android:name="com.example.hp.broadcasttest.MY_BROADCAST"/>
        </intent-filter>
    </receiver>
    
    //动态注册，在代码中注册
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    
        mIntentFilter = new IntentFilter();
        //添加广播想要监听的类型，监听网络状态是否发生变化
        mIntentFilter.addAction("android.net.conn.CONNECTIVITY_CHANGE");
        mNetworkChangeReceiver = new NetworkChangeReceiver();
        //注册广播
        registerReceiver(mNetworkChangeReceiver, mIntentFilter);
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        //取消注册广播接收器
        unregisterReceiver(mNetworkChangeReceiver);
    }
    
    //发送广播，同样通过Intent
    Intent intent = new Intent("com.example.hp.broadcasttest.MY_BROADCAST");
    //发送标准广播
    sendBroadcast(intent);
    
    //接收广播
    public class MyBroadcastReceiver extends BroadcastReceiver {
        public MyBroadcastReceiver() {
        }
    
        @Override
        public void onReceive(Context context, Intent intent) {
            // TODO: This method is called when the BroadcastReceiver is receiving
            // an Intent broadcast.
            Toast.makeText(context, "received", Toast.LENGTH_SHORT).show();
            //将这条广播截断
    //        abortBroadcast();
        }
    }
    ```

4. BroadcastReceiver，LocalBroadcastReceiver 区别
    广播接收者：

    ```java
    （1）用于应用间的传递消息
    （2）由于跨应用，存在安全问题
    ```

    本地广播接收者：

    ```java
    （1）广播数据在本应用范围内传播。  
    （2）不用担心别的应用伪造广播。  
    （3）比发送全局广播更高效、安全。  
    （4）无法使用静态注册  
    ```

5. 在manifest和代码中如何注册和使用BroadcastReceiver
    在AndroidManifest中静态注册，然后直接使用

    代码中，通过registerReceiver来注册

    注册发送后，在BroadcastReceiver（自定义一个接收器继承自BroadcastReceiver）的onReceive中接收广播并处理广播

6. 广播引起anr的时间限制
    前台广播：BROADCAST_FG_TIMEOUT = 10s
    后台广播：BROADCAST_BG_TIMEOUT = 60s

7. 广播是否可以请求网络
    不建议，网络请求一般都是耗时操作，而广播实在主线程运行的，耗时操作会导致线程阻塞，很容易导致ANR。可以在广播中使用子线程进行网络请求，但不建议，更建议在Service中进行。广播可以用于监听网络变化

8. 如何通过广播拦截和abort一条短信
    要拦截首先设置权限：android.permission.SEND_SMS，android.permission.RECEIVE_SMS，设置receiver的优先级为最高1000，acticion为android.provider.Telephony.SMS_RECEIVED。

    在4.4前，短信拦截都是通过动态注册高优先级BroadcastReceiver的方式进行拦截的，主要是用于跟竞品进行短信抢占。而现在ContenetObserver是并行通知的情况下，如果过滤逻辑不够快，依然有可能会被竞品抢先把短信先删除掉，导致拿到的最后一次短信是旧的短信。建议结合BroadcastReceiver和ContenetObserver进行拦截，BroadcastReceiver做内容校正和后备数据，以防拿到的最后一条短信是旧的时候，依然可以进行正常的拦截流程；

    4.4以上引入了default sms机制，我们可以在不成为default sms的前提下实现短信拦截，利用App Ops权限管理功能，

    但由于App Ops从4.3出现到4.4一直牌隐藏的状态，猜想google还在不断调整中；

    Write SMS/MMS的权限开关的存在跟default sms本身是一个矛盾，之所以出现Write SMS/MMS的权限开关，完全是因为App Ops出现在前，而defaultsms出现在后所致。

    6.0以上引入了新的动态权限管理，类似于iOS中的权限。6.0以前是在安装时一次性获取权限，6.0之后是在运行的时候选择。

    ```java
    //onReceive代码
    public class SmsReceiver extends BroadcastReceiver {
        // 当接收到短信时被触发
        @Override
        public void onReceive(Context context, Intent intent) {
            // 如果是接收到短信
            if (intent.getAction().equals("android.provider.Telephony.SMS_RECEIVED")) {
                // 取消广播（这行代码将会让系统收不到短信）
                abortBroadcast();
                StringBuilder sb = new StringBuilder();
                // 接收由SMS传过来的数据
                Bundle bundle = intent.getExtras();
                // 判断是否有数据
                if (bundle != null) {
                    // 通过pdus可以获得接收到的所有短信消息
                    Object[] pdus = (Object[]) bundle.get("pdus");
                    // 构建短信对象array,并依据收到的对象长度来创建array的大小
                    SmsMessage[] messages = new SmsMessage[pdus.length];
                    for (int i = 0; i < pdus.length; i++) {
                        messages[i] = SmsMessage.createFromPdu((byte[]) pdus[i]);
                    }
                    // 将送来的短信合并自定义信息于StringBuilder当中
                    for (SmsMessage message : messages) {
                        sb.append("短信来源:");
                        // 获得接收短信的电话号码
                        sb.append(message.getDisplayOriginatingAddress());
                        sb.append("\n------短信内容------\n");
                        // 获得短信的内容
                        sb.append(message.getDisplayMessageBody());
                    }
                }
                Toast.makeText(context, sb.toString(), 5000).show();
            }
        }
    }
    
    //AndroidManifest中
    <application
        android:icon="@drawable/icon"
        android:label="@string/app_name" >
    
        <receiver android:name=".SmsReceiver" >
    
            <intent-filter android:priority="800" >
    
                <action android:name="android.provider.Telephony.SMS_RECEIVED" />
            </intent-filter>
        </receiver>
    </application>
    <!-- 授予程序接收短信的权限 -->
    
    <uses-permission android:name="android.permission.RECEIVE_SMS" />
    ```

    通过ContentObserver监听短信数据库

    ```java
    private ContentObserver smsContentObserver =  new ContentObserver(new Handler()) {  
        @Override  
        public void onChange(boolean selfChange) {  
            super.onChange(true);  
            /*Cursor cursor = resolver.query( 
                    Uri.parse(SMS_INBOX_URI), 
                    new String[] { "_id", "address", "thread_id", "date", 
                            "protocol", "type", "body", "read" }, 
                    " address=? and read=?", new String[] {SENDER_ADDRESS, "0"}, 
                    "date desc");*/  
            //注释掉的是查未读状态的，但如果你的手机安装了第三放的短信软件时，他们有可能把状态改变了，你就查询不到数据
            Cursor cursor = resolver.query(  
                    Uri.parse(SMS_INBOX_URI),  
                    new String[] { "_id", "address", "thread_id", "date",  
                            "protocol", "type", "body", "read" },  
                    " address=?", new String[] {SENDER_ADDRESS},  
                    "date desc");  
    
            while(cursor.moveToNext()){  
                String address = cursor.getString(cursor.getColumnIndex("address"));  
                String body = cursor.getString(cursor.getColumnIndex("body"));  
                String id = cursor.getString(cursor.getColumnIndex("_id"));  
                resolver.delete(Uri.parse("content://sms/"+id), null, null);  
                Log.d("短信平台发来的短信---", address+":::::"+body);  
                break;  
            }  
        }  
    };
    ```