---
title: 2021-04-02 UOKO面试
date: 2021-04-02 07:42:15
tags:
- 个人日记
---



第1轮：感觉面试官技术很牛，很善良。对我目前使用的架构提出很多质疑，我也问了很多问题，感觉这像一次面试，而是一次技术讨论会。

- [x] 自我介绍
  - [x] ...
- [x] k8s日志收集架构
  - [x] filebaet+java(调优过) -> redis -> logstash(解析) -> es -> kibana
    - [x] filebeat和java同生命周期，java崩了，filebeat也收集完日志了。
    - [x] java怎么调优。必须使用jar选项加内存限制，不然一直耗内存，就OOM. 
    - [x] redis单机挂了？filebaet一直写，会丢失日志。
      - [x] 得高可用的kafka
    - [x] logstash解析，日志多行得合并。日志错误得报警
    - [x] kibana上次面试问的哪些字段慢，后期我问了这个问题，链路追踪完全可以分析出哪里慢了。
- [x] docker网络，常用的。bridge
- [x] 问了shell常用命令，循环, ....， 文件取词数量，.... 太基础了。

- [x] 如果要上200个pod, 你怎么优化
  - [x] 网段
  - [x] api, kubelet, kube-prxoy调优

- [x] istio, servermesh
  - [x] 目前在学习阶段
- [x] 生产使用有状态服务吧？
  - [x] 就使用operator管理，nfs性能差， ceph rbd正在 学习中。

然后让我问，我问

- [x] 运维几个人
  - [x] 2
- [x] 开发几个人
  - [x] 40个
- [x] 机器多少台
  - [x] 60个机器
- [x] 公司啥语言
  - [x] java
- [x] 公司上k8s技术用了哪些
  - [x] 测试，实验环境使用。
- [x] 有状态服务用啥存储？
- [x] 你们使用了istio, servermesh吗？
  - [x] 没有，将来准备上。
- [x] redis单机的问题。kafka
- [x] 监控。他们直接在consolu注册中心获取java的地址直接监控存活性和jvm状态。
- [x] 我发现nginx的upstream_response_time并没有啥用。链路追踪：zipkin, cat ,pinpoint https://www.cnblogs.com/yychuyu/p/13324532.html

第二轮boss面试

- [x] 自我介绍

  - [x] ...

- [x] 几个运维

  - [x] 1个

- [x] 你遇到一个不会的问题，怎么解决的？

  - [x] prometheus监控k8s, 查看网上资料，官方文档

- [x] 你找工作的要求

  - [x] kubernetes, cicd方向

- [x] 你对公司的要求

  - [x] 只要技术符合我的方向就可以了。

- [x] 你将来的规划

  - [x] 架构师，目前我了解现在技术使用的趋势，去学习它，后面学习一门开发语言。

  

<!--more-->
