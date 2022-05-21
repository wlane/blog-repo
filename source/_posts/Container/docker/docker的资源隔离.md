---
title: docker的资源隔离
tags:
- resources
categories:
- docker
---

# 1.原理

1. 什么是namespace

容器本质上是一种轻量级的虚拟化技术，Docker是许多容器引擎中应用最广泛的一种，自然也是如此。既然是虚拟化，那我们不可避免的会想到它一定使用到了某种隔离技术，保证不同的容器之间是不可见的。我们常见的可能是使用chroot的方式来控制资源的使用，但显然这种方式功能单一。我们可以用使用chroot的方式来思考容器的资源隔离，实际上这就是namespace，namespace是Linux内核提供的一种环境隔离方法：

[namespaces(7) - Linux manual page (man7.org)](https://man7.org/linux/man-pages/man7/namespaces.7.html)

它提供了一种抽象的机制，将原本共享的资源分割成不同的集合，集合中的成员独自占有原本需要共享的资源。那为什么linux要提供这样一种功能呢？主要还是从安全的角度来考虑，试想一台运行多个服务的机器，假如其中有一个服务存在漏洞，这个漏洞被入侵者利用，从而控制住了这个服务，此时如果其他服务和这个服务公用资源，包括内存，cpu等资源，那很容易可以获取到其他服务的信息，最总可能导致入侵者控制整个机器。为了防止危害结果的扩大，通过namespace将不同服务的环境隔离开来，可以更好的控制和消除这些风险。

[Linux namespaces - Wikipedia](https://en.wikipedia.org/wiki/Linux_namespaces)

每个进程都会有一个namespace的信息，比如我们以sshd服务为例：

~~~shell
$ ps -ef | grep sshd
root      1003     1  0 Apr30 ?        00:00:00 /usr/sbin/sshd -D
$ sudo ls -l /proc/1003/ns
total 0
lrwxrwxrwx 1 root root 0 May  8 13:11 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 May  8 13:11 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 Apr 30 23:35 net -> net:[4026531956]
lrwxrwxrwx 1 root root 0 Apr 30 23:35 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 May  8 13:11 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 May  8 13:11 uts -> uts:[4026531838]
~~~

上面的每个文件对应一个namespace的文件描述符，可以看出它们都是符号链接文件，后面中括号中的值是namespace的id。如果这个值相同，证明两个进程在同一个namespace，同时进程被删除后，namespace也会被删除。可以通过如下方式查看namespace有哪些进程：

~~~shell
# 查询上面查出来的sshd所在的namespace，可以看出它和1号进程共享一个namespace
$ sudo lsns -p 1003
        NS TYPE  NPROCS PID USER COMMAND
4026531836 pid      100   1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531837 user     110   1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531838 uts       99   1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531839 ipc       99   1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531840 mnt       97   1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531956 net       99   1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
# 可以根据下面结果的pid去查询它的子进程来确定有多少个进程运行在当前的namespace下
$ sudo lsns -t pid
        NS TYPE NPROCS   PID USER COMMAND
4026531836 pid     101     1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026532170 pid       4 11455 root bash
4026532233 pid       3  4027 root nginx: master process nginx -g daemon off
4026532296 pid       3 14080 root nginx: master process nginx -g daemon off
~~~

注意：在namespace被打开（正在被使用）的情况下，即使停止了该进程，这个namesapce仍然存在，不会被删除。

2. 如何使用

应用程序主要调用Liunx内核参数提供的三个系统调用来实现对namespace的控制：

- clone：创建新进程并设置一个独立的namespace。它类似于fork系统调用，通过传递的参数来控制新进程有哪些特性；
- setns：让进程加入已经存在的namespace；
- unshare：将当前进程和所在的namespace分离，并进入到一个新的namespace。

在shell的环境下也就只方便操作unshare：

~~~shell
# 创建一个拥有独立pid namespace的bash环境
$ sudo unshare -u -p --mount-proc --fork bash
[root@VM-12-3-centos figo]#
# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 14:02 pts/0    00:00:00 bash
root        28     1  0 14:02 pts/0    00:00:00 ps -ef
~~~

以上其实可以看出这个命令类似于sudo docker exec -it xxx bash这个命令的结果，我只能看到我所在namespace环境的进程，其他的都看不到。

# 2.namespace的类型

[namespaces(7) - Linux manual page (man7.org)](https://man7.org/linux/man-pages/man7/namespaces.7.html)

| 名称   | 宏定义          | 隔离资源                           | 内核版本 |
| ------ | --------------- | ---------------------------------- | -------- |
| Cgroup | CLONE_NEWCGROUP | Control group root directory       | 4.6      |
| IPC    | CLONE_NEWIPC    | System V IPC, POSIX message queue  | 2.6.19   |
| Net    | CLONE_NEWNET    | Network devices,stacks, ports, etc | 2.6.24   |
| Mnt    | CLONE_NEWNS     | Mount point                        | 2.4.19   |
| PID    | CLONE_NEWPID    | Process ID                         | 2.6.24   |
| Time   | CLONE_NEWTIME   | Boot and monotonic clocks          | 5.6      |
| User   | CLONE_NEWUSER   | User and group ID                  | 3.8      |
| UTS    | CLONE_NEWUTS    | Hostname and NIS domain name       | 2.6.19   |

可以看出这些类型加入的内核版本都不一样，也侧面说明了一个问题，为何docker的运行要求内核最少在3.10以上，因为一些最基础的隔离在低版本的内核中都不支持。

我们这里以net namespace为例来说明一下他们的特点。

注意：一个物理设备只能存在于一个network namespace中，但是虚拟的veth设备成对出现，所以两端可以分别存在于两个namespace中，以类似双向管道的方式通信。参考：[veth(4) - Linux manual page (man7.org)](https://man7.org/linux/man-pages/man4/veth.4.html)

1. 查看是否支持net namespace：

   ~~~shell
   $ grep CONFIG_NET_NS /boot/config-$(uname -r)
   CONFIG_NET_NS=y
   ~~~

2. 创建net namespace

   ~~~shell
   $ sudo ip netns add ns-test
   $ sudo ip netns ls
   ns-test
   # 可以进入namespace查看网络，可以看到只有一个lo口
   $ sudo ip netns exec ns-test bash
   [root@VM-12-3-centos figo]# ip a
   1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
   ~~~

3. 两个net namespace之间的通信

   ~~~shell
   # 创建一对veth设备，veth pair总是成对出现的
   $ sudo ip link add veth10 type veth peer name veth20
   $ ip a
   32: veth20@veth10: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
       link/ether 62:69:b2:60:48:3e brd ff:ff:ff:ff:ff:ff
   33: veth10@veth20: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
       link/ether 5a:7c:78:32:29:5f brd ff:ff:ff:ff:ff:ff
   # 增加一个namespace
   $ sudo ip netns add ns-test2
   # 分别将不同的veth加入不同的namespace
   $ sudo ip link set veth10 netns ns-test
   $ sudo ip link set veth20 netns ns-test2
   # 查看两个不用namespace里面的veth设备，它们此时的状态都是down
   $ sudo ip netns exec ns-test ip addr
   1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
   33: veth10@if32: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
       link/ether 5a:7c:78:32:29:5f brd ff:ff:ff:ff:ff:ff link-netnsid 1
   $ sudo ip netns exec ns-test2 ip addr
   1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
   32: veth20@if33: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
       link/ether 62:69:b2:60:48:3e brd ff:ff:ff:ff:ff:ff link-netnsid 0
   # 给veth设备添加ip，并启动
   $ sudo ip netns exec ns-test ip address add 10.1.1.10/24 dev veth10
   $ sudo ip netns exec ns-test ip link set veth10 up
   $ sudo ip netns exec ns-test2 ip address add 10.1.1.20/24 dev veth20
   $ sudo ip netns exec ns-test2 ip link set veth20 up
   # 查看ip连通性
   $ sudo ip netns exec ns-test ping 10.1.1.20
   PING 10.1.1.20 (10.1.1.20) 56(84) bytes of data.
   64 bytes from 10.1.1.20: icmp_seq=1 ttl=64 time=0.055 ms
   64 bytes from 10.1.1.20: icmp_seq=2 ttl=64 time=0.048 ms
   # 查看ip设备，此时找不到veth10和veth20了，你只能使用sudo ip netns exec ns-test ip a这种方式找到，因为他不在当前用户的namespace了
   $ sudo ip a
   # 删除namespace
   $ sudo ip netns del ns-test
   $ sudo ip netns del ns-test2
   ~~~

4. 多个net namespace之间的通信，此时需要通过bridge转发

   ~~~shell
   $ sudo ip netns add ns0
   $ sudo ip link add br0 type bridge
   $ sudo ip link set dev br0 up
   # 下面只写一个veth设备挂载到bridge的流程，其他都一样
   $ sudo ip link add veth5 type veth peer name veth6
   # 一端在namespace
   $ sudo ip link set dev veth5 netns ns0
   $ sudo ip netns exec ns0 ip link set dev veth5 name eth0
   $ sudo ip netns exec ns0 ip addr add 10.1.1.10/24 dev eth0
   $ sudo ip netns exec ns0 ip link set dev eth0 up
   $ sudo ip netns exec ns0 ip a
   1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
   36: eth0@if35: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
       link/ether 4e:25:ef:57:58:84 brd ff:ff:ff:ff:ff:ff link-netnsid 0
       inet 10.1.1.10/24 scope global eth0
          valid_lft forever preferred_lft forever
   # 一端在bridge上
   $ sudo ip link set dev veth6 master br0
   $ sudo ip link set dev veth6 up
   # 查看此时的设备状态，会发现veth5不见了，veth6还在
   $ sudo ip link ls
   34: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
       link/ether 3a:4b:30:b8:3f:61 brd ff:ff:ff:ff:ff:ff
   35: veth6@if36: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
       link/ether 3a:4b:30:b8:3f:61 brd ff:ff:ff:ff:ff:ff link-netnsid 3
   # 把其他veth pair设备也这样配置好之后，不用的ip之间仍然是可以通的，不同的namespace之间可以互访
   ~~~

   从以上流程可以得知，z在docker启动时会创建一个docker0作为默认的bridge，后续创建的容器都会创建一对veth pair，一端在容器的namespace内，一端在docker0上，所以ip a看到的只是在docker0内的veth设备，它对端的看不到。

   当用户使用bridge的方式再创建一个新的network时，会再生成一个bridge的设备，后续使用这个network的容器生成的veth都绑定在这个bridge上。

# 3. 仍然存在的不足

1. 仍然共享一个宿主机的系统内核，隔离并不彻底；

2. 还有一些资源和对象不能被namespace化，比如在5.6内核版本之前的时间，容器修改了时间会导致宿主机的时间也被修改；

3. 安全性比传统的虚拟机仍然要差一些：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220508153103.png)

   这方面可参考：[Docker容器安全性分析 - 云+社区 - 腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1557811)