---
title: Envoy集群管理
date: 2021-09-03 10:10:49
tags:
- istio
---

# 前言

传统nginx/haproxy，或web框架实现的功能主要有: 

- 流量代理、流量分配
  - 流量代理：接入流量，可以以指定特征识别用户。(canary)
  - 流量分配：将流量传给真实服务器，可以以指定权重、算法来传递流量。(rr, session sticky)
- 上游主机分组管理：固定分组
- 健康状态检测

> envoy均支持以上功能，而且有特色功能：
>
> 1. **1个集群内多个主机，主机分组管理**。
>    - 全局负载 
>      - 以分组指定优先级，请求会优先路由给（健康的、高优先的）分组。分配时，仅部分流量 。
>      - 以分组指定权重。
>      - 优先级和权重混合使用。
>    - 分布式负载均衡
>      - 区域感知路由
>      - wrr
>      - wlc
>      - consistence hash
>
> 2. **基于元数据的选择器**，动态拿出主机，归并成组，也称""集群的子集"
>
> > ```bash
> > group1:
> >   stable
> > group2:
> >   alpha
> >   
> > weight: group1:group2 = 90:10
> > ```
> >
> > 以上表示将stable元数据匹配的主机归并成`group1`, 并有90的权重，90%的用户到达`group1`.  同理10%到达alpha所在分组。
>
> 3. **被动健康状态检测（异常值检测）**：如果后端主机连续响应: `5xx`代码，或者后端响应during大于`300ms`.  即便健康状态检测OK，envoy同样会认为后端主机故障，从而将其从可用主机列表中弹出。 envoy假设后端的问题：
>    - 主机过载
>    - 临时故障
>
> 4. **主机端点发现**：`strict_dns`/`api_config_resource`/`ads`
> 5. **断路器**：对应后端主机超出承载能力、异常值检测生效时（监控多个指标达到），前端主机就不会向后端派发流量了。（如果就卡在等待后端主机的连接上，可能会造成大面积的问题）



<!--more-->
# envoy遇到故障，其处理机制

- 超时
- 有限次数的重试，支持可变的重试延迟
- 主动健康检查与异常探测
- 连接池
- 断路器

> 均可动态配置
>
> 结合流量管理机制，用户可为每个服务的不同版本定制所需的故障恢复机制。

# cluster manager, 集群管理器

envoy的集群管理由cm管理，生成集群：静态、动态： `dns, api_config_resource , ads`, 动态的除dns之外有预热功能，cds->eds, lds->rds->vhds有序加载。

`clusters.[]`

```yaml
transport_socket_matches: []
`name`: '...'
alt_stat_name: '...'
type: '...'
cluster_type: '{...}'
eds_cluster_config: '{...}'
connect_timeout: '{...}'
per_connection_buffer_limit_bytes: '{...}'
lb_policy: '...'
load_assignment: '{...}'
health_checks: []
max_requests_per_connection: '{...}'
circuit_breakers: '{...}'
upstream_http_protocol_options: '{...}'
common_http_protocol_options: '{...}'
http_protocol_options: '{...}'
http2_protocol_options: '{...}'
typed_extension_protocol_options: '{...}'
dns_refresh_rate: '{...}'
dns_failure_refresh_rate: '{...}'
respect_dns_ttl: '...'
dns_lookup_family: '...'
dns_resolvers: []
use_tcp_for_dns_lookups: '...'
dns_resolution_config: '{...}'
wait_for_warm_on_init: '{...}'
outlier_detection: '{...}'
cleanup_interval: '{...}'
upstream_bind_config: '{...}'
lb_subset_config: '{...}'
ring_hash_lb_config: '{...}'
maglev_lb_config: '{...}'
original_dst_lb_config: '{...}'
least_request_lb_config: '{...}'
common_lb_config: '{...}'
transport_socket: '{...}'
metadata: '{...}'
protocol_selection: '...'
upstream_connection_options: '{...}'
close_connections_on_host_health_failure: '...'
ignore_health_on_host_removal: '...'
filters: []
load_balancing_policy: '{...}'
track_timeout_budgets: '...'
upstream_config: '{...}'
track_cluster_stats: '{...}'
preconnect_policy: '{...}'
connection_pool_per_downstream_connection: '...'
```

> 以上为yaml格式
>
> name: 集群名
>
> type: 集群类型, eds/static/logical/strict_dns
>
> eds_cluster_config: 通过eds发现
>
> lb_policy: 本集群内所有endpoints负载均衡算法，有些算法需要额外的同级选项来配置。`lb_subset_config`, `ring_hash_lb_config`, `least_request_lb_config` , `common_lb_config` 轮循、磁悬浮没有专用说明。
>
> hosts 静态配置集群端点，已经废弃
>
> load_assignment：现在静态配置集群端点的方法（type=strict_dns, logical, static)
>
> `circuit_breakers` 定义熔断器
>
> `tls` tls配置
>
> `connection_pool_per_downstream_connection` 连接池
>
> `outlier_detection` 被动健康状态检测
>
> `health_checks` 主动健康状态检测

`load_assignment` 静态配置

```yaml
    load_assignment:
      cluster_name: webcluster1
      policy:
        overprovisioning_factor: 140
      endpoints:
      - locality:
          region: cn-north-1
          zone
          sub_zone
        lb_endpoints:
        - endpoint:
            health_check_config:
            address:
              socket_address:
                address: colorful
                port_value: 80
          load_balancing_weight: # 结合common_lb_config.locality_weighted_lb_config 启用位置加权负载均衡
          metadata:
          health_status:
          
        load_balancing_weight: 
        priority: 0
      policy: 
        drop_overloads:  # 过载保护， 丢弃过量流量 
        overprovisioning_factor: # 超配因子，默认140, 即1.4
        endpoint_stale_after: # 
    common_lb_config:
      heath_panic_threshold: # 恐慌阈值
```



## type: cluster服务发现endpoints

- static
- strict dns
- logical dns
- original destination
- endpoint disvoery service(eds)
- custom cluster

