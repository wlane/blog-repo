---
title: 使用Hexo+GitHub搭建个人博客
tags:
- hexo
categories:
- blog
---

可以和GitHub一起使用的静态网页模板系统有很多，比如：

1、基于ruby的Jekyll;

2、基于nodejs的Hexo；

3、基于go的Hugo；

4、基于python的Pelican；

5、操作最简单的Gridea。

这里面之所以选择Hexo，是因为官方推荐的Jekyll速度感人，而Gridea不适用于多终端，所以就采用了比较活跃的hexo，其他的暂时还没有用过。

## 前提

1、拥有一个GitHub仓库；

2、拥有一个图床，这里为了防止网络问题，我使用的是阿里云OSS；

3、拥有一个云盘，用来同步仓库，以便在多台设备上同时使用；

4、拥有一个域名（可选）。

# 1.使用阿里云OSS配置图床

首先我们需要一个图床，以用来存放markdown里面的图片，当然图片也可以放在本地，但是markdown编辑模式下操作太麻烦，所以选择一个在线的图床。

然后我们这里使用方便上传图片的工具[picGo](https://picgo.github.io/PicGo-Doc/zh/guide/)，它支持图片拖拽上传，并且上传后就立即得到链接地址的功能，如下图：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220426221122.png)

最后一开始考虑使用免费的GitHub，但是存放GitHub静态资源的地址国内无法访问，可能就会出现经常网站打开看不到图片的情况，结合上面picGo中图床设置中支持的种类，最终选择了国内的阿里云OSS。

1、登录阿里云购买阿里云OSS；

2、创建一个Bucket；

3、创建普通用户，开启open api调用，获取accesskey的相关信息；

4、配置到picGo的图床设置中：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220426221613.png)

从此图片都会上传到上面创建的Bucket中。



# 2.创建GitHub仓库

如下图：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220426220122.png)

这里需要注意的是：

1、仓库为public；

2、仓库名一定要以.github.io结尾；

3、创建blog分支，而不用master分支；

4、Custom domain可选，不配置的话，访问域名默认为仓库名；

5、开启https连接。

# 3.配置同步云盘

我这里使用的是我一直在使用的[坚果云](https://www.jianguoyun.com/)，在想要设置的路径下创建一个目录 blog，然后设置成实时同步：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220426223130.png)



# 4.安装Hexo

参考： https://hexo.io/docs/

git以前就安装过，这里略过，node版本为：node-v14.19.0-x64.msi 

基础环境安装完成后：

~~~shell
$ node -v
v14.19.0
$ npm -v
7.12.0
$ git --version
git version 2.1.4
~~~

接下来就是安装Hexo了：

~~~shell
$ pwd
/cygdrive/c/Users/Figo
$ npm install -g hexo-cli	# hexo是基于nodejs的博客框架
$ cd Documents/blog			# 这里的blog就是上面创建的同步目录
$ hexo init blog	# 初始化
$ cd blog		# 初始化后生成一个叫blog的目录
$ npm install	# 安装所有依赖
$ ll
total 256K
-rw-r--r-- 1 Figo None    0 Apr 25 23:50 _config.landscape.yml
-rw-r--r-- 1 Figo None 2.4K Apr 25 23:50 _config.yml
-rw-r--r-- 1 Figo None  20K Apr 26 00:00 db.json
drwxr-xr-x 1 Figo None    0 Apr 25 23:58 node_modules
-rw-r--r-- 1 Figo None  619 Apr 26 00:00 package.json
-rw-r--r-- 1 Figo None 163K Apr 25 23:58 package-lock.json
drwxr-xr-x 1 Figo None    0 Apr 25 23:50 scaffolds
drwxr-xr-x 1 Figo None    0 Apr 25 23:50 source
drwxr-xr-x 1 Figo None    0 Apr 25 23:50 themes
~~~

以上就是安装部署流程，部署完成可以测试一下：

~~~shell
$ hexo s
INFO  Validating config
INFO  Start processing
INFO  Hexo is running at http://localhost:4000/ . Press Ctrl+C to stop.
~~~

此时你可以在浏览器中以http://localhost:4000/ 访问默认页面。

# 5.配置Hexo

1、设置CNAME

这里需要在source目录下设置CNAME文件，文件中填写访问域名：

~~~shell
$ vi source/CNAME
www.xwangwang.com
~~~

2、下载自己想要的主题

参考：[Themes | Hexo](https://hexo.io/themes/)

这里我使用的主题叫[wujun234/hexo-theme-tree (github.com)](https://github.com/wujun234/hexo-theme-tree)：

~~~shell
$ git clone https://github.com/wujun234/hexo-theme-tree.git  themes/tree
$ rm -rf themes/tree/.git	# 防止git配置信息干扰我们自己的同步
$ vi themes/tree/_config.yml	# 修改tree的配置
......
	enableComment: false
......
busuanzi: false
......
tags: true
categories: true
......
$ cd source
$ ll
total 9.0K
drwxr-xr-x 1 Figo None  0 Apr 26 02:10 _posts
-rw-r--r-- 1 Figo None 18 Apr 26 00:17 CNAME
$ hexo new page "tags"		# 创建一个tags的目录
$ vi tags/index.md
---
title: tags
date: 2022-04-26 02:58:43
type: "tags"
layout: "tags"
---
$ hexo new page "categories"	# 创建一个categories的目录
$ vi categories/index.md
---
title: categories
date: 2022-04-26 02:59:43
type: "categories"
layout: "categories"
---
$ hexo new page --path about/index "About"
$ vi about/index.md
......		# 添加一些简介
$ ll
total 13K
drwxr-xr-x 1 Figo None  0 Apr 26 02:10 _posts
drwxr-xr-x 1 Figo None  0 Apr 26 23:12 about
drwxr-xr-x 1 Figo None  0 Apr 26 03:00 categories
-rw-r--r-- 1 Figo None 18 Apr 26 00:17 CNAME
drwxr-xr-x 1 Figo None  0 Apr 26 02:59 tags
~~~

最后替换图标，路径：themes/tree/source/favicon.ico

3、配置hexo

~~~shell
$ vi _config.yml
......
title: repo
......
author: Figo Xue
......
url: https://www.xwangwang.com
......
theme: tree
......
deploy:
  type: 'git'
  repo: git@github.com:wlane/wlane.github.io.git
  branch: blog
$ hexo d -g			# 生成页面并推送到github仓库
~~~

此时浏览器访问：https://www.xwangwang.com 即可。

# 6.Markdown编写

后续文章的编写都在blog/source/\_posts目录下进行，在这个下面可以随意创建目录和md文件。

Markdown语法：

[Markdown 官方教程](https://markdown.com.cn/)

使用工具（需要购买）：

[Typora — a markdown editor, markdown reader.](https://typora.io/)

每次推送时执行下列命令：

~~~shell
$ pwd
/cygdrive/c/Users/Figo/Documents/blog/blog
$ hexo d -g
......
~~~

hexo命令参考：

[Commands | Hexo](https://hexo.io/docs/commands)
