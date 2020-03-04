---
title: Android——Binder机制
tag: Android
category: Android
date: 2019-06-02
---

<meta name="referrer" content="no-referrer" />



# Android——Binder机制

## 介绍



### Linux下的IPC方式

- 管道：创建时会分配一个page大小的内存，缓存区大小比较有限
- 消息队列：信息复制两次，额外的CPU消耗，不适合频繁或信息量大的通信
- 共享内存：无需复制，共享缓存区直接附加到进程虚拟地址空间，速度快；但进程间的同步问题写操作系统不能控制，必须各进程使用同步工具
- 套接字socket：更通用的接口，传输效率低，主要用于不同机器或跨网络的通信
- 信号量：在Linux下常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问资源，可以作为进程间的同步手段
- 信号：不适用于信息交换，更适用于进程中断控制，如非法内存访问、杀死某个进程



>  Android是基于Linux开发的系统，所以很多Linux能用的IPC，Android都可以使用，但是Android开发了自己的一套IPC机制——Binder

### 为什么要用Binder

- 从性能角度，数据拷贝次数来说：

    Binder数据只需要拷贝一次，而管道、消息队列、Socket等都需要两次数据的拷贝，但共享内存一次都不需要，可以直接访问；从性能来说，Binder仅次于共享内存

- 从稳定性角度：

    Binder基于C/S架构（Client和Server，客户端有什么需求直接发送给服务端去完成，两端相对独立），而共享内存实现方式复杂，没有客户端与服务端之别，需要充分考虑到访问临界资源的并发同步问题，否则会出现死锁问题；从稳定性来说，Binder优于共享内存

- 从安全角度：

    Android由于开源，App来源广，容易数据泄露、后台耗电等问题，所以安全特别重要，传统Linux无任何保护措施，完全由上层协议来确保

    传统Linux IPC的接收方无法获取对方进程可靠的UID/PID，从而无法鉴别身份；

    Android为每个安装好的应用程序分配了自己的UID，因此进程的UID就是最好鉴别进程身份的标志；前面说到的C/S架构，Android系统中对外只暴露Client端，Client端将任务发送给Server端，Server端会根据权限控制策略，判断UID/PID是否满足访问权限（目前权限控制很多时候是通过弹出权限对话框，让用户选择，低版本没有）；Android 6.0之前，所有权限申请都是在安装的时候有一个权限清单，一些不需要的权限让用户无法拒绝，因为拒绝后App无法正常使用，在6.0（M）后，Google对此做了调整，安装时不是一并询问所有权限，而是在运行过程中，需要什么权限再由用户授权，对权限进行了细化，这样也导致了更多的弹窗；在后面的8.0、9.0中，对权限的控制更加严格

    传统IPC只能由用户在数据包里填入UID/PID；另外，可靠的身份标记只有由IPC机制本身在内核中添加。其次传统IPC访问接入点是开放的，无法建立私有通道。从安全角度，Binder的安全性更高。

- 从语言层面角度：

    Linux基于C语言，Android基于Java语言；Binder也符合面向对象的思想，将进程间通信转化为通过对某个Binder对象的引用调用该对象的方法，而其独特之处在于Binder对象是一个可以跨进程引用的对象，它的实体位于一个进程，它的引用可以位于多个进程。Binder模糊化了进程边界，淡化考虑进程间的通信过程。从语言层面来说，Binder属于面向对象，使用Android人员

    当然Binder更多用于system_server进程与上传APP层的IPC

> Binder是基于开源的 [OpenBinder](https://link.zhihu.com/?target=http%3A//www.angryredplanet.com/~hackbod/openbinder/docs/html/BinderIPCMechanism.html)实现的，OpenBinder是一个开源的系统IPC机制,最初是由 [Be Inc.](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Be_Inc.) 开发，接着由[Palm, Inc.](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Palm%2C_Inc.)公司负责开发，现在OpenBinder的作者在Google工作，既然作者在Google公司，在用户空间采用Binder 作为核心的IPC机制，再用Apache-2.0协议保护，自然而然是没什么问题，减少法律风险，以及对开发成本也大有裨益的，那么从公司战略角度，Binder也是不错的选择

## 参考博文

- [为什么 Android 要采用 Binder 作为 IPC 机制？](<https://www.zhihu.com/question/39440766>)