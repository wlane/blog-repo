---
title: docker与iptables
tag:
- iptables
categories:
- docker
---

# 1.默认配置

在未安装docker之前的iptables配置如下，此时我们关闭了Firewalld服务，全部链表都为空：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220430215242.png)

安装并启动docker-ce v20.10.14后，增加了虚拟网桥docker0：

​	![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220501221724.png)

默认的filter表结果如下：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220501221057.png)

可以看出FILTER表中增加了DOCKER,DOCKER-ISOLATION-STAGE-1,DOCKER-ISOLATION-STAGE-2,DOCKER-USER四种链，而这四种链都是FORWARD的子链，对于符合FORWARD条件的处理逻辑如下：DOCKER-USER-> DOCKER-ISOLATION-STAGE-1-> DOCKER-ISOLATION-STAGE-2-> DOCKER。

NAT表如下：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220501221254.png)

可以看出NAT表中增加了DOCKER链，这个链是PREROUTING和OUTPUT的子链，并且POSTROUTING链多了一条规则。

# 2.新建服务

此时我新建一个nginx容器，暴露端口80：

~~~shell
$ sudo docker run -d -p 80:80 nginx:latest
3b4167054cbe136161541545f6cfb3d1f0936080df6986fd904db8d3ce3304cc
$ sudo docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                               NAMES
3b4167054cbe   nginx:latest   "/docker-entrypoint.…"   6 seconds ago   Up 5 seconds   0.0.0.0:80->80/tcp, :::80->80/tcp   clever_carson
$ ip a
......
5: veth0c9554e@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether 2a:ff:ea:2e:42:5f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::28ff:eaff:fe2e:425f/64 scope link
       valid_lft forever preferred_lft forever
$ sudo docker inspect --format='{{.NetworkSettings.IPAddress}}' 3b4167054cbe
172.17.0.2
~~~

此时iptables的Filter表规则如下：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220501222345.png)

可以看出DOCKER链多了一条规则。

NAT表如下：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220501222746.png)

可以看出POSTROUTING和DOCKER都多了一条规则。

路由表：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220501234559.png)

# 3.分析

前置知识：[iptables原理 | repo (xwangwang.com)](https://www.xwangwang.com/2022/04/30/Security/iptables原理/)

此时一个客户端想要访问 http://10.0.12.3（走eth0）或者http://localhost（走lo），则防火墙过滤流程如下：

1. 先经过NAT的PREROUTING，这里使用到了DOCKER子链，默认规则为地址类型匹配目的地址是本地地址的就放行走DOCKER子链，这里默认全部放行；
2. 在NAT的DOCKER子链中，由于进来的网络接口不是docker0，而是物理网卡eth0，所以匹配了DNAT的规则，这里将目标地址换成了172.17.0.2:80；
3. 经过路由表，这里直接到达docker0，从而不经过eth0，走到了Filter表的FORWARD链；
4. 默认匹配所有从而经过DOCKER-USER，这里直接返回RETURN；
5. 默认匹配所有符合条件，走DOCKER-ISOLATION-STAGE-1，从而走到DOCKER-ISOLATION-STAGE-2，但是不匹配DOCKER-ISOLATION-STAGE-2的条件，因为它的入口不是docker0，所以还是走下面的RETURN返回到FORWARD链；
6. 下面的ACCEPT不符合条件，因为要求状态是related和established，但是这里是new；
7. 走到下面DOCKER，符合出口是docker0的要求，于是走到了下面的DOCKER子链；
8. 在DOCKER链中发现入口不是docker0，出口时docker0，destination是172.17.0.2，dpt是80，所以符合要求，走这一条规则；
9. 最后是要走NAT的POSTROUTING，这里是两条自动做SNAT伪装的MASQUERADE规则，简单描述就是从哪个口出去就伪装成source是哪个口的ip，但是我们这里的数据库包不符合条件，所以走默认的ACCEPT目标。
10. 至此整个iptables的包过滤流程结束，允许访问docker0上的172.17.0.2:80。

总结：这种流程最重要的是看NAT和Filter的DOCKER链。

