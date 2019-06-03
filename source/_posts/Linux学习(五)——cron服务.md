---
title: Linux学习(五)——cron服务
tag: Linux
date: 2018-04-23
---

# Linux学习(五)——cron服务

**cron：**守护进程

> sudo service cron status：查看cron状态
> sudo /etc/init.d/cron start：开启cron
> sudo /etc/init.d/cron stop：关闭cron
> sudo /etc/init.d/cron restart：重启cron

---

**crontab用法：**
> crontab –e : 修改 crontab 文件,如果文件不存在会自动创建。 (这样可以编辑模式打开个人的 crontab 配置文件,然后加入一下这行:0 0 * * * /home/linrui/XXXXXXXX.sh这将会在每天凌晨运行 指定的.sh 文件)
> crontab –l : 显示 crontab 文件。
> crontab -r : 删除 crontab 文件。
> crontab -ir : 删除 crontab 文件前提醒用户。

在 crontab 文件中写入需要执行的命令和时间,该文件中每行都包括六个域,其中前五个域是指定
命令被执行的时间,最后一个域是要被执行的命令。每个域之间使用空格或者制表符分隔。格式如下:
```
minute hour day-of-month month-of-year day-of-week commands
合法值为:00-59 00-23 01-31 01-12 0-6 (0 is sunday)
```
> 除了数字还有几个特殊的符号:"*"、"/"和"-"、","
>  *代表所有的取值范围内的数字
>  "/"代表每的意思,"/5"表示每 5 个单位
>  "-"代表从某个数字到某个数字
>  ","分开几个离散的数字
> 注:commands 注意以下几点
>  要是存在文件,要写绝对路径
>  即使是打印也不会显示在显示屏,在后台运行,最好重定向日志


以下是 crontab 文件的格式:
>`{minute} {hour} {day-of-month} {month} {day-of-week} {full-path-to-shell-script}`
>o minute: 区间为 0 – 59
>o hour: 区间为 0 – 23
>o day-of-month: 区间为 0 – 31
>o month: 区间为 1 – 12. 1 是 1 月. 12 是 12 月.
>o Day-of-week: 区间为 0 – 7. 周日可以是 0 或 7.

![QQ截图20180527102400.png](https://upload-images.jianshu.io/upload_images/4061843-60f756ac69be4c6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20180527102733.png](https://upload-images.jianshu.io/upload_images/4061843-edd332b37a6730e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)