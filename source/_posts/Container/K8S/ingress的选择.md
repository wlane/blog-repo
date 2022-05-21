---
title: ingress的选择
date: 2022年4月27日 23点39分
tags:
- k8s
categories:
- operation
---

ingress控制器的种类丰富，如下：

[Ingress Controllers | Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)

参考：

曾经社区对比较流行的ingress做过比较，如下图：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220427234937.png)

由于这张图是很久以前的了，并不保证是否匹配新版本。

全文如下：

[Comparing Ingress controllers for Kubernetes – Flant blog](https://blog.flant.com/comparing-ingress-controllers-for-kubernetes/)

这个博客里面的其他技术性文章也推荐一看。

建议：

Istio或者Traefik，最近github都仍然很活跃。