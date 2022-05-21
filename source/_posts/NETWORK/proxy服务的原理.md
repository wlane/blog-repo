---
title: proxy服务的原理
tags:
- proxy
categories:
- network
---

# 1.正向代理

1. 特点

   一般指的是内网服务器或者用户通过代理访问公网，或者是访问某些被屏蔽的网站。

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220506235742.png)

   - 解决ip访问限制的问题
   - 隐藏客户端真实ip
   - 假如代理有缓存的话，还可以提高访问速度

2. 配置

   - 最佳方案：需要代理访问的客户端应用是否自身支持proxy参数，可以的话，直接在应用内配置，这样不会污染其他环境；

   - 如果应用不支持，那就考虑配置系统级别的环境变量：

     RHEL/CentOS：

     ~~~shell
     # 以下对所有使用bash登录的用户生效
     $ sudo vi /etc/profile.d/http_proxy.sh
     export http_proxy="http://10.1.1.1:8080"
     export https_proxy="http://10.1.1.1:808"
     export no_proxy="127.0.0.1,localhost"
     $ source /etc/profile
     
     # 以下对所有应用程序都有效，和是否有用户登录无关
     $ sudo vi /etc/environment
     http_proxy="http://10.1.1.1:8080"
     https_proxy="http://10.1.1.1:808"
     no_proxy="127.0.0.1,localhost"
     ~~~

   - 如果连系统级别的变量也不支持的话，可能就需要一些特殊的配置方式，比如Nginx：

     [http proxy - nginx proxy_pass over https_proxy - Stack Overflow](https://stackoverflow.com/questions/46803431/nginx-proxy-pass-over-https-proxy)

   - windows 环境里面的代理工具多种多样，这里就不介绍了。

# 2.反向代理

一般指的就是我们遇到的Nginx，Haproxy等这一类软件在整个应用架构中所起的作用。

1. 特点

   一般指代理服务器接受外部网络的请求，然后转发给内部网络的服务器，并将结果返回给外部网络客户。

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220507001700.png)

   - 负载均衡
   - 隐藏真实的业务服务器ip
   - 有缓存的情况下提高访问速度
   - 提供一定的安全防护措施，比如提供统一的SSL访问，http访问认证等。

2. 配置

   一般这种代理服务器上的应用都是我们熟知的Nginx，Haproxy之类的，具体配置可以参考：[Nginx的安全配置 | repo (xwangwang.com)](https://www.xwangwang.com/2022/05/02/Web/nginx/Nginx的安全配置/)

# 3.透明代理

简单来讲就是设备上不需要做任何配置，默认就可以走代理，所以是透明的，对用户来讲，感受不到。

可以参考：

[透明代理入门 | Project X (xtls.github.io)](https://xtls.github.io/document/level-2/transparent_proxy/transparent_proxy.html)

主要实现原理仍然是和iptables有关，需要了解：[iptables原理 | repo (xwangwang.com)](https://www.xwangwang.com/2022/04/30/Security/iptables原理/)