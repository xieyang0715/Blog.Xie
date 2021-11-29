---
title: 可观测性应用istio
date: 2021-09-20 11:38:16
tags:
- istio
---



# 统计

- envoy状态统计
  - stats sink
  - 配置案例
  - 将指标数据纳入监控系统: statsd_exporter + prometheus + grafana
- 访问日志
  - 格式规则和命令操作符
  - 配置语法和配置案例
  - 日志存储检索系统：filebeat + elasticsearch + kibana
- 分布式跟踪
  - 分布式追踪基础概念
  - 分布式跟踪的工作机制
  - Envoy的分布式跟踪
  - 使用示例：zipkin跟踪服务
  - 使用示例：jaeger跟踪服务

## 可观测性应用

可观测性应用：监控、指标、报警、日志、分布式跟踪机制

- 日志：时间序列数据，帮助开发者排查问题。对单体应用有效。但是分布式故障通常由多个组件之间互连事件触发。elasticstack/splunk/fluentd
- 指标：时序性收集和记录的固定类型可聚合数据，单体有效。但是无法提供足够的信息理解分布式系统调用 RPC的生命周期。prometheus/statsd
- 跟踪：一个请求事务 过程及流经的组件。适合分布式系统中，定位问题发生的位置。 zipkin
  - 通常需要代码侵入，才可以生成原始数据

> 对于应用（envoy）需要3种不同的系统，来支撑。



# envoy指标

`envoy_host:admin_port/stats`接口可以输出统计数据，3类

- 下游：envoy接收连接的统计信息：listener/http connect manager/tcp proxy filter ..
- 上游:  envoy发送请求的统计信息：连接 池、路由器过滤器、tcp proxy filter
- envoy服务器：记录了envoy服务工作细节，正常运行时间、分配的内存量等。
- 数据类型
  - counter: 累加型，单调递增， 例如：total request
  - gauge:  常规数据，可增可减。 例如 current active request
  - histogram: 柱状图数据，主要用于统计一些数据的分布情况，用于计算在一定范围内的分布情况，同时还提供了度量指标值的总和，例如：upstream request time.

## stats sink

能将envoy统计数据存储到外部指标存储系统中。statsd 实例/集群或statsd_exporter集群。

bootstrap的顶级配置段

```yaml
stats_sinks: []
- name: envoy_statsd
  config:                
    address:            # statsd实例的ip地址 
    tcp_cluster_name:   # envoy上的sink服务器组成的集群，与addresses互斥
    
    
stats_config: {}
stats_flush_interval: {} # 默认5000ms, 仅周期刷写gauges, counters.
```

statsd_exporter 可以暴露给prometheus. 

## prometheus

采集->存储->告警->展示

采集：

- pull
- push：Promtheus,

存储

告警

展示

## stats_sink将数据存入prometheus

envoy的stats_sink定义

```yaml
stats_sinks:
- name: envoy.statsd
  config:
    tcp_cluster_name: statsd-exporter
    prefix: front-envoy
static_resources:
  clusters:
  - name: statsd-exporter
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: statsd-exporter
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: statsd_exporter
                port_value: 9125
```

prometheus定义

```yaml
global:
  scrape_interval:  15s
scrape_configs:
  - job_name: 'statsd'
    scrape_interval: 5s
    static_configs:
      - targets: ['statsd_exporter:9102']
        labels:
          group: 'services'
```

> 5s原因是 statd 5000ms刷新数据
>
> `9102` 抓取端口，相当于采集pushgateway, 原因是envoy将数据推送给statsd, prometheus并没有直接采集envoy, 而是采集statsd的。
>
> `9125`采集的端口



## 架构图

