---
title: kerberos原理
tags:
- kerberos
categories:
- security
---

官网：[Kerberos: The Network Authentication Protocol](https://web.mit.edu/kerberos/)

# 1.概述

​		首先可以看下对于理解kerberos有极大帮助的虚构对话：[**Designing an Authentication System:**
**a Dialogue in Four Scenes**](http://web.mit.edu/kerberos/dialogue.html)

​		Kerberos 是一种计算机网络认证协议，它允许某实体在非安全网络环境下通信，向另一个实体以一种安全的方式证明自己的身份。它是一种第三方认证机制，其中用户和服务依赖于第三方（Kerberos 服务器）来对彼此进行身份验证。

​		kereros服务器本身成为KDC或者密钥分发中心，主要包含三个部分：

- 主体（用户和服务）及其密码的数据库；
- 认证服务器（Authentication Server，简称 AS）：用来验证客户端的身份，通过就会发一张票证授予票证（Ticket Granting Ticket，简称TGT）给客户端；
- 凭据授权服务器（Ticket Granting Server，简称TGS）：使用TGT向TGS获取访问服务端的票据（Server Ticket，简称ST），客户端使用ST访问服务。

​		必须要注意的是，AS返回的TGT使用了用户主体的密码加密，用户主体收到后必须使用自己的kerberos密码解密，但是通常情况下，每次解密的密码不一定都有，所以这个时候需要使用称为keytab的文件来解密，这个文件中包含了资源主体的身份验证凭据。

# 2.验证流程





参考地址：

https://www.cnblogs.com/panpanwelcome/p/13601987.html

https://juejin.cn/post/6844903955416219661