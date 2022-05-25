---
title: etcd解析
tag:
- etcd
categories:
- key-value
---

[v3.5 docs | etcd](https://etcd.io/docs/v3.5/)

# 1. 基本原理

1. 基本架构如下图：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220508210013.png)

   它是一个分布式的，可靠的key-value存储系统，通常由奇数个节点组成，由Raft一致性算法完成分布式一致性协同。关于Raft算法我们参考：

   (TBD)

   boltdb：保存了集群的元数据和用户写入的数据；

   wal：Write Ahead Log（预写式日志），所有数据的修改在提交前，都需要先写入到WAL中，它是保持etcd数据一致性的重要组件：

   - 数据的undo和redo：因为所有的修改操作都被记录在 WAL 中，需要回滚或重做，只需要反向或正向执行日志中的操作即可；
   - 故障恢复：当数据遭到破坏时，可以通过执行WAL中的操作，使得数据恢复到损坏前的状态。

   snapshot：当日志不断增长时，需要snapshot对当前的应用状态做一次snapshot，以便能够将这个时间点之前的日志删除，从而清理日志占用的存储空间。

   gRPC server：发送到etcd服务的API请求都是gRPC调用，它是从etcd v3版本之后实现的，之前的版本还是http的调用。

2. 几个重要概念

   - MVCC：和传统数据库中的概念一致，也是为了避免对相同数据同时读写时发生冲突。简单来讲，就是一个数据可以有对应的多个revision，每次查询默认获取最新版本的值，当然也可以通过指定版本号的值获取历史数据，如下图：

     ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220508215848.png)

     其中revision的ID，是全局递增不重复的；

   - Watch：为了避免客户端的反复轮询， etcd 提供了 watch 机制。客户端 watch 一系列 key，当这些被 watch的 key 更新时， etcd 就会通知客户端：

     ~~~shell
     # 在第一个terminal中操作
     # 需要指定api版本，否则默认使用api v2，则命令不一样
     $ export ETCDCTL_API=3
     $ etcdctl put hello world1
     OK
     $ etcdctl put hello world2
     OK
     $ etcdctl watch hello -w=json
     
     # 打开另外一个terminal
     $ export ETCDCTL_API=3
     $ etcdctl put hello world3
     # 回到第一个terminal，可以看到下面出现了一行返回的记录
     $ etcdctl watch hello -w=json
     
     {"Header":{"cluster_id":14841639068965178418,"member_id":10276657743932975437,"revision":4,"raft_term":2},"Events":[{"kv":{"key":"aGVsbG8=","create_revision":2,"mod_revision":4,"version":3,"value":"d29ybGQz"}}],"CompactRevision":0,"Canceled":false,"Created":false}
     # 可以依次查询历史版本
     $ etcdctl get hello --rev=2
     hello
     world1
     $ etcdctl get hello --rev=3
     hello
     world2
     $ etcdctl get hello --rev=4
     hello
     world3
     ~~~

     这个功能对k8s来讲特别重要，基于这个功能，k8s才能得知部署的应用是否和预期的状态一致。

   - Lease：租约是分布式系统的重要概念，一般用来检查节点是否存活。

     ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220508234015.png)

     上图是一个10s的租约，将 key1 和 key2 两个 key value 绑定到这个租约之上，如果10s之内不做任何操作，那么10s之后自动过期，会清理掉key1和key2的数据。进程在访问etcd后创建key的同时会设置一个租约，并且开启keepalive方法，此时客户端会向服务端发送keepalive请求来不断保持租约，如果此时客户端挂了，则服务器etcd的数据在租约到期之后这个租约下的key都会被删除。

3. 主要版本

   目前etcd的API存在v2和v3版本，主要对应对etcd 2.0和etcd 3.0版本，默认使用v2版本。主要区别如下：

   | 区别类型       | etcd API v2                                        | etcd API v3                                                 |
   | -------------- | -------------------------------------------------- | ----------------------------------------------------------- |
   | 客户端通信方式 | http/1.x+JSON                                      | gRPC，基于http/2实现，基于 protobuf 来声明数据模型和RPC接口 |
   | key的过期机制  | TTL机制，客户端必须定期的进行刷新                  | 租约机制，可以实现服务注册和健康检查                        |
   | watch机制      | 事件机制，有滑动窗口的大小限制                     | 支持get和watch键的任意的历史版本记录                        |
   | 数据存储模型   | 维护了一个历史变更的窗口，默认保存最新的1000个变更 | 实现了MVCC，保存了key的所有历史记录变更                     |

# 2. 性能优化

由上一节的架构图我们可以看出，影响etcd性能的主要在四个方面：

- 数据库本身的各种锁机制，以及所在磁盘的I/O；
- Raft算法通过网络同步数据，所以受到网络I/O的影响；
- WAL的写入受磁盘I/O的影响；
- 客户端调用gRPC的网络影响。

所以我们可以从下面几个方向来优化：

1. 硬件层面：足够的cpu和memory，以及ssd硬盘，尽量独立部署，减少不同应用间的资源争夺；

2. 应用层面：

   可以参考： [【课时 17-深入理解 etcd：etcd 性能优化实践(陈星宇)】- CSDN](https://gitchat.csdn.net/columnTopic/5dba800736f9741cf5c4da4a)

# 3. 安全

1. 开启Basic认证和授权

   [Authentication Guide | etcd](https://etcd.io/docs/v3.2/op-guide/authentication/)

   v2和v3开启方式一样，都需要开启，并且无法通过ip来限制客户端

2. 开启TLS访问

   [Security model | etcd](https://etcd.io/docs/v3.2/op-guide/security/)

   通过证书来验证客户端权限