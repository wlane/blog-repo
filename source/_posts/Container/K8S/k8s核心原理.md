---
title: k8s核心原理
date: 2022年4月28日 00点11分
tags:
- k8s
categories:
- 读书笔记
---

《kubernetes权威指南》第一版

# kubernetes结构图


![](https://images-pigo.oss-cn-beijing.aliyuncs.com/k8s-arch.jpg)

主要分为五个部分：apiserver，controller manager，scheduler，安全机制和网络原理，如下图：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/k8s-principle.jpg)


## apiserver
-------------

1.如何访问k8s的api
	
　　通过编程方式访问
　　通过curl命令访问
　　通过命令行工具kubectl访问

2.通过API Server 访问Node、Pod、Service

　　/api/v1/proxy/nodes/{name}/*

3.集群功能模块之间的通信

　　集群内的功能模块通过API Server将信息存入etcd，其他模块通过API Server读取这些信息，从而实现模块之间的交互


## controller manager
---------------------

1.Replication Controller

　　确保任何时刻一个RC关联的Pod都能保持一定数量的Pod副本处于正常运行状态，实现自动创建、补足、替换、删除Pod副本
　　常用使用模式：
　　(1)重新调度
　　(2)弹性伸缩
　　(3)滚动更新  ***

2.Node Controller

　　发现、管理和监控集群中的各个Node节点，kublet启动时通过API Server将节点信息写入etcd

3.ResourceQuota Controller

　　负责实现k8s的资源配额管理，通过两种准入控制机制（Admission Control）来实现，LimitRanger作用于Pod和Container上，ResourceQuota作用于Namaspace上

4.Namespace Controller

　　定时通过API Server从etcd中读取Namespace信息并对Namespace做相应处理

5.ServiceAccount Controller与Token Controller

　　ServiceAccount Controller在Controller Manager启动时被创建，监听Service Account的删除和Namespace的创建、修改事件
　　Token Controller监听Service Account和Secret的创建、修改和删除事件

6.Service Controller与 Endpoint Controller

　　什么是Service（微服务）
　　Service如何访问到后端的Pod（Endpoint）
　　如何找到Service（内部访问方式（虚拟ip+端口），外部访问方式（Nodeport，LoadBalancer，ingress））
　　Service Controller监控Service的变化，Endpoint Controller监控Service和Pod的变化


## Scheduler
------------

将待调度的Pod按照特定的调度算法和调度策略绑定集群中的某个合适的Node上，并将绑定信息写入到etcd中，接下来kubelet获取到Pod绑定事件，生成具体的Pod

1.预选调度过程（预选策略）

　　（1）NoDiskConfilct
　　（2）PodFitsResource
　　（3）PodSelectorMatches
　　（4）PodFitsHost
　　（5）CheckNodeLabelPresence
　　（6）CheckServiceAffinity
　　（7）PodFitsPorts

2.确定最优节点（优选策略）

　　（1）LeastRequestedPriority
　　（2）CalculateNodeLabelPriority
　　（3）BalancedResourceAllocation


## 安全机制
---------

1.Authentication认证

　　API的调用使用CA、Token和HTTP Base方式实现用户认证

2.Authorization授权

　　授权是认证后的独立步骤，用于API Server主要端口的所有http访问，有三种模式：
　　（1）AlwaysDeny
　　（2）AlwaysAllow
　　（3）ABAC

3.Admission Control准入控制

　　用于拦截所有经过认证和鉴权后的访问API Server请求的可插入代码（或插件），参考ResourceQuota Controller

4.Secret私密凭据

　　保管私密数据，比如密码、OAuthTokens、SSH Keys等信息
　　三种类型：
　　（1）Opaque
　　（2）Dockercfg
　　（3）ServiceAccount
　　三种使用方式：
　　（1）创建Pod时，通过为Pod指定Service Account来自动使用该Secret
　　（2）通过挂载该Secret到Pod来使用它
　　（3）创建Pod时，指定Pod的spc.ImagePullSecrets来引用它

5.Service Account

　　多个Secret的集合，包含两类Secret：
　　（1）普通Secret，用于访问API Server
　　（2）imagePullSecret，用于下载容器镜像
　　会一起工作的功能模块：
　　（1）Admission Controller
　　（2）Token Controller
　　（3）Service Account Controller



## 网络原理
----------

1.K8s网络模型

　　基础原则：每个Pod都有一个独立的ip地址，而且假定所有的Pod都在一个可以直连、偏平的网络空间，不管Pod是否在同一个Node中，都要求能通过对方的ip直接进行访问（IP_per_Pod）
　　要求：
　　（1）所有容器都可以在不用NAT的方式下和别的容器通信
　　（2）所有节点都可以在不用NAT的方式同所有容器通信，反之亦然
　　（3）容器的地址和别人看到的地址是同一个地址

2.Docker的网络基础

　　2.1.网络名称空间
　　　　独立的网络协议栈被隔离到不同的命名空间中后形成了网络命名空间，不同的网络命名空间互相是隔离的，彼此无法通信
　　2.2.Veth设备对
　　　　利用它可以直接将两个网络命名空间连接起来，在不用的网络命名空间之间进行通信
　　2.3.网桥
　　　　提供了网络设备之间互相转发数据的二层设备，linux网桥既可以看做一个二层，也可以看做一个三层设备
　　2.4.iptables/Netfilter
　　　　实现用户自定义数据包的处理
　　2.5.路由
　　　　在IP层决定数据的发送和转发

3.Docker的网络实现

　　Docker支持的4类网络模式：
　　（1）host模式
　　（2）container模式
　　（3）none模式
　　（4）bridge模式，也是k8s管理模式下通常使用的模式
　　问题：
　　在bridge模式下，同一台机器内的容器之间可以相互通信，不同主机上的容器不能够相互通信，甚至可能有相同的网络地址段，因此为了跨节点相互通信，必须通过主机端口来代理，这种做法意味着在要协调好不同容器的代理端口的分配，再加上服务注册发现等机制也十分困难

4.k8s的网络实现

　　4.1.容器到容器的通信
　　　　在同一Pod内的容器（不用Pod内的容器不直接访问）共享同一个网络命名空间，可以直接访问
　　4.2.Pod之间的通信
　　　　（1）同一个Node上
　　　　关联在同一个Docker0网桥上，是可以直接通信的
　　　　（2）不同Node之间
　　　　需要额外的网络配置，谷歌的GCE环境和部分公有云都支持，私有云使用一些开源组件
　　4.3.Pod到Service之间的通信
　　　　Service是一组Pod的抽象，真正将Service作用落实的是背后的kube-proxy进程。每个Node上面都有一个kube-proxy进程，可看作是Service的透明代理和负载均衡器，他会将某个Service的访问请求转发到后端的多个Pod实例上
　　　　访问Service的请求，不论是用Cluster IP+TargetPort的方式还是用节点机IP+NodePort的方式，都被节点机的Iptables规则重定向到kube-proxy监听Service服务代理端口
　　4.4.外部到内部的访问
　　　　主要有三种
　　　　（1）外部没有负载均衡器，直接通过访问内部的负载均衡器来访问Pod（NodePort）
　　　　（2）外面有负载均衡器，通过外部负载均衡器直接访问内部的Pod（LoadBalance）
　　　　（3）ingress

5.开源网络组件

　　（1）Flannel
　　（2）Open vSwitch
　　（3）直接路由





