---
title: '区别metrics, tracing, logging'
date: 2021-07-14 09:24:28
tags:
- kubernetes
- 个人日记
---



> 参考：https://help.aliyun.com/document_detail/68035.html?spm=5176.21213303.J_6028563670.7.69453edaPnj7Yx&scm=20140722.S_help%40%40%E6%96%87%E6%A1%A3%40%4068035.S_hot.ID_68035-RL_jaeger-OR_s%2Bhelpproduct-V_1-P0_0

# 背景
<!--more-->

容器、Serverless编程方式提升了软件交付与部署的效率。架构的演化过程：

- 应用架构从单体 转向 微服务，业务逻辑变成：微服务之间的调用与请求。
- 资源角度来看，传统服务器这个物理单位逐渐淡化，变为了虚拟资源模式。

![img](http://myapp.img.mykernel.cn/p5875.png)

从应用架构和资源角度的2个变化导致的弹性、标准化的架构，使运维与诊断的需求变得复杂。为应对这种架构变化和资源变化，诞生了devops诊断与分析系统，包括：集中式日志系统（Logging）、集中式度量系统（Metrics）和分布式追踪系统（Tracing）。

除Jaeger外，阿里云还提供支持OpenTracing链路追踪产品[XTrace](https://www.aliyun.com/product/xtrace)。

# Logging，Metrics和Tracing的特点

http://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html?spm=a2c4g.11186623.2.12.171f30a8nLGJn5

**metrics** 的特征是可以聚合的（aggregatable），他们是原子，组成一个 一段时间的gauge, counter, histomgram.

例如，队列的当前深度可被定义为一个度量值（gauge），在元素入队或出队时被更新；HTTP请求个数可被定义为一个计数器（ counter），新请求到来时进行累加。将观察到的请求持续时间建模为直方图（histomgram），其更新汇总到时间段中并产生统计摘要。

**logging** 的特征是它处理离散事件。

例如：应用程序调试或错误消息通过旋转文件描述符通过 syslog 发送到 Elasticsearch，审计跟踪事件通过 Kafka 推送到 BigTable 等数据湖；从服务调用中提取并发送到错误跟踪服务（如 NewRelic）的特定于请求的元数据。

trace的唯一定义特征是它处理请求范围内的信息。任何可以绑定到系统中单个事务对象生命周期的数据或元数据。

例如，一次远程方法调用的执行过程和耗时。Tracing是我们排查系统性能问题的利器。

例如：到远程服务的出站 RPC 的持续时间；发送到数据库的实际 SQL 查询的文本；或入站 HTTP 请求的关联 ID。

![修正的、带注释的维恩图](http://myapp.img.mykernel.cn/02.png)

通过以上信息，可以对已有系统进行分类。

例如，Zipkin专注于tracing领域；Prometheus开始专注于metrics，随着时间推移可能会集成更多的tracing功能，但不太可能深入logging领域；ELK、阿里云日志服务这样的系统开始专注于logging领域，但同时也不断地集成其他领域的特性到系统中来，正向上图中的圆心靠近。

# Tracing

流行的Tracing软件有：

- Dapper（Google）：各Tracer的基础
- StackDriver Trace（Google）
- Zipkin（Twitter）
- Appdash（golang）
- 鹰眼（Taobao）
- 谛听（盘古，阿里云云产品使用的Trace系统）
- 云图（蚂蚁Trace系统）
- sTrace（神马）
- X-ray（AWS）

分布式追踪系统发展快，种类多，核心步骤一般有三个：

- 代码埋点
- 数据存储
- 查询展示

下图是一个分布式调用的例子，客户端发起请求，请求首先到达负载均衡器，接着经过认证服务，计费服务，然后请求资源，最后返回结果。

![img](http://myapp.img.mykernel.cn/p5877.png)

数据被采集存储后，分布式追踪系统一般会选择使用包含时间轴的时序图来呈现这个调用链。但在数据采集过程中，由于需要侵入用户代码，并且不同系统的API并不兼容，导致如果切换追踪系统，会有较大改动。

![img](http://myapp.img.mykernel.cn/p5883.png)

# OpenTracing

为了解决不同的分布式追踪系统API不兼容的问题，诞生了[OpenTracing](http://opentracing.io/?spm=a2c4e.11153959.blogcont514488.21.463f30c2m1sv0g)规范。OpenTracing是一个轻量级的标准化层，它位于应用程序/类库和追踪或日志分析程序之间。

![img](http://myapp.img.mykernel.cn/p5878.png)

- OpenTracing优势：
  - OpenTracing已进入CNCF，为全球的分布式追踪，提供统一的概念和数据标准。
  - OpenTracing通过提供平台无关、厂商无关的API，使开发人员能够方便的添加或更换追踪系统。

# Jaeger

Jaeger是Uber推出的一款开源分布式追踪系统，兼容OpenTracing API。

![img](http://myapp.img.mykernel.cn/p5879.png)

如上图所示，Jaeger主要由以下几部分组成。

- Jaeger Client：为不同语言实现了符合OpenTracing标准的SDK。应用程序通过API写入数据，client library把Trace信息按照应用程序指定的采样策略传递给jaeger-agent。
- Agent：它是一个监听在UDP端口上接收span数据的网络守护进程，它会将数据批量发送给collector。它被设计成一个基础组件，部署到所有的宿主机上。Agent将client library和collector解耦，为client library屏蔽了路由和发现collector的细节。
- Collector：接收jaeger-agent发送来的数据，然后将数据写入后端存储。Collector被设计成无状态的组件，因此您可以同时运行任意数量的jaeger-collector。
- Data Store：后端存储被设计成一个可插拔的组件，支持将数据写入cassandra、elastic search。
- Query：接收查询请求，然后从后端存储系统中检索Trace并通过UI进行展示。Query是无状态的，您可以启动多个实例，把它们部署在Nginx负载均衡器后面。



# 配置jaeger对接阿里云

http://cloud.video.taobao.com//play/u/2143829456/p/1/e/6/t/1/50081772711.mp4
