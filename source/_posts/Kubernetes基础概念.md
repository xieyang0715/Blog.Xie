---
title: Kubernetes基础概念
date: 2021-01-13 03:16:58
tags:
toc: true
---

kuberentes迭代非常快，https://github.com/kubernetes/kubernetes  这是官方网址

> 第1个kuberentes是名称空间，用户名
>
> 第2个kuberentes是子项目

 

进入releases界面

![image-20210113112313614](http://myapp.img.mykernel.cn/image-20210113112313614.png)

从左侧的时间看，迭代速度非常快，之所以快，核心的应该出现的功能还没有出现，老版本BUG需要修复。完全进入稳定版本，就不必跟进新版本。

![image-20210113112546564](http://myapp.img.mykernel.cn/image-20210113112546564.png)

<!--more-->

# 目标

新时代的企业级的操作系统

| 开发方式            | 对象     | 环境                         |
| ------------------- | -------- | ---------------------------- |
| c/c++               | 系统 api | 操作系统                     |
| java/python         | jvm/pvm  | 虚拟机                       |
| K8cloud native apps | K8S API  | 云计算环境(kubernetes, ....) |

将来企业使用应用时，找几个裸金属（物理机、VMWARE虚拟机、公有云之上的IAAS环境之上的虚拟机、私有云openstack）跑操作系统的主机，在上面部署haproxy/tomcat/keepalived. 之后部署k8s集群，开发人员针对 k8s api开发程序 。 k8s不能代表云计算环境，但是是最火热的云计算环境，开发出的程序 是cloud native apps。



#  serverless

2019推出serverless, 无服务器的应用程序 ： Knative

**无服务器**: httpd守护进程运行，对系统 资源一直消耗。面serverless是应用程序平时不运行，调用时才运行，每一个应用程序是调用的函数，FaaS: 函数即服务， 函数本身在应用程序服务器上。



# 微服务架构 

开发范式从单体(巨石)应用转向**微服务架构** 

![image](http://myapp.img.mykernel.cn/dubbo-architecture-roadmap.jpg)

>  借用dubbo的图, 网站应用的演进: http://dubbo.apache.org/zh/docs/v2.7/user/preface/background/





**单体架构**：一个程序实现很多功能，多个功能整合在同一个程序 中。模块化、插件、组织逻辑，传统意义上称为单体应用程序。**3-5个服务**

优势：所有应用打包在一起，运行起来稳定。

缺陷：**牵一发而动全身**

- [x] 众多功能在一起，一个小功能的改变，需要整个应用发版。如果一个代码有问题，整个业务会宕机
- [x] 众多功能在一起，启动时间慢。
- [x] 用户量增加，如果简单的扩容后，后面这个节点需要一起在线，扩缩容不灵活。 

**微服务架构**：将单体应用切分**多层**，每层分割成**多个模块**，每个模块是一个微型团队维护。**将3-5个服务抽成30-50个服务，网状。**

优势：

- [x] 快速迭代，应用分割，使用CICD, 可以快速迭代版本而不影响其他应用。
- [x] 启动速度快

问题：

- [x] **服务注册、服务发现**   

  调用不是静态配置实现。Nginx调用tomcat, 静态配置。节点不多的时候可以，万一三五十个服务呢？ 有可能后端 故障，所以这种情况，只能借助服务发现。

  注册 服务提供功能，服务访问接口：**服务总线**

  Client使用功能，不是查询 配置，而是到服务总线查询 有没有自己想要的服务，在通过提供的接口连接server. 一旦服务有故障，总线会移除，cleint会重新请求总线，获取 新的位置。 总线需要高可用。

- [x] 微服务部署

  服务编排系统 ，给一组主机，当作集群对待，运行一个服务，提供对系统 ，系统会找到最适合运行的节点，将此应用程序自动运行在其上。出现故障就需要卸载，重新部署。

  如果手工创建虚拟主机，使用ansible管理运行服务，达到扩容、缩容、故障重建、重新部署，非常难。

- [x] 系统异构

  S1服务是app1应用程序，部署在3个节点上，正好2个节点满足app1的库。第3个节点满足不了。如果3个节点都不满足app1的环境。

  ​    30-50个应用程序 ，部署上去可以，但是运行环境不满足。可以不考虑底层环境，应用程序 自带运行环境，**容器**

  标准化打包应用：**镜像**，一次打包，到处运行。

  CRI接口调用"docker, containerd, ..."把镜像运行为窗口：**容器**

# 容器编排系统

单一容器没有生产价值，编排起来才有价值

##  特点

1） 容器生命周期管理工具

2） 动态环境

## 核心价值 

> 注册和发现  负载均衡、配置和存储、健康、自动扩容、缩容、0宕机

1. 镜像提供和部署

2.  故障的容器自动恢复

3. 应用规模的自动伸缩
   - LB后面server不够，加节点。静态配置，后面节点用或不用都需要放这么节点，**资源闲置**。所以必要时需要缩容。需要请求量大，自动扩容。请求量小时，自动缩容
4. 资源不够迁移容器

5. 暴露容器服务

6. 接入层，只有外部访问。

7. 其他服务只被 前端访问。

8. 健康状态

9. 容器配置

 

## 编排系统 

| 系统                   | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| k8s                    | 1． 最火的，google主导研发，，现在已经损给CNCF(项目：**k8s, prometheus, coredns**,如果项目成熟到可以生产中使用，表示项目在CNCF中毕业，毕业前叫孵化阶段。)组织。成员： google, amazon, Microsoft, ibm, interl, cisco. |
| docker swarm           | swarm是编排系统                                              |
| apache mesos  marathon | mesos 是数据中心，把1万台服务器当成1台服务器使用。marathon是编排系统 |

2017年12月：mesos支持marathon, 转为K8S。 docker EE同时支持swarm和K8S。所以k8s已经成为事实上的标准。

为什么竞争中，2015年诞生的K8S，能一统天下？同样9年义务教育，为什么有些人那么优秀？同样20岁，xxx衣食无忧？k8s诞生于google, 容器技术出现时，容器技术在google中已经运行了10多年了，这个系统叫伯格。10几年的期间，google已经有非常好的经验了。Docker新出来的编排系统，没有经验。K8S是golang把伯格思想沿袭，所以K8S是含着金汤勺出身。



# k8s是什么？

Kubernetes，也称为K8s，是一个开放源码系统，用于自动化部署、扩展和管理容器化应用程序。



# k8s物理结构 

![image-20210115162143039](http://myapp.img.mykernel.cn/image-20210115162143039.png)

运行时的物理结构 

多个底层主机（裸金属、虚拟机…），抽象资源，组成一个平台，统一管理

```
k8s运行一般是直接做行物理机还是虚拟机

你看那些放在阿里云腾讯云的，这不都是运行在虚拟机吗？

云上物理机挂了之后 自动触发热迁移 也就1-3s感知时间 快得很 

这个时候pod也不会被驱逐 就是业务卡了一下
```

每一个方形，代表一个主机

Host分为 master(蜂王)和node(工蜂)





## node

运行容器的主机，在node节点，**data plane**.

Node是负载均衡，多个node同时工作，一个宕机，就在其他node上运行起来。

平时负载均衡，必要时，接管其他node的任务。

## Master

控制中心，**controller plane**, 管理层对外命令统一，所以多个master是冗余 ， 纯粹冗余。 

Master之上，可以实现所有node的CPU/RAM抽离出来，组织成资源池

MASTER明确知道哪个node有多少资源，空闲多少,而且知道哪个node最适合运行，叫**容器调度**  scheduler

K8S运行容器，而调度单元是pod组件



## 角色划分

![image-20210113125131608](http://myapp.img.mykernel.cn/image-20210113125131608.png)

中间：

**master**

**Node**: 运行容器，靠镜像，镜像来自registry



**镜像 registry**

```bash
需要创建、删除很多镜像，自定义镜像时，需要私有registry

公共的: Docker Hub、gcr.io、quay.io；
私服的: harbor
```



**client**

```bash
写程序 ：api

命令：cli, 重点

图形：ui, dashboard, 托管运行于k8s之上。

都是发出请求的发出者，管理者。 Master负责完成容器创建、显示 、修改容器配置、删除容器。必须 把请求发给master, 所以master是控制平面。
```



## master 组件

![image-20210113125312960](http://myapp.img.mykernel.cn/image-20210113125312960.png)

| 组件       | 描述                                                         | 提供者            | 监听端口                                    |
| ---------- | ------------------------------------------------------------ | ----------------- | ------------------------------------------- |
| scheduler  | 1. 选择合适的nodes<br/>2. 根据算法选择合适的node.  默认随机选择一个。 | k8s内建           |                                             |
| controller | 1.通过controller loop 检测当前状态和目标状态，保证pod正常运行。<br/>2. controller扩容或 缩容<br/> | k8s内建           |                                             |
| api server | 整个k8s上，**惟一接收客户端请求的入口**.  请求合符规范，就保存在etcd中 | k8s内建           | https 6443 http 8080(v1.11版本之后废弃8080) |
| etcd       | 保存pod当前状态和目标状态。                                  | coreos 使用go研发 |                                             |

## node组件

![image-20210113130132277](http://myapp.img.mykernel.cn/image-20210113130132277.png)

| 组件       | 描述                                                         | 提供者  |
| ---------- | ------------------------------------------------------------ | ------- |
| kubelet    | watch api，发现是调度到自己，就使用CRI接口调用docker运行容器。  <br/>docker只是k8s的CRI接口支持的容器运行环境之一。环境可以有docker, containerd,  .... | k8s内建 |
| kube-proxy | 创建service, 为其生成iptables, ipvs规则 。删除时，同理删除规则 。 | k8s内建 |
|            |                                                              |         |

## 容器编对象pod

运行的容器是重新封装过的，运行的是**pod**，是容器的另一层外壳。 一个pod中可能多个容器，多个容器当作原子单元。 3个容器只能在同一个节点运行

docker容器利用，内核中6个名称空间技术，实现运行环境隔离：

```bash
PID, Mount, USER, UTS, Network, IPC

UTS:主机名和域名
```



docker网络模型

| 名称   | 描述                          |
| ------ | ----------------------------- |
| closed | 没有网络                      |
| bridge | SNAT                          |
| joined | 两个容器共享UTS, Network, IPC |
| host   | 使用宿主机的网络名称空间      |



pod架构 https://blog.csdn.net/weixin_30794491/article/details/101604904

- [x] 多个容器
- [x] 彼此共享 network, uts, ipc 
- [x] 每个pod底层还有一个容器：infra container, 任何容器加入pod中，共享infra network, uts, ipc
- [x] Docker可以使用存储卷，即infra上的存储卷，如果 加入时，说共享使用infra的存储卷。即可以使用这个存储卷。用得挂载



## 部署环境

| 环境 | master                | node        |
| ---- | --------------------- | ----------- |
| 测试 | 1个master             | 2个node     |
| 生产 | 2个以上master,3个最好 | 至少2个node |

| 角色   | 程序名：应用执行程序的文件名                                 | 描述                                                      |
| ------ | ------------------------------------------------------------ | --------------------------------------------------------- |
| master | kube-apiserver <br/>kube-scheduler<br/>kube-controller-manager<br/> etcd<br/>cloud-controller-manager | cloud-controller-manager把k8s部署在云计算环境中，才会使用 |
| node   | kublet<br/>kube-proxy<br/>container runtime<br/>             | container runtime: docker, containerd, cri-o              |
| addons | dns<br/>web ui(dashboard)<br/>cluster-level logging          | Dns服务完成注册和发现                                     |

# k8s api

整个集群的网关

与其交互：kubectl、dashboard、api

## kubectl

- [x] 陈述式api:  操作过程由调用者决定。                                yaml文件编写 。
- [x] 声明式api:  调用者只管调用，被调用者有自已的逻辑。 kubectl create/run完成。

### 陈述式命令配置

kubectl命令生成，自动生成json，再创建资源对象。

### json/yaml

优点

- [x] 命令今天执行一遍，明天执行会忘记
- [x] 版本控制器(git), 修改后提交，可以查看每一次变更。可以做**内容变动的追踪**



通过yaml文件，kubectl转换为json, 再创建资源对象。

所有资源对象的yaml文件有固定的格式



**陈述式对象**

kubectl create

**声明式对象配置**

kubectl apply 



```bash
apiVersion:  #`kubectl api-versions`群组中某个组的某个版本
kind:         #`kubectl api-resources` 资源类型

metadata:    # 元数据 嵌套的字段
  name: myapp  #  资源名称
  namespace: default # 资源所属的名称空间
  labels:       # 资源的标签
    app: myapp 
  annotations:  # 注解
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2021-01-15T08:09:00Z" # 创建时间 
  selfLink: # url地址
  
spec:       # 用户期望的状态，嵌套的字段 “最重要”        用户期望状态

status:     # 用户不能修改，由k8s自行维护。             当前实际状态
```

> controller loop 不停的检查(死循环)，用户期望状态和当前实际状态不一致，就让当前状态 **接近或吻合**用户期望状态。这个叫**和解循环**(Reconciliation Loop)

## dashboard

## api 

- 支持协议

  - http： api，默认使用json数据(序列化方式)，也可以使用yaml, 因为是json转换成yaml。

    - restful api: **表征状态转移**，是架构范式。设计出来让分布式组件互相调用。范式设计中每一个操作对象，都组织成以下格式

      ```bash
      #协议    地址       端口  服务            版本 资源类型  资源id
      http://localhost:9999/restfulservices/v1/users/{id}
      
      # http/https方法
      method: GET, PUT, POST, DELETE, PATCH, ...
      	GET -> kubectl get
      	DELETE -> kubectl delete
      	...
      	
      	
      # api版本 kubectl api-versions
      # 群组/版本         -> 通常格式，多个版本配置方式不同
      # 版本             -> 表示属于core组
      [root@node01 ~]# kubectl api-versions
      admissionregistration.k8s.io/v1
      admissionregistration.k8s.io/v1beta1
      apiextensions.k8s.io/v1
      apiextensions.k8s.io/v1beta1
      apiregistration.k8s.io/v1
      apiregistration.k8s.io/v1beta1
      
      # deploy相关. v1.10之后deploy位置
      apps/v1
      
      authentication.k8s.io/v1
      authentication.k8s.io/v1beta1
      authorization.k8s.io/v1
      authorization.k8s.io/v1beta1
      autoscaling/v1
      autoscaling/v2beta1
      autoscaling/v2beta2
      batch/v1
      batch/v1beta1
      certificates.k8s.io/v1
      certificates.k8s.io/v1beta1
      coordination.k8s.io/v1
      coordination.k8s.io/v1beta1
      discovery.k8s.io/v1beta1
      events.k8s.io/v1
      events.k8s.io/v1beta1
      extensions/v1beta1
      flowcontrol.apiserver.k8s.io/v1beta1
      networking.k8s.io/v1
      networking.k8s.io/v1beta1
      node.k8s.io/v1
      node.k8s.io/v1beta1
      policy/v1beta1
      
      # 授权相关
      rbac.authorization.k8s.io/v1
      rbac.authorization.k8s.io/v1beta1
      
      scheduling.k8s.io/v1
      scheduling.k8s.io/v1beta1
      storage.k8s.io/v1
      storage.k8s.io/v1beta1
      v1                            # pod属于v1版本中
      
      
      
      # 资源类型(class) 实例化出对象(instance)
      基本对象：pod, servie, namespace, volume
      
      高级对象：建立在基础对象之上，提供额外功能
      
      控制器对象：replicaset,, deployment, daemonset, statefulset, job
      
      
      # k8s支持很多种资源`kubectl api-resources`, 他们均属于某个版本
          # k8s将资源分成多个逻辑组合
      
          # 每个组合可以独立演进
          # 一个组合是 api群组 `kubectl api-versions`
          # 每个组合有不同的版本，不同的版本有不同的功能特性。 
              # [root@node01 ~]# kubectl get deploy myapp -o yaml
              # apiVersion: apps/v1
      
          # 声明式api，手写yaml，可以指定apiVersion
      # 陈述式api, kubectl create完成。
      ```
    
      
  
  - grpc协议，google研发分布式rpc协议，



# deployment

运行容器：pod controller, 也叫deployment

1. 创建pod
2. 暴露服务
3.  健康状态
4. 检测重启

pod崩溃，ip 地址变了，必须服务注册和服务发现：dns完成。

# service、label、label selector

dns解析的是service域名和ip。service通过**label selector**( 标签选择器)查找pod.deployment也可以通过label selector查找 pod.

label: K8s可以给每个资源附加一个标签。相同pod有相同的标签

Service不是服务，只要不删除就有一个ip. 简单理解 service就是ipvs/iptables规则 。Service也可能 被 删除重建。因此k8s需要重要的附件，是动态的。

创建service,有一个名称和IP地址，会转换为A记录，动态注册DNS上。向DNS请求域名，会是建立的名称和IP地址的映射。



# k8s网络

![image-20210113133438796](http://myapp.img.mykernel.cn/image-20210113133438796.png)

Node ip配置在节点的网卡上

Pod 虚拟网卡

Cluster ip 出现在iptables/ipvs/dns解析记录中