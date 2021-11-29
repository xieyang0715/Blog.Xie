---
title: "istio 部署模型"
date: 2021-09-29 16:59:58
tags:
- istio
---

# 前言

网格部署模型

- 单网格 *
  - 单网络单集群
  - 单网络多集群
  - 多网络、多集群、多控制平面
    - 多网络互相访问，跨网络访问时，需要南北的MTLS
- 多网格

网络诊断工具

- istioctl proxy-status
- istioctl proxy-config
- istioctl describe pod | svc



<!--more-->
# 部署模型

https://istio.io/v1.4/docs/ops/deployment/deployment-models/

- 单集群：单个k8s集群
- **多集群**：多个k8s集群，但是网格服务跨集群运行
- **单网络**：网格内的微服务，单一网络。网格内的各微服务可以直接访问
- 多网络：跨网络微服务间访问，需要网关来实现
- **单一控制面板**：
- 多控制面板
- **身份标识和信任模型**
- **单网络**
- 多网格
- 租户，应用需求隔离：**1个名称空间代表租户**。或者1个集群代表一个租户



## 考虑因素

- 单集群或多集群？
- 所有service单个网络还是多网络
- 单个控制平面控制1到多个集群，还是多个控制平面，以确保HA



## 常用部署模型

- 单网格 *
  - 单网络单集群
  - 单网络多集群
  - 多网络、多集群、多控制平面
    - 多网络互相访问，跨网络访问时，需要南北的MTLS
- 多网格

### 单网络单集群

<img src="https://istio.io/v1.4/docs/ops/deployment/deployment-models/single-cluster.svg" title="istio v1.4" alt="图来自istio v1.4 control plane models" width="700" height="500"/>

简单易用, 学习使用

不支持故障转移



### 单网络多集群

- 故障隔离、故障转移。

> <div class="admonition-title note"> 
>     <p>
>         万一一个集群故障，就会转移到另一个集群
>     </p>
> </div>

- 万一两个集群在不同的地域，还可以基于区域感知路由
- 多控制平面，有主副之分。 也能实现配置隔离功能。
- 团队项目隔离。
- 单网络的优势，各workload直接访问，不需要istio网关

<img src="https://istio.io/v1.4/docs/ops/deployment/deployment-models/shared-control.svg" title="istio v1.4" alt="图来自istio v1.4 control plane models" width="700" height="500"/>

### 多集群多网络

- 多个网络，在不同的数据中心
  - 左边2个集群在同一个网络，可以直接通信
  - 右边1个是不同网络，通信时要基于gateway完成通信，意味着，跨网络的互相访问，可能带来较严重的延迟。
- 必要的时候要优先本地访问，就是区域感知路由，不同地域的服务访问优先看本地的。
- 多网络时，网关转发才可以访问。
- 使用单个控制平面不合适

<img src="https://istio.io/v1.4/docs/ops/deployment/deployment-models/multi-cluster.svg" width="700" height="500"/>

- 多控制平面
- 每一个控制平面，配置隔离，只需要管理自己区域的配置
- 跨网络通信就需要借助网关实现

<img src="https://istio.io/v1.4/docs/ops/deployment/deployment-models/multi-control.svg" width="700" height="500" />

跨网络通信的逻辑,走的gateway

<img src="https://istio.io/v1.4/docs/ops/deployment/deployment-models/multi-net.svg" width="700" height="500"/>

## 控制平面可用性排序

提高控制可用性，由低到高排序。region在zone中

1. 每地域1集群 1个控制平面 ，region
2. 每地域多集群1个控制平面 
3. 每区域1个集群 zone1个控制平面 
4. 每区域多集群 1个控制平面 
5. 每个集群一个控制平面



## 常用选择

单网络单集群单控制平面

单网络多集群单控制平面

多网络多集群多控制平面

<div class="admonition-title warning">
    <p>
        为了降低复杂度，就使用1个网格。否则就使用网格联邦和多网络模型。
    </p>
</div>

## 网络联邦

