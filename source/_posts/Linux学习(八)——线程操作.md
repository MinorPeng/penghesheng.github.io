---
title: Linux学习(八)——线程操作
tag: Linux
category: Linux
date: 2018-06-25	

---

<meta name="referrer" content="no-referrer" />



# Linux学习(八)——线程操作

### 线程的基本概念
（1）线程是进程内独立的一条运行路线，处理器调度的最小单元，也称为迷你进程（轻量级进程）
（2）一个进程可以有多个线程，一个进程至少需要一个线程作为它的指令执行体
（3）多核心甚至多CPU的调度

**程序：**存放在磁盘文件中的可执行文件，使用6个exec函数中的一个由内核将程序读入存储器，并使其执行

**进程：**是资源管理的最小单位，是程序的执行实例，是动态过程

**线程：**是程序执行的最小单元，比进程更小的、能独立运行和调度的基本单元，以此来提高程序并行执行的程度，是CPU上被调度执行的实体

进程的所有信息对该进程中的所有线程共享![QQ截图20180625083227.png](https://upload-images.jianshu.io/upload_images/4061843-2feea0ba434c8aa4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)进程来自不同用户，彼此存在竞争关系，一个进程中的线程通常是彼此合作

**线程的状态：**运行、阻塞、就绪、终止，同进程状态一样

---

### 基本操作
- **线程实现：**
（1）用户态线程：把整个线程包放在用户空间中，内核对线程包一无所知，线程表在用户（会导致阻塞）
（2）内核态线程：线程由内核管理，在每一个时间片内，内核负责调度进程内的线程，线程表在内核（并发）

*线程标识：*线程ID，只有在它所属的进程中唯一
` typdef unsigned long int pthread_t`

- **操作函数：**
```
#include <pthread.h>

/**
* thread：创建的线程标识符
* attr：新线程的属性，默认NULL
* *(*start_routine) 线程执行函数
* arg：线程执行函数的输入参数，只有一个返回值
*/
int pthread_create(pthread_t *thread, pthread_attr_t *attr, void *(*start_routine) void *), void *arg);

/**
* 获取线程ID
*/
pthread_t pthread_self();

/**
* 结束线程
*/
void pthread_exit(void *retval);

/**
* 线程挂起
* th：线程标识符
* thread_return：存放其他线程的返回值
*/
int pthread_join(pthread_t th, void **thread_return);

//取消
/**
* 请求终止线程
* 只提出请求，线程继续运行，直到达到某个取消点
*/
int pthread_cancel(pthread_t thread);

/**
* 设置取消状态：state
* （1）PTHREAD_CANCEL_ENABLE(默认),收到信号后设为CANCLED态
* （2）PTHREAD_CANCEL_DISABLE，忽略CANCEL信号继续运行
*/
int pthread_setcancelstate(in state, int *oldstate);

/**
* 设置取消执行动机
* 仅当PTHREAD_CANCEL_ENABLE时有效
* 立即取消(PTHREAD_CANCEL_ASYNCHRONOUS)和
延迟取消至取消点(PTHREAD_CANCEL_DEFERRED，默认)
*/
int pthread_setcaceltype(int type, int *oldtype);

/**
* 设置取消点
*/
void pthread_testcacle();

/**
* 清理，入栈
* routine：子程序
* arg：子程序参数
*/
void pthread_cleanup_push(void (*routine)(void *), void *arg);

/**
* 出栈
* execute：或非0，0表示正常退出时不执行该弹出的程序，异常退出时(响应cancel等， 执行到pthread_cleanup_pop代码之前的退出)执行弹出程序
*/
void pthread_cleanup_pop(int execute);

```

![QQ截图20180630194401.png](https://upload-images.jianshu.io/upload_images/4061843-30b758415b460a70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)![QQ截图20180630194456.png](https://upload-images.jianshu.io/upload_images/4061843-6ed5e09b4c2b34ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **线程同步：**
1. 互斥锁：
锁定对共享资源的访问

*特点：*
（1）原子性：如果一个线程锁定量一个互斥锁，没有其他线程在同一时间可以成功锁定该互斥锁
（2）唯一性：如果一个线程锁定了一个互斥锁，在它解锁之前，没有其他线程可以锁定这个互斥锁
（3）非繁忙等待：如果一个线程锁定了一个互斥锁，第二个线程试图锁定这个互斥锁时，则第二个线程会被挂起，直到第一个线程解除这个互斥锁的锁定，第二个线程会被唤醒执行，同时锁定这个互斥锁

*互斥锁函数：*
```
#include <pthread.h>

//静态方式创建
pthread_mutex_t mutex = PTHREAD_MUTEX_INTIALIZER;
pthread_mutex_t mutex = PTHREAD_RECURSIVE_MUTEX_INITIALIZER_NP;
pthread_mutex_t mutex = PTHREAD_ERRORCHECK_MUTEX_INITIALIZER_NP;

/**
* 动态方式
* mutex：创建的互斥锁
* attr：互斥锁的属性，默认NULL
*/
pthread_mutex_t mutex;
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutex_attr *attr);

//互斥锁的锁定
int ptfhread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);

//解锁，必须处于加锁状态，且调用的线程必须是上锁的同一线程，多个线程同时解锁时由有核心调度程序决定
int pthread_mutex_unlock(pthread_mutex_t *mutex);

//清除互斥锁
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```
![QQ截图20180630205443.png](https://upload-images.jianshu.io/upload_images/4061843-5a855112174d5f5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 条件变量

*特点：*
（1）对互斥锁的补充，允许线程阻塞并等待另一个线程发送的信号
（2）与互斥锁一起使用时，允许线程以无竞争的方式等待特定的条件发生
（3）条件本身是由互斥锁保护的，线程改变条件状态之前必须首先锁住互斥锁。其他线程在获得互斥锁之前不会察觉这种改变，必须锁定后才能计算条件

*函数：*
```
#include <pthread.h>

/**
* 初始化条件变量
* cond：条件变量的标识
* attr：条件变量的属性，默认NULL
*/
int pthread_cond_init(pthread_cond_t *cond, const pthread_cond_attr *attr);

/**
* 条件变量的清除
* 清除任何条件变量的状态，但内存空间不释放
*/
int pthread_cond_destory(pthread_cond_t *cond);

//对互斥锁与线程的操作
/**
* 释放由参数mutex指向的互斥锁，并且使调用线程关于参数cond指向的条件变量阻塞。
* 被阻塞的线程可以被pthread_cond_signal/pthread_cond_broadcast唤醒
*/
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);

/**
* cond：标识
* mutex：互斥锁标识
* abstime：延迟时间
* pthread_cond_timedwait()函数在定时时刻*abstime之前未被条件变量唤醒，则可自动唤醒，获取互斥锁。如果超过*abstime指定的时间，则返回ETIMEOUT
*/
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex, const struct timespec *abstime); 

//唤醒
int pthread_cond_signal(pthread_cond_t *cond);  //至少唤醒一个
int pthread_cond_broadcast(pthread_cond_t *cond);  //可以唤醒所有的等待该条件的线程

//获取系统时间函数，tv秒数(1970-01-01 零时开始至当前的时间)，tz返回时区参数
gettimeofday(struct timeval *tv, struct timezone *tz);

//秒数转化为可理解的时间
struct tm *localtime(time_t *timep);
struct tm{
    int tm_sec;  //目前秒数, 正常范围0-59, 但允许至61 
    int tm_min;  //分数, 范围0-59
    int tm_hour;  //时数, 范围为0-23
    int tm_mday;  //目前月份的日数, 范围01-31
    int tm_mon;  //代表目前月份, 范围从0-11
    int tm_year;  //从1900 年算起至今的年数
    int tm_wday;  //一星期的日数, 从星期一算起, 范围0-6
    int tm_yday;  //从今年1 月1 日算起的天数, 范围0-365
    int tm_isdst;  //日光节约时间的旗标
};
```

3. 信号量
与进程通信使用的一样，特殊的变量，原子操作。通过一个计数器控制对共享资源的访问，信号量的值是一个非负整数，所有通过它的线程都会将值-1。如果计数器大于0，则允许访问，计数器-1，如果计数器为0，则禁止访问，所有通过它的线程处于等待状态

*分类：*
（1）二进制信号量：只允许信号量取0或1，同时只能被一个线程获取
（2）整型信号量：取值是整数，可以被多个线程同时获取，直到信号量的值为0

*操作函数：*
```
#include <semaphore.h>

/**
* 创建
* sem：信号量标识，长整型
* pshared：控制信号量的类型，如果该值设为0，就表示这个信号量是当前进程的局部信号量，只能为当前进程的所有线程共享。不为0表示信号量可以在多个进程之间共享。
* value：初始值
*/
int sem_init(sem_t *sem, int pshared, unsigned int value);

/**
* 清除
*/
int sem_destory(sem_t *sem);

/**
* 控制
* em的值大于0，则sem_wait()函数是以原子操作的方式将信号量sem的值减1。如果sem的值为0，则线程阻塞，直到获取到sem值大于1时才开始执行(P操作)
* sem_post()将信号量sem的值加1，如果此时有正在等待的线程，则唤醒该线程。（V操作）
*/
int sem_wait(sem_t *sem);
int sem_post(sem_t *sem);
 
```

---

### 特殊操作

*特定数据的创建和取消：*
```
#include <pthread.h>

/**
* 创建一个对进程中所有线程都可见的关键字
* 函数可以与析构函数dest_routine关联。当线程退出时，那么析构函数会被调用。
* 线程通常使用malloc为线程特定数据分配内存，析构函数通常需要释放已分配的内存。第二个参数析构函数可用NULL，即采用系统默认的销毁函数
*/
int pthread_key_create(pthread_key_t *key, void(*dest_routine(void *)));

/** 
* 清除指定关键字
*/
int pthread_key_delete(pthread_key_t key);
```

*特定数据的读取和设置：*
```
/**
* 指定由参数pointer指定的指针指向由参数key指定的关键字
*/
int pthread_setspecific(pthread_key_t key, const void *pointer);

/**
* 获取由pthread_setspecific()设置的关键字指针
*/
void * pthread_getspecific(pthread_key_t key);


void *pthread_one_t one_control= PTHERAD_ONCE_INIT;
/**
* 保证某些初始化代码至多只能执行一次
* once_control指向静态的或外部的变量
*/
int pthread_once(pthread_once_t *once_control, void (*init_routine)(void));
```

*线程属性：*
（1）每个POSIX线程有一个相连的属性对象表示其特性
（2）线程属性对象可以与多个线程相连，可根据属性对线程分组。属性对象特性改变时，组中所有线程实体具有新特性
```
//创建和销毁
int pthread_attr_init(pthread_attr_t *attr);
int pthread_attr_destory(pthread_attr_t *attr);

//拆卸状态
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);
int pthread_attr_getdetachstate(const pthread_attr_t *attr, int *detachstate);

//作用域函数
int pthread_attr_setscope(pthread_attr_t *attr, int scope);
int pthread_attr_getscope(phtread_attr_t *attr, int *scope);

//堆栈相关函数
int pthread_attr_getstack(const pthread_attr_t *attr, void **stackaddt, size_t *stacksize);
int pthread_attr_setstack(pthread_attr_t *attr, void *stackaddr, size_t *stacksize);
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);
int pthread_attr_getstacksize(const pthread_attr_t *attr, size_t *stacksize);
int pthread_attr_setstackaddr(pthread_attr_t *attr, void *stack_addr);
int pthread_attr_getstackaddr(const pthread_attr_t *attr, void **stackaddr);

//调度相关函数
/*继承调读*/
int pthread_attr_setinheristched(pthread_attr_t *attr, int inherit);
int pthread_attr_getinheritsc(const pthread_attr_t *attr, int *inherit);
/*调度策略*/
int pthread_attr_setschedpolicy(pthread_attr_t *attr, int policy);
int pthread_attr_getschedpolicy(const pthread_attr_t *attr, int *policy);
/*调度参数*/
int phtread_attr_setschedparam(pthread_attr_t *attr, const struct sched_param *param);
int pthread_attr_getschedparam(const pthread_attr_t *attr, struct sched_param *param);
```