![image-20210920163014815](http://myapp.img.mykernel.cn/image-20210920163014815.png)

```bash
# ls -l servicemesh_in_practise-master/monitoring-and-tracing/statsd-sink-and-prometheus
-rw-r--r--. 1 root root 1939 Apr 23  2020 docker-compose.yaml
-rw-r--r--. 1 root root 1748 Apr 23  2020 front-envoy.yaml
drwxr-xr-x. 2 root root   48 Apr 23  2020 grafana
drwxr-xr-x. 2 root root   25 Apr 23  2020 prometheus
drwxr-xr-x. 2 root root   32 Apr 23  2020 service_blue
drwxr-xr-x. 2 root root   32 Apr 23  2020 service_green
drwxr-xr-x. 2 root root   32 Apr 23  2020 service_red
```

> `docker-compose.yaml` 3个容器 red/blue/green, 1个statsd_exporter, 1个prometheus, 1个grafana.
>
> `front-envoy.yaml` 侦听在80反代到red/blue/green 3个为一个集群。配置2个集群，一个是前面3个合并的，另一个指向statsd_exporter的集群。另一个侦听是 admin接口
>
> `grafana`配置了grafana配置：认证。数据源，指向protheus: 可编辑。
>
> `prometheus` 采集自身和statsd_exporter的9102.

启动

```bash
docker-compose up
```

先看集群状态

```bash
[root@ip-10-0-0-91 statsd-sink-and-prometheus]# docker exec statsd-sink-and-prometheus_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:50:02  
          inet addr:192.168.80.2  Bcast:192.168.95.255  Mask:255.255.240.0
```

```bash
[root@ip-10-0-0-91 statsd-sink-and-prometheus]# curl -s 192.168.80.2:9901/clusters  | grep flag
statsd_exporter::192.168.80.5:9125::health_flags::healthy
mycluster::192.168.80.7:80::health_flags::healthy
mycluster::192.168.80.3:80::health_flags::healthy
mycluster::192.168.80.4:80::health_flags::healthy

```

再访问envoy, 生成标数据。

```bash\
[root@ip-10-0-0-91 statsd-sink-and-prometheus]# while true; do curl -s 192.168.80.2; sleep 0.$[$RANDOM%10] ; done
Hello from App proxied by Envoy, Hostname: 35ac7c3a56f1, Address: 192.168.80.4!
Hello from App proxied by Envoy, Hostname: 3df47c05d2c4, Address: 192.168.80.7!
Hello from App proxied by Envoy, Hostname: 3df47c05d2c4, Address: 192.168.80.7!
Hello from App proxied by Envoy, Hostname: f478957ddc6e, Address: 192.168.80.3!
Hello from App proxied by Envoy, Hostname: 35ac7c3a56f1, Address: 192.168.80.4!
Hello from App proxied by Envoy, Hostname: f478957ddc6e, Address: 192.168.80.3!
Hello from App proxied by Envoy, Hostname: 3df47c05d2c4, Address: 192.168.80.7!
Hello from App proxied by Envoy, Hostname: f478957ddc6e, Address: 192.168.80.3!
```

> 生成离散时间序列数据，被 statsd_exporter获取数据，然后prometheus可以抓取到数据

现在查看prometheus 9090

```bash
front_envoy_cluster_manager_active_clusters # 活动集群，2个

```

现在查看 grafana 3000



# envoy访问日志 `typed_config.access_log`

日志 协助问题的排查，指标查看系统运行状态

日志本身会对系统性能带来开销。

如何采集日志？

- 主机：指定路径下的日志文件中。

tcp_proxy和http_connection_manager过滤器支持可扩展的访问日志，并支持任意数量的访问日志，访问日志过滤器支持自定义日志格式，且允许将不同类型的请求和响应写入不同的访问日志中；

访问日志也支持将数据保存于相应的后端存储系统(sink)中，支持接收器：

- 文件
  - 异步io架构，访问日志记录不会阻塞主线程；
  - 可自定义访问日志格式，使用预定义字段以及HTTP请求响应报文的任意标头。
- GRPC
  - HTTP 2.0实现数据传输
  - 自行开发支持grpc日志记录

## http_connection_manager

访问日志配置语法

```yaml
filter_chains:
  filter:
  - name envoy.http_connection_manager
    config:
      access_log:
        name: # 访问日志实现的名称 这里envoy.http_grpc_access_log, file, tcp
        filter: {} # 信息的filter
        config: {}
          path:             # 本地文件系统的日志文件路径
          format:           # envoy有默认的日志格式
          json_format:      # json格式的访问日志字符串,方便ELK收集，不支持嵌套，与format互斥
```

> LDS发现可以获取

```bash
ls servicemesh_in_practise-master/monitoring-and-tracing/access-log
-rw-r--r--. 1 root root 1011 Apr 23  2020 docker-compose.yaml
-rw-r--r--. 1 root root 2383 Apr 23  2020 front-envoy.yaml
```

> `docker-compose.yaml` 
>
> `front-envoy.yaml` 
>
> ```yaml
> static_resources:
>   listeners:
>   - address:
>       socket_address:
>         address: 0.0.0.0
>         port_value: 80
>     name: listener_http
>     filter_chains:
>     - filters:
>       - name: envoy.http_connection_manager
>         typed_config:
>           "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
>           codec_type: auto
>           access_log:
>           - name: envoy.file_access_log
>             config:
>               path: "/dev/stdout"
>               json_format: {"start": "[%START_TIME%] ", "method": "%REQ(:METHOD)%", "url": "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%", "protocol": "%PROTOCOL%", "status": "%RESPONSE_CODE%", "respflags": "%RESPONSE_FLAGS%", "bytes-received": "%BYTES_RECEIVED%", "bytes-sent": "%BYTES_SENT%", "duration": "%DURATION%", "upstream-service-time": "%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%", "x-forwarded-for": "%REQ(X-FORWARDED-FOR)%", "user-agent": "%REQ(USER-AGENT)%", "request-id": "%REQ(X-REQUEST-ID)%", "authority": "%REQ(:AUTHORITY)%", "upstream-host": "%UPSTREAM_HOST%", "remote-ip": "%DOWNSTREAM_REMOTE_ADDRESS_WITHOUT_PORT%"}
>               #format: "[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" \"%DOWNSTREAM_REMOTE_ADDRESS_WITHOUT_PORT%\"\n"
>           stat_prefix: ingress_http
>           route_config:
>             name: local_route
>             virtual_hosts:
>             - name: vh_001
>               domains: ["*"]
>               routes:
>               - match:
>                   prefix: "/"
>                 route:
>                   cluster: mycluster
>           http_filters:
>           - name: envoy.router
> ```
>
> > `file_access_log` 写入文件
> >
> > `path: "/dev/stdout"` 标准输出
> >
> > `json_format` 格式
> >
> > `%DOWNSTREAM_REMOTE_ADDRESS_WITHOUT_PORT%` 下游客户端地址，不包含端口

```bash
docker-compose up
```

集群状态

```bash
[root@ip-10-0-0-91 ~]# docker exec access-log_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:60:02  
          inet addr:192.168.96.2  Bcast:192.168.111.255  Mask:255.255.240.0


[root@ip-10-0-0-91 ~]# curl 192.168.96.2
Hello from App proxied by Envoy, Hostname: c6e98fa8a451, Address: 192.168.96.3!

```

控制台输出

```bash
front-envoy_1    | {"status":"200","upstream-host":"192.168.96.3:80","upstream-service-time":"1004","bytes-received":"0","url":"/","authority":"192.168.96.2","protocol":"HTTP/1.1","duration":"1005","bytes-sent":"80","request-id":"fc45f467-e78d-427c-89bf-9029dbb25ac1","x-forwarded-for":"-","respflags":"-","method":"GET","user-agent":"curl/7.29.0","remote-ip":"192.168.96.1","start":"[2021-09-20T13:01:40.185Z] "}
```

查看非json时，把非json取消注释，把json_format注释即可。

也可以把所有注释 ，会使用envoy的默认格式

查看日志，通过容器控制台查看非常低效，一旦容器故障或宿主机故障，分析日志很难做到，通常需要集中统一的日志存储、检索、分析平台：早期ELK(elstaicsearch, logstash, kibana)

 后来，采集日志使用filebeat, 而logstash只做日志聚合和格式转换和丰富。较大平台在fileatbeat和logstash之间使用redis/kafka，避免过多的filebeat发来大量的日志信息淹没logstash.

正常每个envoy将日志发给logstash, 聚合转换丰富之后发给eslasticsearch, 最终由1个kibana从elasticsearch展示。

正常情况下，容器日志保存在/var/lib/docker/容器id/*.log文件中，因此只需要在宿主机部署一个filebeat就可以收集当前主机所有容器的日志(json file logging driver).

```bash
ls -l servicemesh_in_practise-master/monitoring-and-tracing/accesslog-with-efk

total 8
-rw-r--r--. 1 root root 1846 Apr 23  2020 docker-compose.yaml
drwxr-xr-x. 2 root root   27 Apr 23  2020 filebeat
-rw-r--r--. 1 root root 2004 Apr 23  2020 front-envoy.yaml

```

> `docker-compose.yaml`kibana与es版本必须一致，而kibana版本不强制一致
>
> `filebeat` 从容器收集日志
>
> ```yaml
> filebeat.inputs:
> - type: container
>   paths: 
>     - '/var/lib/docker/containers/*/*.log'
> 
> processors:
> - add_docker_metadata:
>     host: "unix:///var/run/docker.sock"
> 
> - decode_json_fields:
>     fields: ["message"] # 默认每行Message: 行，这里表示拆开
>     target: "json"
>     overwrite_keys: true
> 
> output.elasticsearch:
>   hosts: ["elasticsearch:9200"]
>   indices: 
>     - index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}" # 以天为索引
> 
> logging.json: true          
> logging.metrics.enabled: false
> ```

```bash
docker-compose up
```

集群状态

```bash
[root@ip-10-0-0-91 accesslog-with-efk]# docker exec accesslog-with-efk_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:70:08  
          inet addr:192.168.112.8  Bcast:192.168.127.255  Mask:255.255.240.0

