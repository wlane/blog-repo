---
title: windows上安装docker
tags:
- docker
categories:
- docker
---

## 1、下载docker安装包

下载地址：https://www.docker.com/get-started 

选择docker desktop版本安装。

注意：因为docker for windows目前默认支持wsl2，所以如果没有安装wsl2，则会要求安装。假如不安装wsl2，只开启hyper-v功能，应该也行（推测）。

##  2、启动

直接点击docker desktop的图标启动，启动后你会发现在windows的terminal或者wsl的linux系统里面都可以运行docker命令。

如下图使用powershell来操作，并且不需要管理员权限：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504221642514.png)

## 3、查看镜像

可以直接在dashboard中查看相关信息，比如查看镜像：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504221912274.png)