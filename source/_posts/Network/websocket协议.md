---
title: websocket协议
tags:
- websocket
categories:
- web
---

# 1.什么是websocket

参考：[The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)

在websocket出现以前，实时web应用或者游戏，聊天工具都是基于http连接来实时更新数据。关于http可参考：[关于tcp和http的一些问题解析](https://www.xwangwang.net/2022/05/27/Network/%E5%85%B3%E4%BA%8Etcp%E5%92%8Chttp%E7%9A%84%E4%B8%80%E4%BA%9B%E9%97%AE%E9%A2%98%E8%A7%A3%E6%9E%90/)。由于http是单向的从客户端到服务端的请求，因此客户端应用程序就要主动对服务器不断发起请求，询问服务器是否有更新信息，如果有的话服务器就返回最新的数据给客户端。此时一般存在以下几种解决方案：

- 轮询：客户端定时发送请求到服务端询问是否有信息更新；
- 长轮询：客户端发起一个超时时间很长的请求，服务器一直保持这个请求存在，从而可以在有需要时直接响应；
- 长连接：注意这里的长连接和http/1.1的持久连接是一样的，并且如果这里的长连接如果是基于socket来创建的，那其实本质上就是websocket，websocket的基础就是http的长连接。不同的一点是http连接仍然是单向的，还是会有实时性不够或者资源浪费等问题。

上面的几种方式由于都需要不断发起新的http请求，但是http请求是无状态的，每次请求都需要传输包括身份验证在内的各种信息，成本很高，于是就会浪费流量和服务器资源，因此理想的方式是有一种双向的，全双工的基于http的通信机制，从而使得服务器可以主动推送数据到客户端。于是就出现了websocket。

websocket诞生于2008年，并在2011年成为国际标准，它是HTML5规范的组成部分之一。它将TCP的socket应用在了web页面上，从而使通信双方建立起一个保持在活动状态的连接，它是全双工的协议，一种有状态协议。

1. 什么是socket

   网络概念中的socket不是一个协议，而是一种对TCP/IP层的封装，是TCP/IP的编程接口，是抽象出来的一层API，位于应用层和传输层之间。对用户来说，只需要去面对这一组简单的API，具体的数据结构由socket去组织，以符合指定的协议。所以socket就是一种标准，各种不用的语言按照这种标准实现自己的网络通信。

   Socket=IP地址+端口+协议

   通过以上的这种唯一标识，提供了一种通信双方的身份认证方式，从而通行双方可以准确的传送信息。

   在linux服务器上，进程间的通信也有socket的概念。一切皆文件的哲学下，这里的socket是一种文件，它是一种打开-读写-关闭的模式。在建立连接后，通过往这个文件写入内容来供对方读取或者通过读取这个文件来获取对方的内容，通信结束关闭该文件。

   websocket是一种应用层协议，不直接和socket产生联系。websocket是基于http的，http通信又是基于tcp连接的。tcp/ip协议又是通过socket来实现的。所以可以说websocket和socket没有太大的直接关系。

2. 与http的关系

   websocket是基于http协议的，借用了http协议来完成了一部分握手工作。以下为websocket的握手过程。

   首先客户端发送握手请求：

   ~~~html
   GET /chat HTTP/1.1
   Host: server.example.com
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
   Sec-WebSocket-Protocol: chat, superchat
   Sec-WebSocket-Version: 13
   Origin: http://example.com
   ~~~

   相比较正常的http协议，这里主要多出了以下几个部分：

   - upgrade：http/1.1中定义的用于转换协议的header，它表示，如果服务器支持的话，客户端希望使用现有的「网络层」已经建立好的这个连接（此处是TCP连接），切换到另外一个「应用层」协议（此处是WebSocket）；
   - connection：由于http/1.1中规定了upgrade只能出现在直接连接中，所以当我们连接中存在代理服务器时，需要使用connection来表示任何接收到消息的实体（代理）在转发之前都需要处理掉connection中指定的header（不能转发upgrade header）。也就是说，客户端和服务端用过代理（比如Nginx）相连时，在发送这个握手消息之前，首先要发送CONNECT消息来建立直接连接；
   - sec-websocket：用来验证的key，以及客户端支持的协议和版本

   然后服务器在接受请求后，返回如下信息：

   ~~~html
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
   Sec-WebSocket-Protocol: chat
   ~~~

   - 101：响应码，表示同意切换到此协议；
   - 其他：告诉客户端升级的协议是什么，并且发送加密后的key来证明，客户端可以通过解密来验证服务端。

   接下来就是按照websocket的协议来进行通信了。

# 2.在代理服务器中的配置

1. nginx

   参考：[WebSocket proxying](https://nginx.org/en/docs/http/websocket.html)

   在明确所有连接一定要升级的情况下，可以直接指定：

   ~~~nginx
   location / {
           proxy_pass   http://1.1.1.1:8080/;
       	# 启用websocket连接
       	proxy_http_version 1.1;
           proxy_read_timeout   3600s; 	# 超时设置
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection "Upgrade";
   }
   ~~~

   在不需要一定升级成websocket的情况下，可以使用map指令，此时默认会升级成websocket，但是如果http_upgrade为空的话，那么Connection header为close：

   ~~~nginx
   http {
   	map $http_upgrade $connection_upgrade {
       	default Upgrade;
       	'' close;
   	}
   	service {
   		......
   		location / {
   			proxy_pass   http://1.1.1.1:8080/;
   			# 启用websocket连接
       		proxy_http_version 1.1;
           	proxy_read_timeout   3600s; 	# 超时设置
           	proxy_set_header Upgrade $http_upgrade;
           	proxy_set_header Connection $connection_upgrade;
   		}
   	}
   }
   ~~~

2. haproxy

   参考：[Websockets Load Balancing with HAProxy](https://www.haproxy.com/blog/websockets-load-balancing-with-haproxy/)

   当客户端请求websocket时，haproxy会自动转换为tunnel模式，因此默认情况下haproxy不需要配置，但是为了防止连接提前关闭，可以配置一个超时设置：

   ~~~nginx
   defaults
   	timeout tunnel        3600s
   ~~~

   存在多个backend需要区分的情况下：
   
   ~~~nginx
   frontend ws
   	bind *:443 ssl crt /path/to/cert
   	mode http
   	acl host_ws hdr_beg(Host) -i ws.test.com
   	acl hdr_connection_upgrade hdr(Connection)  -i upgrade
   	acl hdr_upgrade_websocket  hdr(Upgrade)     -i websocket
   	use_backend bk_ws if hdr_connection_upgrade hdr_upgrade_websocket host_ws
   	use_backend bk_web if !hdr_connection_upgrade !hdr_upgrade_websocket host_ws
   	
   backend bk_ws
   	balance roundrobin
   	......
   	
   backend bk_web
   	balance roundrobin                                             
     	option httpchk HEAD /                                          
     	server websrv1 192.168.10.11:80 maxconn 100 weight 10 cookie websrv1 check
     	server websrv2 192.168.10.12:80 maxconn 100 weight 10 cookie websrv2 check
   ~~~
   
   