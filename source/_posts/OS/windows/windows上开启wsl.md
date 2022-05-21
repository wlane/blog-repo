---
title: windows上开启wsl
tags:
- wsl
categories:
- windows
---



参考： https://docs.microsoft.com/zh-cn/windows/wsl/install-win10#step-4---download-the-linux-kernel-update-package 

关于wsl版本的选择因素，大致需要考虑以下几点：

1、是否需要在linux和windows间共享文件，如果需要的话，使用wsl 1；

2、wsl 2有比较完整的内核，支持docker；

3、wsl 2 使用了hyper-v，使用的时候不能同时使用vmware或者其他虚拟化工具；

4、基本上所有工具都可以在linux中完成，则使用wsl 2。

我们这里选择的版本是wsl 2.

## 1、启动适用于linux的windows子系统

打开powershell，使用管理员权限运行：

dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

如下：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504192416915.png)

## 2、检查windows版本

在cmd命令行中运行winver：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504192749931.png)

可以得到：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504192857498.png)

对比要求：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504193110001.png)

可以看出这个版本是符合要求的。

## 3、启用虚拟机功能

在powershell中执行：

dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

如下图：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504193306320.png)

执行后重新启动计算机。

## 4、下载linux内核更新包

下载地址： https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi 

下载后执行安装：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504194325690.png)

## 5、将wsl 2设置为默认版本

在powershell中执行下列命令：

wsl --set-default-version 2

如下图；

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504194533402.png)

## 6、安装linux

打开microsoft store，我们最佳选择其实是rhel系的fedora remix，但是需要收费，所以我们选用了ubuntu 18.04：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504195612868.png)

稍等片刻，安装完毕后点击启动：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504195716743.png)

出现了报错：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504195933431.png)

报错的解决方式见地址：https://github.com/microsoft/WSL/issues/4299 （修改 C:\Users<your user name>\AppData\Local\Packages下的目录）

重新启动：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504200439074.png)

启动成功，这里需要设置用户名和密码。

这里我们也可以设置把启动命令放到任务栏：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504200946122.png)

## 7、使用powershell操作linux

见下图：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/image-20210504201621290.png)