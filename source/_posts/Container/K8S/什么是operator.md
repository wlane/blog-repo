---
title: 什么是operator
tags:
- operator
categories:
- k8s
---

# 1.概念

参考：[Operator pattern | Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)             [Operator · Kubernetes 中文指南——云原生应用架构实战手册 (jimmysong.io)](https://jimmysong.io/kubernetes-handbook/develop/operator.html)

书籍：[O’Reilly: Kubernetes Operators: Automating the Container Orchestration Platform (redhat.com)](https://www.redhat.com/en/resources/oreilly-kubernetes-operators-automation-ebook)

Operator是一组特定的应用程序控制器，它基于K8S的两个概念建成：

- 资源：对象的状态定义
- 控制器：控制、分析和动作，以调节资源的分布

它使用了K8S的可自定义资源的扩展API机制，使用CRD（CustomResourceDefinition）来创建、配置和管理应用程序。

Opeartor将运维人员对软件操作的流程代码化，同时利用Kubernetes强大的抽象能力来管理大规模的软件应用。

一般Operator适合于以下需求：

- 按需部署
- 需要备份和恢复
- 需要处理应用升级或者配置更改
- 发布一个服务，让不支持k8s api的应用可以发现
- 可以模拟集群故障以用作测试，比如通过Chaos Monkey测试
- 给分布式应用提供选举功能

# 2.举例说明

参考：[How To Setup Nginx Ingress Controller On Kubernetes (devopscube.com)](https://devopscube.com/setup-ingress-kubernetes-nginx-controller/)

接下来我们以nginx ingress为例来说明opertor的使用。

首先明确一下准入控制器的概念：[Using Admission Controllers | Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)。它是一项关于安全的功能，在使用很多高级的安全功能之前，必须要先经过准入控制器，所以其实它就是API的拦截器，修改和验证请求对象。简单举例来说LimitRanger准入控制器它可以按照Limitrange的设置来配置pod的资源请求，也可以验证pod的配置是都满足了limitrange的要求。

我们先看一张图：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220530233724.png)



1. 当要部署yaml时，准入控制器先拦截请求；
2. K8S API将ingress对象发送给ingress准入控制器服务；
3. Ingress准入控制器将请求发往部署在端口8443上的nginx准入控制器，以便生效ingress对象；
4. Nginx准入控制器返回响应给K8S API；
5. K8S API创建ingress。