[root@ip-10-0-0-91 accesslog-with-efk]# curl -s 192.168.112.8:9901/clusters  | grep flag
```

生成日志信息

```bash
[root@ip-10-0-0-91 accesslog-with-efk]# while true; do curl -s 192.168.112.8 ; sleep 0.$[$RANDOM%10]; done
Hello from App proxied by Envoy, Hostname: f679013d56ba, Address: 192.168.112.4!
Hello from App proxied by Envoy, Hostname: 2a891674dfae, Address: 192.168.112.2!
Hello from App proxied by Envoy, Hostname: 2a891674dfae, Address: 192.168.112.2!
Hello from App proxied by Envoy, Hostname: 7b56f6d22d9b, Address: 192.168.112.3!

```

> front envoy查看日志
>
> ```bash
> front-envoy_1    | {"authority":"192.168.112.8","protocol":"HTTP/1.1","duration":"1002","bytes-sent":"81","request-id":"1b839143-409e-445f-89da-cafa7df7c497","x-forwarded-for":"-","respflags":"-","method":"GET","user-agent":"curl/7.29.0","remote-ip":"192.168.112.1","start":"[2021-09-20T13:38:26.457Z] ","status":"200","upstream-host":"192.168.112.3:80","upstream-service-time":"1002","bytes-received":"0","url":"/"}
> ```

查看日志

```bash
[root@ip-10-0-0-91 ~]# curl localhost:9200/_cat/indices
green  open .kibana_task_manager_1           yLp_1r1LTdKDYYTo-XoSvQ 1 0   2 4 45.5kb 45.5kb
green  open .apm-agent-configuration         YACVNIozR3alRYkdc8wnxg 1 0   0 0   230b   230b
green  open .kibana_1                        4pjRPDhgQ42mxNveHE-k_Q 1 0   2 0 11.1kb 11.1kb
yellow open filebeat-7.4.1-2021.09.20-000001 7mxFNCCUQ02qeM5Z41DI2w 1 1   0 0   230b   230b
yellow open filebeat-7.4.1-2021.09.20        8mqIbG2RTdWHiRvn5-hAqQ 1 1 456 0  618kb  618kb
```

查看kibana`0.0.0.0:5601->5601/tcp, :::5601->5601/tcp             accesslog-with-efk_kibana_1`

通过5601查看



# 指标和日志整合在一起使用

## 示例架构

![image-20210920215227643](http://myapp.img.mykernel.cn/image-20210920215227643.png)

## 运行

```bash
ls -l servicemesh_in_practise-master/monitoring-and-tracing/monitoring-and-accesslog
[root@ip-10-0-0-91 monitoring-and-accesslog]# ls -l
total 228
-rw-r--r--. 1 root root   3299 Apr 23  2020 docker-compose.yml
-rw-r--r--. 1 root root  31682 Apr 23  2020 envoy_monitoring.png
drwxr-xr-x. 2 root root     27 Apr 23  2020 filebeat
drwxr-xr-x. 2 root root     31 Apr 23  2020 front_envoy
drwxr-xr-x. 2 root root    120 Apr 23  2020 grafana
-rw-r--r--. 1 root root 190766 Apr 23  2020 grafana.png
drwxr-xr-x. 2 root root     25 Apr 23  2020 prometheus
-rw-r--r--. 1 root root    511 Apr 23  2020 Readme.md
drwxr-xr-x. 2 root root     64 Apr 23  2020 service_a
drwxr-xr-x. 2 root root     64 Apr 23  2020 service_b
drwxr-xr-x. 2 root root     64 Apr 23  2020 service_c
```

> 中间的a, 有ingress和egress. 所有 envoy需要json format accesslog

启动

```bash
[root@ip-10-0-0-91 monitoring-and-accesslog]# docker-compose up
```

查看集群状态

```bash
[root@ip-10-0-0-91 accesslog-with-efk]# docker exec monitoring-and-accesslog_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:80:0A  
          inet addr:192.168.128.10  Bcast:192.168.143.255  Mask:255.255.240.0
