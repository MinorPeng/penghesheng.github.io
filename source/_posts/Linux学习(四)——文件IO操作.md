---
title: Linux学习(四)——文件IO操作
tag: Linux
category: Linux
date: 2018-04-23

---

<meta name="referrer" content="no-referrer" />



# Linux学习(四)——文件IO操作

> 标准I/O 与系统I/O的区别：
>
> 标准I/O建立在系统I/O上，有缓冲，比较耗时，可以实现更多的功能，对内核进行较复杂的操作，使用更人性；
>
> 系统I/O是操作系统为用户态运行的进程和硬件设备(如CPU、磁盘、打印机等)进行交互提供的一组接口，即就是设置在应用程序和硬件设备之间的一个接口层，可以说是操作系统留给用户程序的一个接口。调用系统函数，没有缓冲，不耗时，C 和汇编语言实现，执行效率高
>
> 系统调用是为了更直接的使用操作系统接口，而库函数则是为了人们编程的方便。

> 文件操作的一般过程：
> 打开->读/写-[定位]-关闭

---

- open函数
```
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

fun: int open(const char* pathname, int flags, .../*mode*/);

O_RDONLY：只读形式
O_WRONLY：只写形式
O_RDWR：读写形式
O_APPEND：追加模式
O_TRUNC：若成功打开，则长度截为0
O_ CREAT：文件不存在则创建
```

- close
```
#include <unistd.h>

fun: int close(int fd);
```

- read
```
#include <unistd.h>

fun: ssize_t read(int fd, void *buf, size_t count);
size_t: unsigned int
```

- write
```
#include <unsitd.h>

fun: ssize_t write(int fd, const void *buf, size_t count);
*buf：输出缓冲区地址
将buf缓冲区count字节写入文件
```

- creat
```
#include <sys/type.h>
#include <sys/stat.h>
#include <fcntl.h>

fun: int creat(const char *pathname, mode_t mode);
```

- lseek：自定义读取文件，指定文件偏移量的位置
```
#include <sys/types.h>
#include <unistd.h>

fun: off_t lseek(int fds, off_t offset, int whence);

fds:文件描述符
offset：偏移量
whence：当前位置的基点
1.SEEK_SET：文件的开头
2.SEEK_CUR：文件指针的位置
3.SEEK_END：文件的末尾
计算文件长度：Iseek(fd, 0, SEEK_END)
```

---
## 文件高级操作
- 文件模式的高7位
```
S_IFMT        0170000     文件类型的位遮罩  
S_IFSOCK    0140000     socket  
S_IFLNK       0120000     符号链接(symbolic link)  
S_IFREG       0100000     一般文件  
S_IFBLK       0060000     区块装置(block device)  
S_IFDIR       0040000     目录  
S_IFCHR      0020000     字符装置(character device)  
S_IFIFO        0010000     先进先出(fifo)  

S_ISUID       0004000     文件的(set user-id on execution)位  
S_ISGID       0002000     文件的(set group-id on execution)位  
S_ISVTX      0001000     文件的sticky位  

用st_mode&S_IFMT检测文件类型
```

- umask
```
fun: mode_t umask(mode_t cmask)

cmask==1: 文件模式的相应位的权限被禁止操作，新的屏蔽码
函数返回旧屏蔽码
```

- chmod fchmod
```
fun：int chmod(const *char filename, mode_t mode)
fun：int fchmod(int fd, mode_t mode)

fd: 文件描述符
```