支持单网络不具备的功能

- 环境强隔离
- 服务名、名称空间名可利用

跨网格访问时，服务应该使用完全限定名称，不同网格应该使用不同的域名。

### 不交换trust bundle时

<img src="https://istio.io/v1.4/docs/ops/deployment/deployment-models/multi-mesh.svg" width="700" height="500" />



### 交换信任树

spiffe协议，导入对方的trust bundle后

<img src="https://istio.io/v1.4/docs/ops/deployment/deployment-models/multi-trust.svg" width="700" height="500" />

## 租户模型

https://istio.io/v1.4/docs/ops/deployment/deployment-models/#tenancy-models

根据项目模型选择

- 名称空间模型，单一集群，使用多个名称隔离多个租户。隔离不好，不同名称空间可以访问
- 集群模型，多集群时，完全隔离的。



# 安装多集群部署

https://istio.io/v1.4/docs/setup/install/multicluster/

多网络了，就在每个网络中独立的控制平面了，也可以实现配置隔离



# 诊断工具

https://istio.io/v1.4/docs/ops/diagnostic-tools/

子命令

```bash
istioctl --help
 proxy-status <pod-name.namespace> <flags> # 显示网络各envoy从pilot同步配置信息的当前状态
   # istioctl proxy-status  ${POD}.default
   # 简写: istioctl ps  ${POD}.default
 proxy-config # 配置信息
 experimental describe
 experimental authn
 experimental authz
```

## 配置信息

```bash
istioctl proxy-config|pc [command]
```

获取bootstrap配置

```bash
bootstrap <pod-name.namespace> <flags>
```

> ```bash
> istioctl pc bootstrap ${POD}.default
> ```

获取cluster/listneer/route/endpoint/secret/log配置

```bash
cluster
listener
route
endpoint
secret # SDS模型时，获取每个envoy的secret.
log
```

> <div class="admonition-title info">
>     <p>
>         istioctl pc cluster ${POD}.default
>     </p>
>     <p>
>         istioctl pc listener ${POD}.default
>     </p>
> </div>

## pod之上的网格配置(pod监听、service详情、vs、dr、policy、ingress配置)

获取pod上的网格配置或服务上的路由信息

```bash
istioctl experimental|x  describe pod|svc <pod|svc>
```

获取productpage的网络配置

```bash
POD=$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')
root@ip-10-0-0-245:/usr/local/istio# istioctl experimental describe pod $POD
Pod: productpage-v1-596598f447-l77sh
   Pod Ports: 9080 (productpage), 15090 (istio-proxy) # pod监听的端口， pod监听端口。istio proxy端口
--------------------
Service: productpage # service 详情
   Port: http 9080/HTTP targets pod port 9080 # 服务只会暴露的端口
   
   
DestinationRule: productpage for "productpage" # cluster中定义的配置
   Matching subsets: v1 # 这个pod路由到子集v1
   No Traffic Policy    # 没有流量策略
   
Pod is PERMISSIVE, clients configured automatically # POD的服务端(ingress，policy)的MTLS认证类型。客户端(cluster, dr)MTLS认证类型

VirtualService: productpage # 拥有一个路由
   1 HTTP route(s)

Exposed on Ingress Gateway http://10.0.0.245 # 暴露的ingress gateway地址
VirtualService: bookinfo # ingress网关的引用的vs
   /productpage, /static*, /login, /logout, /api/v1/products* # 网关暴露的url
```

获取productpage的服务配置

```bash
root@ip-10-0-0-245:/usr/local/istio# istioctl experimental describe svc  productpage
Service: productpage
   Port: http 9080/HTTP targets pod port 9080
DestinationRule: productpage for "productpage"
   Matching subsets: v1
   No Traffic Policy
Pod is PERMISSIVE, clients configured automatically
VirtualService: productpage
   1 HTTP route(s)


Exposed on Ingress Gateway http://10.0.0.245
VirtualService: bookinfo
   /productpage, /static*, /login, /logout, /api/v1/products*
```