```

```bash
[root@ip-10-0-0-91 accesslog-with-efk]# curl -s 192.168.128.10:9901/clusters | grep -i flag
statsd-exporter::192.168.128.9:9125::health_flags::healthy
service_a::192.168.128.4:8786::health_flags::healthy
```

现在生成一些日志

```bash
[root@ip-10-0-0-91 accesslog-with-efk]# while true; do curl -s 192.168.128.10 ; sleep 0.$[$RANDOM%10]; done
```

而且在grafana, 上可以看到一个内建的面板。

由于架构图上有 a->b是 abort故障，所以有一部分的5xx

![image-20210920222347685](http://myapp.img.mykernel.cn/image-20210920222347685.png)

a->c 有延迟故障，请求有些达1s

![image-20210920222414454](http://myapp.img.mykernel.cn/image-20210920222414454.png)

这个就模拟了服务网桥内部的通信效果。

5601 kibana也会展示日志



# envoy跟踪

跟踪和分析所有 服务的事务发生请求的路径: 时间、情况、...

```bash
front -> a -> c
           -> b -> 存储
```

> 定位请求的性能问题

分布式跟踪有助于定位如下问题

- 请求通过哪些服务完成
- 系统瓶颈在何处，每一跳通过多长时间完成
- 哪些服务之间进行了什么样的通信
- 针对给定的请求的每项服务发生了什么

## 实现方式

基于模式：自定义日志消息予以公开。与采样机制不兼容。侵入代码

黑盒接口：不侵入代码，推断因果，无法正确解释异步行为、并发、聚合以及专有代码模式。

**白盒接口**：基于元数据传播。时下最为广泛的方法，本质上讲对代码侵入性低。

- 手动在特定跟踪点添加检测机制；另外也可以使用通用RPC库，自动为每个调用添加元数据。
- **单个跟踪或工作流**(**trace id**)，**其中每个特定跟踪点(span id)**，  **span就是一次请求调用**以及**span起始和结束时间**。
- 每个span都记录了当前id, 和父id. 根节点的span id为1

```bash
					span id:1

