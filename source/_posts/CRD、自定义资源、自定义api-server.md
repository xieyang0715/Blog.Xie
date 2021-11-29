---
title: CRD、自定义资源、自定义api server
date: 2021-02-03 09:09:31
tags:
---



# 前言

k8s 主节点3个组件

- apiserver: gateway/数据库，借助于etcd实现状态定义，当前状态记录。
  - etcd分布式kv sotre
  - api server对etcd抽象，用户使用api server只能使用api server定义格式存储数据。

现在管理的k8s资源，无法满足用户需求。

- pod, service, deploy, pv, pvc, cm/secret

现在k8s之上托管redis cluster, 使用statefulset管理，没有办法处理redis cluster内部的扩容、缩容、备份、恢复。这个时候使用operator管理，operator建构在statefulset和k8s基础资源的自定义资源。

- 创建Redis Cluster资源时，就可以创建`redis cluster`. 整个集群抽象为一个单一的资源。



光有资源不行，还得有对应的控制器才可以运行pod.

<!--more-->

# 扩展k8s资源类型

- CRD  <- K8S TPR （api server内部）
  - 定义格式
  - 定义资源

> 内建的资源CRD， 扩展资源类型

- 自定义API server：需要写代码。 api server定义为pod, 用户访问api server -> 内建apiserver,  用户访问自定义资源 -> 用户定义的pod
  - APIService，相当于路由信息
- 修改APIServer的源代码，改动内部的资源类型定义。云计算公司才有这个实力。



kind: "ETCDCluster"

- api server内建的资源类型，CRD是类型(人)，创建自定义类型(地球人)，创建实例化资源

- 自定义api server
- 修改 api server源码 

![image-20210203181035506](http://myapp.img.mykernel.cn/image-20210203181035506.png)

# 自定义控制器

不一定每个资源需要控制器，自定义资源只是调用内建资源类型，就不需要控制。

自定义的资源，不仅仅内建功能时，需要对应的控制器

- 程序：托管k8s之上的pod，pod本身受k8s内建的控制器控制，可以保证不宕机。

  - 认证到apiserver
  - 监听自已关心资源的变动，不仅仅和解循环，可能还需要进行备份、恢复。

  <img src="http://myapp.img.mykernel.cn/image-20210203174812889.png" alt="image-20210203174812889" style="zoom:33%;" />

很多人需要快速创建控制器，就有开发框架SDK，加速开发。

operator sdk：coreos研发，开发工具箱，和自定义控制器没有本身区别，但是取名operator。现在主流开发框架。

> 管理有状态应用



# CRD

定义出类型，类型还可以实例化

```bash
[root@master ~]# kubectl explain crd
KIND:     CustomResourceDefinition
VERSION:  apiextensions.k8s.io/v1
   apiVersion	<string>
   kind	<string>
   metadata	<Object>
   spec	<Object> -required-
   status	<Object>

```



```bash
[root@master ~]# kubectl explain crd.spec | grep '>'
RESOURCE: spec <Object>
   conversion	<Object>
   group	<string> -required- # 资源属于哪个群组
     are served under `/apis/<group>/...`. Must match the name of the
     CustomResourceDefinition (in the form `<names.plural>.<group>`).
   names	<Object> -required- # 资源名即定义yaml的Kind
   preserveUnknownFields	<boolean>
   scope	<string> -required- # 资源是集群还是名称空间
   versions	<[]Object> -required- # 资源属于哪个版本或哪些版本
     number (the minor version). These are sorted first by GA > beta > alpha

```



定义crd示例

```yaml
kind: CustomResourceDefinition # 内建
metadata:
  name: users.auth.ilinux.io # kind
spec:
  group: auth.ilinux.io
  versions:  # 版本
  - name: v1beta1
    served: true
    storage: true
  - name: v1beta2
    served: true
    storage: false
  names:
    kind: User         # 告诉定义本身kind
    plural: users      # 复数，必须小写字母
    singular: user     # 单数
    shortNames:        # 短格式名列表
    - u
    categories:       
    - all
  scope: Namespaced # 作用域
  validation: # 用户可以使用的字段
    openAPIV3Schema: # 开放第3版语法格式定义
      properties:
        spec:
          properties:
            userID:
              type: integer
              minimum: 1
              maximum: 65535
            groups:
              type: array
            email:
              type: string
            password:
              type: string
              format: password
          required: ["userID","groups"]
  additionalPrinterColumns:
    - name: userID
      type: integer
      description: The user ID.
      JSONPath: .spec.userID
    - name: groups
      type: string
      description: The groups of the user.
      JSONPath: .spec.groups
    - name: email
      type: string
      description: The email address of the user.
      JSONPath: .spec.email
    - name: password
      type: string
      description: The password of the user account.
      JSONPath: .spec.password
  subresources: # 定义子资源，监控系统给子资源，避免泄漏信息
    status: {}
    scale:
      specReplicasPath: .spec.replicas
      statusReplicasPath: .status.replicas
      labelSelectorPath: .status.labelSelector 

```

```bash
kubectl get crd
[root@master ~]# kubectl apply -f Kubernetes_Advanced_Practical/chapter13/users-crd-multiversions.yaml 
Warning: apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.io/v1 CustomResourceDefinition
customresourcedefinition.apiextensions.k8s.io/users.auth.ilinux.io created


[root@master ~]# kubectl get crd
users.auth.ilinux.io                                  2021-02-03T10:20:09Z


```



定义cr

```bash
[root@master ~]# cat Kubernetes_Advanced_Practical/chapter13/users-demo.yaml 
apiVersion: auth.ilinux.io/v1beta1 # 群集和版本
kind: User # kind
metadata:
  name: admin
  namespace: default
spec: 
  userID: 1
  email: k8s@ilinux.io
  groups:  # 组
  - superusers
  - administrators
  password: ikubernetes

```

```bash
[root@master ~]# kubectl apply -f Kubernetes_Advanced_Practical/chapter13/users-demo.yaml 
```



# APIServer

自定义api server，k8s托管pod，提供自定义资源。api aggregator添加**APIServer**路由信息



APIServer路由

- 访问api server资源端点
  - /apis/域名/v1/namespaces/名称空间名/deployments/myapp-deploy

```bash
[root@master ~]# cat Kubernetes_Advanced_Practical/chapter13/apiserver-demo.yaml 
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v2beta1.auth.ilinux.io
spec:
  insecureSkipTLSVerify: true # aggregator与pod通信 true为http; 生产必须false使用https
  group: auth.ilinux.io  # 引入哪个组
  groupPriorityMinimum: 1000
  versionPriority: 15
  service:                # 指明pod前端的service
    name: auth-api
    namespace: default
  version: v2beta1  # 引入哪个版本


# front-proxy-ca
```







