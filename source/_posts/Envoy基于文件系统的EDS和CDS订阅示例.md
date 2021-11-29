---
title: Envoy基于文件系统的EDS和CDS订阅示例
date: 2021-03-18 03:56:45
tags:
- istio
---



# 前言

之前使用static_resource顶级字段配置bootstrap, 并不是envoy软件开发的初衷

```bash
static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 80
....
```

>  envoy从配置源获取配置，配置源数据变化，envoy通过周期性的获取变动或 以**订阅**方式数据源推送变动到envoy。
>
> 不同的配置项，均可以以动态方式从XDS获取。
>
> 面向client: 侦听器（4层过滤、7层过滤。），上游：集群。
>
> 集群可以动态加载(**CDS**)， 其内部的端点可以动态加载(**EDS**)，其内部的路由也可以动态获取(**RDS**)，其内部的侦听器可以动态加载(**LDS**), 7层的证书动态加载(**SDS**)，运行时配置发现(**RDS**), 运行时健康状态(**HDS**) 。
>
> 每一种DS均可以由一个服务器扮演，但是这个动态获取非常依赖服务可靠性（每一个均需要冗余 ）。所以将所有DS聚合在一个服务器(**ADS**)，方便冗余，但是envoy作为XDS协议的客户端来请求不同的DS的配置，是由这个服务器的不同逻辑在处理。
>
> > 这些DS可以由envoy官方提供的python, java, .. SDK来编写。

