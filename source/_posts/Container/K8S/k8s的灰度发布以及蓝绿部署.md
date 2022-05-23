---
title: k8s的灰度发布以及蓝绿部署
tags:
- deploy
categories:
- k8s
---

# 1.场景说明

1. 灰度

   灰度又称为金丝雀发布，主要用于新版本的验证。主要使用场景有以下二种：

   - 新版本只允许指定用户登录，等待这部分用户验证没有问题后再全部把用户导入到新版本，关闭旧版本应用；
   - 按照总的访问用户数的百分比切换，等这部分流量全都测试没有问题，再全部把流量切换到新版本，关闭旧版本。

2. 蓝绿部署

   先部署一个新的版本（绿），此时测试整个绿色版本，等待测试没有问题后，将全部流量转移到新的版本，关闭旧版本（蓝）。

# 2.实现

下面的测试我们会以nginx ingress为例来说明它的实施方式。

1. 安装nginx-ingress

   参考：[Welcome - NGINX Ingress Controller (kubernetes.github.io)](https://kubernetes.github.io/ingress-nginx/)

   ~~~shell
   # 使用helm安装，参考：https://helm.sh/docs/intro/quickstart/
   $ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   $ helm pull ingress-nginx/ingress-nginx
   # 由于镜像无法下载，参考：https://github.com/anjia0532/gcr.io_mirror，建议在各个节点先手动下载以下，不然时间很长
   $ cat > ingress-value.yaml << EOF
   controller:
     name: controller 
     image:
       registry: docker.io
       image: anjia0532/google-containers.ingress-nginx.controller
       tag: "v1.2.0"
       digest: ''
       digestChroot: ''
     ingressClassResource:
       enabled: true
       default: true
     admissionWebhooks:
       patch:
         enabled: true
         image:
           registry: docker.io
           image: anjia0532/google-containers.ingress-nginx.kube-webhook-certgen
           tag: v1.1.1
           digest: ''
   defaultBackend:
     enabled: true
     name: defaultbackend
     image:
       registry: docker.io
       image: anjia0532/google-containers.defaultbackend-amd64
       tag: "1.5"
   EOF
   $ kubectl create namespace ingress-nginx
   $ helm upgrade --install ingress-nginx-4 ingress-nginx-4.1.1.tgz --namespace ingress-nginx --values=ingress-value.yaml --debug
   ......
   # 检查状态
   $ helm list
   NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
   ingress-nginx-4         ingress-nginx   3               2022-05-19 23:07:53.953598445 +0800 CST deployed        ingress-nginx-4.1.1     1.2.0
   # 创建一个nginx服务，略过
   # 创建ingress，这里k8s 1.22之后的版本和以前不用
   $ vi ingress-nginx.yaml
   ---
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: nginx-ingress
     namespace: default
   spec:
     ingressClassName: nginx
     rules:
     - host: figo.test.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: nginx-service
               port:
                 number: 8000
   $ 
   ~~~

   
