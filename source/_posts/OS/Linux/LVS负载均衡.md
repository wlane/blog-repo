---
title: LVS负载均衡
tags:
- LVS
categories:
- LB
---

# 1.简介

官网：[LVS项目中的有关中文文档 (linuxvirtualserver.org)](http://www.linuxvirtualserver.org/zh/index.html)

以上官网中关于原理部分已经讲述的十分清楚，重点在于以下两点：

1. 基于ip层和内容请求分发的负载均衡技术，目前的实现是基于ip层，基于内容请求分发的调度还不成熟，但是它整体上来说是工作在3-4层，识别TCP连接，具体转发是工作在ip层，可以基于ip和端口（必须要到4层才能识别到端口）实现请求转发。所以可以说它是一个虚拟的四层交换机集群；
2. ipvs工作在netfilter的INPUT链上，我们可以使用用户态工具ipvadm实现对内核中ipvs的管理，类似使用iptables实现对netfilter的管理，对符合要求的报文做相应的过滤。所以简单来说这一切还是通过防火墙管理来实现的。

# 2. 体系架构

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220503202929.png)

参见上图，主要组件如下：

1. 负载调度器（load balancer），它是整个集群对外面的前端机，负责将客户的请求发送到一组服务器上执行，而客户认为服务是来自一个IP地址（我们可称之为虚拟IP地址）上的。它本身不产生任何流量，只做转发，所以性能比较高；
2. 服务器池（server pool），是一组真正执行客户请求的服务器，执行的服务有WEB、MAIL、FTP和DNS等；
3. 共享存储（shared storage），它为服务器池提供一个共享的存储区，这样很容易使得服务器池拥有相同的内容，提供相同的服务。

# 3. 主要的四种工作模式

