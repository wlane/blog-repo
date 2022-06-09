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

​		kereros服务器本身为KDC或者密钥分发中心，主要包含三个部分：

- 主体（用户和服务）及其密码的数据库；
- 认证服务器（Authentication Server，简称 AS）：用来验证客户端的身份，通过就会发一张票证授予票证（Ticket Granting Ticket，简称TGT）给客户端；
- 凭据授权服务器（Ticket Granting Server，简称TGS）：使用TGT向TGS获取访问服务端的票证（Server Ticket，简称ST），客户端使用ST访问服务。

​		必须要注意的是，AS返回的TGT使用了用户主体的密码加密，用户主体收到后必须使用自己的kerberos密码解密，但是通常情况下，每次解密的密码不一定都有，所以这个时候需要使用称为keytab的文件来解密，这个文件中包含了资源主体的身份验证凭据。

# 2.关键概念

1. realm（域）

   包含KDC、客户机和服务器的整个kerberos网络，类似于域，俗称为域。

2. Principal（主体）

   用于在kerberos系统中标识一个唯一的身份，它可以是用户（比如tom）或者服务（比如sqlserver）。

   根据约定，主题名称分为三个部分，以`tom/node@example.com`为例：

   - 主名称：tom，用户名或者服务名；
   - 实例：node，对于用户主体，实例是可选部分，但是对于服务主体是必须的。用户可能在多个主机上有不同身份，所以有可能它具有多个实例，比如tom/node1和tom/node2。如果是在一台主机上，则tom可能拥有多个不用身份，比如tom/guest或者tom/admin。但是对于服务而言，实例是FQDN的值，比如sqlserver/nodes.example.com；
   - 域：example.com，这就是上面说的kerberos的领域的值。

3. ticket（票证）

   它是一种信息文件，用以将用户身份安全的传递到服务或者服务器，主要包含以下内容：

   - 用户的主机ip；
   - 用户的主体名称；
   - 服务的主体名称；
   - 时间标记；
   - ticket的生命周期；
   - 会话密钥副本。

   一张ticket仅对一台客户机和一项特定的服务有效。

4. keytab文件

   keytab包含了principal和加密的principal key的文件。由于keytab中存在hostname，所以它对每台host而言是唯一的。

   通过这种方式可以在不需要交互和保存文本密码的情况下验证一个principal。

# 3.验证流程

主要分为两个验证流程：初始验证和后续验证。

前提：kerberos系统默认各个principal的密码只有它自己和KDC知道

1. 初始验证

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220609221710.png)

   - 首选客户机会向KDC（AS）请求TGT，一般这个流程在用户登陆时就已经完成（或者通过kinit手动获取）；
   - KDC从数据库中找到客户机口令，使用它解密请求，通过后创建TGT，并依旧采用客户机口令加密的方式返回给客户端；
   - 客户端获取到TGT之后，使用口令解密得到TGT，只要在TGT的有效期内，客户端可以请求所有操作的服务器票证（ST）。

2. 后续验证

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220609223641.png)

   - 客户机向KDC（TGS）发送TGT作为其身份证明，向其请求某个服务的ST；
   - KDC将ST发送给客户机，ST使用服务器服务主体的口令加密，另外还包括使用客户机口令加密的会话密钥；
   - 客户机将ST发送到访问的服务器，并使用解密后得到的会话密钥验证服务器身份；
   - 服务器收到的ST后，使用服务的口令解密ST，这个ST中包括客户机的身份信息和会话密钥，服务器验证客户机身份后，使用得到的会话密钥回应客户机的身份认证请求，客户机确认了服务器身份正确后，就和服务器利用会话密钥使用对称加密的方式通信。

   注意：

   - 上面的客户机是client，可能是用户，也可能是服务；

   - 在整个请求流程前，这里的服务器已经注册在Kerberos网络中。但是最终客户机访问服务器时，并不需要服务器再去KDC上验证一次用户的身份。

# 4.Q&A

1. 为什么要获取TGT？
   - 为了避免客户机一直占有口令而不释放，通过TGT的生命周期管理，我们可以控制客户机的有效期；
   - 使用这种方式，KDC将给两个主体生成对称加密使用的会话密钥，从而可以避免直接使用口令通信，提高安全性。



本文参考了以下地址：

https://juejin.cn/post/6844903955416219661