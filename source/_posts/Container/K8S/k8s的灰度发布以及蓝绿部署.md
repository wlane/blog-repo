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

# 2.灰度实现

参考：[Annotations - NGINX Ingress Controller (kubernetes.github.io)](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/)

下面的测试我们会以nginx ingress为例来说明它的实施方式。

1. 安装nginx-ingress

   参考：

   [Welcome - NGINX Ingress Controller (kubernetes.github.io)](https://kubernetes.github.io/ingress-nginx/)

   [How To Setup Nginx Ingress Controller On Kubernetes (devopscube.com)](https://devopscube.com/setup-ingress-kubernetes-nginx-controller/)

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
       enabled: true
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
   # 创建ingress，这里k8s 1.22之后的版本和以前不同，多了一个ingressclass
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
     - host: xxxx.test.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: nginx-service
               port:
                 number: 8000
   # 报错解决方案：https://github.com/kubernetes/ingress-nginx/issues/5583           
   $ kubectl create -f ingress-nginx.yaml
   ~~~

2. 访问测试

   ~~~shell
   $ kubectl get pods -n ingress-nginx -o wide
   NAME                                              READY   STATUS    RESTARTS   AGE     IP               NODE              NOMINATED NODE   READINESS GATES
   ingress-nginx-4-controller-78997f755-4gmt5        1/1     Running   0          4d23h   192.168.96.199   vm-12-17-centos   <none>           <none>
   ingress-nginx-4-defaultbackend-76865c4844-rlq7x   1/1     Running   0          4d23h   192.168.99.74    vm-12-15-centos   <none>           <none>
   $ kubectl get  svc -n ingress-nginx
   NAME                                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
   ingress-nginx-4-controller             LoadBalancer   10.103.230.29    <pending>     80:31808/TCP,443:32733/TCP   5d22h
   ingress-nginx-4-controller-admission   ClusterIP      10.100.128.222   <none>        443/TCP                      5d22h
   ingress-nginx-4-defaultbackend         ClusterIP      10.103.74.172    <none>        80/TCP                       5d22h
   # 根据上面ingress controller的结果我们到它所在的host上去操作以下步骤，这里nodevm-12-17-centos的ip为10.0.12.17（实际上clusterip下应该是任意一台都可以直接访问controller映射到node的31808，不过这里的service类型是lb的类型，可能有问题，先跳过此问题）
   $ vi /etc/hosts
   10.0.12.17 xxxx.test.com
   $ curl -v http://xxxx.test.com:31808
   # 查看nginx的日志，可以看到正常输出
   $ kubectl logs nginx-deployment-8d545c96d-zkwvc -f
   ~~~

3. 部署一个新的nginx

   ~~~shell
   # 将原先的部署用的nginx.yaml改个名称，再部署一遍
   $ kubectl get pods
   NAME                                 READY   STATUS    RESTARTS   AGE
   nginx-deployment-8d545c96d-zkwvc     1/1     Running   0          7m34s
   nginx2-deployment-667c55f496-d28wh   1/1     Running   0          28s
   $ kubectl get svc -o wide
   NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE     SELECTOR
   kubernetes       ClusterIP   10.96.0.1        <none>        443/TCP    25d     <none>
   nginx-service    ClusterIP   10.109.51.139    <none>        8000/TCP   5d      app=nginx
   nginx-service2   ClusterIP   10.108.232.249   <none>        8000/TCP   2m40s   app=nginx2
   ~~~

4. 根据请求的header来区分流量

   ~~~shell
   # 定义annotations
   $ cat nginx2.yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: nginx-ingress2
     namespace: default
     annotations:
       nginx.ingress.kubernetes.io/canary: "true"
       nginx.ingress.kubernetes.io/canary-by-header: "Region"
       nginx.ingress.kubernetes.io/canary-by-header-pattern: "nj"
   spec:
     ingressClassName: nginx
     rules:
     - host: xxxx.test.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: nginx-service2
               port:
                 number: 8000
   $ kubectl apply -f nginx2.yaml
   $ kubectl get ingress
   NAME             CLASS   HOSTS           ADDRESS   PORTS   AGE
   nginx-ingress    nginx   xxxx.test.com             80      24h
   nginx-ingress2   nginx   xxxx.test.com             80      23h
   $ kubectl get pod
   NAME                                 READY   STATUS    RESTARTS   AGE
   nginx-deployment-8d545c96d-nnjw7     1/1     Running   0          23h
   nginx2-deployment-667c55f496-d28wh   1/1     Running   0          23h
   # 此时默认的流量走旧版本的nginx
   $ curl -v http://xxx.test.com:31808
   $ kubectl logs nginx-deployment-8d545c96d-nnjw7 -f
   # 定义header之后，可以发现请求走到了新版本的nginx2
   $ curl -H "Region: nj" -v http://xxxx.test.com:31808
   $ kubectl logs nginx2-deployment-667c55f496-d28wh -f
   ~~~

5. 根据cookie来区别流量

   ~~~shell
   $ cat nginx2.yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: nginx-ingress2
     namespace: default
     annotations:
       nginx.ingress.kubernetes.io/canary: "true"
       nginx.ingress.kubernetes.io/canary-by-cookie: "NJ"
   spec:
     ingressClassName: nginx
     rules:
     - host: xxxx.test.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: nginx-service2
               port:
                 number: 8000
   # 通过cookie来访问，注意这里cookie一定要是always，参考官网对这个值的说明
   $ curl --cookie "NJ=always" -v http://figo.test.com:31808
   ~~~

6. 根据权重来区分流量

   ~~~shell
   $ cat nginx2.yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: nginx-ingress2
     namespace: default
     annotations:
       nginx.ingress.kubernetes.io/canary: "true"
       nginx.ingress.kubernetes.io/canary-weight: "20"
   spec:
     ingressClassName: nginx
     rules:
     - host: xxxx.test.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: nginx-service2
               port:
                 number: 8000
   # 以下访问大概每五次有一次能够访问到nginx2版本
   $ curl  -v http://xxxx.test.com:31808
   ~~~

7. 优势与不足

   不足：

   - 只支持一个配置canary的服务，也就是说最多只能有一个新版本存在
   - 必须要存在一个旧的版本服务才能配置canary，即使完全不用也不能删除，否则用户访问会超时

   优势：

   - 新版本故障可控
   - 可在升级过程中按需扩缩容，不需要一次性给足资源

# 3.蓝绿部署的实现

1. 部署蓝（旧）版本

   ~~~shell
   #
   $ cat nginx.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deployment
     namespace: default
     labels:
       app: nginx
       version: v1
   spec:
   。。。。。。
   # 存在一个service
   $ cat svc.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-service
     namespace: default
   spec:
     ports:
     - port: 8000
       targetPort: 80
     selector:
       app: nginx
       version: v1
     type: ClusterIP
   # 此时ingres只是指向v1版本
   $ cat ingress.yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: nginx-ingress
     namespace: default
   spec:
     ingressClassName: nginx
     rules:
     - host: xxxx.test.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: nginx-service
               port:
                 number: 8000
   # 此时访问的是旧版本应用
   $ curl  -v http://xxxx.test.com:31808
   ~~~

2. 部署绿（新）版本

   ~~~shell
   # 部署应用的新版本
   $ cat nginx.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deployment
     namespace: default
     labels:
       app: nginx
       version: v2
   spec:
   。。。。。。
   # 修改service的指向
   $ cat svc.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-service
     namespace: default
   spec:
     ports:
     - port: 8000
       targetPort: 80
     selector:
       app: nginx
       version: v2
     type: ClusterIP
   # 此时访问的是新版本的应用
   $ curl  -v http://xxxx.test.com:31808
   ~~~

3. 优势与不足

   不足：

   - 两套生产环境
   - 新版本故障将会影响所有用户

   优势：

   - 部署简单
   - 无需停机，影响时间短



本文参考以下网址：

https://cloud.tencent.com/developer/article/1729580