parent id:1                  parent id:1
span id: 2                   span id: 3 


							parentid:3 							parentid:3
							span id: 4                          span id: 5
```

span内部结构

- CS起始时间: 客户端发起请求的时间
- SR 服务端接收到请求的时间， SR-CS就是请求在网络上经由的时长
- SS 服务端发送响应的时间。     SS-SR就是服务端处理的时间
- CR 客户端接收时间，同时意味着span结束时间。 CR - CS 客户端发起请求到获取回应的所有 时长



## 生成span, 启用跟踪

1. 大量代码中定义span, 代码被侵入
2. 轻量侵入，植入公共组件中，组件支持跟踪，每一次网络请求就会自动生成跟踪。
   -  opentracing框架代码，嵌入代码中即可，自己写入了一套通用的数据上报接口。
   - opencensus框架代码，
   - opentelemetry框架代码，集成opentracing和opencensus优点
3. envoy 以ingress, egress工作在应用代理的前端，自身集成分布式跟踪功能，支持不侵入应用代码而跟踪。**要实现代码内部详细信息检测，需要侵入代码**，调用底层库完成跟踪。

框架代码上报的位置

- zipkin 收集上报信息，跟踪
  - collector 采集
  - storage 存储
  - api 统一查询接口
  - web ui 展示
- jaeger 兼容opentracing
  - jaeger-client
  - jaeger-agent,   节点采集
  - jaeger-collector 统一采集
  - db
  - jaeger-query 查询接口 
  - jaeger-ui 统一展示接口



## envoy启用分布式跟踪

- 生成请求id: UUID, 放在请求报文首部添加x-request-id标头，应用程序会转发标头。
- 集成外部跟踪服务：lightstep, zipkin , jaeger.
- egress envoy 接收前端请求向后端请求

配置

- 定义zipkin|jaeger集群
- tracing顶级
  - 定义收集跟踪数据的集群，已经静态定义好了
  - collector_endpoint 接收信息的api
  - trace_id_128bit:
  - shared_span_content:  client and server共享span id 默认true
- http_connection_manager启用tracing
  - operation_name 要么 ingress要么egress
  - request_headers_for_tags 为span添加标签
  - client sampling 客户端采样
  - random_samp
  - overall_sampling
  - verbose 



配置示例

1. http_connection_manager启用tracing

   ```yaml
   static_resources:
     listeners:
     - name: http_listener
       address:
         socket_address:
           address: 0.0.0.0
           port_value: 80
       filter_chains:
         filters:
         - name: envoy.http_connection_manager
           config:
             generate_request_id: true
             tracing:
               operation_name: egress
             use_remote_address: true
             add_user_agent: true
             access_log:
             - name: envoy.file_access_log
               config:
                 path: "/dev/stdout"
             stat_prefix: ingress_80
             codec_type: AUTO
             generate_request_id: true
             route_config:
               name: local_route
               virtual_hosts:
               - name: http-route
                 domains:
                 - "*"
                 routes:
                 - match:
                     prefix: "/"
                   route:
                     cluster: service_a
             http_filters:
             - name: envoy.router
   ```

2. span传播机制

   ```yaml
   tracing:
     http:
       name: envoy.zipkin
       typed_config:
         "@type": type.googleapis.com/envoy.config.trace.v2.ZipkinConfig
         collector_cluster: zipkin
         collector_endpoint: "/api/v1/spans"
   ```

3. 定义zipkin集群

   ```yaml
     clusters:
     - name: zipkin
       connect_timeout: 1s
       type: strict_dns
       lb_policy: ROUND_ROBIN
       load_assignment:
         cluster_name: zipkin
         endpoints:
         - lb_endpoints:
           - endpoint:
               address:
                 socket_address:
                   address: zipkin
                   port_value: 9411
   ```

   



4. 应用程序需要传递标头， 获取更为丰富的信息

   zipkin: (x-b3-traceid,  x-b3-spanid, x-b3-parentspanid, x-b3-sampled和x-b3-flags)

   datadog: (x-datadog-trace-id, x-datadog-parent-id, x-datadog-sampling-priority)

   lightstep: (x-ot-span-content)

   以go为例

   ```python
   	req.Header.Add("x-request-id", r.Header.Get("x-request-id"))
   	req.Header.Add("x-b3-traceid", r.Header.Get("x-b3-traceid"))
   	req.Header.Add("x-b3-spanid", r.Header.Get("x-b3-spanid"))
   	req.Header.Add("x-b3-parentspanid", r.Header.Get("x-b3-parentspanid"))
   	req.Header.Add("x-b3-sampled", r.Header.Get("x-b3-sampled"))
   	req.Header.Add("x-b3-flags", r.Header.Get("x-b3-flags"))
   	req.Header.Add("x-ot-span-context", r.Header.Get("x-ot-span-context"))
   ```

## envoy之上完成zipkin



```bash
[root@ip-10-0-0-91 ~]# ls -l servicemesh_in_practise-master/monitoring-and-tracing/zipkin-tracing/
total 40
-rw-r--r--. 1 root root  1735 Apr 23  2020 docker-compose.yml
-rw-r--r--. 1 root root 31682 Apr 23  2020 envoy_tracing.png
drwxr-xr-x. 2 root root    31 Apr 23  2020 front_envoy
-rw-r--r--. 1 root root   275 Apr 23  2020 Readme.md
drwxr-xr-x. 2 root root    64 Apr 23  2020 service_a
drwxr-xr-x. 2 root root    64 Apr 23  2020 service_b
drwxr-xr-x. 2 root root    64 Apr 23  2020 service_c
```

> `docker-compose.yaml` 9411的zipkin，web ui.

启动

```bash
docker-compose up
```

集群状态

```bash
[root@ip-10-0-0-91 ~]# docker exec zipkin-tracing_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:A0:09  
          inet addr:192.168.160.9  Bcast:192.168.175.255  Mask:255.255.240.0

