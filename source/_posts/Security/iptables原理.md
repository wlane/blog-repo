---
title: iptables原理
tags:
- iptables
categories:
- security
---

# 1.介绍

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220430221006.png)

如上图，iptables是管理防火墙规则的处于用户态的工具，最终真正实现的是处于内核态的netfilter模块。

netfilter模块是linux内核中的包过滤模块，主要工作在网络层，启动后自动加载，不存在由其他服务控制的说法。

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220430234218.png)

可以看到ip_tables的模块在netfilter目录下。

参考：[2.8.9. IPTables Red Hat Enterprise Linux 6 | Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/security_guide/sect-security_guide-iptables)

# 2.基本概念

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220430221950.png)

1. 链

   链用来对数据包进行过滤或处理。

   上图中的各种chain，主要有PREROUTING,INPUT,OUTPUT,FORWARD,POSTROUTING五种：

   PREROUTING：在进行路由选择前处理数据包（入站过滤）；
   INPUT：处理入站数据包；
   OUTPUT：处理出站数据包；
   FORWARD：处理转发数据包 ；
   POSTROUTING：在进行路由选择后处理数据包（出站过滤）。

2. 表

   表用来存放各种规则链。

   主要有以下四种：

   RAW：确定是否对该数据包进行状态跟踪，包括PREROUTING和OUTPUT链；
   MANGLE：为数据包设置标记，标记之后可以分流、限流，包括所有五种链；
   NAT：修改数据包中的源、目标IP地址或端口，包括PREROUTING，OUTPUT和POSTROUTING三种链（[2.6.35版本的内核之后还有INPUT](https://github.com/torvalds/linux/commit/c68cd6cc21eb329c47ff020ff7412bf58176984e)）；
   FILTER：确定是否放行该数据包，即过滤，包括INPUT，OUTPUT和FORWARD三种链。

3. 其他注意事项

   不指定表名时，默认指filter表；

   选项、链名、控制类型使用大写字母，其余均为小写；

   常见控制类型：

   ​	ACCEPT：允许通过
   ​	DROP：直接丢弃，不给出任何回应
   ​	REJECT：拒绝通过，必要时会给出提示
   ​	LOG：记录日志信息，然后传给下一跳规则继续匹配

# 3.测试

~~~shell
# 默认值
$ sudo iptables -L -n --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination

Chain FORWARD (policy DROP)
num  target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination
# 不允许192.168.1.100访问本机的2222，但是本机可以使用端口2222主动连接192.168.1.100的80端口（NEW）
$ sudo iptables -A INPUT  -s 192.168.1.100 -p tcp --dport 2222 --sport 80 -m conntrack --ctstate ESTABLISHED -j ACCEPT
$ sudo iptables -A OUTPUT -d 192.168.1.100 -p tcp --sport 2222 --dport 80 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
$ sudo iptables -L -n --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    ACCEPT     tcp  --  192.168.1.100        0.0.0.0/0            tcp spt:80 dpt:2222 ctstate ESTABLISHED

Chain FORWARD (policy DROP)
num  target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination
1    ACCEPT     tcp  --  0.0.0.0/0            192.168.1.100        tcp spt:2222 dpt:80 ctstate NEW,ESTABLISHED
# SNAT策略，内网通过一个公网(eth0)出口，在公网出口所在的服务器上配置（POSTROUTING链）
# 之后所有在192.168.10.0/24的网段服务器网关都配置成这台服务器，出口服务器在路由转发的时候会将source地址改成公网地址，此后就可以和公网通信
# iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -o ppp0 -j MASQUERADE （适用于拨号情况下公网地址不固定的网络）
$ sudo iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -o eth0 -j SNAT --to-source 210.106.46.151
$ sudo iptables -t nat -L -n --line-numbers
。。。。。。
Chain POSTROUTING (policy ACCEPT)
num  target     prot opt source               destination
1    SNAT       all  --  192.168.10.0/24      0.0.0.0/0            to:210.106.46.151
# DNAT策略，公网客户端需要访问内网服务，在公网ip所在的服务器上（防火墙）配置（PREROUTING）
# 将公网访问的目标ip改成防火墙的内网ip，以便能够正确找到所在服务
$ sudo iptables -t nat -A PREROUTING -d 210.106.46.151 -p tcp --dport 80 -j DNAT --to-destination 192.168.10.10
$ sudo iptables -t nat -L -n --line-numbers
Chain PREROUTING (policy ACCEPT)
num  target     prot opt source               destination
1    DNAT       tcp  --  0.0.0.0/0            210.106.46.151       tcp dpt:80 to:192.168.10.10
。。。。。。
~~~

