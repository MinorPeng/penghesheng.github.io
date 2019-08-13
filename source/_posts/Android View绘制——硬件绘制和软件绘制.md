---
title: Android View绘制——硬件绘制和软件绘制
date: 2019-08-10
tag: Android
---

<meta name="referrer" content="no-referrer" />



# Android View绘制——硬件绘制和软件绘制



HardWareRenderer是硬件加速绘制的入口，实现是一个ThreadedRenderer对象，跟一个Render线程息息相关，不过ThreadedRenderer是在UI线程中创建的，作用如下：

- 1. 在UI线程中完成DrawOp集的构建
- 1. 负责跟渲染线程通信

ThreadedRenderer RenderProxy ——> RenderThread 单例线程，不会出现多线程并发访问冲突的问题 ——> ThreadedRenderer的draw函数 ——> updateRootDisplayList构建RootDisplayList，构建View的DrawOp树 ——> 递归完成DrawOp树的构建



软件绘制同硬件合成的区别主要是在绘制上，内存分配、合成等整体流程是一样的，只不过硬件加速相比软件绘制算法更加合理，同时减轻了主线程的负担。



1. 构建阶段 ，递归遍历所有视图，将需要的操作缓存下来，之后再交给单独的Render线程利用OpenGL渲染。
2. 绘制阶段 ，View视图被抽象成一个个DrawOp(DisplayListOp)，比如View中drawLine，构建中就会被抽象成一个DrawLintOp，每个DrawOp有对应的OpenGL绘制命令，同时内部也握着绘图所需要的数据。



![](https://img-blog.csdnimg.cn/20190312232704592.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpYW41MjBhbw==,size_16,color_FFFFFF,t_70)

[Android 屏幕绘制机制及硬件加速](<https://blog.csdn.net/qian520ao/article/details/81144167>)