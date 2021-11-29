---
title: Envoy管理接口基础应用
date: 2021-03-17 09:03:43
tags:
- istio
---



# admin管理接口

只要使用envoy的bootstrap首先应该定义admin接口，Envoy内建一个管理admin接口，支持查询和修改操作，查询结果可能会暴露envoy私有数据(证书), 因此有必要访问控制机制避免非授权访问(监听127.0.0.1).



# 启动admin管理接口

## 环境准备

基于egress示例

### envoy.yaml

```diff
+admin:
+  access_log_path: /tmp/admin.log
+  address:
+    socket_address: 
+      address: 127.0.0.1
+      port_value: 9901

static_resources:
  listeners:
  - name: listener_0  # 监听外部地址
    address:
      socket_address: { address: 127.0.0.1, port_value: 80 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager # 4层过滤的HTTP CONNECTION MANAGER
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager"
          stat_prefix: egress_http    # 统计信息前缀, 通常egress_http或ingress_http。 方便监控
          codec_type: AUTO            # HTTP 7层编解码器。AUTO自动；HTTP1: http 1.0;  HTTP2: http 2.0
          http_filters:
          - name: envoy.filters.http.router
          route_config:               # 静态路由配置; 动态配置应该使用rds字段进行指定
            name: test_route      # 路由配置名
            virtual_hosts:       # (多个) 虚拟主机列表, 用于构成路由表;     Downstream -> Host: xxxx -> domains -> match -> route/redirect/direct_response -> route(cluster, cluster_header, weighted_clusters)
            - name: web_service_1  # 虚拟主机逻辑名，用于统计信息，与路由无关
              domains: ["*.ik8s.io","ik8s.io"]     # 当前虚拟主机匹配的域名列表，支持 * 通配; 匹配搜索次序为: 精确匹配: www.magedu.com、前缀匹配: *.magedu.com <= (www|bbs|test).magedu.com、后缀通配: www.magedu.* <= www.magedu.(cn|org|io|com)及完全通配: * <= 只要访问这个端口, 就算匹配；
              routes:       # (RDS动态生成) 路由列表, 按顺序搜索，第一个匹配的路由信息。 
              - name:  # (optional) 路由条目名称
                match: { prefix: "/" }             # match字段可通过 prefix(前缀)、path(路径)、regex(正则表达式)三者"之一"来表示匹配模式 
                  #prefix: # 请求url前缀
                route: { cluster: web_cluster_1 }              # 与match相关的请求由route/redirect/direct_response三个字段"之一"完成路由
                  #cluster: # 目标集群  # route定义的目标必须是cluster(目标集群), cluster_header(根据请求标头的cluster_header值确定目标集群) 或weighted_clusters(路由目标有多个集群，每个集群有一定权重) 其中"之一"


            - name: web_service_2  # 虚拟主机逻辑名，用于统计信息，与路由无关
              domains: ["*.ik8scast.cn","ik8s.cn"]     # 当前虚拟主机匹配的域名列表，支持 * 通配; 匹配搜索次序为: 精确匹配: www.magedu.com、前缀匹配: *.magedu.com <= (www|bbs|test).magedu.com、后缀通配: www.magedu.* <= www.magedu.(cn|org|io|com)及完全通配: * <= 只要访问这个端口, 就算匹配；
              routes:       # (RDS动态生成) 路由列表, 按顺序搜索，第一个匹配的路由信息。 
              - name:  # (optional) 路由条目名称
                match: { prefix: "/" }             # match字段可通过 prefix(前缀)、path(路径)、regex(正则表达式)三者"之一"来表示匹配模式 
                  #prefix: # 请求url前缀
                route: { cluster: web_cluster_2 }              # 与match相关的请求由route/redirect/direct_response三个字段"之一"完成路由
                  #cluster: # 目标集群  # route定义的目标必须是cluster(目标集群), cluster_header(根据请求标头的cluster_header值确定目标集群) 或weighted_clusters(路由目标有多个集群，每个集群有一定权重) 其中"之一"

  clusters:
  - name: web_cluster_1        # 集群惟一名称, 且未提供alt_stat_name时将会被用于统计信息中；
    connect_timeout: 0.25s    
    type: STRICT_DNS              # 解析集群生成endpoint时，使用的类型。STATIC, STRIC_DNS: 解析多个A记录, LOGICAL_DNS: 解析一个A记录，用于大型web场景; ORIGINAL_DEST: 不转换端口，直接同client请求的端口；和EDS: endpoint discovery server等.
    lb_policy: ROUND_ROBIN    # (optional) 负载均衡算法，支持ROUND_ROBIN, LEAST_REQUEST, RING_HASH, RANDOM, MAGLEV, CLUSTER_PROVIDED;
    load_assignment:          # 为STATIC, STRIC_DNS, LOGICAL_DNS类型的集群指定成员获取方式；EDS类型的集群使用eds_cluster_config字段配置;
      cluster_name: web_cluster_1 # 集群 名， 与上面的name一致
      endpoints:                 # 端点列表. 可能在不同的dc, zone, 机柜, 不同服务器.
      - locality: {}             # (optional) 标识upstream的位置. 通常以region、zone、subzone...标识. 没有划分时，可以不定义
        lb_endpoints:            # 指定位置端点列表
        - endpoint:             # 端点定义, Egress所以外部是多个服务
            address:             # 地址
              socket_address: { address: myservice, port_value: 8081 }

  - name: web_cluster_2        # 集群惟一名称, 且未提供alt_stat_name时将会被用于统计信息中；
    connect_timeout: 0.25s    
    type: STRICT_DNS              # 解析集群生成endpoint时，使用的类型。STATIC, STRIC_DNS: 解析多个A记录, LOGICAL_DNS: 解析一个A记录，用于大型web场景; ORIGINAL_DEST: 不转换端口，直接同client请求的端口；和EDS: endpoint discovery server等.
    lb_policy: ROUND_ROBIN    # (optional) 负载均衡算法，支持ROUND_ROBIN, LEAST_REQUEST, RING_HASH, RANDOM, MAGLEV, CLUSTER_PROVIDED;
    load_assignment:          # 为STATIC, STRIC_DNS, LOGICAL_DNS类型的集群指定成员获取方式；EDS类型的集群使用eds_cluster_config字段配置;
      cluster_name: web_cluster_2 # 集群 名， 与上面的name一致
      endpoints:                 # 端点列表. 可能在不同的dc, zone, 机柜, 不同服务器.
      - locality: {}             # (optional) 标识upstream的位置. 通常以region、zone、subzone...标识. 没有划分时，可以不定义
        lb_endpoints:            # 指定位置端点列表
        - endpoint:             # 端点定义, Egress所以外部是多个服务
            address:             # 地址
              socket_address: { address: webserver1, port_value: 8081 }
```

