---
title: 关于如何利用Github Pages搭建个人博客
tag: other

---

<meta name="referrer" content="no-referrer" />



##### 前言

在自己利用Github搭完博客后，决定还是写一篇博客。网上很多博客都是关于搭建的，但我还是决定写这篇博客。因为我在根据网上的教程搭建时，踩了好几个坑，为了避免其他人再一次重复跟我踩入相同的坑，我会在这篇博客写出踩的坑。也正是由于网上的教程都比较详细，所以我写的也许没有网上其他博客详细，但是我会把重要的过程都写出来，会特别注意那些坑的描述，所以欢迎大家查阅。但这篇博客若要转载，请注明出处。

------

#### 一、 安装准备软件：

- [Node.js](http://nodejs.org/)

- [Git](http://git-scm.com/)

    以上直接点击链接下载就行。
    Node.js下载后默认安装就行，关于Git的安装及使用，除了官方文档，本人更推荐[廖雪峰的Git教程](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/)老师的教程，通俗易懂，并让你学会所有的Git基本操作，并配置好后面相关的Git使用，所以后面所有相关Github上的操作均以你学习完Git教程为基础。

------

#### 二、在Github上配置Github Pages

[GitHub Pages](https://link.zhihu.com/?target=https%3A//pages.github.com/)分两种，一种是你的GitHub用户名建立的username.github.io这样的用户&组织页（站），另一种是依附项目的pages。

每个帐号只能有一个仓库来存放个人主页，而且仓库的名字必须是username/[username.github.io](http://username.github.io/)，这是特殊的命名约定。你可以通过[username.github.io](http://username.github.io/) 来访问你的个人主页。

这里特别提醒一下，需要注意的个人主页的网站内容是在master分支下的。

**廖雪峰老师**已经教会你如何在github上建立一个仓库了，所以这一步也就变得很简单，我们只需建立一个新的仓库，命名为形如username.github.io就可以了，这里需要注意的是，username是你的github的帐号，也就是你自己的用户名，后面的github.io保持一样就行了，建完仓库，通过访问[username.github.io](http://username.github.io/) ，你访问进入这个Pages的页面，能成功跳转页面，说明你的github上的配置也就基本完成了，当然，由于还没有推送博客上来，所以都还是空的。

------

#### 三、安装Hexo

在你的电脑中任何一个你觉得合适的位置，新建一个文件夹，文件夹的命名没有什么硬性要求，但一般我们都用Hexo来命名，因为在这个文件夹下，我们要安装Hexo，用于博客。

建立好文件夹后，在文件夹里右键打开Git Bash，相信这一点廖雪峰老师已经教会你了，打开Git Bash后，输入以下命令：

先安装Hexo：

> npm install -g hexo

初始化Hexo文件夹：

> hexo init

这步过后，你的文件夹里会生成相应的Hexo的相关文件
[![img](http://jycloud.9uads.com/web/GetObject.aspx?filekey=b6f92cee3776a76d6d529a60df428026)](http://jycloud.9uads.com/web/GetObject.aspx?filekey=b6f92cee3776a76d6d529a60df428026)
接下来，你可以先用命令hexo -v查询hexo是否安装好，出现如下的图片便表示安装好(**注意**：Hexo只需要安装一次就可以了)
[![img](http://ojplrudb4.bkt.clouddn.com/QQ%E6%88%AA%E5%9B%BE20170302215635.png)](http://ojplrudb4.bkt.clouddn.com/QQ截图20170302215635.png)
接下来我们现在本地上查询，本地是否已经搭建好了博客
输入以下命令：

> hexo -g
> hexo -s

然后打开你的浏览器，输入<localhost:4000/>来查看你的博客，出现如下图中的样子说明你的本地博客就搭建好了。
[![img](http://ojplrudb4.bkt.clouddn.com/QQ%E6%88%AA%E5%9B%BE20170302220110.png)](http://ojplrudb4.bkt.clouddn.com/QQ截图20170302220110.png)

一下是Hexo的几个常用命令：

> 新建文章：hexo new
> 新建页面：hexo new page
> 生成静态页面至public：hexo generate
> 开启预览访问端口：hexo server
> 将。
> deploy目录部署到github：hexo deploy
> 生成+部署：hexo d -g
> 预览+部署：hexo s -g

更多详情可以访问官方文档[Hexo](https://hexo.io/zh-cn/)

------

#### *四、部署Hexo到Github Pages*

**这一步及其重要，很多人搭博客的时候就出现了本地博客搭好了，Github上的仓库这些也搭好了，但就是本地的博客无法推送到云端，这就是这一步出现了问题，然而很多博客都没有对这一点着重讲解，所以这一步一定要细心。**

首先进入你的Hexo文件夹(就是你搭博客所用的文件夹)，然后打开Git Bash，首先我们需要安装一个扩展，这个扩展用于Hexo与Github连接，前面所提到的问题也差不多是因为这个扩展没装。

输入命令：

> npm install hexo-deployer-git –save

**注意，这里一定要把这个扩展安装好，否则始终不能推到云端上去，重要的事说三遍。**

然后我们需要配置一下Hexo文件夹里的文件，修改项目目录的_config.yml，找到deploy，作如下修改：

```
deploy:
	  type: git
	  repo: git@github.com:jiji262/jiji262.github.io.git
	  branch: master
```

*需要注意： **repo:**后面的是你建立博客的仓库名，在这里你只需要将用户名和仓库名改成你的就行了。*

------

#### 五、将本地博客推送到Github Pages上，并在云端访问。

输入以下命令：

> hexo -g
> hexo -d

推送成功后，在Github上能查询你的推送记录，访问cnfeat.github.io就可以查看你的博客了。

------

#### 六、进阶性的操作

关于将域名和自己的博客结合，如何购买域名这类问题，我想我不需要作太多的描述，网上的资源很多，在[如何搭建一个独立博客——简明Github Pages与Hexo教程](https://link.zhihu.com/?target=http%3A//www.jianshu.com/p/05289a4bc8b2)这篇博客中已有一定的描述。
如果你想把你的博客修改得更好看一点，你可以克隆别人的主题，也可以自己修改成自己所喜欢的样子，在这里克隆主题这一块我就不详细讲解，我给你一些博客自己去查看。

> - [手把手教你使用Hexo + Github Pages搭建个人独立博客](https://www.google.com.hk/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwj84NfB7rfSAhXo7YMKHV7zDfsQFggbMAA&url=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F20760507&usg=AFQjCNHWtLDmniPQ_AOcJM-SnDlSzG_QxA)
> - [如何搭建一个独立博客——简明Github Pages与Hexo教程](https://link.zhihu.com/?target=http%3A//www.jianshu.com/p/05289a4bc8b2)

这两篇博客也十分优秀，可以看一下，当然，里面肯定有主题的克隆。

------

至于如何写博客，并将发表，在此我也不多言，很多优秀的博客都已明确写出，也可以看一下我给出的参考链接

### 参考链接并致谢以下博主

> - [Hexo主页](https://link.zhihu.com/?target=https%3A//hexo.io/)
> - [手把手教你使用Hexo + Github Pages搭建个人独立博客](https://www.google.com.hk/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwj84NfB7rfSAhXo7YMKHV7zDfsQFggbMAA&url=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F20760507&usg=AFQjCNHWtLDmniPQ_AOcJM-SnDlSzG_QxA)
> - [如何搭建一个独立博客——简明Github Pages与Hexo教程](https://link.zhihu.com/?target=http%3A//www.jianshu.com/p/05289a4bc8b2)
> - [Github建站系列教程](http://www.pchou.info/ssgithubPage/2013-01-03-build-github-blog-page-01.html)

------

若有什么不对之处，还请多多指教。

**希望大家多多支持！你的鼓励便是我的动力！**