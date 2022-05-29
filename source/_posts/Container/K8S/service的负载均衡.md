---
title: service的负载均衡
tags:
- service
categories:
- k8s
---

# 1.原理

参见：[Virtual IPs and service proxies](https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies)

1. 服务发现，也就是说怎么找到服务

   k8s目前支持两种类型的服务发现机制：

   - 环境变量：一个pod和其对应的service创建好后，kubelet会往这个pod中注册所有和service相关的环境变量。所以也就是为什么可以在pod中发现多出了很多变量；

     ~~~shell
     # 创建的service名称为nginx-service，并且port为8000
     $ kubectl exec -it nginx-deployment-8d545c96d-nnjw7 -- bash
     root@nginx-deployment-8d545c96d-nnjw7:/# export
     declare -x HOME="/root"
     declare -x HOSTNAME="nginx-deployment-8d545c96d-nnjw7"
     declare -x KUBERNETES_PORT="tcp://10.96.0.1:443"
     declare -x KUBERNETES_PORT_443_TCP="tcp://10.96.0.1:443"
     declare -x KUBERNETES_PORT_443_TCP_ADDR="10.96.0.1"
     declare -x KUBERNETES_PORT_443_TCP_PORT="443"
     declare -x KUBERNETES_PORT_443_TCP_PROTO="tcp"
     declare -x KUBERNETES_SERVICE_HOST="10.96.0.1"
     declare -x KUBERNETES_SERVICE_PORT="443"
     declare -x KUBERNETES_SERVICE_PORT_HTTPS="443"
     declare -x NGINX_SERVICE_PORT="tcp://10.109.51.139:8000"
     declare -x NGINX_SERVICE_PORT_8000_TCP="tcp://10.109.51.139:8000"
     declare -x NGINX_SERVICE_PORT_8000_TCP_ADDR="10.109.51.139"
     declare -x NGINX_SERVICE_PORT_8000_TCP_PORT="8000"
     declare -x NGINX_SERVICE_PORT_8000_TCP_PROTO="tcp"
     declare -x NGINX_SERVICE_SERVICE_HOST="10.109.51.139"
     declare -x NGINX_SERVICE_SERVICE_PORT="8000"
     declare -x NGINX_VERSION="1.21.6"
     ~~~

   - DNS：这是使用的附件功能组件实现，比如CoreDNS，它会监视API服务对象，在发现新服务时创建DNS记录。

     ~~~shell
     # 查询dns信息，这里是两个coredns的pod
     $ kubectl get svc -n kube-system -o wide
     NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE   SELECTOR
     kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   28d   k8s-app=kube-dns
     $ kubectl get pods -n kube-system --show-labels
     NAME                                     READY   STATUS    RESTARTS     AGE   LABELS
     coredns-64897985d-52t4j                  1/1     Running   2 (8d ago)   28d   k8s-app=kube-dns,pod-template-hash=64897985d
     coredns-64897985d-pg6d6                  1/1     Running   2 (8d ago)   28d   k8s-app=kube-dns,pod-template-hash=64897985d
     # 查看pod中配置的dns，这里的ndots指的是查询的域名中包含的点数小于5个先走search域，再按FQDN查找，否则直接按FQDN查找
     $ kubectl exec -it nginx-deployment-8d545c96d-nnjw7 -- bash
     root@nginx-deployment-8d545c96d-nnjw7:/# cat /etc/resolv.conf
     nameserver 10.96.0.10
     search default.svc.cluster.local svc.cluster.local cluster.local
     options ndots:5
     ~~~

2. 服务类型

   通常来讲目前的service有以下四种类型：

   - clusterIP：k8s默认的类型，用于集群内部通信，外部无法直接访问，必须通过其他代理；

     ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220528001437.png)

   - NodePort：在集群所有节点上打开一个端口，用户可以直接通过这个端口访问内部应用；

     ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220528001659.png)

   - Ingress：统一作为多个服务的入口，类似于传统的nginx，haproxy一类的应用；

     ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220528001904.png)

   - LoadBalancer：需要和LB组件结合，将集群暴露到公网，通过访问LB的ip来访问内部应用。

     ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220528002149.png)