> 这些集群类型的定义在[静态cluster + 基于filesystem动态EDS](http://blog.mykernel.cn/2021/03/18/Envoy%E5%9F%BA%E4%BA%8E%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E7%9A%84EDS%E5%92%8CCDS%E8%AE%A2%E9%98%85%E7%A4%BA%E4%BE%8B/#%E9%9D%99%E6%80%81cluster-%E5%9F%BA%E4%BA%8Efilesystem%E5%8A%A8%E6%80%81EDS)中，`clusters.type`详细说明了不同场景

集群发现端点后，流量是否能达到端点？

| discovery status | health check ok | health check failed |
| ---------------- | --------------- | ------------------- |
| discovered       | route           | don't route         |
| absent           | route           | don't route/delete  |

> 发现后，健康检测ok, 就可以路由。未发现(发现信道有问题，但是健康检查是另一个信道)，但是健康检测ok, 也可以路由。



## upstreams 健康状态检测

主动结合被动，提高服务韧性非常有帮助的工具。

### 主动健康检测

 `health_checks` 来定义主动健康检测。

- http
- tcp
- redis ping

```yaml
  clusters:
  - name: webcluster1
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      cluster_name: webcluster1
      endpoints:
      - lb_endpoints:
        - endpoint:
            health_check_config:
              port_value: 
            address:
              socket_address:
                address: myservice
                port_value: 80
    health_checks:
    - timeout: 5s
      interval: 10s
      unhealthy_threshold: 2
      healthy_threshold: 2
      tcp_health_check: {}

```

>  health_checks 可以有多个，一个集群可以有多个健康检测机制。 默认对集群中每个endpoint下的`address.port_value`检测，否则将使用endpoint下的`health_check_config.port_value` 发起检测。
>
> interval 间隔多久检测一次
>
> timeout 此次检测的超时，并设置失败
>
> initial_jitter:   后端多个endpoints，一起检测时，初始检测怎么分散
>
> interval_jitter: 后端多个endpoints，一起检测时，间隔检测怎么分散
>
> unhealthy_threshold: 将主机标记为不健康状态的检测阈值，即至少多少次不健康的检测后才将其标记为不可用；
>
> healthy_threshold: 将主机标记为健康状态的检测阈值，但初始检测成功一次即视主机为健康； 
>
>   定义检测类型: 
>
> - http_health_check: 后端输出http协议 
>     -  **path** 健康状态检测路径
>     - **expected_status** 默认期望对方返回200
>     - host 不定义时，默认当前集群名
>     - request_headers_to_add 添加标头
>     - request_headers_to_remove
>     - use_http2
> - tcp_health_check: 后端输出tcp协议 . {} 意味着 仅通过连接状态检测结果 。      非空负载的tcp检测可以使用send指定请求负载和receive指定响应报文中模糊匹配的结果。
> - grpc_health_check: 后端输出grpc协议 
> - custom_health_check: 自定义检测，调用外置脚本完成。
>
>  reuse_connection: 每一次检测时，会重用之前检测的连接，默认 true. 会提升检测的性能。
>
>  
>
> - no_traffic_interval:  刚开始集群未启用时，没有流量来，检测没有必要太频繁。此选项定义没有流量路由到当前集群时，健康状态检测时间间隔。一旦有流量到来时，才使用上面的Interval作为健康检测检测。
> - unhealthy_interval:  标记为不健康的endpoint的时间间隔。转为正常时，恢复间隔。
> -  unhealthy_edge_interval: 端点刚被标记为unhealthy时，使用此定义，随后转为unhealthy_interval.
> - healthy_edge_interval: 与unhealthy_edge_interval同理，刚健康时的间隔。

#### 配置

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# tree cluster-manager/health-check/
cluster-manager/health-check/
├── docker-compose.yaml
├── front-envoy-tcp.yaml
├── front-envoy.yaml
└── README.md
```

> `docker-compose.yaml` 其中的3个服务均是myservice. 只是每个服务都有不同的端点， 均使用版本 `v0.5` , 镜像内部存在/healthz的url，会响应200。如果删除/var/www/html/healthz时，就返回404，就模拟了健康检测失败。
>
> `envoy` 提供代理

#### 启动

```bash
[root@ip-10-0-0-91 health-check]# docker-compose up 
loading 1 cluster(s)
loading 1 listener(s)
service_green_1  | 127.0.0.1 - - [04/Sep/2021 12:49:42] "GET /healthz HTTP/1.1" 200 -
service_blue_1   | 127.0.0.1 - - [04/Sep/2021 12:49:42] "GET /healthz HTTP/1.1" 200 -
service_red_1    | 127.0.0.1 - - [04/Sep/2021 12:49:42] "GET /healthz HTTP/1.1" 200 -
```

获取envoy容器的集群信息, 并可以看到envoy已经通过strict_dns获取到3个端点，并且每个端点输出一个指标为healthy

```bash
docker exec -it  health-check_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:14:00:04  
          inet addr:172.20.0.4  Bcast:172.20.255.255  Mask:255.255.0.
```

```bash
[root@ip-10-0-0-91 ~]# curl 172.20.0.4:9901/clusters
webcluster1::172.20.0.3:80::health_flags::healthy
webcluster1::172.20.0.5:80::health_flags::healthy
webcluster1::172.20.0.2:80::health_flags::healthy
```

> 显示正常的原因，因为从控制台上也显示了此3个容器均有健康检测日志。均响应200

现在可以测试主机, 发现3个主机均正常。现在有意的人为将某个容器的headlthz接口对应的文件删除时，再次检测，可以发现此端点已经被自动踢出。

```bash
[root@ip-10-0-0-91 ~]#  while true; do curl -s 172.20.0.4/service/color  | grep -i '^hello'; sleep .5; done
Hello from red (hostname: 1e3c33bdeace resolvedhostname:172.20.0.3)
Hello from green (hostname: 127320ed26eb resolvedhostname:172.20.0.5)
Hello from blue (hostname: 65f168b4bd48 resolvedhostname:172.20.0.2)
```

```bash
[root@ip-10-0-0-91 docker exec health-check_service_red_1 rm -f /var/www/html/health.html
```

```bash
[root@ip-10-0-0-91 ~]# curl -s 172.20.0.4:9901/clusters  | grep health
webcluster1::172.20.0.3:80::health_flags::/failed_active_hc
webcluster1::172.20.0.5:80::health_flags::healthy
webcluster1::172.20.0.2:80::health_flags::healthy

[root@ip-10-0-0-91 ~]#  while true; do curl -s 172.20.0.4/service/color  | grep -i '^hello'; sleep .5; done
Hello from green (hostname: 127320ed26eb resolvedhostname:172.20.0.5)
Hello from green (hostname: 127320ed26eb resolvedhostname:172.20.0.5)
Hello from blue (hostname: 65f168b4bd48 resolvedhostname:172.20.0.2)
Hello from green (hostname: 127320ed26eb resolvedhostname:172.20.0.5)
Hello from blue (hostname: 65f168b4bd48 resolvedhostname:172.20.0.2)
Hello from green (hostname: 127320ed26eb resolvedhostname:172.20.0.5)
Hello from blue (hostname: 65f168b4bd48 resolvedhostname:172.20.0.2)

```

> RED已经不正常了, 所以请求时，流量也不会到达这个endpoints.

同样的，创建回来，将恢复。

以上为http检测，现在使用tcp时，将默认的yaml和tcp对调，启动即可。效果同上。

### 被动健康检测

只是一种处理异常的机制，不能取代主动健康检测。对繁忙的集群非常有帮助。

- 异常驱逐的机制
  1. 主机一定异常
  2.  驱逐的数量/总主机数 所占百分比 低于max_ejection_percent，立即驱逐主机。
  3.  驱逐状态达到`base_ejection_time` * `异常次数`所求得的时间， 自动恢复服务。

- 被动: `outlier_detection` envoy不向后端发报文，通过后端响应状态码状态检测，也叫异常值检测。
  - http/tcp/redis
  - 连续5xx、连续网关故障（5xx子集: 502, 503, 504）
  - 无法连接上游主机
  - 成功率:  有2个指标：1）端点数量不低于N，2）主机检测次数（最近至少请求多少次请求）， 某主机最近检测磁成功率比所有主机成功率低，就认为异常。离平均值1个标准差以内，就正常。1个标准差以外，可以认为不正常。

```yaml
  clusters:
  - name: webcluster1
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      cluster_name: webcluster1
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: myservice
                port_value: 80
    health_checks:
    - timeout: 5s
      interval: 10s
      unhealthy_threshold: 2
      healthy_threshold: 2
      http_health_check:
        path: /healthz
        expected_status: 200
    outlier_detection:
      consecutive_5xx: 3
      base_ejection_time: 30s
```

> consecutive_5xx: # 连续几次5xx错误，认为错误而弹出。默认5
>
> interval: 每多长时间探测一次，默认10000ms或10s
>
> base_ejection_time: # 主机因异常被弹出/驱逐时，默认弹出时间30s, 多次时，越来越长的时间 此时间*次数。
>
> max_ejection_percent: #  主机因异常，弹出主机数/总主机数 所占百分比。此选项定义百分比的最大值，默认10%, 不过无论如何，至少要弹出1个主机。
>
> > 5个端点，弹出一个，就弹出20%了，由于默认10%， 所以至少要弹一个主机。如果没有此选项限制，出现问题就弹出，那么从5个端点因为异常，全部弹出，最终将1个都不剩下了。
>
> enforcing_consecutive_5xx: # 连续5xx的错误，主机被弹出的几率。默认100%.
>
> enforcing_success_rate: 因为成功率检测异常时，主机被弹出的几率。默认100。
>
> > 降低`enforcing_success_rate` 允许缓慢弹出，调高此值时，允许出现故障时，立即弹出。
>
> success_rate_minimum_hosts # 成功率检测第1个指标，最少主机数，默认5
>
> success_rate_request_volume: # 对一次成功率检测，其中最少收集的总请求数。默认100， 即达到100个请求所求的成功率才有效。
>
> success_rate_stdev_factor: 到达1个标准差之外就弹出，此处定义1000，1300则表示到达1.3标准差之外就弹出。
>
> consecutive_gateway_failure: # 连续几次网关错误，认为错误而弹出。默认5
>
> enforcing_consecutive_gateway_failure：连续网关错误，主机被弹出的几率。默认0。
>
> > 可以调高。
>
> split_external_local_origin_errors: 是否区分本地原因故障和外部故障，默认false. 仅当此值为true时，以下3个生效。
>
> - consecutive_local_origin_failure: 连续几次本地故障，认为错误而弹出。默认5
> - enforcing_consecutive_local_origin_failure：连续本地故障，主机被弹出的几率。默认100。
> - enforcing_consecutive_local_origin_success_rate: 本地故障探测成功率异常时，主机被弹出的几率。默认100。

使用异常检测

- 连续5xx 3次，弹出30s*次数

  ```yaml
  consecutive_5xx: "3"
  base_ejection_time: "30s"
  ```

- 连续网关故障3次，弹出30s*次数，弹出概率10%

  ```yaml
  consecutive_gateway_failure: "3"
  base_ejection_time: "30s"
  enforcing_consecutive_gateway_failure: "10"
  ```

- 高流量稳定的服务，使用统计信息（成功率）弹出异常主机：集群内主机至少10个，每个主机10s内对不少于500个请求(少于500次的主机不评估，多的主机才评估)，判断成功率是否低于平均率 1个标准差。异常就弹出30s*次数

  ```yaml
  success_rate_minimum_hosts: "10"
  success_rate_request_volume: "500"
  success_rate_stdev_factor: "1000"
  base_ejection_time: "30s"
  interval: "10s"
  ```

#### 配置

```bash
[root@ip-10-0-0-91 cluster-manager]# tree outlier-detection/
outlier-detection/
├── docker-compose.yaml
└── front-envoy.yaml
```

由于有示例，但是演示不出来效果，就不进行测试了。



## 负载均衡策略

### 算法概览

| 均衡           | 作用域              | 算法                           | 依赖             |
| -------------- | ------------------- | ------------------------------ | ---------------- |
| 分布式负载均衡 | 同cluster中多个端点 | wrr, wlc, ring-hash, 磁县浮    | 主动健康检测正常 |
| 全局负载均衡   | 跨集群的            | 位置优先、位置权重、均衡器子集 |                  |

整个集群内，定义多个端点

```yaml
  clusters:
  - name: webcluster1
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    common_lb_config:
      locality_weighted_lb_config: {}
    load_assignment:
      cluster_name: webcluster1
      policy:
        overprovisioning_factor: 140
      endpoints:
      - locality:
          region: cn-north-1
        priority: 0
        load_balancing_weight: 10
        lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: colored
                port_value: 80
      - locality:
          region: cn-north-2
        priority: 0
        load_balancing_weight: 20
        lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: myservice
                port_value: 80
```

> endpoints中多个端点， 每个端点组的优先级`priority`, 位置`locality` ，是一个端点组。
>
> 一个端点组可以有多个endpoint. 由 `lb_endpoints`定义
>
> 定义负载均衡策略: `clusters.load_assignment.policy`
>
> 定义算法 `clusters.lb_policy`
>
> 某些算法，需要专用选项：`lb_subset_config`, `ring_hash_lb_config`, `least_request_lb_config` , `common_lb_config` 轮循、磁悬浮没有专用说明。
>
> `common_lb_config.health_panic_threshold` panic阈值，默认50%
>
> `common_lb_config.zone_aware_lb_config` 区域感知路由相关配置
>
> `common_lb_config.locality_weighted_lb_config` 位置权重负载均衡算法 相关配置
>
> `common_lb_config.ignore_new_hosts_until_first_hc` 是否在新加入主机经历第一次健康状态检查之前不予考虑进负载均衡。

### 分布式算法比较

均对`lb_endpoints`某个组内多个endpoint生效。

| 算法                                  | 描述                                                         | 适用场景                                                     | 为什么出现                                                   | 原理 |
| ------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ---- |
| 加权轮询(weighted round robin)        | 更多资源的主机，按权重将划分成多份。，然后再次进行轮询       | 非保持连接的http                                             | 轮询：权重为1, 按次序调度。                                  |      |
| 加权最少请求(weighted least request)  | 按后端负载情况分配，负载最低的，优先被调度                   | 适用于保持连接的mysql/redis. 如果此方法需要检测后端连接需要消耗资源。 | 集群轮询调度时，可能过一段时间，某些后端连接占用多，有些占用少，还是轮询的话。对请求均衡不好。 |      |
| 环哈希                                | 一致性哈希，基于请求属性，将请求与后端始终建立关联关系。     | 同一个ip地址/cookie/header/param /url 到 同一个后端。 **url： 后端为缓存。ip/cookie: session sticky** <br/> 由于整个环上的主机是扩大了的，所以也需要消耗计算能力。envoy算法，可以自定义环大小，定义虚拟节点数量。 | 哈希算法 ：分布主机：每个主机的ip地址的hash值 % 2^32, 对应环上的点。调度：按我们指定的属性取hash值%2^32，按顺时针找与此值相邻的节点。  <br/> 节点数少时，会出现某个节点上接受大量流量。 <br/>环哈希算法/一致性算法，可以将少量节点，扩大虚拟成数倍，让主机均衡到整个环上。同时节点的减少不会影响到整个环。 |      |
| 磁悬浮                                | 环哈希的特殊，固定大小65537                                  | 可以确保主机虚拟出来的数量一定要填充满整个环。所以效果比环哈希好。 |                                                              |      |
| 随机                                  | 随机选                                                       | 不配置健康检测时，比轮循更好。                               |                                                              |      |
| 原始目标集群负载机制: original_dst_lb | 仅适用于 original_dst类型的集群，[此处有说明此集群](http://blog.mykernel.cn/2021/03/18/Envoy%E5%9F%BA%E4%BA%8E%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E7%9A%84EDS%E5%92%8CCDS%E8%AE%A2%E9%98%85%E7%A4%BA%E4%BE%8B/#%E9%9D%99%E6%80%81cluster-%E5%9F%BA%E4%BA%8Efilesystem%E5%8A%A8%E6%80%81EDS) |                                                              |                                                              |      |

#### 加权轮循

`lb_endpoints.endpoint.load_balancing_weight`  定义之后，就会把cluster.type的roundrobin调整为加权轮循。

示例

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# tree cluster-manager/weighted-rr/
cluster-manager/weighted-rr/
├── docker-compose.yaml
├── front-envoy.yaml
└── send-request.sh

```

> compose: enovy + 3个容器
>
> `front-envoy.yaml` round_robin, load_assignment, 静态定义了3个端点，并分配不同的权重。

启动

```yaml
  clusters:
  - name: webcluster1
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      cluster_name: webcluster1
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: service_red
                port_value: 80
          load_balancing_weight: 1
        - endpoint:
            address:
              socket_address:
                address: service_blue
                port_value: 80
          load_balancing_weight: 3
        - endpoint:
            address:
              socket_address:
                address: service_green
                port_value: 80
          load_balancing_weight: 5
```



```bash
[root@ip-10-0-0-91 weighted-rr]# docker-compose up
 loading 1 cluster(s)
loading 1 listener(s)
```

集群信息

```bash
[root@ip-10-0-0-91 ~]# docker exec weighted-rr_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:15:00:03  
          inet addr:172.21.0.3  Bcast:172.21.255.255  Mask:255.255.0.0
```

```bash
[root@ip-10-0-0-91 ~]# curl 172.21.0.3:9901/clusters
```

使用脚本测试WRR

```bash
[root@ip-10-0-0-91 weighted-rr]# cat send-request.sh 
#!/bin/bash
declare -i red=0
declare -i blue=0
declare -i green=0

#interval="0.1"
counts=300

for ((i=1; i<=${counts}; i++)); do
        content="$(curl -s http://$1/service/colors)" 
        if echo "$content" | grep -i "red" &> /dev/null; then
                # $1 is the host address of the front-envoy.
                red=$[$red+1]
        elif echo "$content" | grep -i "blue" &> /dev/null; then
                blue=$[$blue+1]
        else
                green=$[$green+1]
        fi
#       sleep $interval
done

echo "Red:Blue:Green = $red:$blue:$green"
```

```bash
[root@ip-10-0-0-91 weighted-rr]# ./send-request.sh 172.21.0.2
Red:Blue:Green = 32:100:168
[root@ip-10-0-0-91 weighted-rr]# ./send-request.sh 172.21.0.2
Red:Blue:Green = 34:99:167
```

> 几乎达到 了 `1:3:5`

#### 加权最少请求

适用于长连接场景

所有主机权重相同时，先随机选N(默认2）个连接数最少的主机。然后再从中轮询。

所有主机权重并不完全相同，传统的wlc.

```   yaml
    lb_policy: LEAST_REQUEST
    least_request_lb_config:
      choice_count: 2       # 默认是2个主机
```

准备

```bash
[root@ip-10-0-0-91 least-requests]# pwd
/root/servicemesh_in_practise-master/cluster-manager/least-requests
[root@ip-10-0-0-91 least-requests]# ls
docker-compose.yaml  front-envoy.yaml  send-request.sh
```

> envoy.yaml: 相同权重时，先选择2个负载低的，再调度。不同权重时，按wlc调度。

先调整所有endpoints内每个分组，权重为1

```diff
  clusters:
  - name: webcluster1
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: LEAST_REQUEST
    least_request_lb_config:
      choice_count: 2
    http2_protocol_options: {}
    load_assignment:
      cluster_name: webcluster1
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: service_red
                port_value: 80
+          load_balancing_weight: 1
        - endpoint:
            address:
              socket_address:
                address: service_blue
                port_value: 80
+          load_balancing_weight: 1
        - endpoint:
            address:
              socket_address:
                address: service_green
                port_value: 80
+          load_balancing_weight: 1

```

启动, 使用脚本测试。

```bash
[root@ip-10-0-0-91 weighted-rr]# docker exec least-requests_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:16:00:02  
          inet addr:172.22.0.2  Bcast:172.22.255.255  Mask:255.255.0.0

```

```bash
#!/bin/bash
declare -i red=0
declare -i blue=0
declare -i green=0

#interval="0.1"
counts=300

for ((i=1; i<=${counts}; i++)); do
        if curl -s http://$1/service/colors | grep "red" &> /dev/null; then
                # $1 is the host address of the front-envoy.
                red=$[$red+1]
        elif curl -s http://$1/service/colors | grep "blue" &> /dev/null; then
                blue=$[$blue+1]
        else
                green=$[$green+1]
        fi
#       sleep $interval
done

echo "Red:Blue:Green = $red:$blue:$green"

```

```bash
[root@ip-10-0-0-91 least-requests]# ./send-request.sh 172.22.0.2
Red:Blue:Green = 109:65:126
[root@ip-10-0-0-91 least-requests]# ./send-request.sh 172.22.0.2
Red:Blue:Green = 109:74:117

```

> 可以发现，有2个端点选达到几乎相等，才是另一个接受负载。

按以上方法现在调整为权重 `1:3:5`, 现在再次测试, 接近 `1:3:5`

```bash
[root@ip-10-0-0-91 least-requests]# ./send-request.sh 172.22.0.5
Red:Blue:Green = 37:101:162
```

#### 环哈希算法

把固定请求属性与后端特点端点建立特定的映射关系，环哈希可以自己指定大小`

```yaml
  clusters:
  - name: webcluster1
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: RING_HASH
    ring_hash_lb_config:
      maximum_ring_size: 1048576 # 默认8M. 越大越消耗资源
      minimum_ring_size: 512     # 最小1024， 最大8M. 越大越接近权重比。
      hash_function: # 建议使用默认算法：XX_HASH  ，另外支持MURMUR_HASH_2。
    http2_protocol_options: {}
    load_assignment:
      cluster_name: webcluster1
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: myservice
                port_value: 80
```

除了配置hash, 还需要配置路由时，`routes.route.hash_policy` 可以3种方式指定哪些属性来hash

```yaml
static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 80
    name: listener_http
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          codec_type: auto
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: webcluster1
                  hash_policy:
                  - header:
                      header_name: User-Agent
          http_filters:
          - name: envoy.router

```

> ```yaml
> routes:
> - route:
>     hash_policy:
>     - header:
>         header_name: #指定首部值来hash计算
>     -  cookie: #指定cookie来hash计算
>         name:
>         ttl:
>         path:
>     -  connection_properties: # 基于请求报文源ip做hash计算
>         source_ip: 
>     - terminal: true/false  #如果定义了多个策略来生成哈希值，此选项表示从上到下匹配第1个策略，就终止。
> ```
>
> > hash_policy是一个列表，每个列表项只能3选1.同样的header可以出现多次。第1次基于user-agent, 第2次基于host. 
> >
> > 如果指定了多个。默认每个策略都会生效。如果使用terminal时，就最上面的定义才生效。

配置

```bash
[root@ip-10-0-0-91 ring-hash]# tree 
.
├── docker-compose.yaml
└── front-envoy.yaml

```

```bash 
docker-compose up
```

```bash
[root@ip-10-0-0-91 least-requests]# docker exec ring-hash_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:17:00:05  
          inet addr:172.23.0.3  Bcast:172.23.255.255  Mask:255.255.0.0
```

先仅保留header, user-agent, 验证不同浏览器绑定到不同的后端

```bash
[root@ip-10-0-0-91 ring-hash]#  curl -s -H 'User-Agent: Googlebot/2.1 (+http://www.google.com/bot.html)'  172.23.0.3/service/color
<body bgcolor="blue"><span style="color:white;font-size:4em;">
Hello from blue (hostname: 40c4fde5f4d5 resolvedhostname:172.23.0.5)
</span></body>

[root@ip-10-0-0-91 ring-hash]# curl -s -H 'User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:47.0) Gecko/20100101 Firefox/47.0'  172.23.0.3/service/color
<body bgcolor="green"><span style="color:white;font-size:4em;">
Hello from green (hostname: 9e76a8f95304 resolvedhostname:172.23.0.2)
</span></body>

```

查看虚拟节点, 最小512，3个端点，分别171，环大小513

```bash
[root@ip-10-0-0-91 ring-hash]# curl 172.23.0.3:9901/stats?filter=hash
cluster.webcluster1.ring_hash_lb.max_hashes_per_host: 171
cluster.webcluster1.ring_hash_lb.min_hashes_per_host: 171
cluster.webcluster1.ring_hash_lb.size: 513
```

#### 磁悬浮

需要监视 min_entries_per_host, max_entries_per_host, 确保主机没有出现异常。

#### 区域感知路由

不需要太多的配置， 服务间通信，envoy更倾向将流量调度到与客户端同区域的端点。

先决条件：

- 非恐慌模式
- 始发集群与上游集群同区域
- 上游集群能承载所有请求流量
- 仅支持0优先级，也与位置加权与斥
- 同区域中，始发健康率>上游健康率，其余请求路由到其他区域。
- 同区域中，始发健康率<上游健康率，所有请求路由到本地区域，并可以承载一部分其他区域的路由。

```yaml
common_lb_config:
  zone_aware_lb_config:
    "routing_enabled": {}     # 多大规模的流量上启用区域感知路由调度。 默认100%
    "min_cluster_size": {}    # 配置区域感知路由最小集群大小。默认值为6，可用值64位整数
```

#### 原始目标负载均衡

专用于原始目标集群

没有经过iptables多次转发，用到几率不高

原始目标集群与其他负载均衡策略不兼容

### 全局负载均衡算法

#### 算法概览

Locality 位置:

- region 不同数据中心： cn-north-1
- zone    同数据中心中的不同区域: zone1, zone2, zone3, ...
- subzone sz1, sz2, ...

> 基于位置时，不同zone可以有不同的权重。同一个zone内多个主机时，就使用3.3.2 分布式负载均衡

Priority 优先：按优先级时会忽略位置

位置加权优先级负载均衡：结合locality+priority

负载均衡子集：每个节点施加元数据，主机基于元数据标签归组，流量基于元数据标签来分配。金丝雀发布、蓝绿部署、AB测试。

- 节点优先级及优先级调度

归组：LocalityLBEndpoints, 相同locality, load_balancing_weight 可选, priority

envoy 仅会将所有流量会发给最高优先级(0为最高)组。

除非此组25%端点健康检测失败，处理方式以下之一：

1. 25%流量会选择次优先级组，次优先级失败部分时，又按以上方式处理。

2. 避免一点点检测失败，就转移流量 ，可以为优先组分配超配因子。默认超配因子1400，1.4 ，假如此组对应端点70%都健康(70%*1.4=98%)，则A组需要承载98%流量，2%流量转移。80%都健康时(112%), 所有流量将在此组。  故障比率<100-100/1.4(28.6%), 所有流量当前组处理。大于此值，流量将会转移。或者健康小于72%就会转移。大于72%就不会转移。 所有健康评分之和(`30%*1.4+20%*1.4+10%*1.4=60%*1.4`)小于100，envoy就认为没有足够的健康端点, 就会把健康评分按权重使用，`3:2:1`

3. 组内端点，降级端点不接受流量。正常端点不正常比率大于28%时，降级端点将会 接受部分流量 。

- panic(恐慌)阈值

一旦健康率小于指定值(默认50%），所有流量将到达次优先级所有端点。

#### 优先级示例

```bash
[root@ip-10-0-0-91 cluster-manager]# tree priority-levels/
priority-levels/
├── docker-compose.yaml
├── front-envoy.yaml
└── README.md
```

> compose定义了5个容器(service),  red， green, blue 归组为colorful
>
> envoy: strict_dns发现，第1组为colorful, 第2组为service（black)

启动

```bash
  clusters:
  - name: webcluster1
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      cluster_name: webcluster1
      policy:
        overprovisioning_factor: 140
      endpoints:
      - locality:
          region: cn-north-1
        priority: 0
        lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: colorful
                port_value: 80
      - locality:
          region: cn-north-2
        priority: 1
        lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: myservice
                port_value: 80
    health_checks:
    - timeout: 5s
      interval: 10s
      unhealthy_threshold: 2
      healthy_threshold: 1
      http_health_check:
        path: /healthz
        expected_status: 200

```

```bash
docker-compose up
```

模拟将高优先级流量转移到低优先级：由于高优先级是3个节点，低于72%就会转移，失败一个就是3/2=0.6666， 即健康为66%, 已经低于72%, 所以将会转移流量， envoy通过超配因子，算出来的健康是(66*1.4=92.4%), 所以还有8%的流量不正常，所以将转移7.6%的流量到低优先级。

现在请求clusters,确保所有均健康

```bash
[root@ip-10-0-0-91 cluster-manager]# docker exec priority-levels_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:18:00:07  
          inet addr:172.24.0.7  Bcast:172.24.255.255  Mask:255.255.0.0
```

```bash
[root@ip-10-0-0-91 cluster-manager]# curl -s 172.24.0.7:9901/clusters | grep flags
webcluster1::172.24.0.5:80::health_flags::healthy
webcluster1::172.24.0.4:80::health_flags::healthy
webcluster1::172.24.0.2:80::health_flags::healthy
webcluster1::172.24.0.6:80::health_flags::healthy
webcluster1::172.24.0.3:80::health_flags::healthy
```

> 所有正常

```bash
 while true; do curl -s 172.24.0.7/service/color | grep -i '^hello'; sleep 1; done 
```

> 均为有色端点

现在将有色1个端点，清理health.html, 当此端点不正常后， 上面的输出将有无色端点被调度流量 

```bash
docker exec  priority-levels_service_green_1 rm /var/www/html/health.html
```

```bash
[root@ip-10-0-0-91 ~]#  curl -s 172.24.0.7:9901/clusters | grep flags
webcluster1::172.24.0.5:80::health_flags::healthy
webcluster1::172.24.0.4:80::health_flags::healthy
webcluster1::172.24.0.2:80::health_flags::/failed_active_hc
webcluster1::172.24.0.6:80::health_flags::healthy
webcluster1::172.24.0.3:80::health_flags::healthy
```

> 已经不正常

置回健康状态后，此前转移到第2个组的流量 将消失。



如果破坏2个有色点，健康只是1/3=33%, 另一个组健康评分还是高，流量还是可以完全承载。只是第1个组承载了33%*1.4的流量，第2个组承载了`(1-33%*1.4)`的流量

```bash
[root@ip-10-0-0-91 ~]# docker exec  priority-levels_service_green_1 rm /var/www/html/health.html
[root@ip-10-0-0-91 ~]# docker exec  priority-levels_service_red_1 rm /var/www/html/health.html
```

> 所以blue占大量的流量 

现在破坏第2个组的black, 所以第2个组的健康50%， 小于72%，将不能承载所有流量。现在总体健康比`( 33+50)*1.4`总体流量还是够用。现在把第2组全部搞坏，总体的健康评分只是`33%*1.4=46%`低于 50%, 触发恐慌阈值，所以流量将在第1个组随机调度。

如果总健康比大于50%小于100%, 就按健康比率当作权重调度。当第2个组有6个节点时，仅保留1个节点`(33+1/6*100)*1.4=68.6%`，就会出现这个效果。

 ```bash
 [root@ip-10-0-0-91 ~]#  curl -s 172.24.0.7:9901/clusters | grep flags
 webcluster1::172.24.0.5:80::health_flags::healthy
 webcluster1::172.24.0.4:80::health_flags::/failed_active_hc
 webcluster1::172.24.0.2:80::health_flags::/failed_active_hc
 webcluster1::172.24.0.6:80::health_flags::/failed_active_hc
 webcluster1::172.24.0.3:80::health_flags::/failed_active_hc
 ```

```bash
[root@ip-10-0-0-91 priority-levels]# while true; do curl -s 172.24.0.7/service/color | grep -i '^hello'; sleep 1; done 
Hello from red (hostname: 2bdf4ae87460 resolvedhostname:172.24.0.4)
Hello from green (hostname: 89431bed3c8a resolvedhostname:172.24.0.2)
Hello from blue (hostname: d5f57a46dea0 resolvedhostname:172.24.0.5)
Hello from red (hostname: 2bdf4ae87460 resolvedhostname:172.24.0.4)
Hello from green (hostname: 89431bed3c8a resolvedhostname:172.24.0.2)
Hello from blue (hostname: d5f57a46dea0 resolvedhostname:172.24.0.5)
Hello from blue (hostname: d5f57a46dea0 resolvedhostname:172.24.0.5)
Hello from red (hostname: 2bdf4ae87460 resolvedhostname:172.24.0.4)
Hello from green (hostname: 89431bed3c8a resolvedhostname:172.24.0.2)
Hello from blue (hostname: d5f57a46dea0 resolvedhostname:172.24.0.5)
```

#### 位置加权负载均衡

```yaml
    load_assignment:
      cluster_name: webcluster1
      policy:
        overprovisioning_factor: 140
      endpoints:
      - locality:
          region: cn-north-1
          zone
          sub_zone
        load_balancing_weight: # 当前组权重 结合cluster.common_lb_config.locality_weighted_lb_config 启用位置加权负载均衡
        priority: 0
        lb_endpoints:
        - endpoint:
            health_check_config:
            address:
              socket_address:
                address: colorful
                port_value: 80
          load_balancing_weight: # 单端点权重，
          metadata:
          health_status:
          
        
```

让每个组都负载流量。优先级默认高优先有流量 。

与优先级负载类似，某个组内节点不健康了，会动态调整组权重，而非端点权重

A组80%端点健康，权重就变成了 `组权重*0.8`, 也和优先级相同方式配置超配因子：140，1.4。 所以动态权重 `权重*min(1,健康端点比率*1.4)` 如果小于100，则健康率小于72%，就`权重*比率`, 如果健康率大于72%, `权重*100%`, 就不动权重。

```bash
[root@ip-10-0-0-91 cluster-manager]# tree locality-weighted/
locality-weighted/
├── docker-compose.yaml
├── front-envoy.yaml
├── README.md
└── send-request.sh

```

> envoy的yaml, 优先级0，均为最高优先。存在`common_lb_config.locality_weighted_lb_config`, 表示是位置加权LB, 静态定义了2个组：属于2个位置， north-1, north-2. 每个位置的权重是10:20, 即`1:2`. 流量来时，都到高优先， 按1:2流量到各个组。第1/2个组内，按rr调度。 
>
> 如果第2个组失败1个端点 ，就是50%成功率，而`policy.overprovisioning_factor` 表示成功率`0.5*1.4`= `70%`, 小于100%， 所以位置权重会动态变化为：`1:2 => 1:2*0.7 => 1:1.4`, 大量流量将转移到组1.
>
> ```yaml
>   clusters:
>   - name: webcluster1
>     connect_timeout: 0.25s
>     type: STRICT_DNS
>     lb_policy: ROUND_ROBIN
>     http2_protocol_options: {}
>     common_lb_config:
>       locality_weighted_lb_config: {}
>     load_assignment:
>       cluster_name: webcluster1
>       policy:
>         overprovisioning_factor: 140
>       endpoints:
>       - locality:
>           region: cn-north-1
>         priority: 0
>         load_balancing_weight: 10
>         lb_endpoints:
>         - endpoint:
>             address:
>               socket_address:
>                 address: colored
>                 port_value: 80
>       - locality:
>           region: cn-north-2
>         priority: 0
>         load_balancing_weight: 20
>         lb_endpoints:
>         - endpoint:
>             address:
>               socket_address:
>                 address: myservice
>                 port_value: 80
>     health_checks:
>     - timeout: 5s
>       interval: 10s
>       unhealthy_threshold: 2
>       healthy_threshold: 1
>       http_health_check:
>         path: /healthz
>         expected_status: 200
> 
> ```

启动

```bash
docker-compose up
```

集群状态

```bash
[root@ip-10-0-0-91 priority-levels]# docker exec locality-weighted_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:19:00:07  
          inet addr:172.25.0.7  Bcast:172.25.255.255  Mask:255.255.0.0
```

```bash
[root@ip-10-0-0-91 priority-levels]# curl -s 172.25.0.7:9901/clusters | grep  flags
webcluster1::172.25.0.6:80::health_flags::healthy
webcluster1::172.25.0.5:80::health_flags::healthy
webcluster1::172.25.0.3:80::health_flags::healthy
webcluster1::172.25.0.4:80::health_flags::healthy
webcluster1::172.25.0.2:80::health_flags::healthy
```

现在请求查看

```bash
[root@ip-10-0-0-91 locality-weighted]# cat send-request.sh 
#!/bin/bash
declare -i colored=0
declare -i colorless=0

interval="0.1"

while true; do
        if curl -s http://$1/service/colors | grep -E "red|blue|green" &> /dev/null; then
                # $1 is the host address of the front-envoy.
                colored=$[$colored+1]
        else
                colorless=$[$colorless+1]
        fi
        echo $colored:$colorless
        sleep $interval
done
```

查看请求比

```bash
[root@ip-10-0-0-91 locality-weighted]# ./send-request.sh 172.25.0.7
0:1
0:2
1:2
1:3
1:4
1:5
2:5
2:6
3:6
4:6
4:7
5:7
5:8
5:9
5:10
5:11
5:12
6:12
6:13
6:14
6:15
7:15
7:16
7:17
7:18
8:18
8:19
9:19
9:20
10:20
10:21
10:22
10:23
11:23
11:24
```

> 接近`1:2`

现在将第2个组内的一个端点破坏时, 再次执行脚本

```bash
docker exec locality-weighted_service_black_1 rm -f /var/www/html/health.html
```

```bash
[root@ip-10-0-0-91 priority-levels]# curl -s 172.25.0.7:9901/clusters | grep  flags
webcluster1::172.25.0.6:80::health_flags::healthy
webcluster1::172.25.0.5:80::health_flags::healthy
webcluster1::172.25.0.3:80::health_flags::healthy
webcluster1::172.25.0.4:80::health_flags::healthy
webcluster1::172.25.0.2:80::health_flags::/failed_active_hc
```

```bash
[root@ip-10-0-0-91 locality-weighted]# ./send-request.sh 172.25.0.7
0:1
1:1
2:1
2:2
3:2
3:3
3:4
4:4
4:5
4:6
5:6
5:7
6:7
6:8
6:9
7:9
7:10
8:10
9:10
9:11
9:12
10:12
```

> 接近`1:1.4`

#### 混合优先级和位置权重

先满足优先级，后满足位置权重



#### 负载均衡器子集调度

以上是将endpoint分成逻辑上的组：按优先级、按权重划分。

使用负载均衡器子集：

1. endpint添加键值的元数据，一个主机上可以有多个键值数据。

   ```bash
   # EDS发现的端点， load_assignment静态定义的端点才支持
     clusters:
     - name: webcluster1
       connect_timeout: 0.25s
       type: STRICT_DNS
       lb_policy: ROUND_ROBIN
       http2_protocol_options: {}
       load_assignment:
         cluster_name: webcluster1
         endpoints:
         - lb_endpoints:
           - endpoint:
               address:
                 socket_address:
                   address: e1
                   port_value: 80
             metadata:
               filter_metadata:
                 envoy.lb:      
                   stage: "prod"
                   version: "1.0"
                   type: "std"
                   xlarge: true
   ```

   > 标签必须定义在`metadata.filter_metadata.envoy.lb`下，是map, 不支持嵌套。 
   >
   > 地址可以是域名，多个主机将有相同的元数据。
   >
   > 如果使用eds发现端点，配置元数据将可以一下分配给多个主机。

   

2. `subset_selectors` 指定键即可，所以拥有键集合的主机，其值相同的，划分成一个组，也叫一个子集。

   ```yaml
     clusters:
     - name: webcluster1
       load_assignment: {}
       lb_subset_config:
         fallback_policy: DEFAULT_SUBSET            
         default_subset:
           stage: "prod"
           version: "1.0"
           type: "std"
         subset_selectors:
         - keys: ["stage", "type"]
         - keys: ["stage", "version"]
         - keys: ["version"]
         - keys: ["xlarge", "version"]
         locality_weight_aware: # 考虑位置，目前存在缺陷
         scale_locality_weight: # 
         panic_mode_any:
         list_as_any:  
   ```

   > 路由不匹配时，回退策略：
   >
   > 1. 失败 fall
   > 2. 所有主机间调度 any_subset
   > 3. 默认子集 default_subset
   >
   > `subset_selectors` 子集选择器列表。 每一个项，定义一个子集选择器，一个选择器由多个键组成的列表。相同值就是一个子集。 

3. 调度时，需要路由器上指定元数据，挑选出适配到`subset_selectors`选择出来的某一个子集上。

   ```yaml
                 routes:
                 - match:
                     prefix: "/"
                     headers:
                     - name: x-custom-version
                       value: pre-release
                   route:
                     cluster: webcluster1
                     metadata_match:
                       filter_metadata:
                         envoy.lb:
                           version: "1.2-pre"
                           stage: "dev"
                 - match:
                     prefix: "/"
                   route:
                     weighted_clusters:
                       clusters:
                       - name: webcluster1
                         weight: 90
                         metadata_match:
                           filter_metadata:
                             envoy.lb:
                               version: "1.0"
                       - name: webcluster1
                         weight: 10
                         metadata_match:
                           filter_metadata:
                             envoy.lb:
                               version: "1.1"
                     metadata_match:
                       filter_metadata:
                         envoy.lb:
                           stage: "prod"
   
   ```

   > 路由指定了子集，但是没有适配，就使用回退策略。
   >
   > routes.route下仅支持以下2者之一
   >
   > - cluster: 路由到单个集群，      
   >
   > - weighted_cluster： 路由到**相同集群**，即将待路由的流量 ，按权重分配到集群上去，并且每一个weighted_cluster.clusters 中单个集群，也可以使用元数据匹配。如果在route下定义了metadata_match，又在weighted_clusters.clusters单个集群中又定义了metadata_match, 两者作交集使用。 prod环境下，     1.0版本承载90% 并且 1.1承载10%的流量



```bash
[root@ip-10-0-0-91 cluster-manager]#   tree lb-subsets/
lb-subsets/
├── docker-compose.yaml
├── front-envoy.yaml
├── README.md
└── test.sh

```

> `front-envoy.yaml` 定义的端点如下：
>
> | 端点 | stage | version | type   | xlarge |
> | ---- | ----- | ------- | ------ | ------ |
> | e1   | prod  | 1.0     | std    | true   |
> | e2   | prod  | 1.0     | std    |        |
> | e3   | prod  | 1.1     | std    |        |
> | e4   | prod  | 1.1     | std    |        |
> | e5   | prod  | 1.0     | bigmem |        |
> | e6   | prod  | 1.1     | bigmem |        |
> | e7   | dev   | 1.2-pre | std    |        |
>
> > type ： std标准的硬件配置
> >
> > stage: 阶段， 生产、开发环境
> >
> > version: 版本，1.0, 1.1, 1.2-pre预发
>
> 默认子集`default_subset`如下所示，所以匹配以上表格，仅`e1, e2`符合条件。
>
> ```yaml
>       default_subset:
>         stage: "prod"
>         version: "1.0"
>         type: "std"
> ```
>
> 子集选择器`subset_selectors`
>
> ```yaml
>       - keys: ["stage", "type"]
>       - keys: ["stage", "version"]
>       - keys: ["version"]
>       - keys: ["xlarge", "version"]
> ```
>
> 第1个选择器，仅对stage,type考虑，综合以上表格，
>
> stage=prod,dev.     type=std, bigmem,std. 所以就有6种组合，但是实际上，只有3个子集
>
> e1, e2, e3, e4: stage=prod, type=std
>
> e5,e6:  stage=prod, type=bigmem
>
> e7: stage=dev, type=std
>
> 路由到目前集群时，metadata_match, 来限制集群
>
> `docker-compose.yaml` 给每个服务传递不同的服务环境变量，e3就会响应e3

启动

```bash
docker-compose up
```

查看集群状态

```bash
[root@ip-10-0-0-91 cluster-manager]# docker exec lb-subsets_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:1A:00:04  
          inet addr:172.26.0.4  Bcast:172.26.255.255  Mask:255.255.0.0
```

```bash
[root@ip-10-0-0-91 cluster-manager]# curl -s 172.26.0.4:9901/clusters | grep flags
webcluster1::172.26.0.8:80::health_flags::healthy
webcluster1::172.26.0.9:80::health_flags::healthy
webcluster1::172.26.0.7:80::health_flags::healthy
webcluster1::172.26.0.5:80::health_flags::healthy
webcluster1::172.26.0.3:80::health_flags::healthy
webcluster1::172.26.0.2:80::health_flags::healthy
webcluster1::172.26.0.6:80::health_flags::healthy
```

> 所有均正常

通过脚本，测试不带header的weighted_cluster中的1.0与1.1比例。prod的1.0版本对应e1, e2, e5. 

```bash
[root@ip-10-0-0-91 lb-subsets]# cat test.sh 
#!/bin/bash
declare -i v10=0
declare -i v11=0

for ((counts=0; counts<200; counts++)); do
        if curl -s http://$1/service/host | grep -E "e[125]" &> /dev/null; then
                # $1 is the host address of the front-envoy.
                v10=$[$v10+1]
        else
                v11=$[$v11+1]
        fi
done

echo "Requests: v1.0:v1.1 = $v10:$v11"

```

```bash
[root@ip-10-0-0-91 lb-subsets]# bash test.sh 172.26.0.4
Requests: v1.0:v1.1 = 182:18
```

> 近似`9:1`

测试带了header, x-custom-version=pre-release, 到达dev

```bash
[root@ip-10-0-0-91 lb-subsets]# curl -H 'x-custom-release: pre-release' 172.26.0.4/service/host
<body bgcolor="e1"><span style="color:white;font-size:4em;">
Hello from e1 (hostname: cfb3eb6f33fe resolvedhostname:172.26.0.8)
</span></body>

[root@ip-10-0-0-91 lb-subsets]# curl -H 'x-custom-version: pre-release' 172.26.0.4/service/host
<body bgcolor="e7"><span style="color:white;font-size:4em;">
Hello from e7 (hostname: cb51b3f87682 resolvedhostname:172.26.0.6)
</span></body>

```

> 不一致时，到达默认的

测试`x-hardware-test: memory`, 到e5, e6. 并且使用lb_policy的轮循。

```bash
[root@ip-10-0-0-91 lb-subsets]# curl -s -H  'x-hardware-test: memory' 172.26.0.4/service/host | grep -i 'hello'
Hello from e5 (hostname: 6fd408aa899f resolvedhostname:172.26.0.3)
[root@ip-10-0-0-91 lb-subsets]# curl -s -H  'x-hardware-test: memory' 172.26.0.4/service/host | grep -i 'hello'
Hello from e6 (hostname: 81ca7089103e resolvedhostname:172.26.0.2)
[root@ip-10-0-0-91 lb-subsets]# curl -s -H  'x-hardware-test: memory' 172.26.0.4/service/host | grep -i 'hello'
Hello from e5 (hostname: 6fd408aa899f resolvedhostname:172.26.0.3)
[root@ip-10-0-0-91 lb-subsets]# curl -s -H  'x-hardware-test: memory' 172.26.0.4/service/host | grep -i 'hello'
Hello from e6 (hostname: 81ca7089103e resolvedhostname:172.26.0.2)
```



## 体现服务韧性

envoy： 熔断 -> istio: 连接池

envoy:  异常值检测 -> istio: 熔断

### 熔断

多级调用中，被调用服务故障，会级联故障，可能导致系统整体不可用。例如：C/D -> B -> A , A不可用，B大量阻塞，B消耗完套接字，B不可用，然后C/D又不可用。整个系统不可用。

**熔断**：上游服务压力过大，响应过慢甚至失败时，下游服务暂时切断对上游服务的请求。熔断是分布式应用常用的一种流量管理模式，它能够让应用程序免受上游服务失败、延迟峰值或其他网络异常的侵害。

Envoy没有在应用程序级别进行断路限制，是网络级别强制进行限制，不必独立配置和编码每个应用。envoy以sidecar工作时，其后面的应用，都可以应用网络级熔断。

工作逻辑：

- open, 固定时间窗口内，检测的失败指标达到定义的阈值，就启动熔断。断开，即所有请求快速失败(503)，而不是阻塞。

  如果open状态之后，一旦下一次请求成功，立即转换为close.

  如果一直失败，一直是open状态。

- close, 熔断本身关闭，回路通了。

- half open: 中间状态，根据下次请求结果确定转换为open/close状态。 



envoy支持5种断路器（连接池）

- 集群最大连接数：envoy与上游集群建立的最大连接数，仅适用于HTTP/1.1, 因为HTTP/2可以链路复用。
- 集群最大请求数：在给定的时间内，集群中的所有主机未完成的最大请求数，仅适用于HTTP/2;
- 集群可挂起的最大请求数：集群最大连接数是1024，满载时，可挂起的最大请求数。
- 集群最大活动并发重试次数：请求失败时，可以重试在集群中，每个主机都有可能重试，此选项定义对每个主机同时发起重试并发的重试次数。
- 集群最大并发连接池：envoy与后端建立的连接池最大数量。溢出时，集群的upstream_rq_pending_overflow计数器就会递增。



isito的熔断功能：连接池 和 被动健康状态检测 进行定义，envoy的断路器通常仅对应于istio的连接池功能。

> traefik的熔断，是被动健康检测

```yaml
  clusters:
  - name: webcluster1
    connect_timeout: 0.25s # TCP连接超时
    max_reqeusts_per_connection: 
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: webcluster1
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: colored
                port_value: 80
    circuit_breakers:
      thresholds:
      - priority:
        max_connections: 1
        max_pending_requests: 1
        max_retries: 3
        max_requests: 
        max_connection_pools:
        track_remaining:
```

> 1. 集群级别限制: 
>
> `max_reqeusts_per_connection`： 每个连接可以承载最大请求数，HTTP/1.1和HTTP/2的连接池均受此限制，不设置则不限制，能发多少就多少。1表示每个连接1个请求，表示禁用keep-alive.
>
> `connect_timeout` 连接超时。
>
> `circuit_breakers` 熔断设置
>
> `thresholds` 多个连接池定义
>
> `priority`: 当前连接池优先级
>
> `max_connections` HTTP/1.1 最大并发连接数，默认1024
>
> `max_pending_requests` 集群最大连接数是1024，满载时，可挂起的最大请求数。默认1024
>
> `max_requests` 仅适用于HTTP/2，最大请求数，默认1024.
>
> `max_retries` 最大并发重试数（retry_policy配置情况下，此选项有有意义）；默认 3
>
> `max_connection_pools` 每个集群可以同时打开的最大连接池数量，默认为无限制；
>
> `track_remaining` 其值为true时，公布统计数据以显示断路器open前所剩余的资源数量；默认false

#### 架构

![image-20210908111958252](http://myapp.img.mykernel.cn/image-20210908111958252.png)

```bash
[root@ip-10-0-0-91 cluster-manager]# tree circuit-breaker/
circuit-breaker/
├── docker-compose.yaml
├── front-envoy.yaml
├── send-requests.sh
└── service-envoy.yaml
```

> `front-envoy.yaml`是前端envoy的配置
>
> 1. 第1个集群，前端代理定义了熔断器, 后端也是envoy代理，但是没有加envoy熔断器。
> 2. 第2个集群，前端代理定义了异常值检测，后端的集群主机的sidecar才定义了熔断器。通常这样使用。
>
> `service-envoy.yaml`是作为sidecar的envoy的配置
>
> `docker-compose.yaml`定义了5个容器，前3个是有色的。后2个是无色的。无色容器，挂载envoy配置

#### 启动

```bash
docker-compose up
front-envoy_1    | [2021-09-08 03:14:31.025][1][info][config] [source/server/configuration_impl.cc:67] loading 2 cluster(s)
service_black_1  | [2021-09-08 03:14:31.150][9][info][config] [source/server/configuration_impl.cc:67] loading 1 cluster(s)
service_gray_1   | [2021-09-08 03:14:31.035][8][info][config] [source/server/configuration_impl.cc:67] loading 1 cluster(s)
```

验证有颜色和无颜色

```bash
$ curl 172.27.0.7/service/colored
$ curl 172.27.0.7/service/colorless
```



现在给集群施加压力，看看熔断的效果。

```bash
./send-requests.sh 172.27.0.7/service/colored 50
#无法模拟较大并发压力的效果
```

现在使用[fortio](https://github.com/fortio/fortio), 模拟压力测试

```bash
$ fortio load -c 2 -qps 0 -n 20 -loglevel Warning  172.27.0.7/service/colored
Code 200 : 20 (100.0 %)
```

> 虽然cluster1, 定义的断路器，最大连接1，挂起1，但是后端集群规模大，所以很快响应了

现在调整并发

```bash
# fortio load -c 4 -qps 0 -n 1000 -loglevel Warning  172.27.0.7/service/colored
Code 200 : 999 (99.9 %)
Code 503 : 1 (0.1 %)
```

> 现在有503了，获取webcluster1状态
>
> ```bash
> curl -s 172.27.0.7:9901/stats | grep cluster.webcluster1
> cluster.webcluster1.circuit_breakers.default.cx_open: 1
> cluster.webcluster1.external.upstream_rq_200: 1458
> cluster.webcluster1.external.upstream_rq_2xx: 1458
> cluster.webcluster1.external.upstream_rq_503: 28
> cluster.webcluster1.external.upstream_rq_5xx: 28
> cluster.webcluster1.upstream_cx_overflow: 134  # 溢出
> cluster.webcluster1.upstream_cx_pool_overflow: 0
> cluster.webcluster1.upstream_rq_pending_overflow: 28 # 溢出，open.
> cluster.webcluster1.upstream_rq_pending_total: 135 # 总挂起
> ```
>
> 随着并发量增加，失败变多，熔断打开再所难免

现在请求没有颜色

```bash
fortio load -c 6 -qps 0 -n 100 -loglevel Warning  172.27.0.6/service/colorless
```

> 第2个集群请求，异常检测，会弹出部分节点。

