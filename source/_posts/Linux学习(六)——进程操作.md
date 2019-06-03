---
title: Linux学习(六)——进程操作
tag: Linux
date: 2018-05-14
---

# Linux学习(六)——进程操作

## 什么是进程
> 程序：程序是静态的，它是一些保存在磁盘上的指令的有序集合，是可执行文件，但没有任何执行的概念
> 进程：一个程序的一次执行的过程，是一个动态的概念，是程序执行时的一个实例
---
## 进程的状态
（是程序执行的过程）
* 执行态：程序在运行，占用CPU
* 就绪态：具备执行条件，等待分配CPU的时间片
* 等待态：等待除CPU外的其他资源或条件（不能运行）
---
## Linux中进程结构
* 代码段
* 数据段
* 堆栈段

---

## 进程控制块（PCB，又称进程描述符）
（进程是Linux系统的调度和管理资源的基本单位，通过进程控制块来描述的）
> 控制块：包含进程的描述信息、控制信息、资源信息
>
> 进程的静态描述，每一项是一个task_struct结构，系统进程数受task_struct数组大小限制，一般为512，新线程先分配task_struct结构并加入task数组

#### task_struct：

1.*进程状态（State）*：除三态外还有stopped（挂起状态，被暂停）、zombie（僵尸态，进程结束但未消亡）

2.*进程调度信息和策略*
- 用户模式
- 系统模式（内核模式）
进程通过系统调用或中断在两种模式间切换

 **时间片轮调度**：将所有的就绪任务按照First Come First Served原则排成队列，每次调度时将处理器分配给队首任务，让其执行一小段时间，一个时间片内任务未完成暂定送往队尾，将时间片给下一个队首

3.*进程标识号（Identifiers）*：系统分配一个唯一的数值给进程
 - 进程号（PID）和父进程号（PPID）
 - PID唯一的标识一个进程
 - PID和PPID都是非零证书
 - 系统调用getpid和getppid

4.*进程通信有关的信息（IPC）*：

5.*时间和定时器信息（Times and Timers）*

6.*文件系统信息（Files System）*

## 一些进程
1、孤儿进程：如果父进程先退出，子进程还没退出，那么子进程将被托孤给init进程（所有进程的父进程，PID为1）
2、僵尸进程：进程终止，但是父进程还未获取其状态。会消耗一定的资源
![image.png](https://upload-images.jianshu.io/upload_images/4061843-12ebfac4c99f5d74.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3、守护进程：在后台运行，不和任何终端关联，通常在系统启动时运行，命名以d结尾

## 进程管理
- 启动进程：手工启动和调度启动
- 相关命令：
```
ps:查看系统中的进程
jobs:显示当前shell下已启动的任务
kill:向进程发送信号
    kill -s 9 pid
    kill %num
fg:将后台进程转为前台进程
    fg %num
bg:将暂停的进程放到后台运行
    bg %num
ctrl+z:将前台运行程序，转为后台暂停
```


## 进程控制

#### 进程创建
- fork()/vfork
```
#include <sys/types.h>
#include <unistd.h>

fun:pid_t fork(void);
```
返回：单调用双返回，子进程中返回为0，父进程中返回子进程ID，出错-1，可以判定子进程父进程
得到的子进程是父进程的一个复制品，fork后谁先执行不确定

vfork创建与fork基本相同，但是vfork并不完全复制父进程的数据段，而是和父进程共享数据段，vfork对于父子进程的执行次序有所限制，保证子线程先运行，直到exec或exit后父进程才可能被调用

- exec
提供在进程中启动另一个进程的方法，某进程一旦调用，正在执行的程序将会替换成新订单，只有进程号会被保留，进程还是原来的进程但是执行的程序被替换了，调用exec成功时无返回值，其后面的代码也不会执行。fork和exec启动一个新的指定的进程
```
fun: int execl(const char *pathname, const char *arg0, ..., NULL);

fun: int execlp(const char *filename, const char *arg0, ...,NULL);

fun: int execle(const char *pathname, const char *arg0, ...,NULL);

fun: int execv(const char *pathname, const char *arg0, ..., NULL);

fun: int execvp(const char *filename, const char *arg0, ..., NULL);

fun: int execve(const char *path, const char *argv[], const char *envp[]);
path：被执行应用程序的完整路径
argv：传给被执行应用程序的命令行参数
envp：传给被执行应用程序的环境变量
argv[0]必须是程序的可执行文件的名字
```
execve才是真正的系统调用
#### 进程结束
- exit/_exit
8种方式终止进程：
1、main返回
2、exit
3、_exit或_Exit
4、最后一个线程从启动例程返回
5、最后一个线程调用pthread_exit
6、abort异常终止
7、接到信号终止，异常终止
8、最后一个线程对取消请求做出响应
![image.png](https://upload-images.jianshu.io/upload_images/4061843-4e01f1d47dba7427.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
#include <stdlib.h>
void exit(int status)

#include<unistd.h>
void _exit(int status)

status:0表示正常退出
```
exit在_exit（直接使进程停止运行）上进行包装，生成终止状态字，可用wait或waitpid获取
调用exit后变为僵尸状态，保留PCB信息供wait收集

#### 进程等待
wait/waitpid
```
#include <sys/types.h>
#include <sys/wait.h>

fun:pid_t wait(int *status)
fun: pid_t wait(pid_t pid, int *status, int options);
```
立即阻塞自己，如果找到僵尸线程（此处是它的子进程），会销毁后返回。没有找到就会一直阻塞
（1）如果所有子进程还在运行，则阻塞
（2）有子进程结束，得到子进程的终止状态和进程号
（3）没有子进程，返回-1
![QQ截图20180521083757.png](https://upload-images.jianshu.io/upload_images/4061843-f4457494c2391c60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 程序中使用
system函数
```
#include <stdlib.h>

fun: int system(const char *cmdstring);
```
利用fork、exec、waitpid实现。若cdmstring为空指针，system成功时返回非零指针，失败时返回0，可用于测试system是否有效。cdmstring不为空，fork或waitpid出现错误返回-1。exec错误返回，表示shell无法执行这个命令行，返回与exit(127)的返回相同。都调用成功时返回shell的结束态

#### 进程标识号管理

**进程用户的标识管理**
- 用户标识：real user id、effective user id
- 组标识号：real group id、 effective group id
```
#include <sys/types.h>
#include <unistd.h>

fun: signed short getuid(void);   //进程的真实用户id
fun: signed short geteuid(void);  //取得目前进程有效的用户识别id
fun: signed short getgid(void):  //组识别符
fun: signed shor getegid(void);  //有效的组识别符
fun: int setuid(uid_t uid);  //设置实际进程的实际用户、有效用户标识号
fun: int setgid(gid_t gid);  //设置进程的实际组、有效组标识号
```
只有root用户才更改实际用户id

**进程标识号**
```
#include <sys/types.h>
#include <sys/unistd.h>

fun: pid_t getpid(void);  //获取当前进程的id
fun: pid_t getpgrp(void);  //获取当前进程所属的组识别码
fun: pid_t getppid(void);  //取得目标进程的父进程识别码
fun: int setpgrp(void);  //将当前进程的组识别码设为当前进程的进程识别码，可脱离控制终端
```