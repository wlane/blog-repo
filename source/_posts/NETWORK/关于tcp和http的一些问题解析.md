---
title: 关于tcp和http的一些问题解析
tags:
- http
categories:
- tcp
---

前置知识：[TCP连接的建立和关闭 | repo (xwangwang.com)](https://www.xwangwang.com/2022/05/04/Network/TCP连接的建立和关闭/)

# 1.一个TCP上可以有几个HTTP请求

http/1.0：默认一个HTTP请求对应于一个TCP连接，每次有新的HTTP请求都需要打开一个新的TCP连接，但是后期也部分支持持久连接；

http/1.1： 头部加入Connection: keep-alive，所有连接默认都是持久连接（长连接），建立一次TCP连接后可以进行多次的HTTP请求和响应；

http/2：加入了多路复用的技术，以前的HTTP通信都是需要前一个请求收到回复之后才能发送下一个请求，但是在HTTP/2中可以直接发送和接受多个请求，不用等待。

以上协议版本需要客户端和服务器同时支持才能生效，所以我们比如在Nginx上配置h2时需要浏览器也支持，当然现代的浏览器基本上都支持。

# 2.TCP什么时候断开

在http/1.1之后，由于存在keep alive持久连接，建立一个tcp连接后会进行多次的http连接，那http请求结束后，那到底什么时候断开tcp连接呢？

首先要明确一点，keepalive可以分为http keepalive和tcp keepalive，http keepalive由应用完成，tcp keepalive由操作系统内核保证。我们这里的参数指的是http keepalive。

1. 配置了keepalive timeout的参数

   比如Nginx我们配置了keepalive_timeout 10s; 这会导致当TCP连接空闲10s钟后，Nginx服务会向客户端发送FIN请求，要求关闭连接。

2. 没有配置keepalive timeout的参数的情况

   假如某些应用没有自己设定心跳规则，也就是没有设置http keepalive，这时候就需要操作系统的tcp keepalive来判断了。

   Linux：

   ~~~shell
   $ sudo sysctl -a |grep -i tcp_keep
   net.ipv4.tcp_keepalive_intvl = 75
   net.ipv4.tcp_keepalive_probes = 9
   net.ipv4.tcp_keepalive_time = 7200
   ~~~

   Windows 10 Pro：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220505233404.png)

   可以看出linux服务器的值为2个小时，windows电脑的值为30分钟。所以如果连接空闲的话，则启动系统的tcp保活机制，默认执行间隔是2个小时或者30分钟。

   这部分可以参考：

   [聊聊 TCP 中的 KeepAlive 机制 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/28894266)

   [TCP Keepalive机制刨根问底 - SegmentFault 思否](https://segmentfault.com/a/1190000021057175)

# 3.浏览器建立的tcp连接有没有限制

在默认http/1.1的条件下，虽然一个tcp连接就可以多次传输http，但是在某些页面资源非常多的情况下按照依次请求回复的运行模式下，仍然需要很长的时间，所以还是有必要建立多个tcp连接，那么tcp连接到底能建立多少呢？

比如chrome： [Network features reference - Chrome Developers](https://developer.chrome.com/docs/devtools/network/reference/)

>- There are already six TCP connections open for this origin, which is the limit. Applies to HTTP/1.0 and HTTP/1.1 only.

他最多允许由6个TCP连接。不同的浏览可能有不同限制，需要去查看不同浏览器的开发文档。

整体上来讲，一般是这种请求方式：在使用http/1.1的情况下，尽量建立多的TCP连接，以便快速请求，在使用http/2的情况下，由于支持多路复用，所以对多个TCP连接没有那么急迫，可能就存在只使用一个TCP连接的情况。
