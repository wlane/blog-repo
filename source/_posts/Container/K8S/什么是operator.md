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

首先明确一下准入控制器的概念：[Using Admission Controllers | Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)。它是一项关于安全的功能，在使用很多高级的安全功能之前，必须要先经过准入控制器，所以其实它就是API的拦截器，修改和验证请求对象。简单举例来说LimitRanger准入控制器它可以按照Limitrange的设置来配置pod的资源请求，也可以验证pod的配置是否满足了Limitrange的要求。

我们先看一张图：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220530233724.png)



1. 当要部署yaml时，触发一个验证准入的webhook，它先拦截请求；
2. K8S API将ingress对象发送给验证准入控制器服务的endpoint；
3. 准入控制器服务将请求发往部署在端口8443上的Nginx Ingress准入控制器，以便生效ingress对象；
4. Nginx Ingress准入控制器返回响应给K8S API；
5. K8S API创建ingress。

接下来我们部署一个nginx ingress看看，部署文件来自于：[kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx/blob/main/deploy/static/provider/cloud/deploy.yaml)

1. 根据上面的信息我们可以得知准入控制器是一定要开启的，但是k8s默认不开启

   配置参考：[kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/#options)

   ~~~shell
   # 检查集群中已经启用了准入注册
   $ kubectl api-versions |grep admission
   admissionregistration.k8s.io/v1
   # 修改apiserver的配置文件，修改后会自动重启apiserver
   $ sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
   ......
       - --enable-admission-plugins=NodeRestriction,ServiceAccount,NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota,MutatingAdmissionWebhook,ValidatingAdmissionWebhook
   ......
   ~~~

2. 创建namespace

   ~~~shell
   $ cat namespace.yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     labels:
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
     name: ingress-nginx
   $ kubectl apply -f namespace.yaml
   namespace/ingress-nginx created
   ~~~

3. 创建用于准入控制的SA和Role

   ~~~shell
   $ cat admission-service-account.yaml
   ---
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     labels:
       app.kubernetes.io/component: admission-webhook
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx-admission
     namespace: ingress-nginx
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     labels:
       app.kubernetes.io/component: admission-webhook
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx-admission
     namespace: ingress-nginx
   rules:
   - apiGroups:
     - ""
     resources:
     - secrets
     verbs:
     - get
     - create
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     labels:
       app.kubernetes.io/component: admission-webhook
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx-admission
   rules:
   - apiGroups:
     - admissionregistration.k8s.io
     resources:
     - validatingwebhookconfigurations
     verbs:
     - get
     - update
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     labels:
       app.kubernetes.io/component: admission-webhook
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx-admission
     namespace: ingress-nginx
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: Role
     name: ingress-nginx-admission
   subjects:
   - kind: ServiceAccount
     name: ingress-nginx-admission
     namespace: ingress-nginx
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     labels:
       app.kubernetes.io/component: admission-webhook
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx-admission
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: ingress-nginx-admission
   subjects:
   - kind: ServiceAccount
     name: ingress-nginx-admission
     namespace: ingress-nginx
   $ kubectl apply -f admission-service-account.yaml
   serviceaccount/ingress-nginx-admission created
   role.rbac.authorization.k8s.io/ingress-nginx-admission created
   clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
   rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
   clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
   ~~~

4. 创建验证准入的wenhook配置

   ~~~shell
   $ cat validating-webhook.yaml
   ---
   apiVersion: admissionregistration.k8s.io/v1
   kind: ValidatingWebhookConfiguration
   metadata:
     labels:
       app.kubernetes.io/component: admission-webhook
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx-admission
   webhooks:
   - admissionReviewVersions:
     - v1
     clientConfig:
       service:
         name: ingress-nginx-controller-admission
         namespace: ingress-nginx
         path: /networking/v1/ingresses
     failurePolicy: Fail
     matchPolicy: Equivalent
     name: validate.nginx.ingress.kubernetes.io
     rules:
     - apiGroups:
       - networking.k8s.io
       apiVersions:
       - v1
       operations:
       - CREATE
       - UPDATE
       resources:
       - ingresses
     sideEffects: None
     $ kubectl apply -f validating-webhook.yaml
   validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
   ~~~

5. 由于ValidatingWebhookConfiguration运行在https上，所以我们需要有一个机制来生成CA证书

   ~~~shell
   # 将生成的证书放在一个叫ingress-nginx-admission的secret里面，之后使用这个证书来对上面创建好的ValidatingWebhookConfiguration打patch
   $ cat webhook-jobs.yaml
   ---
   apiVersion: batch/v1
   kind: Job
   metadata:
     labels:
       app.kubernetes.io/component: admission-webhook
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx-admission-create
     namespace: ingress-nginx
   spec:
     template:
       metadata:
         labels:
           app.kubernetes.io/component: admission-webhook
           app.kubernetes.io/instance: ingress-nginx
           app.kubernetes.io/name: ingress-nginx
           app.kubernetes.io/part-of: ingress-nginx
           app.kubernetes.io/version: 1.2.1
         name: ingress-nginx-admission-create
       spec:
         containers:
         - args:
           - create
           - --host=ingress-nginx-controller-admission,ingress-nginx-controller-admission.$(POD_NAMESPACE).svc
           - --namespace=$(POD_NAMESPACE)
           - --secret-name=ingress-nginx-admission
           env:
           - name: POD_NAMESPACE
             valueFrom:
               fieldRef:
                 fieldPath: metadata.namespace
           image: anjia0532/google-containers.ingress-nginx.kube-webhook-certgen:v1.1.1
           imagePullPolicy: IfNotPresent
           name: create
           securityContext:
             allowPrivilegeEscalation: false
         nodeSelector:
           kubernetes.io/os: linux
         restartPolicy: OnFailure
         securityContext:
           fsGroup: 2000
           runAsNonRoot: true
           runAsUser: 2000
         serviceAccountName: ingress-nginx-admission
   ---
   apiVersion: batch/v1
   kind: Job
   metadata:
     labels:
       app.kubernetes.io/component: admission-webhook
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx-admission-patch
     namespace: ingress-nginx
   spec:
     template:
       metadata:
         labels:
           app.kubernetes.io/component: admission-webhook
           app.kubernetes.io/instance: ingress-nginx
           app.kubernetes.io/name: ingress-nginx
           app.kubernetes.io/part-of: ingress-nginx
           app.kubernetes.io/version: 1.2.1
         name: ingress-nginx-admission-patch
       spec:
         containers:
         - args:
           - patch
           - --webhook-name=ingress-nginx-admission
           - --namespace=$(POD_NAMESPACE)
           - --patch-mutating=false
           - --secret-name=ingress-nginx-admission
           - --patch-failure-policy=Fail
           env:
           - name: POD_NAMESPACE
             valueFrom:
               fieldRef:
                 fieldPath: metadata.namespace
           image: anjia0532/google-containers.ingress-nginx.kube-webhook-certgen:v1.1.1
           imagePullPolicy: IfNotPresent
           name: patch
           securityContext:
             allowPrivilegeEscalation: false
         nodeSelector:
           kubernetes.io/os: linux
         restartPolicy: OnFailure
         securityContext:
           fsGroup: 2000
           runAsNonRoot: true
           runAsUser: 2000
         serviceAccountName: ingress-nginx-admission
   $ kubectl apply -f webhook-jobs.yaml
   job.batch/ingress-nginx-admission-create created
   job.batch/ingress-nginx-admission-patch created
   ~~~

6. 创建ingress controller的SA和Role

   ~~~shell
   $ cat ingress-service-account.yaml
   ---
   apiVersion: v1
   automountServiceAccountToken: true
   kind: ServiceAccount
   metadata:
     labels:
       app.kubernetes.io/component: controller
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx
     namespace: ingress-nginx
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     labels:
       app.kubernetes.io/component: controller
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx
     namespace: ingress-nginx
   rules:
   - apiGroups:
     - ""
     resources:
     - namespaces
     verbs:
     - get
   - apiGroups:
     - ""
     resources:
     - configmaps
     - pods
     - secrets
     - endpoints
     verbs:
     - get
     - list
     - watch
   - apiGroups:
     - ""
     resources:
     - services
     verbs:
     - get
     - list
     - watch
   - apiGroups:
     - networking.k8s.io
     resources:
     - ingresses
     verbs:
     - get
     - list
     - watch
   - apiGroups:
     - networking.k8s.io
     resources:
     - ingresses/status
     verbs:
     - update
   - apiGroups:
     - networking.k8s.io
     resources:
     - ingressclasses
     verbs:
     - get
     - list
     - watch
   - apiGroups:
     - ""
     resourceNames:
     - ingress-controller-leader
     resources:
     - configmaps
     verbs:
     - get
     - update
   - apiGroups:
     - ""
     resources:
     - configmaps
     verbs:
     - create
   - apiGroups:
     - ""
     resources:
     - events
     verbs:
     - create
     - patch
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     labels:
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx
   rules:
   - apiGroups:
     - ""
     resources:
     - configmaps
     - endpoints
     - nodes
     - pods
     - secrets
     - namespaces
     verbs:
     - list
     - watch
   - apiGroups:
     - ""
     resources:
     - nodes
     verbs:
     - get
   - apiGroups:
     - ""
     resources:
     - services
     verbs:
     - get
     - list
     - watch
   - apiGroups:
     - networking.k8s.io
     resources:
     - ingresses
     verbs:
     - get
     - list
     - watch
   - apiGroups:
     - ""
     resources:
     - events
     verbs:
     - create
     - patch
   - apiGroups:
     - networking.k8s.io
     resources:
     - ingresses/status
     verbs:
     - update
   - apiGroups:
     - networking.k8s.io
     resources:
     - ingressclasses
     verbs:
     - get
     - list
     - watch
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     labels:
       app.kubernetes.io/component: controller
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx
     namespace: ingress-nginx
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: Role
     name: ingress-nginx
   subjects:
   - kind: ServiceAccount
     name: ingress-nginx
     namespace: ingress-nginx
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     labels:
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: ingress-nginx
   subjects:
   - kind: ServiceAccount
     name: ingress-nginx
     namespace: ingress-nginx
   $ kubectl apply -f ingress-service-account.yaml
   serviceaccount/ingress-nginx created
   role.rbac.authorization.k8s.io/ingress-nginx created
   clusterrole.rbac.authorization.k8s.io/ingress-nginx created
   rolebinding.rbac.authorization.k8s.io/ingress-nginx created
   clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
   ~~~

7. 通过configmap创建nginx配置，这里只采用默认配置，自定义参考：[Nginx Configuration options](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/)

   ~~~shell
   $ cat configmap.yaml
   ---
   apiVersion: v1
   data:
     allow-snippet-annotations: "true"
   kind: ConfigMap
   metadata:
     labels:
       app.kubernetes.io/component: controller
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx-controller
     namespace: ingress-nginx
   $ kubectl apply -f configmap.yaml
   configmap/ingress-nginx-controller created
   ~~~

8. 创建nginx ingress的service，这里service需要在deployment之前创建

   ~~~shell
   # 由于前面没有lb，所以这里使用nodeport
   $ cat services.yaml
   ---
   apiVersion: v1
   kind: Service
   metadata:
     labels:
       app.kubernetes.io/component: controller
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx-controller
     namespace: ingress-nginx
   spec:
     externalTrafficPolicy: Local
     ports:
     - appProtocol: http
       nodePort: 30080
       name: http
       port: 80
       protocol: TCP
       targetPort: http
     - appProtocol: https
       nodePort: 30443
       name: https
       port: 443
       protocol: TCP
       targetPort: https
     selector:
       app.kubernetes.io/component: controller
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
     type: NodePort
   ---
   apiVersion: v1
   kind: Service
   metadata:
     labels:
       app.kubernetes.io/component: controller
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx-controller-admission
     namespace: ingress-nginx
   spec:
     ports:
     - appProtocol: https
       name: https-webhook
       port: 443
       targetPort: webhook
     selector:
       app.kubernetes.io/component: controller
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
     type: ClusterIP
   $ kubectl apply -f services.yaml
   service/ingress-nginx-controller created
   service/ingress-nginx-controller-admission created
   ~~~

9. 创建部署nginx ingress的deployment

   ~~~shell
   $ cat deployment.yaml
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     labels:
       app.kubernetes.io/component: controller
       app.kubernetes.io/instance: ingress-nginx
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
       app.kubernetes.io/version: 1.2.1
     name: ingress-nginx-controller
     namespace: ingress-nginx
   spec:
     minReadySeconds: 0
     revisionHistoryLimit: 10
     selector:
       matchLabels:
         app.kubernetes.io/component: controller
         app.kubernetes.io/instance: ingress-nginx
         app.kubernetes.io/name: ingress-nginx
     template:
       metadata:
         labels:
           app.kubernetes.io/component: controller
           app.kubernetes.io/instance: ingress-nginx
           app.kubernetes.io/name: ingress-nginx
       spec:
         containers:
         - args:
           - /nginx-ingress-controller
           - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
           - --election-id=ingress-controller-leader
           - --controller-class=k8s.io/ingress-nginx
           - --ingress-class=nginx
           - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
           - --validating-webhook=:8443
           - --validating-webhook-certificate=/usr/local/certificates/cert
           - --validating-webhook-key=/usr/local/certificates/key
           env:
           - name: POD_NAME
             valueFrom:
               fieldRef:
                 fieldPath: metadata.name
           - name: POD_NAMESPACE
             valueFrom:
               fieldRef:
                 fieldPath: metadata.namespace
           - name: LD_PRELOAD
             value: /usr/local/lib/libmimalloc.so
           image: anjia0532/google-containers.ingress-nginx.controller:v1.2.1
           imagePullPolicy: IfNotPresent
           lifecycle:
             preStop:
               exec:
                 command:
                 - /wait-shutdown
           livenessProbe:
             failureThreshold: 5
             httpGet:
               path: /healthz
               port: 10254
               scheme: HTTP
             initialDelaySeconds: 10
             periodSeconds: 10
             successThreshold: 1
             timeoutSeconds: 1
           name: controller
           ports:
           - containerPort: 80
             name: http
             protocol: TCP
           - containerPort: 443
             name: https
             protocol: TCP
           - containerPort: 8443
             name: webhook
             protocol: TCP
           readinessProbe:
             failureThreshold: 3
             httpGet:
               path: /healthz
               port: 10254
               scheme: HTTP
             initialDelaySeconds: 10
             periodSeconds: 10
             successThreshold: 1
             timeoutSeconds: 1
           resources:
             requests:
               cpu: 100m
               memory: 90Mi
           securityContext:
             allowPrivilegeEscalation: true
             capabilities:
               add:
               - NET_BIND_SERVICE
               drop:
               - ALL
             runAsUser: 101
           volumeMounts:
           - mountPath: /usr/local/certificates/
             name: webhook-cert
             readOnly: true
         dnsPolicy: ClusterFirst
         nodeSelector:
           kubernetes.io/os: linux
         serviceAccountName: ingress-nginx
         terminationGracePeriodSeconds: 300
         volumes:
         - name: webhook-cert
           secret:
             secretName: ingress-nginx-admission
   $ kubectl apply -f deployment.yaml
   deployment.apps/ingress-nginx-controller created
   ~~~

10. 从1.0.0开始ingress nginx需要安装IngressClass对象，以便在多个ingress nginx的情况下，ingress对象可以知道它服务于哪个ingress nginx，这里虽然我们只有一个ingress nginx实例，但是还是按照标准来创建

    ~~~shell
    $ cat ingressclass.yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: IngressClass
    metadata:
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
        app.kubernetes.io/version: 1.2.1
      name: nginx
    spec:
      controller: k8s.io/ingress-nginx
    $ kubectl apply -f ingressclass.yaml
    ingressclass.networking.k8s.io/nginx created
    ~~~

11. 最终结果如下

    ~~~shell
    $ kubectl get all -n ingress-nginx
    NAME                                           READY   STATUS      RESTARTS   AGE
    pod/ingress-nginx-admission-create-xvpng       0/1     Completed   0          50m
    pod/ingress-nginx-admission-patch-dxbm7        0/1     Completed   0          50m
    pod/ingress-nginx-controller-99c7c69b6-65gc6   1/1     Running     0          23s
    
    NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
    service/ingress-nginx-controller             NodePort    10.98.63.81     <none>        80:30080/TCP,443:30443/TCP   2m34s
    service/ingress-nginx-controller-admission   ClusterIP   10.106.101.65   <none>        443/TCP                      2m34s
    
    NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/ingress-nginx-controller   1/1     1            1           23s
    
    NAME                                                 DESIRED   CURRENT   READY   AGE
    replicaset.apps/ingress-nginx-controller-99c7c69b6   1         1         1       23s
    
    NAME                                       COMPLETIONS   DURATION   AGE
    job.batch/ingress-nginx-admission-create   1/1           4s         50m
    job.batch/ingress-nginx-admission-patch    1/1           4s         50m
    ~~~

12. 此时我们创建一个nginx服务

    ~~~shell
    $ cat nginxtest.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginxtest
      namespace: default
      labels:
        app: nginxtest
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginxtest
      template:
        metadata:
          labels:
            app: nginxtest
        spec:
          containers:
          - name: nginxtest
            image: nginx:latest
            ports:
            - containerPort: 80
    
    ---
    
    apiVersion: v1
    kind: Service
    metadata:
      name: nginxtest
      namespace: default
      labels:
        app: nginxtest
    spec:
      ports:
      - port: 8000
        targetPort: 80
      selector:
        app: nginxtest
      type: ClusterIP
    
    ---
    
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: nginxtest-ingress
      namespace: default
    spec:
      ingressClassName: nginx
      rules:
      - host: nginx.test.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginxtest
                port:
                  number: 8000
    $ kubectl apply -f nginxtest.yaml
    deployment.apps/nginxtest created
    service/nginxtest created
    ~~~

13. 访问测试

    ~~~shell
    # 配置本地hosts
    $ sudo vi /etc/hosts
    127.0.0.1 nginx.test.com
    # 访问测试
    $ curl http://nginx.test.com:30080
    ~~~

13.问题

- 部署ingress过程中出错

  ~~~shell
  $ kubectl apply -f nginx3.yaml
  deployment.apps/nginxtest created
  service/nginxtest created
  Error from server (InternalError): error when creating "nginx3.yaml": Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": failed to call webhook: Post "https://ingress-nginx-controller-admission.ingress-nginx.svc:443/networking/v1/ingresses?timeout=10s": context deadline exceeded
  ~~~

  参考：[enable-admission-plugins](https://github.com/cert-manager/cert-manager/issues/2640)

  可能：

  - ingress-nginx-controller-admission.ingress-nginx.svc  这个 域名应该使用 ingress-nginx-controller-admission.ingress-nginx.svc.cluster.local：[webhook.cert-manager.io](https://github.com/cert-manager/cert-manager/issues/2640)
  - calico有问题，访问上游dns报错：[coredns](https://github.com/kubernetes/kubernetes/issues/86762)