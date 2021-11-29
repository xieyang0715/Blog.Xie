---
title: dapr结合istio
tags:
- istio
- dapr
photos:
- http://myapp.img.mykernel.cn/service-mesh.png
date: 2021-11-11 17:51:45
---

# 前言

开发.net时，结合dapr完成状态存储、消息发布与订阅、服务调用、观测，需要dapr



大概看了架构如下 

<!--more-->



# 官方网站

dapr架构: ![](http://myapp.img.mykernel.cn/overview_kubernetes.png)

> 1. placement, actor
>
> 2. dapr injector: 在deploy添加注释，确定是否注入dapr的sidecar
>
>    dapr的sidecar与服务风格的sidecar区别，服务网格的sidecar是拦截流量。而dapr的sidecar是提供给app访问, 提供给其他dapr的sidecar访问
>
> 3. dapr sentry哨兵，MTLS证书的管理，一般服务网格和dapr只启用一个。dapr启动的话，可能部分应用没有使用dapr就不能TLS。所以一般禁用dapr的mtls
>
> 4. dapr operator管理state状态, 所有state通过`Commponent` K8S资源完成定义，而且这个组件通过dapr的api来让app调用。app定义如下：
>
>    > https://docs.dapr.io/zh-hans/concepts/building-blocks-concept/



官方示例: https://github.com/dapr/samples

quickstart示例:  https://github.com/dapr/quickstarts/tree/master/hello-kubernetes

![img](http://myapp.img.mykernel.cn/Architecture_Diagram.png)

> 1. dapr -> app: https
> 2. app -> dapr: token auth
> 3. dapr -> dapr: app-id
> 4. dapr -> state management, dapr的分布式跟踪才可以跟踪，istio网格不能跟踪
> 5. dapr的metrics和服务风格的可以一同使用

# 开发桥接k8s进行调试

 https://docs.dapr.io/developing-applications/debugging/bridge-to-kubernetes/