![image-20210830221651436](http://myapp.img.mykernel.cn/image-20210830221651436.png)

> XDS典型实现：Istio Piolt

Envoy XDS

- [x] filesystem inotify
- [x] rest-json api
- [x] grpc api

> 无论是rest, grpc 均会使用到XDS 协议的API。

![image-20210830211455267](http://myapp.img.mykernel.cn/image-20210830211455267.png)

> XDS典型实现：istio 核心组件之一，就是为envoy提供核心配置 ADS。paliot

<!--more-->
# Envoy动态配置及配置源

envoy 基于XDS实现了动态配置，所以也称XDS为 数据平面API（Data plane api)

| 动态配置源    | 工作模式                   | 实用性                                                       | 订阅模式     | 格式                                                         | 集群规模 | 获取配置                     |
| ------------- | -------------------------- | ------------------------------------------------------------ | ------------ | ------------------------------------------------------------ | -------- | ---------------------------- |
| 文件系统      | 本地、网络文件系统         | 实用性低、动态性低                                           | 全量订阅     | inotify监听，文件内容本身为discovery response proto.  均可以使用json, yaml, proto文本。 | 小规模   | inotify                      |
| rest-json api | 独立套接字, **http/https** | envoy发送配置请求信息（discovery request）。server以json响应。 | 全量订阅     |                                                              | 大规模   | 轮询获取                     |
| grpc api      | 独立套接字, **http2**      | envoy发送配置请求信息。server响应。                          | 全量订阅     | 双向grpc流，就是始终在线的会话，每个流上面有独立的版本。     | 大规模   | **订阅**，双向可传输数据流。 |
| delta grpc    | 同grpc api                 | 同grpc api                                                   | **增量订阅** | 同grpc api                                                   | 大规模   | **订阅**，双向可传输数据流。 |

> ADS仅支持grpc
>
> **动态配置源**，需要存储数据，示例中均为内存中存储。重启就会失效，生产中使用resdis、consolu集群、etcd集群、... 强一致服务。

# xds 协议

https://github.com/envoyproxy/envoy/blob/main/api/envoy/api/v2/endpoint.proto

新版本没有写了，回到旧版本

https://github.com/envoyproxy/envoy/blob/v1.10.0/api/envoy/api/v2/discovery.proto

```bash
syntax = "proto3";
package envoy.api.v2;
option java_outer_classname = "DiscoveryProto";
option java_multiple_files = true;
option java_package = "io.envoyproxy.envoy.api.v2";
option go_package = "v2";
import "envoy/api/v2/core/base.proto";
import "google/protobuf/any.proto";
import "google/rpc/status.proto";
import "gogoproto/gogo.proto";
option (gogoproto.equal_all) = true;
option (gogoproto.stable_marshaler_all) = true;

# 资源发现的请求报文
message DiscoveryRequest {
  # 初始请求version为空，没有意义。之后的请求才有意义
  string version_info = 1;
  # 哪个节点发来的请求
  core.Node node = 2;
  # 请求的资源名
  repeated string resource_names = 3;
  # 请求的资源类型：EDS, CDS, RDS, LDS, SDS, HDS,...
  string type_url = 4;
  # 加密标识
  string response_nonce = 5;
  # client拒绝的原因
  google.rpc.Status error_detail = 6;
}
# 资源发现的响应报文
message DiscoveryResponse {
  string version_info = 1;
  repeated google.protobuf.Any resources = 2 [(gogoproto.nullable) = false];
  bool canary = 3;
  string type_url = 4;
  string nonce = 5;
  core.ControlPlane control_plane = 6;
}
# 增量格式的请求报文，用于请求单个资源
message DeltaDiscoveryRequest {
  core.Node node = 1;
  string type_url = 2;
  repeated string resource_names_subscribe = 3;
  repeated string resource_names_unsubscribe = 4;
  map<string, string> initial_resource_versions = 5;
  string response_nonce = 6;
  google.rpc.Status error_detail = 7;
}
# 增量格式的响应报文
message DeltaDiscoveryResponse {
  string system_version_info = 1;
  repeated Resource resources = 2 [(gogoproto.nullable) = false];
  repeated string removed_resources = 6;
  string nonce = 5;
}
# 跟踪的资源
message Resource {
  string name = 3;
  repeated string aliases = 4;
  string version = 1;
  google.protobuf.Any resource = 2;
}
```

更详细：http://skyao.io/

## type.urls

一次grpc请求和grpc响应区别不同的资源，使用typeurl： typeurl 采用`type.googleapis.com/<resource type>`形式, 以下为 resource type简写及url使用的格式：

LDS: envoy.api.v2.Listener

RDS: envoy.api.v2.RouteConfiguration

VHDS: envoy.api.v2.Vhds

CDS: envoy.api.v2.Cluster

EDS: envoy.api.v2.ClusterLoadAssignment

SDS: envoy.api.v2.Auth.Secret

RTDS: envoy.service.discovery.v2.Runtime

> EDS对应于`type.googleapis.com/envoy.api.v2.ClusterLoadAssignment`



## Discovery Request和Response报文示例

### Discovery Request

每个流都以envoy的DiscoveryRequset开始，指定订阅资源列表、与订阅的资源对应的类型URL、节点标识符和空的version_info等信息，示例EDS

```bash
version_info:
node {id: envoy} # 请求所在节点
resource_names:
- foo
- bar
type_url: type.googleapis.com/envoy.api.v2.ClusterLoadAssignment #EDS类型
response_nonce:
```

### Discovery Response 

管理服务器可能立即回复，或当请求资源可用时进行回复

```bash
version_info: X # 多次请求时，仅显示最一个应用成功的版本号
node {id: envoy} 
resources:
- foo 配置信息...
- bar 配置信息...
type_url: type.googleapis.com/envoy.api.v2.ClusterLoadAssignment #EDS类型
nonce: A # 加密字符
```

> 文件系统发现，应该以response格式定义

### ack/nack and verisoning

versioning机制为envoy和ms之间提供了共享当前应用配置的概念，以及通过ack/nack来进行配置更新的机制 ；

- [ ] ms应该只身envoy client发送上次discoveryResponse更新过的资源；
- [ ] envoy则会根据自身能否接受discoveryResponse的情况立即回复包含ACK/NACK的discoveryReqeust请求
- [ ] 另外，同一个流中，Envoy可能会连接发送多个discoveryRquest请求， MS响应给定资源最新请求即可。

discoveryRequest中的resource_names仅用作资源提示信息

- [ ] 诸如cluster和listener一类的资源类型需要使用**空的resource_names**, 因为envoy需要获取MS对应节点标识的所有**CDS/LDS**资源定义，这意味着本地已经 配置但响应报文中不存在的资源将会被删除。
- [ ] 对于**RDS, EDS一类资源**，Envoy明确枚举resource_names；但是MS不需要为每个请求的资源进行响应，而且还可能提供额外未请求的资源，**resource_names只是提示**。

### Discovery Request和Response 工作过程、

每个envoy流以discoveryRequest开始，包括了列表订阅的资源、订阅资源对应的类型URL，节点标识符和空的version_info。

MS发送响应

Envoy处理响应后，通过stream长连接发送新请求，包含**成功的最后一个版号和MS的nonce。**

成功，version_info值last successful version；

不成功，envoy请求发送error_detail(错误详情)和last version;

不成功，MS会提供新的版本号，Envoy可能在新的version上完成；

### REST-JSON轮询订阅

通过rest 端点进行的同步 （长）轮询也可用于XDS单例API

上面的消息顺序类型的，除了没有长连接的stream

ADS不适用于rest-json轮询

轮询周期小，避免发送DiscoveryResponse, 除非发生对底层资源更新

# filesystem eds or filesystem cds

## 静态cluster + 基于filesystem动态EDS

```diff
  clusters:
  - name: web_cluster_1        # 集群惟一名称, 且未提供alt_stat_name时将会被用于统计信息中；
    connect_timeout: 0.25s    
+    type: EDS              # 解析集群生成endpoint时，使用的类型。STATIC, STRIC_DNS: 解析多个A记录, LOGICAL_DNS: 解析一个A记录，用于大型web场景; ORIGINAL_DEST: 不转换端口，直接同client请求的端口；和EDS: endpoint discovery server等.
    lb_policy: ROUND_ROBIN    # (optional) 负载均衡算法，支持ROUND_ROBIN, LEAST_REQUEST, RING_HASH, RANDOM, MAGLEV, CLUSTER_PROVIDED;
+    eds_cluster_config: 
+      service_name: web_cluster_1       # 同cluster.name
+      eds_config:                       # filesystem加载eds配置
+        path: "/etc/envoy/eds.conf"
```

type为eds，则静态的CDS中 的动态获取Endpoints.

eds_config: 动态获取: [Envoy动态配置及配置源](#Envoy动态配置及配置源) path, api_config: type指定rest-json或grpc

- path格式应该基于json, discovery response格式定义。envoy可以基于Inotify监控此path文件变动。envoy在docker中，inotify监控不会实时加载。需要人为添加步骤，直接把编辑好的覆盖到此配置文件，而不是就地修改。
- api_config
  - rest-json
  - grpc

> admin 接口 本地，安全
>
> egress, 所以Listener在127，所以制作镜像需要有curl命令
>
> 请求任意域的/到web_cluster_1集群 

### 获取基于文件系统的eds配置

```bash
git clone https://github.com/slcnx/servicemesh_in_practise.git
```

以下就是基于eds文件系统发现endpoints

```bash
[root@alyy servicemesh_in_practise-master]# tree eds-filesystem/
eds-filesystem/
|-- docker-compose.yaml
|-- Dockerfile-envoy  
|-- eds.conf          # v0
|-- eds.conf.v1       # v1
|-- eds.conf.v2       # v2
`-- envoy.yaml
```

> 新版本的type:  `type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment`
>
> dockerfile可以添加以下命令，完成安装更多工具
>
> ```dockerfile
> RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
> RUN apk update && apk --no-cache add curl tzdata iotop gcc gcc libgcc libc-dev libcurl libc-utils pcre-dev zlib-dev libnfs make pcre pcre2 zip unzip net-tools pstree wget libevent libevent-dev iproute2 tcpdump vim
> ```
>
> `envoy.yaml` 部分解释
>
> ```yaml
> static_resources:
>   listeners:
>   - name: listener_0  # 监听外部地址
>     address:
>       socket_address: { address: 127.0.0.1, port_value: 80 }
>     filter_chains:
>     - filters:
>       - name: envoy.http_connection_manager # 4层过滤的HTTP CONNECTION MANAGER
>         typed_config:
>           "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager"
>           stat_prefix: egress_http    # 统计信息前缀, 通常egress_http或ingress_http。 方便监控
>           codec_type: AUTO            # HTTP 7层编解码器。AUTO自动；HTTP1: http 1.0;  HTTP2: http 2.0
>           http_filters:
>           - name: envoy.filters.http.router
>           route_config:               # 静态路由配置; 动态配置应该使用rds字段进行指定
>             name: test_route      # 路由配置名
>             virtual_hosts:       # (多个) 虚拟主机列表, 用于构成路由表;     Downstream -> Host: xxxx -> domains -> match -> route/redirect/direct_response -> route(cluster, cluster_header, weighted_clusters)
>             - name: web_service_1  # 虚拟主机逻辑名，用于统计信息，与路由无关
>               domains: ["*"]     # 当前虚拟主机匹配的域名列表，支持 * 通配; 匹配搜索次序为: 精确匹配: www.magedu.com、前缀匹配: *.magedu.com <= (www|bbs|test).magedu.com、后缀通配: www.magedu.* <= www.magedu.(cn|org|io|com)及完全通配: * <= 只要访问这个端口, 就算匹配；
>               routes:       # (RDS动态生成) 路由列表, 按顺序搜索，第一个匹配的路由信息。 
>               - name:  # (optional) 路由条目名称
>                 match: { prefix: "/" }             # match字段可通过 prefix(前缀)、path(路径)、regex(正则表达式)三者"之一"来表示匹配模式 
>                   #prefix: # 请求url前缀
>                 route: { cluster: web_cluster_1 }              # 与match相关的请求由route/redirect/direct_response三个字段"之一"完成路由
>                   #cluster: # 目标集群  # route定义的目标必须是cluster(目标集群), cluster_header(根据请求标头的cluster_header值确定目标集群) 或weighted_clusters(路由目标有多个集群，每个集群有一定权重) 其中"之一"
> ```



`docker-compose.yaml`

```yaml
version: '3.3'
services:
  envoy:
    build:
      context: .
      dockerfile: Dockerfile-envoy
    networks:
      envoymesh:
        aliases:
        - envoy
    expose:
    - "80"
    ports:
    - "8080:80"

  webserver1:
    image: ikubernetes/mini-http-server:v0.3
    networks:
      envoymesh:
        aliases:
        - webserver1
    expose:
    - "8081"
    depends_on:
    - "envoy"

  webserver2:
    image: ikubernetes/mini-http-server:v0.3
    networks:
      envoymesh:
        aliases:
        - webserver2
    expose:
    - "8081"
    depends_on:
    - "envoy"

networks:
  envoymesh: {}
```

> 最终server1, server2不知道地址，所以envoy不知道后端地址，所以使用version0
>
> 一旦webserver1启动后，我们可以拿到地址。放在eds.conf.v1中。覆盖到eds.conf
>
> 一旦webserver2启动后，我们可以拿到地址。放在eds.conf.v2中。覆盖到eds.conf
>
> >  将来地址来自k8s的服务

### 启动

```bash
[root@alyy eds-filesystem]# docker-compose up
admin address: 0.0.0.0:9901
loading 1 cluster(s)
loading 1 listener(s)
webserver1_1  | Listening on http://0.0.0.0:8081
webserver2_1  | Listening on http://0.0.0.0:8081
```

查看envoy，可以发现ip为`172.18.0.2`, 端口为`9901`, `80`

```bash
/ # [root@alyy eds-filesystem]# docker exec -it eds-filesystem_envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:12:00:02  
          inet addr:172.18.0.2  Bcast:172.18.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:3 errors:0 dropped:0 overruns:0 frame:0
          TX packets:5 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:142 (142.0 B)  TX bytes:258 (258.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

[root@alyy eds-filesystem]# docker exec -it eds-filesystem_envoy_1 netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 0.0.0.0:9901            0.0.0.0:*               LISTEN      
tcp        0      0 127.0.0.11:36623        0.0.0.0:*               LISTEN      
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN  
```

现在直接在宿主机请求地址

```bash
[root@alyy eds-filesystem]# curl 172.18.0.2; echo
no healthy upstream # 因为endpoint为空，所以没有健康的上游服务器
```

通过管理端口查看集群定义, 存在`webcluster1`集群，但是没有endpoints

```bash
[root@alyy eds-filesystem]# curl 172.18.0.2:9901/clusters; echo
webcluster1::default_priority::max_connections::1024
webcluster1::default_priority::max_pending_requests::1024
webcluster1::default_priority::max_requests::1024
webcluster1::default_priority::max_retries::3
webcluster1::high_priority::max_connections::1024
webcluster1::high_priority::max_pending_requests::1024
webcluster1::high_priority::max_requests::1024
webcluster1::high_priority::max_retries::3
webcluster1::added_via_api::false
```

通过查看listener, 可以看出监听在`80`

```bash
[root@alyy eds-filesystem]# curl 172.18.0.2:9901/listeners
listener_http::0.0.0.0:80
```

查看证书

```bash
[root@alyy eds-filesystem]# curl 172.18.0.2:9901/certs
{
 "certificates": []
}
```

查看所有配置

```bash
[root@alyy eds-filesystem]# curl -s 172.18.0.2:9901/config_dump  | yq -y

```

查看内存

```bash
[root@alyy eds-filesystem]# curl -s 172.18.0.2:9901/memory
{
 "allocated": "3986880",
 "heap_size": "5242880",
 "pageheap_unmapped": "0",
 "pageheap_free": "606208",
 "total_thread_cache": "352696"
}
```

### 动态配置endpoint1

动态配置endpoint 1, 将webserver1地址和商品填入到v1中，覆盖到eds.conf

```bash
docker inspect eds-filesystem_webserver1_1 
  "IPAddress": "172.18.0.3",

```

编辑v1, 将以下高亮地方做以上IP的替换。

```diff
                           "socket_address": {
+                              "address": "172.18.0.3",
+                              "port_value": 8081
```

覆盖到conf

```bash
/etc/envoy # mv eds.conf.v1 eds.conf
```

验证envoy已经动态加载endpoints

```bash
[root@alyy eds-filesystem]# curl 172.18.0.2:9901/clusters; echo
webcluster1::default_priority::max_connections::1024
webcluster1::default_priority::max_pending_requests::1024
webcluster1::default_priority::max_requests::1024
webcluster1::default_priority::max_retries::3
webcluster1::high_priority::max_connections::1024
webcluster1::high_priority::max_pending_requests::1024
webcluster1::high_priority::max_requests::1024
webcluster1::high_priority::max_retries::3
webcluster1::added_via_api::false
webcluster1::172.18.0.3:8081::cx_active::0
webcluster1::172.18.0.3:8081::cx_connect_fail::0
webcluster1::172.18.0.3:8081::cx_total::0
webcluster1::172.18.0.3:8081::rq_active::0
webcluster1::172.18.0.3:8081::rq_error::0
webcluster1::172.18.0.3:8081::rq_success::0
webcluster1::172.18.0.3:8081::rq_timeout::0
webcluster1::172.18.0.3:8081::rq_total::0
webcluster1::172.18.0.3:8081::hostname::
webcluster1::172.18.0.3:8081::health_flags::healthy
webcluster1::172.18.0.3:8081::weight::1
webcluster1::172.18.0.3:8081::region::
webcluster1::172.18.0.3:8081::zone::
webcluster1::172.18.0.3:8081::sub_zone::
webcluster1::172.18.0.3:8081::canary::false
webcluster1::172.18.0.3:8081::priority::0
webcluster1::172.18.0.3:8081::success_rate::-1
webcluster1::172.18.0.3:8081::local_origin_success_rate::-1
```

请求envoy的hostname， 会egress转发到172.18.0.3上。所以请求的结果的hostname正好是webserver1的容器id.

```bash
[root@alyy eds-filesystem]# curl 172.18.0.2/hostname
Hostname: 169ab1431d01.
```

对应容器id，正好是webserver1

```bash
[root@alyy eds-filesystem]# docker ps 
CONTAINER ID        IMAGE                                                                         COMMAND                  CREATED             STATUS              PORTS                                            NAMES
169ab1431d01        ikubernetes/mini-http-server:v0.3                                             "/bin/httpserver"        24 minutes ago      Up 24 minutes       8081/tcp                                         eds-filesystem_webserver1_1
```



### 动态配置endpoint2

如上方式，添加webserver2到envoy, 并检验结果，集群会有2个端点，并且hostname结果是轮询的。

```bash
[root@alyy eds-filesystem]# curl 172.18.0.2/hostname
Hostname: 169ab1431d01.
[root@alyy eds-filesystem]# curl 172.18.0.2/hostname
Hostname: 7651055d3801.
[root@alyy eds-filesystem]# curl 172.18.0.2/hostname
Hostname: 169ab1431d01.
[root@alyy eds-filesystem]# 
```



## 基于filesystem动态CDS 

cluster提供：

- static_resources.cluster
- dynamic_resources.cds_config
  - path
  - grpc
  - rest-json

endpoint提供:

- static
- eds
  - api_config
    - rest-json
    - grpc
  - path
- strict_dns: 域名
- logical_dns

组合：

- 动态cds + 静态endpoints.

- 动态cds + 动态eds

```diff
+dynamic_resources:
+  cds_config:
+    path: /etc/envoy/cds.conf
```

### filesystem cds + strict_dns endpoints

```bash
git clone https://github.com/slcnx/servicemesh_in_practise.git
[root@alyy servicemesh_in_practise-master]# tree cds-eds-filesystem/
cds-eds-filesystem/
|-- cds.conf        # filesystem cds + strict_dns eds， discovery response 格式
|-- cds.conf.v2     # filesystem cds + filesystem eds， discovery response 格式
|-- docker-compose.yaml
|-- Dockerfile-envoy
|-- eds.conf        # discovery response 格式
`-- envoy.yaml
```

> `envoy.yaml`
>
> ```yaml
> static_resources:
>   listeners:
>   - name: listener_0  # 监听外部地址
>     address:
>       socket_address: { address: 127.0.0.1, port_value: 80 }
>     filter_chains:
>     - filters:
>       - name: envoy.http_connection_manager # 4层过滤的HTTP CONNECTION MANAGER
>         config:
>           stat_prefix: egress_http    # 统计信息前缀, 通常egress_http或ingress_http。 方便监控
>           codec_type: AUTO            # HTTP 7层编解码器。AUTO自动；HTTP1: http 1.0;  HTTP2: http 2.0
>           http_filters:
>           - name: envoy.router
>           route_config:               # 静态路由配置; 动态配置应该使用rds字段进行指定
>             name: test_route      # 路由配置名
>             virtual_hosts:       # (多个) 虚拟主机列表, 用于构成路由表;     Downstream -> Host: xxxx -> domains -> match -> route/redirect/direct_response -> route(cluster, cluster_header, weighted_clusters)
>             - name: web_service_1  # 虚拟主机逻辑名，用于统计信息，与路由无关
> +              domains: ["*"]     # 当前虚拟主机匹配的域名列表，支持 * 通配; 匹配搜索次序为: 精确匹配: www.magedu.com、前缀匹配: *.magedu.com <= (www|bbs|test).magedu.com、后缀通配: www.magedu.* <= www.magedu.(cn|org|io|com)及完全通配: * <= 只要访问这个端口, 就算匹配；
>               routes:       # (RDS动态生成) 路由列表, 按顺序搜索，第一个匹配的路由信息。 
>               -  match: { prefix: "/" }             # match字段可通过 prefix(前缀)、path(路径)、regex(正则表达式)三者"之一"来表示匹配模式 
>                   #prefix: # 请求url前缀
>                 route: { cluster: web_cluster_1 }              # 与match相关的请求由route/redirect/direct_response三个字段"之一"完成路由
>                   #cluster: # 目标集群  # route定义的目标必须是cluster(目标集群), cluster_header(根据请求标头的cluster_header值确定目标集群) 或weighted_clusters(路由目标有多个集群，每个集群有一定权重) 其中"之一"
> ```
>
> `docker-compose.yaml`
>
> 由于webserver1, webserver2有相同的aliases, webservice, 所以cds + strict_dns指定webservice就可以获取到对应的端点了
>
> `Dockerfile-envoy`
>
> 在安装包之前，先配置镜像，这样快一点
>
> ```dockerfile
> RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
> ```



#### 启动

```bash
# docker-compose up
loading 0 cluster(s)
loading 1 listener(s)
webserver1_1  | Listening on http://0.0.0.0:8081
webserver2_1  | Listening on http://0.0.0.0:8081

```

进入envoy，查看启动的端口，并进行一系列的观测

```bash
[root@alyy eds-filesystem]# docker exec -it cds-eds-filesystem_envoy_1 sh
/ # netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 127.0.0.11:35804        0.0.0.0:*               LISTEN      
tcp        0      0 0.0.0.0:9901            0.0.0.0:*               LISTEN      
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      

```

查看clusters及其endpoints, 可以发现有2个发现的端点，此端点来自docker-compose的aliaes的webservice解析，此webservice定义在cluster获取endpoint的方式为strict_dns的名称为`webservice`

```bash
/ # curl localhost:9901/clusters
```

查看hostname, 现在可以发现是轮循获取主机名。

```bash
/ # curl localhost/hostname
Hostname: 2763c1e0043f.
/ # curl localhost/hostname
Hostname: ee066257c4d3.
/ # curl localhost/hostname
Hostname: 2763c1e0043f.
```

### filesystem cds  + filesystem eds

```bash
git clone https://github.com/slcnx/servicemesh_in_practise.git
[root@alyy servicemesh_in_practise-master]# tree cds-eds-filesystem/
cds-eds-filesystem/
|-- cds.conf        # filesystem cds + strict_dns eds， discovery response 格式
|-- cds.conf.v2     # filesystem cds + filesystem eds， discovery response 格式
|-- docker-compose.yaml
|-- Dockerfile-envoy
|-- eds.conf        # discovery response 格式
`-- envoy.yaml
```

> 以下示例基于文件系统的cds. 并引用了基于文件系统的eds
>
> `cds.conf.v2` 
>
> ```bash
> [root@alyy cds-eds-filesystem]# cat cds.conf.v2  | yq -y
> version_info: '1'
> resources:
>   - '@type': type.googleapis.com/envoy.api.v2.Cluster
>     name: webcluster1
>     connect_timeout: 0.25s
>     lb_policy: ROUND_ROBIN
>     type: EDS
>     eds_cluster_config:
>       service_name: webcluster1
>       eds_config:
>         path: /etc/envoy/eds.conf
> ```
>
> 基于文件系统的eds，需要指定eds每一个端点
>
> `eds`
>
> ```yaml
> [root@alyy cds-eds-filesystem]#  yq -y < eds.conf 
> version_info: '2'
> resources:
>   - '@type': type.googleapis.com/envoy.api.v2.ClusterLoadAssignment
>     cluster_name: webcluster1
>     endpoints:
>       - lb_endpoints:
>           - endpoint:
>               address:
>                 socket_address:
>                   address: 172.24.0.3
>                   port_value: 8081
>           - endpoint:
>               address:
>                 socket_address:
>                   address: 172.24.0.4
>                   port_value: 8081
> ```

#### 启动

```bash
docker-compose up
```

获取webserver1, 2的ip

```bash
[root@alyy eds-filesystem]# docker inspect cds-eds-filesystem_webserver2_1  | grep -i IPADDRESS
            "SecondaryIPAddresses": null,
            "IPAddress": "",
                    "IPAddress": "172.20.0.3",
[root@alyy eds-filesystem]# docker inspect cds-eds-filesystem_webserver1_1  | grep -i IPADDRESS
            "SecondaryIPAddresses": null,
            "IPAddress": "",
                    "IPAddress": "172.20.0.4",
```

进入envoy, 使用v2版本的配置

```bash
[root@alyy eds-filesystem]# docker exec -it cds-eds-filesystem_envoy_1 sh
/ # cd /etc/envoy/
/etc/envoy # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:14:00:02  
          inet addr:172.20.0.2  Bcast:172.20.255.255  Mask:255.255.0.0
```

检验此ip对应的集群配置, 可以看出是STRICT_DNS. 指向的是webservice.

```bash
[root@alyy cds-eds-filesystem]# curl -s 172.20.0.2:9901/config_dump | yq -y
  - '@type': type.googleapis.com/envoy.admin.v2alpha.ClustersConfigDump
    version_info: '1'
    dynamic_active_clusters:
      - version_info: '1'
        cluster:
          name: webcluster1
          type: STRICT_DNS
          connect_timeout: 0.250s
          load_assignment:
            cluster_name: webcluster1
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: myservice
                          port_value: 8081

```

现在切换成cds v2版本

```bash
/etc/envoy # ls
cds.conf     cds.conf.v2  eds.conf     envoy.yaml
/etc/envoy # mv cds.conf.v2 cds.conf
```

再次检验, 可以发现已经走的是EDS了。

```bash
[root@alyy cds-eds-filesystem]# curl -s 172.20.0.2:9901/config_dump | yq -y
  - '@type': type.googleapis.com/envoy.admin.v2alpha.ClustersConfigDump
    version_info: '1'
    dynamic_active_clusters:
      - version_info: '1'
        cluster:
          name: webcluster1
          type: EDS
          eds_cluster_config:
            eds_config:
              path: /etc/envoy/eds.conf
            service_name: webcluster1
```

获取clusters及endpoints.

```bash
[root@alyy cds-eds-filesystem]# curl 172.20.0.2:9901/clusters
```

获取主机名，失败了，原因是现在我们的webserver1, webserver2的ip不对。所以需要我们修改eds.conf。然后覆盖回eds.conf(docker 分层存储机制导致inotify失效)

```bash
[root@alyy cds-eds-filesystem]# curl 172.20.0.2/hostname
upstream connect error or disconnect/reset before headers. reset reason: connection failure
```

此处按4.1.3动态配置endpoint1来完成，namely.