### dockerfile

```bash
root@harbor01:~/admininterface# cat Dockerfile-envoy 
FROM envoyproxy/envoy-alpine:v1.17-latest

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
RUN apk update && apk --no-cache add curl tzdata iotop gcc gcc libgcc libc-dev libcurl libc-utils pcre-dev zlib-dev libnfs make pcre pcre2 zip unzip net-tools pstree wget libevent libevent-dev iproute2 tcpdump vim
```



### docker-compose

```bash
root@harbor01:~/admininterface# cat docker-compose.yaml 
version: '3'
services:
  sidecar:
    build:
      context: .
      dockerfile: Dockerfile-envoy
    image: envoyproxy/envoy-alpine:v1.17-curl
    volumes:
    - ./envoy.yaml:/etc/envoy/envoy.yaml
    networks:
      envoymesh:
        aliases:
        - envoy
    depends_on: # 先于当前服务运行的服务
    - webserver1
    - webserver2
    expose:
    - "80"

  webserver1:
    image: ikubernetes/mini-http-server:v0.3
    networks:
      envoymesh:
        aliases:
        - webserver1
        - myservice
    expose:
    - "8081"

  webserver2:
    image: ikubernetes/mini-http-server:v0.3
    networks:
      envoymesh:
        aliases:
        - webserver2
        - myservice
    expose:
    - "8081"

networks:
  envoymesh: {}
```

### 启动