3. 原理

   从最上方官方的介绍页面我们可以看到service的访问是通过kube-proxy组件完成的。kube-proxy实现了一种使用VIP来访问service的形式，目前有三种工作模式：userspace、iptables和IPVS三种代理模式。

   我们以目前默认的运行的iptables模式来说明：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220528010202.png)
   
   - kube-proxy组件会实时的监控server和它的endpoints对象，当一个新的serivce创建出来之后，k8s会主动创建一个同名的endpoints对象，用于保存该service下的pod信息；
   - kube-proxy感知到service和endpoings的变化之后，会在所有的k8s节点上设置对应的iptables的规则。综合service和endpoint的信息，配置clusterIP到各个pod ip的规则，并且默认使用随机的负载均衡策略。
   - 配置完成后，用户可以通过访问clusterIP来访问内部应用，这就回到上面的服务类型的实现了。

# 2.具体实现

关于iptables的信息可以参考：[iptables原理](https://www.xwangwang.net/2022/05/27/Security/iptables%E5%8E%9F%E7%90%86/)

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220528155020.png)

如上图，在iptables模式下，kube-proxy通过在目标node节点上的iptables中的NAT表的PREROUTING和POSTROUTING链中创建自定义链，通过这些链对流经的数据包做DNAT或者SNAT的操作从而实现路由、负载均衡等目的。

- 查看目前我们使用的模式

  ~~~shell
  # 可以看到目前集群支持iptables和ipvs，mode参数为空，则为默认值iptables
  kubectl get cm -n kube-system kube-proxy -o yaml
  。。。。。。
  detectLocalMode: ""
      enableProfiling: false
      healthzBindAddress: ""
      hostnameOverride: ""
      iptables:
        masqueradeAll: false
        masqueradeBit: null
        minSyncPeriod: 0s
        syncPeriod: 0s
      ipvs:
        excludeCIDRs: null
        minSyncPeriod: 0s
        scheduler: ""
        strictARP: false
        syncPeriod: 0s
        tcpFinTimeout: 0s
        tcpTimeout: 0s
        udpTimeout: 0s
      kind: KubeProxyConfiguration
      metricsBindAddress: ""
      mode: ""
      nodePortAddresses: null
  。。。。。。
  ~~~

- 我们以访问nginx service（后端有2个pod）为例，查看nginx的serivce：

  ~~~shell
  $ kubectl get pods -o wide
  NAME                                 READY   STATUS    RESTARTS   AGE     IP               NODE              NOMINATED NODE   READINESS GATES
  nginx-deployment-8d545c96d-ms2mc     1/1     Running   0          11m     192.168.99.81    vm-12-15-centos   <none>           <none>
  nginx-deployment-8d545c96d-nnjw7     1/1     Running   0          3d14h   192.168.96.201   vm-12-17-centos   <none>           <none>
  $ kubectl get svc -o wide
  nginx-service    ClusterIP   10.107.7.183     <none>        8000/TCP   2d14h   app=nginx
  ~~~

- 首先集群内部访问service使用OUTPUT链，而外部访问使用PREROUTING链：

  ~~~shell
  # 集群内部访问走到KUBE-SERVICES链
  $ sudo iptables -t nat -nL OUTPUT
  Chain OUTPUT (policy ACCEPT)
  target     prot opt source               destination
  cali-OUTPUT  all  --  0.0.0.0/0            0.0.0.0/0            /* cali:tVnHkvAo15HuiPy0 */
  KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
  DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL
  # 外部访问也会走到KUBE-SERVICES链
  $ sudo iptables -t nat -nL PREROUTING
  Chain PREROUTING (policy ACCEPT)
  target     prot opt source               destination
  cali-PREROUTING  all  --  0.0.0.0/0            0.0.0.0/0            /* cali:6gwbT8clXdHdC1b1 */
  KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
  DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
  
  ~~~

- 查看KUBE-SERVICES链中和nginx相关的链：

  ~~~shell
  # 这里的每一个service都会有一条对应的链
  $ sudo iptables -t nat -nL KUBE-SERVICES | grep nginx-service
  KUBE-SVC-V2OKYYMBY3REGZOG  tcp  --  0.0.0.0/0            10.107.7.183         /* default/nginx-service cluster IP */ tcp dpt:8000
  
  ~~~

