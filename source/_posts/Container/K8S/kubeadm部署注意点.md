---
title: kubeadm部署注意点
tags:
- k8s
categories:
- deploy
---

[Installing kubeadm | Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

# 1.内核参数

1. 添加了了br_netfilter模块之后需要重启服务器；

2. 为什么要开启bridge这个参数：

   [为什么 kubernetes 环境要求开启 bridge-nf-call-iptables ? | roc云原生 (imroc.cc)](https://imroc.cc/post/202105/why-enable-bridge-nf-call-iptables/)

3. 建议修改打开文件数和进程的limit值；

4. 注意配置net.ipv4.ip_forward = 1;

5. 关于内核参数的调整可以参考下页，而不需要在宿主机上操作：

   [Using sysctls in a Kubernetes Cluster | Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/)

# 2.CR的选择

​	[Container Runtimes | Kubernetes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)	

1. 关于docker

   ​	k8s 1.24开始不自带dockershim，也就是无法直接调用docker了，但是docker build出来的镜像仍然可以在其他的CR上运行。

   ​	参考：[Check whether Dockershim deprecation affects you | Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/check-if-dockershim-deprecation-affects-you/)

2. cgroupdriver需要修改

   ​	以docker为例，需要在/etc/docker/daemon.json中配置：

   ​	{"exec-opts": ["native.cgroupdriver=systemd"]}

3. 未来建议使用containerd，否则还需要在安装docker的基础上再安装一个dockershim，而dockershim的维护也会是一个问题。

   ​	以后容器平台大概会是宿主机 podman，容器集群 k8s+containerd，镜像通用。

# 3.下载仓库

​	1.yum仓库

​		在安装kubeadm的时候，默认的google无法打开，需要使用国内的镜像仓库，比如阿里云：

~~~shell
# 下面的gpgcheck=1会报错
$ cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
~~~

​	2.镜像仓库

​		默认镜像仓库地址为k8s.gcr.io，大陆也是连接不上的，所以这里仍然改成了阿里云的镜像仓库：

~~~shell
# coredns的镜像tag不一样，需要做额外处理
$ vi pull_k8s_image.sh
#!/bin/bash

images=$(kubeadm config images list)
for image in ${images[@]};do
    if echo $image | grep coredns > /dev/null;then
        image_name_v=${image##*/}
        image_name=$(echo $image_name_v | sed "s/v\([0-9].*\)/\1/g")
    else
        image_name=${image#*/}
    fi
    echo "image name is: $image_name"
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$image_name
    if echo $image | grep coredns > /dev/null;then
        docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$image_name k8s.gcr.io/coredns/$image_name_v
    else
        docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$image_name k8s.gcr.io/$image_name
    fi
    echo "----------------------------------"
done
$ sudo sh pull_k8s_image.sh
~~~

# 4. 安装参数

​	[Dual-stack support with kubeadm | Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/dual-stack-support/)

 1. 指定参数

    可以指定pod和service的cidr，防止与内网网段冲突，这个比较重要

 2. 使用kubeadm-config.yaml文件

    可以配置更多的值，比如apiserver的地址

# 5.CNI的选择

​	[Cluster Networking | Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model)

​	默认kubeadm安装过程中是不会安装CNI插件的，这个需要单独安装，关于一些CNI插件的比较如下：

​	[Comparing Kubernetes Container Network Interface (CNI) providers | Kubevious.io](https://kubevious.io/blog/post/comparing-kubernetes-container-network-interface-cni-providers)

​	建议：[About Calico (tigera.io)](https://projectcalico.docs.tigera.io/about/about-calico)，它还可以和Istio集成，执行service mesh的网络策略

​		注意：使用operator安装calico的过程可能会有挂载相关的报错，但是稍后会自行解决。

# 6.自动补全

自动补全：[Tools Included | Kubernetes](https://kubernetes.io/docs/tasks/tools/included/)

# 7.插件

​	[Extend kubectl with plugins | Kubernetes](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/)

​	这里可以编写自己想要的操作命令，但是建议可以使用开源工具：[kubernetes-sigs/krew: 📦 Find and install kubectl plugins (github.com)](https://github.com/kubernetes-sigs/krew/)

