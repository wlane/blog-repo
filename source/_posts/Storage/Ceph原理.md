---
title: Ceph原理
tags:
- ceph
categories:
- storage
---

参考：[<font color=\#6495ED><u>Ceph</u></font>](https://docs.ceph.com/en/latest/)

# 1.介绍

1. 简介

   Ceph是一个基于软件定义的分布式存储系统，它可以提供对象存储和块存储服务，同时还能创建文件系统以提供文件存储服务。

2. 特点

   [<font color=\#6495ED><u>Benefits of Ceph</u></font>](https://ceph.io/en/discover/)

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

- [<font color=\#6495ED><u>RADOS</u></font>](https://ceph.com/assets/pdfs/weil-rados-pdsw07.pdf)：可靠、自治、分布式对象存储（Reliable Autonomic Distributed Object Store）。Ceph中的一切都是以对象的形式存储，它是Ceph存储系统的核心，RADOS就负责存储这些对象。当Ceph集群接收到来自客户端的写请求时，CRUSH算法首先计算出存储位置，然后这些信息传递到RADOS层进行进一步处理。RADOS以小对象的形式将数据分发到集群内的所有节点，最后将这些对象存储在OSD中。它负责数据的复制、恢复和故障检测，以及数据的平衡和迁移。RADOS包含两个核心组件：MON和OSD。

  - [<font color=\#6495ED><u>MON</u></font>](https://docs.ceph.com/en/latest/cephadm/services/mon)：监视器（Monitor）。它用来跟踪整个集群的健康状态，一个MON为每一个组件维护一个独立的map（元数据），不存储实际数据。集群节点和客户端定期与MON通信确保自己持有的是最新的map。通常一个集群有多个MON，但是一个时刻只有一个活动状态的MON；
  - [<font color=\#6495ED><u>OSD</u></font>](https://docs.ceph.com/en/latest/cephadm/services/osd/)：Object Storage Daemon，Ceph集群中最重要的一个基础组件，它负责将数据以对象的形式存储在节点的磁盘中。任何读写操作，客户端都是先从MON那里拿到组件的map，然后直接和OSD通信进行I/O操作。Ceph集群中推荐的做法是为节点上的每一个物理磁盘创建一个OSD的守护进程，然后根据副本配置每个对象有一个主副本和几个辅副本，存放辅副本的OSD受主副本的OSD控制，一旦主副本出现故障，辅副本可以转化为主副本。目前Ceph推荐的文件系统是XFS。对数据的写操作都是先到日志，再到存储。

- [<font color=\#6495ED><u>librados</u></font>](https://docs.ceph.com/en/latest/rados/api/librados-intro)：本地的C语言库，RADOS对外的接口，支持多种语言。它是对外提供各种存储服务的基础；

- [<font color=\#6495ED><u>radosgw/rgw</u></font>](https://docs.ceph.com/en/latest/radosgw/)：RADOS对象网关，通过这个网关对外提供对象存储功能。它是一个http请求的代理，它可以将http请求转换为RADOS，也可以把RADOS转换为http请求，从而提供restful接口，兼容s3和swift接口。它的调用架构如下：

  ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220711232214.png)

  如上图，Rados网关通过守护进程（RGW）与librgw交互，通过librgw又直接与librados交互，它支持的API有上图中的三种；

- [<font color=\#6495ED><u>rbd</u></font>](https://docs.ceph.com/en/latest/rbd/)：RADOS Block Device，通过rbd对外提供块存储，可以像其他磁盘一样挂载到服务器上使用。它的主要调用架构如下：

  ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220711232901.png)

  RADOS将块设备的对象以分布式模式存储在Ceph集群中，然后一旦操作系统的KRBD通过librados的接口来映射块设备到主机，它可以被视为裸设备或者格式化为具体的文件系统来挂载。当客户端写入数据时，librbd库会将数据块映射到块设备的对象，从而实现对数据的操作；

- [<font color=\#6495ED><u>MDS</u></font>](https://docs.ceph.com/en/latest/cephfs/add-remove-mds/)：元数据服务器，它只有存在文件存储Cephfs时才用到，主要用来跟踪文件层次结构并存储元数据；

- [<font color=\#6495ED><u>Cephfs</u></font>](https://docs.ceph.com/en/latest/cephfs/)：通过此组件对外提供文件存储服务。它在RADOS层之上提供了一个兼容POSIX的文件系统。使用MDS作为守护进程，负责管理其元数据并将它和其他数据分开。CephFS使用cephfuse模块（FUSE）扩展其在用户空间文件系统方面的支持（就是将CephFS挂载到客户端机器上）。它还允许直接与应用程序交互，使用libcephfs库直接访问RADOS集群；

- [<font color=\#6495ED><u>MGR</u></font>](https://docs.ceph.com/en/latest/mgr/)：Manager daemon，ceph的管理组件，用来收集整个集群的状态。

# 3.原理

参考：[<font color=\#6495ED><u>Architecture — Ceph Documentation</u></font>](https://docs.ceph.com/en/latest/architecture/)

1. 集群map

   如下图，来自于上面参考的地址：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220715001828.png)

   需要注意这里的map代表了组件的各类元数据，client都是首先通过map来获取集群参数，再进行下一步的操作。

2. 数据存储的方式

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220712224217.png)

   如上图，无论对外提供的是对象存储、块存储还是文件存储，Ceph集群接受到的数据都是以RADOS对象来存储的。每个对象都保存在OSD上，由OSD守护进程来处理在磁盘上的读、写和复制。在以前后端使用Filestore存储引擎的时候，一个OSD对象就是一个文件，但是在默认使用新的BlueStore存储引擎后，OSD对象以一种类似数据库的方式来存储。

   关于FileStore和BlueStore的区别，可以看这里：[Storage Devices — Ceph Documentation](https://docs.ceph.com/en/latest/rados/configuration/storage-devices/#osd-backends)

   OSD对象结构如下：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220712225824.png)

   每一个对象都包含指针、二进制数据和包含一系列name/value集合的元数据。这里的指针是整个集群内全局唯一的标识符。

3. CRUSH算法

   为了实现高可用和可扩展性，实现去中心化，Ceph使用了一种叫CRUSH的算法来控制数据的存储和检索。

   CRUSH算法，全称Controlled Replication Under Scalable Hashing，是一个可控的、可扩展的、分布式的副本数据放置算法， 它通过计算数据存储位置来确定如何存储和检索数据。关于它的具体说明，可参考：[CRUSH](https://ceph.com/assets/pdfs/weil-crush-sc06.pdf)

   - PG

     归置组（[Placement Groups — Ceph Documentation](https://docs.ceph.com/en/latest/rados/operations/placement-groups/)），它是一组对象的逻辑集合，通过复制它到不同的OSD上实现存储的可靠性。根据Pool的复制级别，每个PG的数据会被复制分发到多个OSD上，可以将PG看成是一个个的容器，这个容器内部包含了多个对象并被映射到多个OSD上。关于它的取值可以参考前面PG说明连接中的：

     ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220714225832.png)

   - Pool

     ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220714233327.png)

     存储池（[Pools — Ceph Documentation](https://docs.ceph.com/en/latest/rados/operations/pools/#pools)），它是存储对象的逻辑分区，每个Pool都包含一定数量的PG，可以把它看成namespace，包含多个可看成容器的PG，从而能把对象映射到不同的OSD上。由于pool分布在整个集群的节点上，所以可以提供的弹性。Pool还支持快照功能以及故障隔离域：

   - CRUSH算法

     ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220714232958.png)

     可参考：[CRUSH Maps — Ceph Documentation](https://docs.ceph.com/en/quincy/rados/operations/crush-map/)

     如上图，首先客户端通过和MON联系获取到集群map值，然后使用对象名称和池名将数据转换为一个对象，接下来对生成的对象名进行hash处理得到一个十六进制的值，然后将这个值跟总的PG数进行取模处理，这时根据pool的id值可以得到最终是分配到了哪个pool的哪个PG上面，上图中的结果是4.32的PG。接下来就是CRUSH算法主要做的事情：计算PG-->OSD的映射关系。

     首先给不同的OSD不同的权重，比如根据容量，1T大小的权重为1，500G的就是0.5。然后使用CURSH算法中的Straw算法来将权重和相关id一起运算得出一个值，将每个OSD的值比较得到的结果最高的OSD，就是目标OSD。整体上看，OSD的权重越大，选中的概率越高。确定一个一个OSD之后，那怎么在剩下的OSD中选择副本呢？其实就是将前面计算的过程中涉及的常量加1，然后再重复计算一次，选择和之前编号不一样的OSD即可。

4. 数据写入流程

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220715002114.png)

   如上图，客户端首先写入到Pool中的主OSD，然后主OSD会将同样的数据复制到其它OSD中，等待它们确认写入成功后，返回ACK给主OSD，此时主OSD才会返回一个确认写入的信号给客户端。

# 4.性能调优