```bash
root@harbor01:~/admininterface# docker-compose up
# 次序
Starting admininterface_webserver2_1 ... 
Starting admininterface_webserver1_1 ... 
Starting admininterface_webserver2_1
Starting admininterface_webserver2_1 ... done
Starting admininterface_sidecar_1 ... 
Starting admininterface_sidecar_1 ... done


# 监听
webserver1_1  | Listening on http://0.0.0.0:8081
webserver2_1  | Listening on http://0.0.0.0:8081

# admin listen
sidecar_1     | [2021-03-17 09:18:41.091][1][info][main] [source/server/server.cc:486] admin address: 127.0.0.1:9901


# 配置
sidecar_1     | [2021-03-17 09:18:41.095][1][info][config] [source/server/configuration_impl.cc:91] loading 2 cluster(s)
sidecar_1     | [2021-03-17 09:18:41.098][1][info][config] [source/server/configuration_impl.cc:95] loading 1 listener(s)
...
sidecar_1     | [2021-03-17 09:18:41.109][1][info][upstream] [source/common/upstream/cluster_manager_impl.cc:191] cm init: all clusters initialized
```

```bash
root@harbor01:~/admininterface# docker ps
CONTAINER ID   IMAGE                                COMMAND                  CREATED         STATUS         PORTS               NAMES
`83a55d7847bf   envoyproxy/envoy-alpine:v1.17-curl   "/docker-entrypoint.…"   5 minutes ago   Up 3 minutes   80/tcp, 10000/tcp   admininterface_sidecar_1`
7207aa377f77   ikubernetes/mini-http-server:v0.3    "/bin/httpserver"        5 minutes ago   Up 3 minutes   8081/tcp            admininterface_webserver2_1
2b49786b3648   ikubernetes/mini-http-server:v0.3    "/bin/httpserver"        5 minutes ago   Up 3 minutes   8081/tcp            admininterface_webserver1_1
root@harbor01:~/admininterface# docker exec -it admininterface_sidecar_1 sh
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:12:00:04  
          inet addr:172.18.0.4  Bcast:172.18.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:11 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:866 (866.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:376 errors:0 dropped:0 overruns:0 frame:0
          TX packets:376 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:24440 (23.8 KiB)  TX bytes:24440 (23.8 KiB)

/ # netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:9901          0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.11:45709        0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:80            0.0.0.0:*               LISTEN     
/ # curl 127.0.0.1:9901

```



## admin配置解析

### 日志



### 获取帮助

```bash
/ # curl 127.0.0.1:9901/help
admin commands are:
  /: Admin home page #GET
  /certs:  #TLS通信时，打印数字证书，暴露了用户私有数据。所以需要访问控制。 #GET 已加载所有TLS证书信息
  /clusters: upstream cluster status。#上游服务器状态              #GET, GET /clusters?format=json
  /config_dump: dump current Envoy configs (experimental)。#转储envoy配置 #GET Envoy加载各类配置
  /contention: dump current Envoy mutex contention stats (if enabled)。 # GET 互斥跟踪
  /cpuprofiler: enable/disable the CPU profiler # POST 启动或禁用cpuprofier. 请求启动，再请求禁用
  /drain_listeners: drain listeners
  /healthcheck/fail: cause the server to fail health checks。#强制upstream为fail # POST, 强制设定http健康状态检查为失败。
  /healthcheck/ok: cause the server to pass health checks。#强制upstream为ok     # POST, 强制设定http健康状态检查为成功
  /heapprofiler: enable/disable the heap profiler    # POST 启动或禁用heapprofier. 请求启动，再请求禁用
  /help: print out list of admin commands
  /hot_restart_version: print the hot restart compatibility version #GET 热重启次数
  /init_dump: dump current Envoy init manager information (experimental)
  /listeners: print listener info # GET 打印lisenter信息。支持使用GET /listeners?format=json
  /logging: query/change logging levels # POST,  启动或禁用不同子组件上的不同日志记录级别
  /memory: print current allocation/heap usage # POST, 打印当前内存分配信息，以字节为单位；
  /quitquitquit: exit the server # POST 干净退出envoy
  /ready: print server state, return 200 if LIVE, otherwise return 503
  /reopen_logs: reopen access logs
  /reset_counters: reset all counters to zero # POST 重置计数器
  /runtime: print runtime values # GET json获取runtime相关值 
  /runtime_modify: modify runtime values # POST 修改runtime值  /runtime_modify?key1=value1&key2=value2, 添加或修改param中传递的运行时值
  /server_info: print server version/status information # GET Envoy服务相关信息
  /stats: print server stats # 按需要输出统计数据，例如 ： /stats?filter=`regex`, 另外支持json/prometheus 2种格式输出 
  /stats/prometheus: print server stats in prometheus format  # prometheus格式数据
  /stats/recentlookups: Show recent stat-name lookups
  /stats/recentlookups/clear: clear list of stat-name lookups and counter
  /stats/recentlookups/disable: disable recording of reset stat-name lookup names
  /stats/recentlookups/enable: enable recording of reset stat-name lookup names
```