这里可以查看详细解释：[Linux服务器集群系统(三)--LVS集群中的IP负载均衡技术 (linuxvirtualserver.org)](http://www.linuxvirtualserver.org/zh/lvs3.html)

1. VS/NAT模式

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220503205726.png)

   上图中VIP指的是调度器的公网ip，DIP指的是调度器的内网IP。

   这种配置方式的条件或者特点是：

   - 所有后端应用服务器的网关必须要指定为调度器的地址DIP，这样保证响应数据返回到调度器；
   - 调度器同时负责处理请求和响应，因此I/O负载较高，性能是瓶颈；
   - 配置相对简单，调度器和后端应用服务器可在不用网段，只要可互连即可；
   - 适合后端有windows的服务集群；
   - 支持端口映射；
   - 大多数硬件负载均衡的实现原理都是这种，比如F5的Big/IP。

   具体操作可参考：[LVS-NAT 原理介绍和配置实践 | HelloDog (wsgzao.github.io)](https://wsgzao.github.io/post/lvs-nat/)

2. VS/TUN模式

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220503211531.png)

   IP隧道的概念：将一个IP报文封装在另一个IP报文内。

   这种配置方式的条件或者特点是：

   - 后端应用服务器必须要支持ip隧道功能，以便配置VIP，这个VIP和调度器的VIP地址一样；
   - 一般要求VIP,DIP和后端应用服务器都是公网地址，但是如果只是内部应用则无所谓；
   - 调度器收到请求后在原报文外面封装一层报文，目的地址改为RIP，源地址改为DIP，后端服务器接受到报文后解开第一层后发现目的地址为自己的VIP地址，则会回复响应，然后直接直接发送给客户端；
   - 可以看出它不支持按照端口转发；
   - 比较适合跨机房的需求。

   具体操作可参考：[LVS-TUN 原理介绍和配置实践 | HelloDog (wsgzao.github.io)](https://wsgzao.github.io/post/lvs-tun/)

3. VS/DR模式

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220503221351.png)

   类似于TUN模式。

   这种配置方式的条件或者特点是：

   - 调度器和后端应用服务器必须在一个物理网络，并且共享一个VIP；
   - 后端应用服务器VIP配置在一个Non-ARP的设备上，用于接受内部网络请求，对外不可见；
   - 调度器将数据帧的MAC地址改为后端应用服务器的MAC地址，再将修改后的数据帧在与后端应用服务器组的局域网上发送；
   - 后端应用服务器获取报文后，发现报文的目标地址VIP是在本地的网络设备上，就处理这个报文，然后根据路由表将响应报文直接返回给客户；
   - 也不支持端口映射。

   具体操作可参考：[LVS-DR 原理介绍和配置实践 | HelloDog (wsgzao.github.io)](https://wsgzao.github.io/post/lvs-dr/)

4. VS/FULLNAT

   注意：

    - 这种模式不是LVS原生支持的，是阿里开源的：[ FULLNAT](https://github.com/alibaba/LVS) 
    - <font face="微软雅黑" color=red size=3>实现原理与kube-proxy一致，可由此思考k8s内部网络基于ipvs的实现。</font>

   

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220503224750.png)

   NAT的升级版。

   这种配置方式的条件或者特点是：

   - 调度器在收到请求和回复时会同时修改源和目的地ip；
   - 不需要将后端应用服务器的网关修改为调度器的DIP；
   - 这种模式没有组网要求；
   - 支持端口映射；
   - 所有的流量都经过调度器，因此和NAT模式一样，调度器是瓶颈。

   具体操作可参考：[Lvs的Fullnat模式部署安装_蓝叶子Water的技术博客_51CTO博客](https://blog.51cto.com/dellinger/2299662)

# 4.比较

| 要求         | NAT模式            | TUN模式            | DR 模式               |
| ------------ | ------------------ | ------------------ | --------------------- |
| 服务器       | 任意操作系统       | 支持IP隧道         | 支持Non-arp的虚拟设备 |
| 网络         | LAN                | LAN/WAN            | LAN                   |
| 支持的节点数 | 10~20              | 100                | 100                   |
| 网关         | 调度器             | 自己的网关         | 自己的网关            |
| 安全性       | 较好，后端应用隐藏 | 较差，节点全部暴露 | 较差，节点全部暴露    |
| 特点         | NAT                | 封装ip             | 修改MAC地址           |
| 配置复杂度   | 简单               | 复杂               | 复杂                  |

由于FULLNAT不是内核原生支持的，需要patch，这里就不放进去比较了。实际上综合下来NAT适合相对的小型应用以及快速部署，TUN适合跨机房的异地应用集群，DR适合性能要求比较高的场合。

# 5.关于Keepalived

官方文档：[Introduction — Keepalived 1.2.15 documentation](https://keepalived.readthedocs.io/en/latest/introduction.html)

我们了解了上面的LVS后，可以发现调度器的单点故障可以通过heartbeat服务来实现，但是并没有提到后端服务的可用性，所以后来为了使得调度器可以自动识别后端出现故障的服务，便出现了Keepalived。它当初一开始也只是为了监控LVS后端应用服务的状态，但是后来又加入了VRRP协议实现了调度器自身的高可用，避免了调度器的单点故障，所以在heartbeat配置复杂而keepalived更适合web业务的情况下都转向了Keepalived。关于heartbeat和Keepalived，可以参考Haproxy作者的观点： [Hypermail (formilux.org)](http://www.formilux.org/archives/haproxy/1003/3259.html)

一般我们实现HA会经历以下过程：

1. 只是用Keepalived+Nginx或者其他LB软件，实现较为简单的HA，这时候所有的流量都只走LB的Master节点。此时类似NAT模式，瓶颈都在一台节点上；
2. 如果流量升高，发现前端的LB已经成为瓶颈，这时候就需要同时使用LB的所有节点来分担负载，就有必要考虑加入LVS。此时可以先尝试使用NAT或者FULLNAT的模式，把Keepalived+调度器+LB软件放在相同的节点上（其他模式在同节点无法配置），这样不需要增加硬件，但是LVS可以将流量同时分发到LB的Master和Backup节点，从而降低每个LB的单点负载，只不过这种方式只是减轻了LB处理的负载，流量最后仍然是要走调度器的Master节点的，还是有网络I/O瓶颈；
3. 当上面的网络I/O已经开始影响业务时，需要把调度器+Keepalived独立出来，此时调度器和Keepalived所在的节点配置可以较低，因为作为调度器，在TUN或者DR模式下，它只有请求流量，不会对性能造成太大负担。此时，一般来讲足以应付绝大多数场景。
4. 当I/O仍然存在瓶颈时，我们可以将调度器纵向扩展，或者LB节点纵向或者横向扩展。



本文参考了以下网页：

[LVS负载均衡原理 - 乐章 - 博客园 (cnblogs.com)](https://www.cnblogs.com/zhangxingeng/p/10497279.html)

[超详细！一文带你了解 LVS 负载均衡集群！ - 云+社区 - 腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1657962)