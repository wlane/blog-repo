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

核心组件及概念：

- [RADOS](https://ceph.com/assets/pdfs/weil-rados-pdsw07.pdf)：可靠、自治、分布式对象存储（Reliable Autonomic Distributed Object Store）。Ceph中的一切都是以对象的形式存储，它是Ceph存储系统的核心，RADOS就负责存储这些对象。当Ceph集群接收到来自客户端的写请求时，CRUSH算法首先计算出存储位置，然后这些信息传递到RADOS层进行进一步处理。RADOS以小对象的形式将数据分发到集群内的所有节点，最后将这些对象存储在OSD中。它负责数据的复制、恢复和故障检测，以及数据的平衡和迁移。RADOS包含两个核心组件：MON和OSD。

  - [MON](https://docs.ceph.com/en/latest/cephadm/services/mon)：监视器（Monitor）。它用来跟踪整个集群的健康状态，一个MON为每一个组件维护一个独立的map（元数据），不存储实际数据。集群节点和客户端定期与MON通信确保自己持有的是最新的map。通常一个集群有多个MON，但是一个时刻只有一个活动状态的MON；
  - [OSD](https://docs.ceph.com/en/latest/cephadm/services/osd/)：Object Storage Daemon，Ceph集群中最重要的一个基础组件，它负责将数据以对象的形式存储在节点的磁盘中。任何读写操作，客户端都是先从MON那里拿到组件的map，然后直接和OSD通信进行I/O操作。Ceph集群中推荐的做法是为节点上的每一个物理磁盘创建一个OSD的守护进程，然后根据副本配置每个对象有一个主副本和几个辅副本，存放辅副本的OSD受主副本的OSD控制，一旦主副本出现故障，辅副本可以转化为主副本。目前Ceph推荐的文件系统是XFS。对数据库的写操作都是先到日志，再到存储。

- [librados](https://docs.ceph.com/en/latest/rados/api/librados-intro)：本地的C语言库，RADOS对外的接口，支持多种语言。它是对外提供各种存储服务的基础；

- [radosgw/rgw](https://docs.ceph.com/en/latest/radosgw/)：RADOS对象网关，通过这个网关对外提供对象存储功能。它是一个http请求的代理，它可以将http请求转换为RADOS，也可以把RADOS转换为http请求，从而提供restful接口，兼容s3和swift接口。它的调用架构如下：

  ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220711232214.png)

  如上图，Rados网关通过守护进程（RGW）与librgw交互，通过librgw又直接与librados交互，它支持的API有上图中的三种；

- [rbd](https://docs.ceph.com/en/latest/rbd/)：RADOS Block Device，通过rbd对外提供块存储，可以像其他磁盘一样挂载到服务器上使用。它的主要调用架构如下：

  ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220711232901.png)

  RADOS将块设备的对象以分布式模式存储在Ceph集群中，然后一旦操作系统的KRBD通过librados的接口来映射块设备到主机，它可以被视为裸设备或者格式化为具体的文件系统来挂载。当客户端写入数据时，librbd库会将数据块映射到块设备的对象，从而实现对数据的操作；

- [MDS](https://docs.ceph.com/en/latest/cephfs/add-remove-mds/)：元数据服务器，它只有存在文件存储Cephfs时才用到，主要用来跟踪文件层次结构并存储元数据；

- [Cephfs](https://docs.ceph.com/en/latest/cephfs/)：通过此组件对外提供文件存储服务。它在RADOS层之上提供了一个兼容POSIX的文件系统。使用MDS作为守护进程，负责管理其元数据并将它和其他数据分开。CephFS使用cephfuse模块（FUSE）扩展其在用户空间文件系统方面的支持（就是将CephFS挂载到客户端机器上）。它还允许直接与应用程序交互，使用libcephfs库直接访问RADOS集群；

- [MGR](https://docs.ceph.com/en/latest/mgr/)：Manager daemon，ceph的管理组件，用来收集整个集群的状态。

# 3.原理

