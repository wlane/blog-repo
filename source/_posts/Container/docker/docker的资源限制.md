---
title: docker的资源限制
tags:
- limit
categories:
- docker
---

[Runtime options with Memory, CPUs, and GPUs](https://docs.docker.com/config/containers/resource_constraints/)

# 1.概念

通常宿主机上都会运行多个容器，容器之间的隔离通过[Namespace](http://www.xwangwang.com/2022/05/08/Container/docker/docker%E7%9A%84%E8%B5%84%E6%BA%90%E9%9A%94%E7%A6%BB/)的机制来实现，那容器的资源使用量又通过什么来实现呢，毕竟容器在宿主机上也只是一种进程，不同的进程互相之间会有竞争关系。在有大量进程存在的情况下，我们都需要考虑对各个进程资源的限制，否则就会导致比如OOM killer(TBD)的结果。严重情况下会导致宕机事件的发生。

docker在这里采用了linux内核提供的一种叫CGroup的功能。它全称为control group，是用来限制进程组上使用的资源上限的，包括cpu，内存，磁盘，带宽等，还可以配置进程的优先级，审计，挂起和恢复等操作。

1. 几个基本概念

   - 任务（task）：任务指的是系统的一个进程；
   - 控制组（control group）：控制组就是受相同资源限制的一组进程；
   - 层级（hierarchy）：控制组可以组织成层级的形式，即一棵控制组组成的树。控制组树上的子节点控制组是父节点控制组的孩子，继承父控制组的特定的属性；
   - 子系统（subsystem）：控制器，比如CPU子系统就是控制CPU时间分配的控制器，子系统必须要附加到某一个具体的层级以后才会起作用，这个层级上的所有控制组都收到这个控制器的控制。

   详细概念可以参照：[CGroup 介绍](https://cloud.tencent.com/developer/article/1685657)

   我们以memory为例来说明：

   ~~~shell
   # 查看memory子系统下面的cgroup
   $ ll /sys/fs/cgroup/memory/
   total 0
   -rw-r--r--  1 root root 0 Apr 30 23:34 cgroup.clone_children
   。。。。。。
   -rw-r--r--  1 root root 0 Apr 30 23:34 memory.memsw.max_usage_in_bytes
   -r--r--r--  1 root root 0 Apr 30 23:34 memory.usage_in_bytes
   -rw-r--r--  1 root root 0 Apr 30 23:34 memory.use_hierarchy
   -rw-r--r--  1 root root 0 Apr 30 23:34 notify_on_release
   -rw-r--r--  1 root root 0 Apr 30 23:34 release_agent
   drwxr-xr-x 75 root root 0 May 13 23:51 system.slice
   -rw-r--r--  1 root root 0 Apr 30 23:34 tasks
   drwxr-xr-x  2 root root 0 May  1 20:09 user.slice
   drwxr-xr-x  3 root root 0 Apr 30 23:35 YunJing
   ~~~

   以上结果中memory父目录是一个根cgroup，YunJing目录是腾讯云服务器自带的子cgroup，可以看出这里的memory目录和下面的YunJing目录组成了一个层级，这个层级附加了一个叫memory的子系统，它控制了memory和YunJing下面的资源。

2. 具体实现

   CGroup使用了虚拟文件系统（[VFS](https://www.kernel.org/doc/html/latest/filesystems/vfs.html)）来将功能暴露给进程来调用，关于VFS可以参考：[虚拟文件系统(VFS)](https://www.modb.pro/db/44881)。既然涉及到文件系统，我们可以先看一下文件挂载：

   ~~~shell
   # 查看当前挂载的cgroup子系统
   $ mount  | grep cgroup
   tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
   cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
   cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_prio,net_cls)
   cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct,cpu)
   cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
   cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
   cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
   cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
   cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
   cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
   cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
   cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
   ~~~

   以上/sys/fs/cgroup/后面的值就是支持的cgroup子系统。系统挂载了这些cgroup系统之后就能通过直接在这些目录下面建立文件夹来创建对资源的控制和管理。

   从RHEL7的版本之后对cgroup的管理工具从libcgroup转移到了systemctl。systemctl管理的服务为systemd，而systemd就是用来管理系统资源的。为了管理的方便，这里面涉及到一个叫unit的概念，参考：（TBD）。cgroup中主要使用到了slice，scope和service三种类型：

   | 类型    | 作用                                                         |
   | ------- | ------------------------------------------------------------ |
   | slice   | 用于关联 Linux Control Group 节点，根据关联的 slice 来限制进程。一个管理单元的组。Slice 并不包含任何进程，仅仅管理由 service 和 scope 组成的层级结构。 |
   | scope   | systemd 从 bus 接口收到消息后自动创建。Scope 封装了任意进程通过 `fork()` 函数开启或停止的进程，并且在 `systemd` 运行时注册。例如：用户 sessions，容器和虚拟机。 |
   | service | 一个服务或者一个应用，具体定义在配置文件中。                 |

# 2.测试

1. 使用systemctl控制普通服务

   ~~~shell
   # 创建一个叫cgrouptest的服务，在名叫test的slice中运行
   $ sudo systemd-run --unit=cgrouptest --slice=test sleep 3600
   Running as unit cgrouptest.service.
   # 查看服务状态
   $ sudo systemctl status cgrouptest.service
   ● cgrouptest.service - /bin/sleep 3600
      Loaded: loaded (/run/systemd/system/cgrouptest.service; static; vendor preset: disabled)
     Drop-In: /run/systemd/system/cgrouptest.service.d
              └─50-Description.conf, 50-ExecStart.conf, 50-Slice.conf
      Active: active (running) since Sat 2022-05-14 21:40:58 CST; 5min ago
    Main PID: 9129 (sleep)
      CGroup: /test.slice/cgrouptest.service
              └─9129 /bin/sleep 3600
   
   May 14 21:40:58 VM-12-3-centos systemd[1]: Started /bin/sleep 3600.
   # 查询cgroup内容
   $ sudo systemd-cgls
   ├─1 /usr/lib/systemd/systemd --switched-root --system --deserialize 22
   ├─test.slice
   │ └─cgrouptest.service
   │   └─9129 /bin/sleep 3600
   ├─user.slice
   。。。。。。
   └─system.slice
   。。。。。。
   # cgroup信息
   $ cat /proc/9129/cgroup
   11:freezer:/
   10:memory:/
   9:pids:/
   8:perf_event:/
   7:hugetlb:/
   6:devices:/
   5:cpuset:/
   4:blkio:/
   3:cpuacct,cpu:/
   2:net_prio,net_cls:/
   1:name=systemd:/test.slice/cgrouptest.service
   # 限制内存和cpu
   $ sudo systemctl set-property cgrouptest.service CPUShares=100 MemoryLimit=100M
   # 再次查看cgroup信息
   $ cat /proc/9129/cgroup
   11:freezer:/
   10:memory:/test.slice/cgrouptest.service
   9:pids:/
   8:perf_event:/
   7:hugetlb:/
   6:devices:/
   5:cpuset:/
   4:blkio:/
   3:cpuacct,cpu:/test.slice/cgrouptest.service
   2:net_prio,net_cls:/
   1:name=systemd:/test.slice/cgrouptest.service
   # 可以看到在根cgroup的目录下多出了这个目录
   $ ll /sys/fs/cgroup/memory/test.slice/cgrouptest.service/
   total 0
   -rw-r--r-- 1 root root 0 May 14 21:50 cgroup.clone_children
   --w--w--w- 1 root root 0 May 14 21:50 cgroup.event_control
   -rw-r--r-- 1 root root 0 May 14 21:50 cgroup.procs
   -rw-r--r-- 1 root root 0 May 14 21:50 memory.failcnt
   --w------- 1 root root 0 May 14 21:50 memory.force_empty
   。。。。。。
   -rw-r--r-- 1 root root 0 May 14 21:50 memory.limit_in_bytes
   。。。。。。
   -rw-r--r-- 1 root root 0 May 14 21:50 notify_on_release
   -rw-r--r-- 1 root root 0 May 14 21:50 tasks
   # 查看文件内容
   $ cat /sys/fs/cgroup/memory/test.slice/cgrouptest.service/tasks
   9129
   $ cat /sys/fs/cgroup/memory/test.slice/cgrouptest.service/memory.limit_in_bytes
   104857600
   ~~~

2. docker容器的资源控制

   ~~~shell
   # 运行一个容器
   $ sudo docker run -d nginx
   4f6c2a45846f5b48ee0496ee47bb28ab8a72ea09d1b0fe294a236e30ca4b9c68
   # 查看cgroup信息，以memory为例，可以看出这里是容器运行的最大值，所以默认是没有限制的
   $ sudo systemd-cgls | grep -v grep|grep -i  nginx
     │ ├─13667 nginx: master process nginx -g daemon off;
     │ ├─13724 nginx: worker process
     │ └─13725 nginx: worker process
   $ cat /proc/13667/cgroup
   11:freezer:/system.slice/docker-4f6c2a45846f5b48ee0496ee47bb28ab8a72ea09d1b0fe294a236e30ca4b9c68.scope
   10:memory:/system.slice/docker-4f6c2a45846f5b48ee0496ee47bb28ab8a72ea09d1b0fe294a236e30ca4b9c68.scope
   。。。。。。
   $ cat /sys/fs/cgroup/memory/system.slice/docker-4f6c2a45846f5b48ee0496ee47bb28ab8a72ea09d1b0fe294a236e30ca4b9c68.scope/memory.limit_in_bytes
   9223372036854771712
   # 配置内存限制
   $ sudo docker update 4f6 --memory='200M' --memory-swap='-1'
   4f6
   # 再次查看，可以看到这里的值已经被设定为200M
   $ cat /sys/fs/cgroup/memory/system.slice/docker-4f6c2a45846f5b48ee0496ee47bb28ab8a72ea09d1b0fe294a236e30ca4b9c68.scope/memory.limit_in_bytes
   209715200
   ~~~

由以上的操作内容可以看出，充分使用systemd-cgls，/proc/\<pid\>/cgroup以及/sys/fs/cgroup就可以了解cgroup的配置信息。





此文参考以下地址：

https://cloud.tencent.com/developer/article/1684499

https://tech.meituan.com/2015/03/31/cgroups.html