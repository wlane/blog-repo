---
title: Firewalld原理
tags:
- firewalld
categories:
- security
---

# 1.与iptables的区别

见下图：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220501005710.png)

可以看出Firewalld只是另外一种防火墙管理工具而已，最终都是netfilter模块起作用。它可以基于CLI和[GUI](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-installing_firewall-config)来操作。

# 2.zone

firewalld加入了zone的概念，也就是说它预先提供了一些防火墙策略的合集，用户可以根据不用使用场景快速切换，而不再需要不断修改iptables规则：

| 区域     | 默认规则策略                                                 |
| :------- | :----------------------------------------------------------- |
| trusted  | 允许所有的数据包                                             |
| home     | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh、mdns、ipp-client、amba-client与dhcpv6-client服务相关，则允许流量 |
| internal | 等同于home区域                                               |
| work     | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh、ipp-client与dhcpv6-client服务相关，则允许流量 |
| public   | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh、dhcpv6-client服务相关，则允许流量 |
| external | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh服务相关，则允许流量 |
| dmz      | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh服务相关，则允许流量 |
| block    | 拒绝流入的流量，除非与流出的流量相关                         |
| drop     | 拒绝流入的流量，除非与流出的流量相关                         |

# 3.CLI操作

~~~shell
# 获取默认的策略
$ sudo firewall-cmd --get-default-zone
public
$ sudo firewall-cmd --zone=public --list-all
public
  target: default
  icmp-block-inversion: no
  interfaces:
  sources:
  services: dhcpv6-client ssh
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:

# 切换区域
$ sudo firewall-cmd --set-default-zone=external
$ sudo firewall-cmd --get-default-zone
external

# 加入某端口，并且永久生效。这里可以看出并不需要重启相应服务，也就是说它是动态更新的
$ sudo firewall-cmd --zone=public --add-port=8080-8081/tcp
success
$ sudo firewall-cmd --permanent --zone=public --add-port=8080-8081/tcp
success
# 或者下列方法
$ sudo firewall-cmd --permanent --zone=public --add-port=8080-8081/tcp
success
$ sudo firewall-cmd --reload
success
# 都可以得出下列结果
$ sudo firewall-cmd --zone=public --list-ports
8080-8081/tcp
~~~

未来我们将都会转向使用Firewalld，参考：

[Chapter 5. Using Firewalls Red Hat Enterprise Linux 7 | Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-using_firewalls#sec-Getting_started_with_firewalld)

