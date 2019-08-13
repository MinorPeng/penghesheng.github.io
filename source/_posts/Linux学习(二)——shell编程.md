---
title: Linux学习(二)——shell编程
tag: Linux
date: 2018-03-26

---

<meta name="referrer" content="no-referrer" />



# Linux学习(二)——shell编程

**shell脚本**
C语言通过调用API系统接口也可实现shell脚本功能

解释器：Bash、Csh、Ksh

bash hello  强制使用Bash解释器执行
sh hello 软连接，指向一个解释器

> shell
> `$n	$1 表示第一个参数，$2 表示第二个参数 ... `
> `$#	命令行参数的个数 `
> `$0	当前程序的名称`
> `$?	前一个命令或函数的返回码`
> `$*	以“参数1 参数2 ... ” 形式保存所有参数`
> `$@	以“参数1” “参数2” ... 形式保存所有参数`
> `$$	本程序的(进程ID号)PID  `
> `$! 	上一个命令的PID `
> read：读取信息（键盘或者文件）

在Linux中第一个字符代表这个文件是目录、文件或链接文件等等。

当为[ d ]则是目录
当为[ - ]则是文件；
若是[ l ]则表示为链接文档(link file)；
若是[ b ]则表示为装置文件里面的可供储存的接口设备(可随机存取装置)；
若是[ c ]则表示为装置文件里面的串行端口设备，例如键盘、鼠标(一次性读取装置)。

默认是字符串，表达式加空格、expr、let、$(( ))

![1.png](https://upload-images.jianshu.io/upload_images/4061843-32d09b434147cbc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![2.png](https://upload-images.jianshu.io/upload_images/4061843-503cad7ba1598907.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![3.png](https://upload-images.jianshu.io/upload_images/4061843-8155e0d274b98b15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![4.png](https://upload-images.jianshu.io/upload_images/4061843-b96463388f1fba52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![5.png](https://upload-images.jianshu.io/upload_images/4061843-52ab68a86a39769d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![6.png](https://upload-images.jianshu.io/upload_images/4061843-ddbabbb91d203ec1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![7.png](https://upload-images.jianshu.io/upload_images/4061843-d9424622db57bfe5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![8.png](https://upload-images.jianshu.io/upload_images/4061843-0ab3da2608014041.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)