- 查看KUBE-SVC链：

  ~~~shell
  # 这里的操作是对所有非内部访问的请求走KUBE-MARK-MASQ给数据包打上标记，以便最终发送数据的时候在POSTROUTING上做SNAT，但是内部的请求直接就可以走KUBE-SEP链
  # 这里可以看到第一条SEP使用了随机算法来做负载均衡
  $ sudo iptables -t nat -nL KUBE-SVC-V2OKYYMBY3REGZOG
  Chain KUBE-SVC-V2OKYYMBY3REGZOG (1 references)
  target     prot opt source               destination
  KUBE-MARK-MASQ  tcp  -- !192.168.0.0/16       10.107.7.183         /* default/nginx-service cluster IP */ tcp dpt:8000
  KUBE-SEP-QNYGGPIFZJ3D3DNW  all  --  0.0.0.0/0            0.0.0.0/0            /* default/nginx-service */ statistic mode random probability 0.50000000000
  KUBE-SEP-SJ3GVCAX25ZLTA5X  all  --  0.0.0.0/0            0.0.0.0/0            /* default/nginx-service */
  ~~~

- 查看KUBE-SEP链：

  ~~~shell
  # 这里会对pod自身内部的访问做MARK操作，然后将所有的请求做DNAT发往pod的地址
  $ sudo iptables -t nat -nL KUBE-SEP-QNYGGPIFZJ3D3DNW
  Chain KUBE-SEP-QNYGGPIFZJ3D3DNW (1 references)
  target     prot opt source               destination
  KUBE-MARK-MASQ  all  --  192.168.96.201       0.0.0.0/0            /* default/nginx-service */
  DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/nginx-service */ tcp to:192.168.96.201:80
  $ sudo iptables -t nat -nL KUBE-SEP-SJ3GVCAX25ZLTA5X
  Chain KUBE-SEP-SJ3GVCAX25ZLTA5X (1 references)
  target     prot opt source               destination
  KUBE-MARK-MASQ  all  --  192.168.99.81        0.0.0.0/0            /* default/nginx-service */
  DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/nginx-service */ tcp to:192.168.99.81:80
  ~~~

- 数据处理完毕后，通过POSTROUTING链发出：

  ~~~shell
  # 这里使用了KUBE-POSTROUTING链
  $ sudo iptables -t nat -nL POSTROUTING
  Chain POSTROUTING (policy ACCEPT)
  target     prot opt source               destination
  cali-POSTROUTING  all  --  0.0.0.0/0            0.0.0.0/0            /* cali:O3lYWMrLQYEMJtB5 */
  MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0
  KUBE-POSTROUTING  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
  
  ~~~

- 查看KUBE-POSTROUTING链：

  ~~~shell
  # 对于没有打标记的请求，也就是内部请求全部直接RETURN，有标记的走SNAT
  $ sudo iptables -t nat -nL KUBE-POSTROUTING
  Chain KUBE-POSTROUTING (1 references)
  target     prot opt source               destination
  RETURN     all  --  0.0.0.0/0            0.0.0.0/0            mark match ! 0x4000/0x4000
  MARK       all  --  0.0.0.0/0            0.0.0.0/0            MARK xor 0x4000
  MASQUERADE  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */
  ~~~

以上就是在iptables模式下访问的整个流程。

存在的不足：

- 规则匹配成本高：每个service都会产生一条KUBE-SVC链，请求需要按顺序匹配，当存在大量的service时，每次请求都需要消耗大量时间；
- 规则更新成本高：iptables的特性决定了每次更新规则都是使用全量替换（非增量式）的方式写入内核，规则数多时，过程会很慢；
- 可用性：扩、缩容时，iptables规则的刷新会导致链接断开，服务可能会有暂时不可用；
- 扩展性：规则多时，对规则的修改会出现内核锁，这是只能等待修改完成，这限制了服务数和节点数的扩展。

基于以上原因，后来出现了使用IPVS的代理模式，但由于现在k8s和OCP默认使用的仍然是iptables的代理模式，而且需要安装IPVS组件，我们在部署k8s集群之前并没有安装，所以这里先不讨论。可以通过最上面官网的信息了解一下，或者也可以先查看这里：[LVS负载均衡](https://www.xwangwang.net/2022/05/27/OS/Linux/LVS%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1/)。kube-proxy在IPVS模式下，工作在NAT模式。
