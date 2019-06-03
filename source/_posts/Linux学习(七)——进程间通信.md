---
title: Linux学习(七)——进程间通信
tag: Linux
date: 2018-06-11
---

# Linux学习(七)——进程间通信

### 进程间通信的目的
1、数据传输：一个进程需要将它的数据发送给另一个进程
2、共享数据：多个进程想要操作共享数据，一个进程对共享数据进行修改，别的进程应该立即看到
3、通知事件：一个进程需要向另一个或一组进程发送消息，通知=它们发生了某种事件（如进程终止时要通知父进程）
4、资源共享：多个进程间共享同样的资源，需要内核提供锁和同步机制
5、进程控制：有些进程希望完全控制另一个进程的执行（如debug进程），此时控制进程希望能够拦截另一个进程的所有陷入和异常，能够及时知道它的状态改变

![QQ截图20180613141119.png](https://upload-images.jianshu.io/upload_images/4061843-89b00a5c9c5aa834.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

### 进程间的通信机制
##### 信号（signal）
信号是对在软件层次上对中断机制的一种模拟，是比较复杂的通信方式，用于通知进程有某事件发生，一个进程收到一个信号的与处理器收到一个中断请求效果上可以说是一样的

1. 用来向进程通知异步事件的发生，最初用来处理错误，最基本的IPC机制
2. linux中大部分信号都有预先定义好的意义，但至少有两个信号SIGUSER1和SIGUSER2可以由应用程序定义
3. 每个信号都有默认的动作，典型的是终止进程，进程可以通过提供信号处理函数来代替默认动作
4. 信号的产生：
 a）某些终端键产生，如ctrl+c中断信号
 b）硬件产生异常信号，如除数为0、无效的存储访问
 c）kill()函数将信号发送给另一个进程或进程组
> 常见的信号：
> 1、SIGHUP：挂断终端信号，终端被切断时产生
> 2、SIGINT：来自键盘的中断信号ctrl+c
> 3、SIGQUIT：来自键盘的退出信号ctrl+\
> 4、SIGFPE：浮点异常信号（浮点运算溢出）
> 5、SIGKILL：该信号结束接收信号的进程
> 6、SIGALRM：进程的定时器到期时发送
> 7、SIGTERM：kill命令发出的信号
> 8、SIGCHLD：标识子进程停止或结束的信号
> 9、SIGSTOP：ctrl+z或调试程序的停止执行信号
> 10、SIGUSER1：用户自定义信号1
> 11、SIGUSER2：用户自定义信号2

- **进程对信号的处理方式：**
1. 忽略此信号
（1）大多数信号都可以这样处理
（2）SIGKILL和SIGSTOP不能被忽略
2. 缺省处理方式：
执行系统默认动作，默认终止该进程
3. 暂时搁置信号
4. 捕捉信号
（1）通知内核在某种信号发生时，调用一个用户函数
（2）在用户函数中，可执行用户希望对这种事件进行的处理
（3）如果捕捉到SIGCHLD，表示子进程已经终止，所以此信号可以调用waitpid以取得该子进程的进程ID以及终止状态

*注册信号处理函数：*
（1）signal()，只有两个参数，不支持信号传递信息，主要用于前32种非实时信号注册
```
#include <signal.h>

void (*signal(int signumber, void(*func))(int)))(int);

/*
signumber：所注册函数针对的信号
func：调用函数的函数指针，信号处理函数，func有一个整数参数，返回值void，可以是自定义也可以是SIG_IGN（忽略对信号signumber的处理）或SIG_DFL（对信号默认处理）
不能对SIGKILL和SIGSTOP设置信号处理函数
signal失败返回1
*/
```
（2）sigaction()，三个参数，支持信号传递信息，同样支持signal的功能
```
#include <signal.h>

int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);

/*
1、第一个参数signum为信号的值，可以是出SIGKILL和SIGSTOP外的任何一个特定有效的信号
2、act是指定流对特定信号的处理，可以为空（以缺省方式处理），指向的sigaction包含流对指定信号的处理、信号所传递的信息、处理过程屏蔽的信号
3、oldact用来保存返回的原来对相应信号的处理，可以为NULL
*/

//sigaction结构
struct sigaction {
     void(*sa_handler)(int);
    void(*sa_sigaction)(int, struct siginfo_t*, void *);
    sigset_t sa_mask;
    int sa_flags;
}
/**
* sa_handler：信号处理函数地址
* sa_sigacation：指向信号处理函数的指针
* sa_handler和sa_sigaction某些体系结构上被定义为共用体
* sa_mask：指定哪些信号应当被阻塞
* sa_flags：标志位
*/
```
> sa_flags中的标志位。各位可通过“或”运算串接
> 1、0表示默认选项；
> 2、SA_NOCLDSTOP: 如果参数signum为SIGCHLD,当子进程被中断时，并不通知父进程。
> 3、SA_NOCLDWAIT:当信号为SIGCHLD,时可避免子进程僵死。
> 4、SA_NODEFER= SA_NOMASK :当信号处理函数正在进行时，不堵塞对于信号处理函数自身信号功能。
> 5、SA_ONESHOT= SA_RESETHAND:  用户注册的信号处理函数只执行一次，随后该信号的处理函数被设为系统默认的处理函数。
> 6、SA_RESTART:是本来不能重新运行的系统调用自动重新运行。
> 7、SA_SIGINFO:如果设置了该标志，则信号处理函数由三参数的sa_sigaction指定而不是sa_handler指定。

*信号集及操作函数：*被定义为一种数据类型，表示多个信号，描述信号的集合
```
#include <signal.h>

//操作相关
int sigemptyset(sigset_t *set);  //清空set指定的信号集
int sigfillset(sigset_t *set);  //set指向的信号集包含所有信号
int sigaddset(sigset_t *set, int signum);  
int sigdelset(sigset_t *set, int signum);
int ismenmber(const sigset_t *set, int signum);

typedef struct {
      unsigned long_sig[_NSIG_WORDS];
} sigset_t

//阻塞相关
/**
* oset非空，oset返回进程当前的信号掩码
* set非空，how指定当前的信号掩码如何改变
* SIG_BLOCK：set所指向的信号集包含的要增加的被阻塞的信号
* SIG_UNBLOCK：set所指向的信号包含了要增加的不被阻塞的信号
* SIG_SETMASK：进程的新信号掩码为set所指的信号集
*/
int sigprocmask(int how, const sigset_t *set, sigset_t *oset);

/**
* 获取当前已递送到进程却被阻塞的所有信号
*/
int sigpending(sigset_t *set);

/**
* 用于在接收到某个信号执勤，临时用mask替换进程的信号掩码，暂停进程执行，直到收到信号
* 返回之前的信号掩码，系统调用始终返回-1
* 返回条件：信号发生且信号不是屏蔽，信号必须处理，返回在处理函数后
*/
int sigsuspend(const sigset_t *mask);
```

*信号发送函数：*
```
#include <sys.types.h>
#include <signal.h>

/**
* 向进程或进程组发送信号
* pid>0，pid的进程
* pid=0，同一个进程组的进程
* pid<-1，进程组ID为pid绝对值的所有进程
* pid=-1，将信号广播传送给系统内所有有权限发的进程
*/
int kill(pid_t pid, int signo);

/**
* 用于向进程自身发送信号，失败-1，成功0
*/
int raised(int signo);

```

- **信号优缺点：**
> 1、信号数量有限，传递的信息比较粗糙，只是一个整数
> 2、系统开销大，要管理堆栈
> 3、由于传递的信息量少，便于管理和使用
> 4、经常用于系统管理相关

---

##### 管道（pipe）/有名管道（named pipe，FIFO）
管道可用于具有亲缘关系进程间的通信
有名管道除了具有管道的的功能还允许无亲缘关系进程间通信

将一个进程的标准输出和另一个进程的标准输入连到一起，以供两个进程互相通信
> 1、管道是单向的、先入先出的、无结构的、固定大小的数据流，
> 2、从管道读出后，数据移走，其他进程无法再读取
> 3、进程试图读空管道时，在有数据写入管道前，进程将一直阻塞，同样管道已满时，写进程也阻塞
> 4、pipe函数生成管道，每个管道可以有多个读进程和写进程
> 5、两个进程都终结时，管道自动消失
> 6、将一个程序的输出作为另一个的输入，管道符 | 

*局限性*
> 1、不能用来对多个接受者广播数据
> 2、无法识别信息的边界
> 3、一个管道有多个读进程，那么写进程不能发送数据到指定的读进程
> 4、一个管道有多个写进程，那么无法判断那个写进程发送了数据

- **命名管道（FIFOs，named PIPE）**
> 1、由于fork机制，管道只能用于父子进程之间，或者拥有相同祖先的两个子进程之间
> 2、FIFOs(First in, First out)为一种特殊的文件类型，有对应的路径
> 3、当一个进程读打开，另一个写打开，内核就建立管道
> 4、FIFOs是已经存在的对象，非命名管道是一个临时文件
> 5、多个写进程时，通pipe一样，可能发生写交错现象
> 6、删除FIFO文件，管道消失
> 7、可以通过文件的路径来识别管道，从而让没有亲缘关系的进程之间建立连接  
```
#include <sys/types.h>
#include <sys/stat.h>

/**
* pathname：FIFO路径名+文件名
* mode：权限描述符，与open中的相同
*/
int mkfifo(const char *pathname, mode_t mode);
int mknod(char *pathname, mode_t mode, dev_t dev);
```

- **匿名管道（单向管道）**
>1、半双工，数据在同一时刻只能在一个方向上流动
>2、数据只能从管道的一端写入，从另一端写出
>3、写入管道的数据遵循先入先出
>4、传送的数据是无格式的，所以输入输出端必须先定义好数据的格式
>5、管道不是普通的文件，不属于某个文件系统，只存在内存中
>6、读数据是一次性操作，读取后，从管道抛弃
>7、没有名字，只能在具有公共祖先的进程之间使用

*pipe创建管道：*
```
#include <unistd.h>
/**
* 1、创建的管道包含两个文件描述符fdes[0]和fdes[1]
* 2、管道两端的任务固定，fdes[0]只能用于读，fdes[1]只能用于写
* 3、使用这个文件描述符必须使用read和write（系统I/O）
* 4、如果对管道两端进行相反的工作会导致错误发生
* 5、管道不允许文件定位
*/
int pipe(int fdes[2]);
```

*管道读取数据：*
>如果管道的写端不存在
>1、则任务数据读到末尾，读出字节数返回为0
>如果管道的写端存在
>1、如果请求的字节数目大于PIPE_BUF（一般为4096），则返回管道中现有的数据字节数
>2、如果请求的字节数不大于，则返回管道中现有的字节数（小于请求字节数）或者返回请求的字节数（大于请求字节数）

*管道写入数据：*
>1、linux不保证写入的原子性，管道缓冲区一有空闲区域，写进程就会试图向管道写入数据
>2、当向读描述符关闭的管道中写数据时，会产生SIGPIPE信号，write返回-1
>3、缓冲区已满，写操作一直阻塞
>4、编程时可通过fcntl函数设置文件的阻塞特性（阻塞：fcntl(fd, F_SETFL, 0);  非阻塞：fcntl(fd, F_SETFL, O_NONBLOCK);）

*创建管道的简单方法：*
```
#include <stdio.h>

/**
* popen内部调用fork和exec执行命令行，返回FILE结构指针
* type参数只能定义成只读或只写，不同同时，第一个为准
* r开头则标准输出，w开头则标准输入
*/
FILE *popen(const char *cmdstring, const char *tyoe);
int pclose(FILE *fp);
```

---

##### System V IPC机制
> 1、使用相同的认证方法
> 2、进程通过系统调用向内核传递一个唯一的引用标识符才能访问资源
> 3、进程访问这些IPC资源先要经过权限检查
> 4、IPC对象包含一个ipc_perm结构

*基本概念：*每一个IPC有两个唯一的标志相连
1. 标识符
（1）一个非负整数，进程通过标识符访问IPC对象，IPC对象的内部名
（2）局限在IPC对象的类别里，不同类别的IPC可能有相同的标识符
2. 关键字key
（1）IPC对象的外部名，key由内核变换成标识符
（2）创建一个IPC对象时，必须指定一个key
（3）key可以是IPC_PRIVATE（0），表示总是创建一个新的IPC资源，创建进程私有，子进程共享
（4）key也可以是其他整数，最好用ftok函数得到
```
#include <sys/ipc.h>

/**
* path：文件路径名
* id：只有低8位有效，0-255，同样的id返回同样的key
* 成功返回key，失败-1。返回值根据文件inod确定
* ey_t值算法：将文件的索引节点号取出，前面加上id号得到key_t的返回值。
* 如指定文件的索引节点号为65538，换算成16进制为 0x010002，而你指定的ID值为38，换算成16进制为0x26，则最后的key_t返回值为0x26010002。
*/
key_t ftok(const char *path, int id);
```
ipc_perm结构：
```
#include <sys/ipc.h>

struct ipc_perm
{
  key_t key;   //关键字
  uid_t uid;  //所有者的有效用户ID
  gid_t gid;  //所有者所属组的有效组ID
  uid_t cuid;  //创建者的有效用户ID
  gid_t cgid;  //创建者所属组的有效组ID
  mode_t mode;  //访问权限
  unsigned short seq;  //应用序列号，不确定的值，每次被使用值增加，到最大值后从0开始
```

- **消息队列**
>1、消息的链表，信息是一个数据结构，包括1个32位类型值，其余为数据区
>2、克服流前两种的信息量有限的缺点，具有写权限的进程可以按照一定的顺序向消息队列中添加新消息，读权限的进程可以读取消息
>3、每个消息可以带有一个整数识别符（message_type），对消息分类、查询
>4、允许多个进程放入消息，也允许多个进程取出消息
>5、可以按照先进先出取出，也可以只取某个识别符的消息
>6、操作包含创建、打开队列、添加消息、读取消息、控制消息队列

```
strcut msqid_ds
{
  struct ipc_perm msg_perm;   //ipc_perm
  struct msg *msg_first;  //队列中第一个消息指针unused
  struct msg *msg_last;  //队列中最后一个消息指针unused
  _keynel_time_t msg_stime;  //最后发送消息的时间
  _keynel_time_t msg_rtime;  //最后接收消息的时间
  _keynel_time_t msg_ctime;  //最后队列的改变时间
  unsigned short msg_cbytes;  //当前队列中字节总数
  unsigned short msg_qnum;   //当前队列中消息个数
  unsigned short msg_qbytes;  //队列的最大字节数
  _kernel_ipc_pid_t msg_lspid;  //最后发送消息的进程pid
  _kernel_ipc_pid_t msg_lrpid;  //最后接收消息的进程pid
}
```

*操作函数：*
```
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

/**
* 创建或打开一个消息队列
* key：
* （1）0(IPC_PRIVATE)：建立新的消息队列（用于父子进程通信）
* （2）大于0的32位整数：视参数flag来确定操作。通常要求此值来源于ftok返回的IPC键值
* flag：
* （1）0：取消息队列标识符，若不存在则函数会报错
IPC_CREAT(未设置IPC_EXCL)：如果内核中不存在键值与key相等的消息队列，则新建一个消息队列；如果存在这样的消息队列，返回此消息队列的标识符
* （2）IPC_CREAT|IPC_EXCL：如内核中不存在键值与key相等的消息队列，则新建一个消息队列；如果存在这样的消息队列则报错
* （3）使用时模式参数要与IPC对象存取权限（如0600）进行|运算来确定消息队列的存取权限，如flag=IPC_CREAT|0666
*/
int msgget(key_t key, int flag);

/**
* 发送消息
* msqid：消息队列标识符
* nbytes：消息大小，不含消息类型的占4字节
* ptr：指向一个长正整数的指针，正整数后跟所传数据
* flag：0消息队列满时，msgsnd阻塞，IPC_NOWAIT队满时msgsnd不等带立即返回
* 成功返回0，失败-1
*/
int msgsnd(int msqid, const void *ptr, size_t nbytes, int flag);

/**
* 接收消息
* msqid：msgsnd类似
* type：指定要接收哪条消息，不按FIFO
* （1）=0，返回第一条
* （2）>0，返回队列中类型域=该值的第一条信息
* （3）<0，返回队列中类型域<=该值绝对值的消息中，类型域最小的第一个消息
* flag：
* （1）PC_NOWAIT被设置：在指定的type无效的情况下，msgrcv将不等待而直接带错返回。否则msgrcv将被阻塞，直到所希望的信息已放置在队列中。
* （2）IPC_NOERROR：若发送的消息大于size字节，则msgrcv把该消息截断，截断部分将被丢弃，且不通知发送进程。
*/
int msgrcv(int msqid, void *ptr, size_t nbytes, int flag);

/**
* 控制消息队列
* msqid：
* cmd：
* （1）IPC_STAT:获得msgid指定的消息队列的控制数据结构msqid_ds到buf中
* （2）IPC_SET：设置消息队列的属性，要设置的属性需先存储在buf中，可设置的属性包括：msg_perm.uid、msg_perm.gid、msg_perm.mode以及msg_qbytes
* （3）IPC_RMID:删除msqid指定的消息队列及相连的数据结构
* （4）IPC_STAT 选项需具有队列的读权限，IPC_SET和IPC_RMID选项需要队列的创建者，所有者或特权进程
*/
int msgctl(int msgif, int cmd, strcut msqid_ds *buf);
```

- **信号量（semaphore）**
> 1、主要作为进程间及同一进程的不同线程间的同步和互斥手段，为了控制进程对资源的使用
> 2、具有整数值的对象，可表示当前可用的某种资源数
> 3、支持两种原子操作（最小操作，不可分割）P和V，P操作减小信号量的值（请求资源），信号量<0阻塞。V操作增加信号量的值（释放资源），>0，唤醒一个等待进程，=0表示没有资源可以用
> 4、可实现临界区的概念，一个临界区是一段代码，任何时刻只能有一个进程执行它，访问临界资源的代码叫临界区，临界区本身也是临界资源
> 5、临界资源同一时刻只允许有限个（通常1个）进程可以访问或修改
```
struct semid_ds {
    struct ipc_perm sem_perm;
    struct sem *sem_base;   //信号数组（sem结构）指针
    ushort  sem_nsem;  //此集中信号个数
    time_t sem_otime;  //最后一次semop时间
    time_t sem_ctime;  //最后一次改变时间
};

struct sem {
    ushort_t semval;   //信号量的值
    short sempid;  //最后一个调用semop的进程ID
    ushort semncnt;  //等待该信号量值大于当前值的进程数
    ushort semzcnt;  //等待资源完全空闲的进程数
};
```
*信号量集操作函数：*
```
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>

/**
* 创建或打开一个信号量集
* 成功返回信号量的标识符，semop和semctl使用
* 参数nsem指定集合中的信号量数，若用于访问一个已经存在的集合，指定为0
* flag参数和消息队列类似
* 当一个新的信号集被创建时，相连的semid_ds结构被初始化
*/
int semget(key_t key, in nsems, int flag); 

/**
* 信号量操作
* 参数nops规定ops数组元素个数
* ops是一个指针，指向一个信号量操作数组，信号量操作由sembuf结构表示
*/
int semop(int semid, struct sembuf *sops, size_t nops);

struct sembuf {
    short sem_num;  //要处理的信号量在信号集中的序号
    short sem_op;   //信号量在一次操作中要改变的数据，通常是-1（P 操作）或+1（V操作），为0表示进程等待信号量变为0
    short sem_flag;  //通常为SEM_UNDO，使操作系统跟踪信号，并在进程没有释放该信号量而终止时，由操作系统释放信号量
};

/**
* 控制
* 第四个参数可选，取决于cmd
* semnum指定信号集中的哪个信号
* cmd指定以下10中命令，在semid指定的信号量集合上执行
* （1）PC_STAT   读取一个信号量集的数据结构semid_ds，并将其存储在semun中的buf参数中。
* （2）IPC_SET     设置信号量集的数据结构semid_ds中的元素ipc_perm，其值取自semun中的buf参数。
* （3）IPC_RMID  将信号量集从内存中删除。
* （4）GETALL      用于读取信号量集中的所有信号量的值。
* （5）GETNCNT  返回正在等待资源的进程数目。
* （6）GETPID      返回最后执行semop操作的进程PID。
* （7）GETVAL      返回信号量集中的一个单个信号量的值。
* （8）GETZCNT   返回在等待完全空闲的资源的进程数目。
* （9）SETALL       设置信号量集中的所有的信号量的值。
* （10）SETVAL      设置信号量集中的一个单独的信号量的值。
*/
int semctl(int semid, int semnum, int cmd, [union semun arg]);
```

- **共享内存**
> 1、最有用的进程间通信方式
> 2、使多个进程可以访问同一块内存空间，不同进程可以及时看到对方进程中对共享内存数据的更新
> 3、需要依靠同步机制（互斥锁、信号量）
```
struct shmid_ds
{
  struct ipc_perm shm_perm;  //操作权限
  int shm_segsz;  //段的大小
  time_t shm_atime;  //最后一个进程附加到该段的时间
  time_t shm_dtime;  //最后一个进程离开该段的时间
  time_t shm_ctime;  //最后一个进程修改该段的时间
  unsigned short shm_cpid;  //创建该段进程的pid
  unsigned short shm_lpid;  //该段上操作的最后一个进程pid
  short shm_nattach;  //当前附加导该段的进程的个数
};
```
*操作函数：*
```
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>

/**
* 打开或创建
* key_t key是共享内存的关键字
* size：内存大小
* flag：内存的模式|读写权限，至少需要IPC_CREAT
* （1）IPC_CREAT 新建（若已创建则返回目前id）
* （2）IPC_EXCL和IPC_CREAT结合使用，若已创建则返回错误
* 创建内存时，shmid_ds结构初始化，ipc_perm中的各项被设置为相应值，shm_lpid/shm_nattach/shm_atime 和shm_dtime被置为0，shm_ctime被置为当前时间
*/
int shmget(key_t key, size_t size, int flag);

/**
* 内存附加，允许本进程访问一块内存
* shmid：内存id
* addr：内存起始地址，若为NULL，内核自动查找进程地址空间，将共享内存附着在第一块有效内存区，此时flag无效。不为NULL，附加到addr指定位置，flag可为SHM_RDONLY（只读），否则为可读写
*/
void *shmat(int shmid, char *addr, int flag);

/**
* 内存分离但不删除
*/
int shmdt(void *addr);

/**
* 控制
* cmd：控制命令
* （1）IPC_STAT：得到共享内存状态
* （2）IPC_SET：改变共享内存状态
* （3）IPC_RMID：删除共享内存，调用此操作，不会再接受新的连接
* （4）SHM_LOCK：共享区上锁，需要超级用户
* （5）SHM_UNLOCK：解锁，超级用户
* buf：结构体指针
* 成功0，失败-1
*/
int shmctl(int shmid, int cmd, shmid_ds *buf);
```

---
##### 套接字（socket）
可用于网络中的不同机器之间的进程间通信