### server_info

 print server version/status information # GET Envoy服务相关信息

```bash
/ # curl -s 127.0.0.1:9901/server_info  
{
 "version": "d6a4496e712d7a2335b26e2f76210d5904002c26/1.17.1/Clean/RELEASE/BoringSSL",
 "state": "LIVE", # 正常
 "hot_restart_version": "11.104",
 "command_line_options": {
  "base_id": "0",
  "use_dynamic_base_id": false,
  "base_id_path": "",
  "concurrency": 4,
  "config_path": "/etc/envoy/envoy.yaml", # 配置路径
...
  "log_format": "[%Y-%m-%d %T.%e][%t][%l][%n] [%g:%#] %v", # 日志格式

```



### stats

print server stats # 按需要输出统计数据，例如 ： /stats?filter=`regex`, 另外支持json/prometheus 2种格式输出 

```bash
/ # curl -s 127.0.0.1:9901/stats
http1.response_flood: 0 
listener.127.0.0.1_80.downstream_cx_active: 0

#字段1 哪个子系统
#字段xx, 子系统相关内部统计数据
```

#### cluster相关统计数据

```bash
*_cx_total: 0                    	# total connections
*_cx_active: 0 					# total active connections
cluster.web_cluster_1.upstream_cx_connect_fail: 0 				# total connections failure
*_rq_total: 0                      # 总请求
*_rq_timeout: 0                    # 总超时请求
```

#### json

```bash
/ # curl -s 127.0.0.1:9901/stats?format=json
```

#### prometheus

```bash
/ # curl -s 127.0.0.1:9901/stats?format=prometheus
```

#### 过滤

```bash
/ # curl -s 127.0.0.1:9901/stats?filter=^listener

/ # curl -s 127.0.0.1:9901/stats?filter=^cluster
```

### clusters

GET /custers 列出所有已配置集群，包括每个集群中发现的所有upstream servers及每个主机统计信息，支持json输出

- 集群管理器信息 version_info string, 没有使用Cluster Discovery Server时，显示为 version_info::static
- 集群相关信息：断路器、异常点检测和用于表示是否通过CDS添加的标识 add_via_api
- 每个主机统计信息：包括总连接，活动连接，总请求，主机健康状态, 不健康原因有以下3种
  - failed_active_hc 未通过健康状态检测
  - failed_eds_health   被EDS标记为不健康
  - failed_outlier_check 未通过异常检测机制的检查

