---
title: "Envoy的4层过滤器(Tcp Proxy)和基于4层过滤实现7层过滤(Http connection manager + http filter)"
date: 2021-03-16 06:18:17
tags:
- istio
---



# envoy

[Filter](https://www.envoyproxy.io/docs/envoy/v1.17.1/api-v3/config/filter/filter)

- 4: Network filters [TCP Proxy](https://www.envoyproxy.io/docs/envoy/v1.17.1/api-v3/extensions/filters/network/tcp_proxy/v3/tcp_proxy.proto)
- 7:  [filter](https://www.envoyproxy.io/docs/envoy/v1.17.1/api-v3/config/filter/filter): ( [network filter](https://www.envoyproxy.io/docs/envoy/v1.17.1/api-v3/config/filter/network/network):[http connection manager](https://www.envoyproxy.io/docs/envoy/v1.17.1/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto) + [http filter](https://www.envoyproxy.io/docs/envoy/v1.17.1/api-v3/config/filter/http/http): [Router](https://www.envoyproxy.io/docs/envoy/v1.17.1/api-v3/extensions/filters/http/router/v3/router.proto))

![image-20210316210628440](http://myapp.img.mykernel.cn/image-20210316210628440.png)





## 4层过滤器

client -> listener -> cluster

https://www.envoyproxy.io/docs/envoy/v1.17.1/api-v3/config/filter/filter

Network filters即4层过滤

<img src="http://myapp.img.mykernel.cn/image-20210316181706588.png" alt="image-20210316181706588" style="zoom: 67%;" />

![image-20210316142741788](http://myapp.img.mykernel.cn/image-20210316142741788.png)

- [x] downstream与upstream执行`1:1`网络连接代理
- [x] 同其他过滤器结合使用
- [x] 连接限制 
- [x] 将请求路由至指定集群，基于权重进行调度转发

## 7层过滤器

- [x] 基于4层过滤，只不过是借助4层的http connection manager的http_filter引入7层过滤
- [x] 基于domain匹配 -> 基于path匹配 -> cluster/加权cluster/合成响应报文

client -> listener(router) -> cluster

https://www.envoyproxy.io/docs/envoy/v1.17.1/api-v3/config/filter/filter

HTTP filters即7层过滤，由4层的httpconnectionmanager调用

<img src="http://myapp.img.mykernel.cn/image-20210316181735903.png" alt="image-20210316181735903" style="zoom:67%;" />

![image-20210316142748351](http://myapp.img.mykernel.cn/image-20210316142748351.png)

# 区分ingress和egress

## ingress

- [x] filter_chains.name 为 tcp_proxy
- [x] cluster.load_assignment.endpoints.lb_endpoints.endpoint 一般只有1个，就是内部的微服务。不需要负载均衡
- [x] 通信：client(外部客户端) -> listener(0.0.0.0:80) -> cluster endpoint(127.0.0.1:8081) -> upstream(envoy旁的微服务，1个)

![image-20210316143634093](http://myapp.img.mykernel.cn/image-20210316143634093.png)

## egress

- [x] filter_chains.name 为 tcp_proxy
- [x] cluster.load_assignment.endpoints.lb_endpoints.endpoint 一般只有多个，外部的微服务，可以负载均衡
- [x] 通信：client(envoy旁的微服务) -> listener -> cluster -> endpoint -> upstream(外部微服务列表)

![image-20210316143634093](http://myapp.img.mykernel.cn/image-20210316143634093.png)



# tcp proxy示例



## 环境介绍

| 操作系统    | 环境   | 主机        |
| ----------- | ------ | ----------- |
| Ubuntu 1804 | docker | 192.168.0.8 |

docker-compose 编排2个容器，模拟在pod中运行，共享同一网络名称空间

```bash
# 安装docker-compose
root@ubuntu-template:~# apt install docker-compose -y
```

## ingress

### envoy配置

https://www.envoyproxy.io/docs/envoy/latest/api/api

https://www.envoyproxy.io/docs/envoy/v1.17.1/api-v3/extensions/filters/network/tcp_proxy/v3/tcp_proxy.proto

`filters.name`

@type, 固定格式: ` type.googleapis.com/envoy.<name>`

![image-20210316162707073](http://myapp.img.mykernel.cn/image-20210316162707073.png)

```diff
mkdir tcpproxy
cd tcpproxy
root@ubuntu-template:~# cat envoy.yaml 
static_resources:
  listeners:
+  - name: listener_0  # 监听外部地址
    address:
      socket_address: { address: 0.0.0.0, port_value: 80 }
    filter_chains:
    - filters:
+      - name: envoy.filters.network.tcp_proxy # TCP代理 
        typed_config:
+          "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
          stat_prefix: tcp
          cluster: test_cluster
  clusters:
  - name: test_cluster        # 集群惟一名称, 且未提供alt_stat_name时将会被用于统计信息中；
    connect_timeout: 0.25s    
    type: STATIC              # 解析集群生成endpoint时，使用的类型。STATIC, STRIC_DNS: 解析多个A记录, LOGICAL_DNS: 解析一个A记录，用于大型web场景; ORIGINAL_DEST: 不转换端口，直接同client请求的端口；和EDS: endpoint discovery server等.
    lb_policy: ROUND_ROBIN    # (optional) 负载均衡算法，支持ROUND_ROBIN, LEAST_REQUEST, RING_HASH, RANDOM, MAGLEV, CLUSTER_PROVIDED;
    load_assignment:          # 为STATIC, STRIC_DNS, LOGICAL_DNS类型的集群指定成员获取方式；EDS类型的集群使用eds_cluster_config字段配置;
      cluster_name: test_cluster # 集群 名， 与上面的name一致
      endpoints:                 # 端点列表. 可能在不同的dc, zone, 机柜, 不同服务器.
      - locality: {}             # (optional) 标识upstream的位置. 通常以region、zone、subzone...标识. 没有划分时，可以不定义
        lb_endpoints:            # 指定位置端点列表
+        - endpoint:             # 端点定义, Ingress所以内部一个服务
            address:             # 地址
              socket_address: { address: 127.0.0.1, port_value: 8081 }
```

一个endpoint监听在127.0.0.1， 即主容器服务不接收任何请求。也可以所有地址，确保127.0.0.1可以给envoy访问即可。



### docker-compose配置

```bash
root@ubuntu-template:~# cat docker-compose.yaml 
version: '3'
services:
  sidecar:
    # https://www.envoyproxy.io/docs/envoy/v1.17.1/api/api
    image: envoyproxy/envoy-alpine:v1.17-latest
    volumes:
    - ./envoy.yaml:/etc/envoy/envoy.yaml
    network_mode: "service:mainserver"    # 与mainserver同享network ns
    depends_on: # 先于当前服务运行的服务
    - mainserver
  mainserver:
    image: ikubernetes/mini-http-server:v0.3
    networks:
      envoymesh:
        aliases:
        - webserver
        - httpserver


networks:
  envoymesh: {}
```

启动

```bash
root@ubuntu-template:~# docker-compose up
sidecar_1     | [2021-03-16 08:25:54.221][1][info][upstream] [source/common/upstream/cluster_manager_impl.cc:191] cm init: all clusters initialized # 输出此行表示OK
```

### 了解运行特征

查看容器

```bash
root@ubuntu-template:~# docker ps
CONTAINER ID   IMAGE                                  COMMAND                  CREATED             STATUS         PORTS      NAMES
70e7877de74f   envoyproxy/envoy-alpine:v1.17-latest   "/docker-entrypoint.…"   About an hour ago   Up 3 minutes              root_sidecar_1
a8840adb2e52   ikubernetes/mini-http-server:v0.3      "/bin/httpserver"        About an hour ago   Up 3 minutes   8081/tcp   root_mainserver_1
```

进入envoy

```bash
root@ubuntu-template:~# docker exec -it root_sidecar_1 sh
/ # netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 127.0.0.11:39657        0.0.0.0:*               LISTEN      
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN       # listener监听
tcp        0      0 :::8081                 :::*                    LISTEN       # mainserver监听，因为同network ns. 所以可见
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:12:00:02  
          inet addr:172.18.0.2  Bcast:172.18.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:14 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:1116 (1.0 KiB)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

请求envoy的lisener

```bash
root@ubuntu-template:~# curl 172.18.0.2
This is a website server by a Go HTTP server.
```

查看日志

```bash
sidecar_1     | [2021-03-16 08:25:54.223][1][info][main] [source/server/server.cc:731] starting main dispatch loop
mainserver_1  | GET / # mainserver日志
```



## egress

### envoy配置

```diff
mkdir tcpproxy
cd tcpproxy
root@ubuntu-template:~# cat envoy.yaml 
static_resources:
  listeners:
+  - name: listener_0  # 监听外部地址
    address:
      socket_address: { address: 0.0.0.0, port_value: 80 }
    filter_chains:
    - filters:
+      - name: envoy.filters.network.tcp_proxy # TCP代理 
        typed_config:
+          "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
          stat_prefix: tcp
          cluster: test_cluster
  clusters:
  - name: test_cluster        # 集群惟一名称, 且未提供alt_stat_name时将会被用于统计信息中；
    connect_timeout: 0.25s    
    type: STATIC              # 解析集群生成endpoint时，使用的类型。STATIC, STRIC_DNS: 解析多个A记录, LOGICAL_DNS: 解析一个A记录，用于大型web场景; ORIGINAL_DEST: 不转换端口，直接同client请求的端口；和EDS: endpoint discovery server等.
    lb_policy: ROUND_ROBIN    # (optional) 负载均衡算法，支持ROUND_ROBIN, LEAST_REQUEST, RING_HASH, RANDOM, MAGLEV, CLUSTER_PROVIDED;
    load_assignment:          # 为STATIC, STRIC_DNS, LOGICAL_DNS类型的集群指定成员获取方式；EDS类型的集群使用eds_cluster_config字段配置;
      cluster_name: test_cluster # 集群 名， 与上面的name一致
      endpoints:                 # 端点列表. 可能在不同的dc, zone, 机柜, 不同服务器.
      - locality: {}             # (optional) 标识upstream的位置. 通常以region、zone、subzone...标识. 没有划分时，可以不定义
        lb_endpoints:            # 指定位置端点列表
+        - endpoint:             # 端点定义, Egress所以外部是多个服务
            address:             # 地址
              socket_address: { address: 192.168.0.7, port_value: 80 }
+        - endpoint:             
            address:             # 地址
              socket_address: { address: 192.168.0.9, port_value: 80 }
```

### docker-compose配置

```bash
root@ubuntu-template:~/tcpproxy# cat docker-compose.yaml 
version: '3'
services:
  sidecar:
    image: envoyproxy/envoy-alpine:v1.17-latest
    volumes:
    - ./envoy.yaml:/etc/envoy/envoy.yaml
    network_mode: "service:mainserver"    # 与mainserver同享network ns
    depends_on: # 先于当前服务运行的服务
    - mainserver
  mainserver:
    image: ikubernetes/mini-http-server:v0.3
    networks:
      envoymesh:
        aliases:
        - webserver
        - httpserver


networks:
  envoymesh: {}
```

启动

```bash
root@ubuntu-template:~/tcpproxy# docker-compose up
```

### 了解运行特性

在7和9启动一个apache

```bash
root@ubuntu-template:~# apt install apache2

```

进入envoy

```bash
root@ubuntu-template:~# docker exec -it tcpproxy_sidecar_1 sh
```

请求本地的80端口

```bash
/ # wget -O - -q 127.0.0.1:80
wget: error getting response: Connection reset by peer
/ # wget -O - -q 127.0.0.1:80
wget: error getting response: Connection reset by peer
# 安装apache之前



# 安装apache之后
/ # wget -O - -q 127.0.0.1:80
0.7
/ # wget -O - -q 127.0.0.1:80
0.9
/ # wget -O - -q 127.0.0.1:80
0.7
/ # wget -O - -q 127.0.0.1:80
0.7
/ # wget -O - -q 127.0.0.1:80
0.7
/ # wget -O - -q 127.0.0.1:80
0.9
/ # wget -O - -q 127.0.0.1:80
0.9
/ # wget -O - -q 127.0.0.1:80
0.7
```



# 基于4层过滤实现7层过滤(Http connection manager + http filter)

实现流量韧性、流量治理的框架

## egress

https://www.envoyproxy.io/docs/envoy/v1.17.1/api-v3/api#envoy-v3-api-reference

https://www.cnblogs.com/charlieroro/p/13644368.html typed_config

### envoy配置

```diff
root@ubuntu-template:~/http_egress# cat envoy.yaml 
static_resources:
  listeners:
  - name: listener_0  # 监听外部地址
    address:
      socket_address: { address: 127.0.0.1, port_value: 80 }
    filter_chains:
    - filters:
+      - name: envoy.http_connection_manager # 4层过滤的HTTP CONNECTION MANAGER
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager"
+          stat_prefix: egress_http    # 统计信息前缀, 通常egress_http或ingress_http。 方便监控
+          codec_type: AUTO            # HTTP 7层编解码器。AUTO自动；HTTP1: http 1.0;  HTTP2: http 2.0
          http_filters:
+          - name: envoy.filters.http.router
          route_config:               # 静态路由配置; 动态配置应该使用rds字段进行指定
            name: test_route      # 路由配置名
+            virtual_hosts:       # (多个) 虚拟主机列表, 用于构成路由表;     Downstream -> Host: xxxx -> domains -> match -> route/redirect/direct_response -> route(cluster, cluster_header, weighted_clusters)
            - name: web_service_1  # 虚拟主机逻辑名，用于统计信息，与路由无关
+              domains: ["*.ik8s.io","ik8s.io"]     # 当前虚拟主机匹配的域名列表，支持 * 通配; 匹配搜索次序为: 精确匹配: www.magedu.com、前缀匹配: *.magedu.com <= (www|bbs|test).magedu.com、后缀通配: www.magedu.* <= www.magedu.(cn|org|io|com)及完全通配: * <= 只要访问这个端口, 就算匹配；
              routes:       # (RDS动态生成) 路由列表, 按顺序搜索，第一个匹配的路由信息。 
              - name:  # (optional) 路由条目名称
+                match: { prefix: "/" }             # match字段可通过 prefix(前缀)、path(路径)、regex(正则表达式)三者"之一"来表示匹配模式 
                  #prefix: # 请求url前缀
+                route: { cluster: web_cluster_1 }              # 与match相关的请求由route/redirect/direct_response三个字段"之一"完成路由
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
+  - name: web_cluster_1        # 集群惟一名称, 且未提供alt_stat_name时将会被用于统计信息中；
    connect_timeout: 0.25s    
+    type: STRICT_DNS              # 解析集群生成endpoint时，使用的类型。STATIC, STRIC_DNS: 解析多个A记录, LOGICAL_DNS: 解析一个A记录，用于大型web场景; ORIGINAL_DEST: 不转换端口，直接同client请求的端口；和EDS: endpoint discovery server等.
    lb_policy: ROUND_ROBIN    # (optional) 负载均衡算法，支持ROUND_ROBIN, LEAST_REQUEST, RING_HASH, RANDOM, MAGLEV, CLUSTER_PROVIDED;
    load_assignment:          # 为STATIC, STRIC_DNS, LOGICAL_DNS类型的集群指定成员获取方式；EDS类型的集群使用eds_cluster_config字段配置;
+      cluster_name: web_cluster_1 # 集群 名， 与上面的name一致
      endpoints:                 # 端点列表. 可能在不同的dc, zone, 机柜, 不同服务器.
      - locality: {}             # (optional) 标识upstream的位置. 通常以region、zone、subzone...标识. 没有划分时，可以不定义
        lb_endpoints:            # 指定位置端点列表
        - endpoint:             # 端点定义, Egress所以外部是多个服务
            address:             # 地址
+              socket_address: { address: myservice, port_value: 8081 }

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



### docker-compose配置

```bash
root@ubuntu-template:~/http_egress# cat docker-compose.yaml 
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

### Dockerfile

```bash
root@harbor01:~/admininterface# cat Dockerfile-envoy
FROM envoyproxy/envoy-alpine:v1.17-latest

RUN apk update && apk --no-cache curl
```



启动

```bash
root@ubuntu-template:~/http_egress# docker-compose up


sidecar_1     | [2021-03-16 12:29:39.223][1][info][config] [source/server/configuration_impl.cc:91] loading 2 cluster(s)
sidecar_1     | [2021-03-16 12:29:39.227][1][info][config] [source/server/configuration_impl.cc:95] loading 1 listener(s)
sidecar_1     | [2021-03-16 12:29:39.240][1][info][upstream] [source/common/upstream/cluster_manager_impl.cc:191] cm init: all clusters initialized
```

### 了解运行特性

#### domains匹配

```bash
root@ubuntu-template:~/http_egress# docker exec -it httpegress_sidecar_1  sh
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:14:00:04  
          inet addr:172.20.0.4  Bcast:172.20.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:13 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:946 (946.0 B)  TX bytes:138 (138.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:136 errors:0 dropped:0 overruns:0 frame:0
          TX packets:136 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:8840 (8.6 KiB)  TX bytes:8840 (8.6 KiB)

/ # netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 127.0.0.1:80            0.0.0.0:*               LISTEN      
tcp        0      0 127.0.0.11:36255        0.0.0.0:*               LISTEN      


# 无结果
/ # curl 127.0.0.1:80
/ # 
```

原因, 2个domains都没有定义* ,所以一定匹配不上

```bash
              domains: ["*.ik8s.io","ik8s.io"]    
              domains: ["*.ik8scast.cn","ik8s.cn"]  
```



#### strict_dns

##### 解析多个A记录

以上没有加域名，现在加域名

```bash
/ # curl -H 'host:www.ik8s.io' 127.0.0.1:80
This is a website server by a Go HTTP server.
/ # curl -H 'host:www.ik8s.io' 127.0.0.1:80
This is a website server by a Go HTTP server.
/ # curl -H 'host:www.ik8s.io' 127.0.0.1:80/hostname
Hostname: 640f93afa0b9.
/ # curl -H 'host:www.ik8s.io' 127.0.0.1:80/hostname
Hostname: bc2f7de37bb9.

# 负载均衡
```

日志

```bash
webserver2_1  | GET /
webserver1_1  | GET /
webserver1_1  | GET /hostname
webserver2_1  | GET /hostname
```

原因, 请求`www.ik8s.io`, 并且路径默认`/`, 所以就代理至`myservice`

```diff
            virtual_hosts:       # (多个) 虚拟主机列表, 用于构成路由表;     Downstream -> Host: xxxx -> domains -> match -> route/redirect/direct_response -> route(cluster, cluster_header, weighted_clusters)
            - name: web_service_1  # 虚拟主机逻辑名，用于统计信息，与路由无关
+              domains: ["*.ik8s.io","ik8s.io"]     # 当前虚拟主机匹配的域名列表，支持 * 通配; 匹配搜索次序为: 精确匹配: www.magedu.com、前缀匹配: *.magedu.com <= (www|bbs|test).magedu.com、后缀通配: www.magedu.* <= www.magedu.(cn|org|io|com)及完全通配: * <= 只要访问这个端口, 就算匹配；
              routes:       # (RDS动态生成) 路由列表, 按顺序搜索，第一个匹配的路由信息。 
              - name:  # (optional) 路由条目名称
+                match: { prefix: "/" }             # match字段可通过 prefix(前缀)、path(路径)、regex(正则表达式)三者"之一"来表示匹配模式 
                  #prefix: # 请求url前缀
+                route: { cluster: web_cluster_1 }              # 与match相关的请求由route/redirect/direct_response三个字段"之一"完成路由
                  #cluster: # 目标集群  # route定义的目标必须是cluster(目标集群), cluster_header(根据请求标头的cluster_header值确定目标集群) 或weighted_clusters(路由目标有多个集群，每个集群有一定权重) 其中"之一"


 clusters:
  - name: web_cluster_1        # 集群惟一名称, 且未提供alt_stat_name时将会被用于统计信息中；
    connect_timeout: 0.25s    
+    type: STRICT_DNS              # 解析集群生成endpoint时，使用的类型。STATIC, STRIC_DNS: 解析多个A记录, LOGICAL_DNS: 解析一个A记录，用于大型web场景; ORIGINAL_DEST: 不转换端口，直接同client请求的端口；和EDS: endpoint discovery server等.
    lb_policy: ROUND_ROBIN    # (optional) 负载均衡算法，支持ROUND_ROBIN, LEAST_REQUEST, RING_HASH, RANDOM, MAGLEV, CLUSTER_PROVIDED;
    load_assignment:          # 为STATIC, STRIC_DNS, LOGICAL_DNS类型的集群指定成员获取方式；EDS类型的集群使用eds_cluster_config字段配置;
      cluster_name: web_cluster_1 # 集群 名， 与上面的name一致
      endpoints:                 # 端点列表. 可能在不同的dc, zone, 机柜, 不同服务器.
      - locality: {}             # (optional) 标识upstream的位置. 通常以region、zone、subzone...标识. 没有划分时，可以不定义
        lb_endpoints:            # 指定位置端点列表
        - endpoint:             # 端点定义, Egress所以外部是多个服务
            address:             # 地址
+              socket_address: { address: myservice, port_value: 8081 }
```

查看compose的解析: myservice对应2个记录

```diff
  webserver1:
    image: ikubernetes/mini-http-server:v0.3
    networks:
      envoymesh:
        aliases:
        - webserver1
+        - myservice
    expose:
    - "8081"

  webserver2:
    image: ikubernetes/mini-http-server:v0.3
    networks:
      envoymesh:
        aliases:
        - webserver2
+        - myservice

```



虽然此集群只有一个endpoint, 但是定义为STRICT_DNS，表示只要这个address的域名解析多少A记录，将会自动生成多少记录的endpoint

LOGICAL_DNS 无论这个address的域名解析多少A记录，将会自动生成1个记录的endpoint



所以我们看到多个结果，并且为`ROUND_ROBIN`

##### 解析1个A记录

```bash
/ # curl -H 'host:www.ik8scast.cn' 127.0.0.1:80/hostname
Hostname: 640f93afa0b9.
/ # curl -H 'host:www.ik8scast.cn' 127.0.0.1:80/hostname
Hostname: 640f93afa0b9.
/ # curl -H 'host:www.ik8scast.cn' 127.0.0.1:80/hostname
Hostname: 640f93afa0b9.
/ # curl -H 'host:www.ik8scast.cn' 127.0.0.1:80/hostname
Hostname: 640f93afa0b9.
```

日志

```bash
webserver1_1  | GET /hostname
webserver1_1  | GET /hostname
webserver1_1  | GET /hostname
```

原因: 解析域 -> / -> cluster 2 -> webserver1

```diff
            - name: web_service_2  # 虚拟主机逻辑名，用于统计信息，与路由无关
+              domains: ["*.ik8scast.cn","ik8s.cn"]     # 当前虚拟主机匹配的域名列表，支持 * 通配; 匹配搜索次序为: 精确匹配: www.magedu.com、前缀匹配: *.magedu.com <= (www|bbs|test).magedu.com、后缀通配: www.magedu.* <= www.magedu.(cn|org|io|com)及完全通配: * <= 只要访问这个端口, 就算匹配；
              routes:       # (RDS动态生成) 路由列表, 按顺序搜索，第一个匹配的路由信息。 
              - name:  # (optional) 路由条目名称
+                match: { prefix: "/" }             # match字段可通过 prefix(前缀)、path(路径)、regex(正则表达式)三者"之一"来表示匹配模式 
                  #prefix: # 请求url前缀
+                route: { cluster: web_cluster_2 }              # 与match相关的请求由route/redirect/direct_response三个字段"之一"完成路由
                  #cluster: # 目标集群  # route定义的目标必须是cluster(目标集群), cluster_header(根据请求标头的cluster_header值确定目标集群) 或weighted_clusters(路由目标有多个集群，每个集群有一定权重) 其中"之一"



  - name: web_cluster_2        # 集群惟一名称, 且未提供alt_stat_name时将会被用于统计信息中；
    connect_timeout: 0.25s    
+    type: STRICT_DNS              # 解析集群生成endpoint时，使用的类型。STATIC, STRIC_DNS: 解析多个A记录, LOGICAL_DNS: 解析一个A记录，用于大型web场景; ORIGINAL_DEST: 不转换端口，直接同client请求的端口；和EDS: endpoint discovery server等.
    lb_policy: ROUND_ROBIN    # (optional) 负载均衡算法，支持ROUND_ROBIN, LEAST_REQUEST, RING_HASH, RANDOM, MAGLEV, CLUSTER_PROVIDED;
    load_assignment:          # 为STATIC, STRIC_DNS, LOGICAL_DNS类型的集群指定成员获取方式；EDS类型的集群使用eds_cluster_config字段配置;
      cluster_name: web_cluster_2 # 集群 名， 与上面的name一致
      endpoints:                 # 端点列表. 可能在不同的dc, zone, 机柜, 不同服务器.
      - locality: {}             # (optional) 标识upstream的位置. 通常以region、zone、subzone...标识. 没有划分时，可以不定义
        lb_endpoints:            # 指定位置端点列表
        - endpoint:             # 端点定义, Egress所以外部是多个服务
            address:             # 地址
+              socket_address: { address: webserver1, port_value: 8081 }
```

而docker-compose只定义了一个解析

```diff
  webserver1:
    image: ikubernetes/mini-http-server:v0.3
    networks:
      envoymesh:
        aliases:
+        - webserver1
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

```

所以结果就1个



### 实现redirect

```diff
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
+                match: { prefix: "/" }             # match字段可通过 prefix(前缀)、path(路径)、regex(正则表达式)三者"之一"来表示匹配模式 
-                route: { cluster: web_cluster_2 }              # 与match相关的请求由route/redirect/direct_response三个字段"之一"完成路由
-                  #cluster: # 目标集群  # route定义的目标必须是cluster(目标集群), cluster_header(根据请求标头的cluster_header值确定目标集群) 或weighted_clusters(路由目标有多个集群，每个集群有一定权重) 其中"之一"
+                redirect:                   # 与match相关的请求由route/redirect/direct_response三个字段"之一"完成路由
+                  host_redirect: "www.ik8s.io"

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

启动

```bash
sidecar_1     | [2021-03-16 12:47:12.754][1][info][config] [source/server/configuration_impl.cc:91] loading 2 cluster(s)
sidecar_1     | [2021-03-16 12:47:12.758][1][info][config] [source/server/configuration_impl.cc:95] loading 1 listener(s)
manager_impl.cc:191] cm init: all clusters initialized
```



## ingress

### ingress配置

```diff
static_resources:
  listeners:
  - name: listener_0  # 监听外部地址
    address:
+      socket_address: { address: 0.0.0.0, port_value: 80 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager # 4层过滤的HTTP CONNECTION MANAGER
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager"
+          stat_prefix: ingress_http    # 统计信息前缀, 通常egress_http或ingress_http。 方便监控
          codec_type: AUTO            # HTTP 7层编解码器。AUTO自动；HTTP1: http 1.0;  HTTP2: http 2.0
          http_filters:
          - name: envoy.filters.http.router
          route_config:               # 静态路由配置; 动态配置应该使用rds字段进行指定
+            name: local_route      # 路由配置名
            virtual_hosts:       # (多个) 虚拟主机列表, 用于构成路由表;     Downstream -> Host: xxxx -> domains -> match -> route/redirect/direct_response -> route(cluster, cluster_header, weighted_clusters)
            - name: local_service  # 虚拟主机逻辑名，用于统计信息，与路由无关
+              domains: ["*"]     # 当前虚拟主机匹配的域名列表，支持 * 通配; 匹配搜索次序为: 精确匹配: www.magedu.com、前缀匹配: *.magedu.com <= (www|bbs|test).magedu.com、后缀通配: www.magedu.* <= www.magedu.(cn|org|io|com)及完全通配: * <= 只要访问这个端口, 就算匹配；
              routes:       # (RDS动态生成) 路由列表, 按顺序搜索，第一个匹配的路由信息。 
              - name:  # (optional) 路由条目名称
                match: { prefix: "/" }             # match字段可通过 prefix(前缀)、path(路径)、regex(正则表达式)三者"之一"来表示匹配模式 
                  #prefix: # 请求url前缀
+                route: { cluster: local_service }              # 与match相关的请求由route/redirect/direct_response三个字段"之一"完成路由
                  #cluster: # 目标集群  # route定义的目标必须是cluster(目标集群), cluster_header(根据请求标头的cluster_header值确定目标集群) 或weighted_clusters(路由目标有多个集群，每个集群有一定权重) 其中"之一"



  clusters:
+  - name: local_service        # 集群惟一名称, 且未提供alt_stat_name时将会被用于统计信息中；
    connect_timeout: 0.25s    
+    type: STATIC              # 解析集群生成endpoint时，使用的类型。STATIC, STRIC_DNS: 解析多个A记录, LOGICAL_DNS: 解析一个A记录，用于大型web场景; ORIGINAL_DEST: 不转换端口，直接同client请求的端口；和EDS: endpoint discovery server等.
    lb_policy: ROUND_ROBIN    # (optional) 负载均衡算法，支持ROUND_ROBIN, LEAST_REQUEST, RING_HASH, RANDOM, MAGLEV, CLUSTER_PROVIDED;
    load_assignment:          # 为STATIC, STRIC_DNS, LOGICAL_DNS类型的集群指定成员获取方式；EDS类型的集群使用eds_cluster_config字段配置;
+      cluster_name: local_service # 集群 名， 与上面的name一致
      endpoints:                 # 端点列表. 可能在不同的dc, zone, 机柜, 不同服务器.
      - locality: {}             # (optional) 标识upstream的位置. 通常以region、zone、subzone...标识. 没有划分时，可以不定义
        lb_endpoints:            # 指定位置端点列表
        - endpoint:             # 端点定义, Egress所以外部是多个服务
            address:             # 地址
+              socket_address: { address: 127.0.0.1, port_value: 8081 }
```

> 接收外部请求，所以0.0.0.0
>
> 通用状态接口
>
> 路由名称
>
> domains * 只要请求80端口，就转发
>
> route 给本地
>
> address是127, 原因sidecar相当于在同一个network ns

### docker-compose配置

```bash
"docker-compose.yaml" 22L, 457C written
root@ubuntu-template:~/http_ingress# cat docker-compose.yaml
version: '3'
services:
  sidecar:
    image: envoyproxy/envoy-alpine:v1.17-latest
    volumes:
    - ./envoy.yaml:/etc/envoy/envoy.yaml
    network_mode: "service:mainserver"    # 与mainserver同享network ns
    depends_on: # 先于当前服务运行的服务
    - mainserver
  mainserver:
    image: ikubernetes/mini-http-server:v0.3
    networks:
      envoymesh:
        aliases:
        - webserver
        - httpserver


networks:
  envoymesh: {}
```

启动

```bash
root@ubuntu-template:~/http_ingress# docker-compose up

sidecar_1     | [2021-03-16 12:58:54.453][1][info][config] [source/server/configuration_impl.cc:91] loading 1 cluster(s)
sidecar_1     | [2021-03-16 12:58:54.455][1][info][config] [source/server/configuration_impl.cc:95] loading 1 listener(s)
mon/upstream/cluster_manager_impl.cc:191] cm init: all clusters initialized
```

### 了解运行特性

```bash
root@ubuntu-template:~/http_egress# docker exec -it httpingress_sidecar_1 sh
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:15:00:02  
          inet addr:172.21.0.2  Bcast:172.21.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:15 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:1242 (1.2 KiB)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ # netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 127.0.0.11:40495        0.0.0.0:*               LISTEN      
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      
tcp        0      0 :::8081                 :::*                    LISTEN      
/ # root@ubuntu-template:~/http_egress# 
root@ubuntu-template:~/http_egress# 
root@ubuntu-template:~/http_egress# curl 172.21.0.2
This is a website server by a Go HTTP server.
root@ubuntu-template:~/http_egress# curl 172.21.0.2/hostname
Hostname: 16ab66d7f850.
root@ubuntu-template:~/http_egress# curl 172.21.0.2/hostname
Hostname: 16ab66d7f850.
root@ubuntu-template:~/http_egress# curl 172.21.0.2/hostname
Hostname: 16ab66d7f850.
```

```bash
root@ubuntu-template:~/http_egress# docker ps
CONTAINER ID   IMAGE                                  COMMAND                  CREATED              STATUS              PORTS      NAMES
16ab66d7f850   ikubernetes/mini-http-server:v0.3      "/bin/httpserver"        About a minute ago   Up About a minute   8081/tcp   httpingress_mainserver_1
#主机名正好是mainserver的CONTAINERID
```





