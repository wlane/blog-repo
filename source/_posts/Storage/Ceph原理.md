---
title: Ceph原理
tags:
- ceph
categories:
- storage
---

参考：[Ceph](https://docs.ceph.com/en/latest/)

# 1.介绍

1. 简介

   Ceph是一个基于软件定义的分布式存储系统，它可以提供对象存储和块存储服务，同时还能创建文件系统以提供文件存储服务。

2. 特点

   [Benefits of Ceph](https://ceph.io/en/discover/)

   - 可扩展性
     - 去中心化架构
     - 扩展灵活，没有停机时间
     - 不依赖于特定硬件，可以指数方式增长
   - 高可用性
     - 灵活控制副本数
     - 支持故障隔离，数据具有强一致性
     - 具有自动管理和恢复的功能
     - 没有单点故障和可伸缩性的限制
   - 高性能
     - 采用CRUSH算法，数据均匀分布，并行度高
     - 运行节点不需要专有ICT基础架构，支持上千个存储节点
     - 可以实现各类副本放置规则，可实现跨机柜或者机房的容灾

# 2.架构

如下图：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220710182757.png)

核心组件：

- RADOS：可靠、自治、分布式对象存储（Reliable Autonomic Distributed Object Store）。Ceph中的一切都是以对象的形式存储，RADOS就负责存储这些对象。它负责数据的复制、恢复和故障检测，以及数据的平衡和迁移；
- librados：RADOS对外的接口，支持多种语言。它是对外提供各种存储服务的基础；
- MON：监视器（Monitor）。它用来跟踪整个集群的健康状态，一个MON为每一个组件维护一个独立的map（元数据），不存储实际数据。集群节点和客户端的定期与MON通信确保自己持有的是最新的map；
- 