[root@ip-10-0-0-91 ~]# curl 192.168.160.9
Calling Service B: Hello from service B.
Hello from service A.
Hello from service C.

```

现在发起一些请求

```bash
[root@ip-10-0-0-91 zipkin-tracing]# while true; do curl -s 192.168.160.9 ; sleep 0.$[$RANDOM%10]; done
Calling Service B: Hello from service B.
Hello from service A.
Hello from service C.
Calling Service B: Hello from service B.
Hello from service A.
Hello from service C.
Calling Service B: Hello from service B.
Hello from service A.
Hello from service C.
```

访问9411， 由于c注入了10%的1s延迟，而b注入了10%的abort, 就可以看到异常和持续时间不同了

![image-20210921083246205](http://myapp.img.mykernel.cn/image-20210921083246205.png)

> 控制台输出了日志`service_c_envoy_1  | [2021-09-21T00:33:25.895Z] "GET / HTTP/1.1" 200 DI 0 22 1000 0 "-" "Go-http-client/1.1" "c35c0d2d-da3d-9e21-b33f-0dcd25c5ebd2" "service_a_envoy:8791" "192.168.160.2:8083"`  DI表示delay注入
>
> `service_b_envoy_1  | [2021-09-21T00:33:55.178Z] "GET / HTTP/1.1" 503 FI 0 18 0 - "-" "Go-http-client/1.1" "bd8a3861-66b7-9b62-9314-eaa664f755dd" "service_a_envoy:8788" "-"` FI表示fault注入，会返回 503



## jaeger

shared_span_content必须false, jaeger独有

```bash
ls -l servicemesh_in_practise-master/monitoring-and-tracing/jaeger-tracing
-rw-r--r--. 1 root root  1837 Apr 23  2020 docker-compose.yml
-rw-r--r--. 1 root root 31682 Apr 23  2020 envoy_tracing.png
drwxr-xr-x. 2 root root    31 Apr 23  2020 front_envoy
-rw-r--r--. 1 root root   276 Apr 23  2020 Readme.md
drwxr-xr-x. 2 root root    64 Apr 23  2020 service_a
drwxr-xr-x. 2 root root    64 Apr 23  2020 service_b
drwxr-xr-x. 2 root root    64 Apr 23  2020 service_c

```

> `docker-compose.yml` jaeger all-in-one, 使用环境变量兼容zipkin, 9411
>
> `envoy.yml`
>
> ```yaml
> tracing:
>   http:
>     name: envoy.zipkin
>     typed_config:
>       "@type": type.googleapis.com/envoy.config.trace.v2.ZipkinConfig
>       collector_cluster: jaeger
>       collector_endpoint: "/api/v1/spans"
>       shared_span_context: false
> ```
>
> > shared_span_context: false

```bash
docker-compose up
```

```bash
[root@ip-10-0-0-91 jaeger-tracing]# docker exec jaeger-tracing_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:B0:07  
          inet addr:192.168.176.7  Bcast:192.168.191.255  Mask:255.255.240.0
```

```bash
[root@ip-10-0-0-91 jaeger-tracing]# while true; do curl -s 192.168.176.7; sleep 0.$[$RANDOM%10]; done
Calling Service B: Hello from service B.
Hello from service A.
Hello from service C.
Calling Service B: Hello from service B.
```

现在访问16686

 