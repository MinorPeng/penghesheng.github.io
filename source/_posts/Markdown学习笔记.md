---
title: Markdown学习笔记
tag: 其他
category: 其他
date: 2016-11-01
---



# Markdown学习笔记

![](http://upload-images.jianshu.io/upload_images/4061843-a066d691214d2292.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 一级标题
## 二级标题
### 三级标题

#### 四级标题

##### 五级标题

###### 六级标题

//单个字符保留一个空格，#越多，字体反而越小，而且只能放在首位

# 有序和无序文本

![](http://upload-images.jianshu.io/upload_images/4061843-9b5d76816b3e3711.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

无序
- 文本
- 文本

有序
1. 文本1
2. 文本2
//有序和无序之间要隔一个空行,否则将以在前的格式显示,空行结束文本显示格式

# 链接和图片的写入

[简书](http://www.jianshu.com)
//链接要紧跟[]，同时注意这些符号是在英文状态下的半角输入

![](http://upload-images.jianshu.io/upload_images/4061843-26a41c75417f8196.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![psbY7HX20PM.jpg](http://upload-images.jianshu.io/upload_images/4061843-6bcc90e2e4728a02.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#文本引用
//注意保留一个空格，同时使用三个*时，便是粗斜体，当使用斜体和粗体相邻时，注意用空格分隔，当使用多个时，要注意个数的匹配和分隔

![](http://upload-images.jianshu.io/upload_images/4061843-10a690a20e6d1aea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> *一盏灯， 一片昏黄；* **一简书**， 一杯淡茶。 守着那一份淡定， 品读属于自己的寂寞。 ***保持淡定， 才能欣赏到最美丽的风景！*** 保持淡定， 人生从此不再寂寞。

# 代码引用

//代码引用“`”总是成对出现，前后都有，当三个及以上出现时，便是段的引用，需要注意个数的搭配

![](http://upload-images.jianshu.io/upload_images/4061843-866d96a07894f819.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`Hello world`
``Good``
`hello world
 hello world``
//个数不匹配
```
第一段代码
the second
```
#表格
//"|"左边不能省略，右边可省，左边如果要省略第一个，则下面所有行都要省略，“:”靠近哪边的“|”，表格中的内容偏向哪一边，两边都有时，靠中间，需要与“-”一起用，并且第二行至少有两个“|”，前两个“|”中间不能为空或者其他无意义符号，可以是“-”和“:”

![](http://upload-images.jianshu.io/upload_images/4061843-1be5aea0bc67fb07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

| Tables        |      Are      |  Cool |
| :------------ | :-----------: | ----: |
| *col 3 is*    | right-aligned | $1600 |
| **col 2 is**  |   centered    |   $12 |
| zebra stripes |   are neat    |    $1 |

| dog  | bird |  cat |
| :--- | :--: | ---: |
| foo  | foo  |  foo |
| bar  | bar  |  bar |
| baz  | baz  |  baz |

# 显示连接中带括号的图片
//第一个[]显示的是名字，第二个和第二行的[]中的数字必须相同，不论数字为多少

![](http://upload-images.jianshu.io/upload_images/4061843-50fb4220c923cd9d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![**显示括号中的图片**][22]

[22]: http://latex.codecogs.com/gif.latex?\prod%20\(n_{i}\)+1