```bash
/ # curl -s 127.0.0.1:9901/clusters
# 断路器、异常点检测和用于表示是否通过CDS添加的标识
web_cluster_1::default_priority::max_connections::1024
web_cluster_1::default_priority::max_pending_requests::1024
web_cluster_1::default_priority::max_requests::1024
web_cluster_1::default_priority::max_retries::3
web_cluster_1::high_priority::max_connections::1024
web_cluster_1::high_priority::max_pending_requests::1024
web_cluster_1::high_priority::max_requests::1024
web_cluster_1::high_priority::max_retries::3
web_cluster_1::added_via_api::false
# 第1个集群识别2个主机及统计信息
web_cluster_1::172.18.0.3:8081::cx_active::0
web_cluster_1::172.18.0.3:8081::cx_connect_fail::0
web_cluster_1::172.18.0.3:8081::cx_total::0
web_cluster_1::172.18.0.3:8081::rq_active::0
web_cluster_1::172.18.0.3:8081::rq_error::0
web_cluster_1::172.18.0.3:8081::rq_success::0
web_cluster_1::172.18.0.3:8081::rq_timeout::0
web_cluster_1::172.18.0.3:8081::rq_total::0
web_cluster_1::172.18.0.3:8081::hostname::myservice
web_cluster_1::172.18.0.3:8081::health_flags::healthy # health
web_cluster_1::172.18.0.3:8081::weight::1
web_cluster_1::172.18.0.3:8081::region::
web_cluster_1::172.18.0.3:8081::zone::
web_cluster_1::172.18.0.3:8081::sub_zone::
web_cluster_1::172.18.0.3:8081::canary::false
web_cluster_1::172.18.0.3:8081::priority::0
web_cluster_1::172.18.0.3:8081::success_rate::-1.0
web_cluster_1::172.18.0.3:8081::local_origin_success_rate::-1.0
web_cluster_1::172.18.0.2:8081::cx_active::0
web_cluster_1::172.18.0.2:8081::cx_connect_fail::0
web_cluster_1::172.18.0.2:8081::cx_total::0
web_cluster_1::172.18.0.2:8081::rq_active::0
web_cluster_1::172.18.0.2:8081::rq_error::0
web_cluster_1::172.18.0.2:8081::rq_success::0
web_cluster_1::172.18.0.2:8081::rq_timeout::0
web_cluster_1::172.18.0.2:8081::rq_total::0
web_cluster_1::172.18.0.2:8081::hostname::myservice
web_cluster_1::172.18.0.2:8081::health_flags::healthy  # health
web_cluster_1::172.18.0.2:8081::weight::1
web_cluster_1::172.18.0.2:8081::region::
web_cluster_1::172.18.0.2:8081::zone::
web_cluster_1::172.18.0.2:8081::sub_zone::
web_cluster_1::172.18.0.2:8081::canary::false
web_cluster_1::172.18.0.2:8081::priority::0
web_cluster_1::172.18.0.2:8081::success_rate::-1.0
web_cluster_1::172.18.0.2:8081::local_origin_success_rate::-1.0
# 第2个集群识别1个主机及统计信息
web_cluster_2::default_priority::max_connections::1024
web_cluster_2::default_priority::max_pending_requests::1024
web_cluster_2::default_priority::max_requests::1024
web_cluster_2::default_priority::max_retries::3
web_cluster_2::high_priority::max_connections::1024
web_cluster_2::high_priority::max_pending_requests::1024
web_cluster_2::high_priority::max_requests::1024
web_cluster_2::high_priority::max_retries::3
web_cluster_2::added_via_api::false
web_cluster_2::172.18.0.2:8081::cx_active::0
web_cluster_2::172.18.0.2:8081::cx_connect_fail::0
web_cluster_2::172.18.0.2:8081::cx_total::0
web_cluster_2::172.18.0.2:8081::rq_active::0
web_cluster_2::172.18.0.2:8081::rq_error::0
web_cluster_2::172.18.0.2:8081::rq_success::0
web_cluster_2::172.18.0.2:8081::rq_timeout::0
web_cluster_2::172.18.0.2:8081::rq_total::0
web_cluster_2::172.18.0.2:8081::hostname::webserver1
web_cluster_2::172.18.0.2:8081::health_flags::healthy
web_cluster_2::172.18.0.2:8081::weight::1
web_cluster_2::172.18.0.2:8081::region::
web_cluster_2::172.18.0.2:8081::zone::
web_cluster_2::172.18.0.2:8081::sub_zone::
web_cluster_2::172.18.0.2:8081::canary::false
web_cluster_2::172.18.0.2:8081::priority::0
web_cluster_2::172.18.0.2:8081::success_rate::-1.0
web_cluster_2::172.18.0.2:8081::local_origin_success_rate::-1.0


# CDS
/ # curl -s 127.0.0.1:9901/clusters | grep add
web_cluster_1::added_via_api::false
web_cluster_2::added_via_api::false
```

### listeners

GET /listeners 列出所有已配置的侦听器，包括侦听器的名称及监听的地址；支持输出为json格式

```bash
/ # curl -s 127.0.0.1:9901/listeners
listener_0::127.0.0.1:80
```

### 配置信息

GET /config_dump 以json格式打印当前 从Envoy的各种组件 加载的配置信息

```bash
/ # curl -s 127.0.0.1:9901/config_dump
```

### 计数器重置

POST /reset_counters 将所有计数器重置为0；不过，它只会影响server本地的输出，对于已发送到外部存储系统的数据无效

```bash
/ # curl -X POST 127.0.0.1:9901/reset_counters
OK
```

### ready

GET /ready 获取server 就绪与否的状态，LIVE状态为200，否则503

```bash
/ # curl -s 127.0.0.1:9901/ready
LIVE
```