![QQ截图20180502151447.png](https://upload-images.jianshu.io/upload_images/4061843-938741ebbaaed9b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- chown fchown：修改权限
```
fun: int chown(const *char path, uid_t owner, gid_t group)
fun: int fchown(int fd, uid_t owner, gid_t group)
```
![QQ截图20180502151808.png](https://upload-images.jianshu.io/upload_images/4061843-1859b6c3d6fe3098.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- rename：重命名文件
```
#include <stdio.h>

fun: int rename(char* oldname, char* newname)

可用于移动或复制文件
```

- truncate ftruncate：对文件的大小进行修改
```
#include <unistd.h>

fun: int truncate(const *char path, size_t length)
fun: int ftruncate(int fd, size_t length)
```

- access(const *char pathname, int mode)
![QQ截图20180502152313.png](https://upload-images.jianshu.io/upload_images/4061843-db9e930e79fd9d2c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- utime  utimes
```
#include <utime.h>

fun: int utime(const char* filename, struct utimebuf* buf)
fun: int utimes(const char* filename, struct timeval tvp[2])

改变一个文件的访问时间和修改时间,uimes具有更高的时间解析度
```
![QQ截图20180502152750.png](https://upload-images.jianshu.io/upload_images/4061843-64d6dafae279f367.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- stat fstat lstat
```
fun: int stat(const char* pathname, struct stat *buf)
fun: int fstat(int fd, struct stat *buf)
fun: int lstat(const char* pathname,struct stat *buf)

函数返回指定文件的信息
```

- duo dup2
```
#include <unistd.h>

fun: int dup(int oldfd)
fun: int dup2(int odlfd, int newfd)
```

- fcntl
```
#include <unistd.h>
#include <fcntl.h>

fun: int fcntl(int fd, int cmd)
fun: int fcntl(int fd, int cmd, long arg)
fun: int fcntl(int fd, int cmd, struct flock * lock)
```
![QQ截图20180502153507.png](https://upload-images.jianshu.io/upload_images/4061843-7409b6a707da7f83.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![QQ截图20180502153601.png](https://upload-images.jianshu.io/upload_images/4061843-c97d61321990bf8d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- getwd getcwd
```
fun: char * getwd(char *pathbuf)
fun: char * getcwd(char *pathbuf, size_t size)

绝对路径
```

- chdir fchdir
```
fun: int chdir(const char *path)
fun: int fchdir(int fd)
```

- mkdir rmdir
```
fun: int mkdir(const char *pathname, mode_t mode)

创建目录、删除目录
```

- opendir
```
#include <sys/types.h>
#include <dirent.h>

fun: DIR * opendir(const char * name)
```

- closedir
```
fun: int closedir(DIR *dir)
```

- readdir
```
fun: struct dirent * readdir*DIR *dp)
```

- mknod
```
fun: lint mknod(const char *path, mode_t mode, dev_t dev)
```
![QQ截图20180502154308.png](https://upload-images.jianshu.io/upload_images/4061843-82e727ea72bae56d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- mount umount
```
#include <sys/mount.h>

fun: int mount(const char *source, const char *target, const char *dilesystemtype, unsigned long mountflags, const void *data)
fun: int umount(const char *target)
fun: int umount2(const char *target, int flags)
```
![QQ截图20180502154609.png](https://upload-images.jianshu.io/upload_images/4061843-0edcfc9e68252e4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- link
```
#include <unistd.h>

fun: int link(const char *oldpath, const char *newpath)
```

- symlink readlink
```
#include <unistd.h>

fun: int symlink(const char *oldpath, const char *newpath)
fun: int readlink(const char *path, char *buf, size_t bufsiz)
```
![QQ截图20180502155706.png](https://upload-images.jianshu.io/upload_images/4061843-0186cb1de76e2bb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 标准I/O 
又称流，文件I/O流

![QQ截图20180514075035.png](https://upload-images.jianshu.io/upload_images/4061843-598a5af0a7b32f2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180514075052.png](https://upload-images.jianshu.io/upload_images/4061843-4a37cf8e531d4a94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180514075143.png](https://upload-images.jianshu.io/upload_images/4061843-7def174eda6539ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
![QQ截图20180514075238.png](https://upload-images.jianshu.io/upload_images/4061843-c61afc39b82badab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180514075250.png](https://upload-images.jianshu.io/upload_images/4061843-0af092b244d092db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180514075312.png](https://upload-images.jianshu.io/upload_images/4061843-8089a950d70028a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180514075347.png](https://upload-images.jianshu.io/upload_images/4061843-d8ed939c44c0729a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180514075359.png](https://upload-images.jianshu.io/upload_images/4061843-7745cf7f3512dcaa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180514075411.png](https://upload-images.jianshu.io/upload_images/4061843-e5d969b105f5e2dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180514075423.png](https://upload-images.jianshu.io/upload_images/4061843-feeebd21a776a6c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180514075503.png](https://upload-images.jianshu.io/upload_images/4061843-bcc4524d8b313370.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180514075520.png](https://upload-images.jianshu.io/upload_images/4061843-198130cf31f8a9a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180514075543.png](https://upload-images.jianshu.io/upload_images/4061843-e4d932217168d571.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180514075555.png](https://upload-images.jianshu.io/upload_images/4061843-9e9d65c1e310ab8a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)