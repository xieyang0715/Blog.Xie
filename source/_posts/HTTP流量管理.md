---
title: HTTP流量管理
date: 2021-09-08 13:00:37
tags:
- istio
---



# 前言

envoy使用4层的`http connection manager`可以转换4层为7层

本节课: 主要讲 http manager <-> read/write <-> backend servers

1. 虚拟主机及路由配置概述

2. 路由配置

   - 路由匹配
     - 基础：prefix, path, pregex(safe_regex)
     - 高级: header, query_parameters
   - 路由
     - 路由:         route
     - 重定向:     redirect
     - 直接响应:  direct_response, envoy自己合成响应，错误响应、..。

   > 请求 -> 路由匹配+路由 -> 后端
   >
   > 匹配是短路逻辑

3. 高级流量管理
   - 流量迁移：基于流量的灰度发布
   - 流量分割：用户请求流量按比例侵害，蓝绿部署
   - 流量镜像：
   - 故障注入：破坏服务网格，看是否有韧性。
   - 超时和重试：客户端超时
   - `CORS`(跨域资源共享)

<!--more-->
# 虚拟主机及路由配置概述

`envoy.http_connection_manager` 特性

- 不支持SPDY，仅支持HTTP/1.1、WebSocket和HTTP/2。
- 关联的路由表可静态配置，亦可经由XDS API中的RDS动态生成。
- 内建重试插件，配置重试行为
  - host predicates
  - priority predicates
- 内建支持302重定向，收到上游的302时，envoy直接再请求一次，并直接返回给客户端。
- 连接或流级别的超时：
  - 连接：空闲、排空
  - 流：空闲、每路由相关的上游端点超时和每路由相关的grpc最大超时时长
- 基于自定义集群的动态转发代理。



过滤器实现

- router(`envoy.router`): 常用，基于路由表转发或重定向，重试，生成统计信息等。

router过滤器完成的高级路由机制：

- 基于域名的虚拟主机
- path前缀、精确或正则表达式匹配。
- 虚拟主机的TLS重定向
- path/host重定向
- envoy生成响应报文
- host rewrite
- prefix rewrite
- http标头或路由配置的请求重试与请求超时
- 基于运行时参数的流量迁移
- 基于权重或百分比的跨集群流量分割
- 基于任意标头匹配路由规则 
- 基于优先级的路由
- 基于hash策略的路由
- ...

## 虚拟主机定义 `virtual_hosts`

```yaml
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
          http_filters:
          - name: envoy.router
          route_config:
            name: local_route
            virtual_hosts:
            - name: vh_001
              domains: ["ilinux.io", "*.ilinux.io", "ilinux.*"]
              routes:
              - match:
                  path: "/service/blue"
                route:
                  cluster: blue
                redirect: {}
                direct_response: {}
                metadata: {}
                decorator: {}
                per_filter_config: {}
                typed_per_filter_config: {}
                request_headers_to_add: []
                request_headers_to_remove: []
                response_headers_to_add: []
                response_headers_to_remove: []
                tracing: {}
              require_tls: ""
              virtual_clusters: []
              rate_limits: []
              request_headers_to_add: []
              request_headers_to_remove: []
              response_headers_to_add: []
              response_headers_to_remove: []
              cors: {}
              per_filter_config: {}
              typed_per_filter_config: {}
              include_request_attempt_count: ""
              retry_policy: {}
              hedge_policy: {}
```

> virtual_hosts 顶级元素
>
> name 每一个虚拟主机项
>
> domains 对应http请求的host首部的值
>
> routes: 路由匹配+路由
>
> 限速
>
> cors控制
>
> 虚拟主机级别重试策略
>
> 虚拟主机级别的报文标头添加或删除
>
> > 以上为虚拟主机级别的策略，路由配置上自定义属性。后者优先级更高。
>
> virtual_clusters 虚拟集群列表



## routes的组成部分定义 `virtual_hosts.routes`

```yaml
            virtual_hosts:
            - routes:
              - match:
                  path: "/service/blue"
                route:
                  cluster: blue
                redirect: {}
                direct_response: {}
                metadata: {}
                decorator: {}
                per_filter_config: {}
                typed_per_filter_config: {}
                request_headers_to_add: []
                request_headers_to_remove: []
                response_headers_to_add: []
                response_headers_to_remove: []
                tracing: {}
```

> match:
>
> ​    path|prefix|regex(safe_regex)
>
> route
>
> ​    cluster|cluster_header|weighted_cluster

## 域名映射到虚拟主机

host 从上到下匹配`virtual_hosts`每一项的domains

- exact domain: 从上到下找每一个virtual_host项中的domains列表中是否包含精确匹配。有就先匹配。
- prefix domain:  `*.ilinux.io`  前缀*通配
- suffix domain:  `ilinux.*, ilinux-*.` 后缀*通配
- special wildcard `*`matching any domain.

## 一次请求的过程：从虚拟主机到路由目标

1. **映射到虚拟主机**：检测http请求的host标头，与domains匹配检查
2. **映射到虚拟主机之上的路由表项**：到了虚拟主机上，就检查多个match, 一旦匹配到就停止向下检查，短路工作逻辑。然后进行路由
3. （可选）如果定义了虚拟集群，按顺序检查虚拟主机，直到第1个匹配为止。

![envoy路由基础配置框架](http://myapp.img.mykernel.cn/envoy路由基础配置框架.png)

> match中为与逻辑

请求`/`，并且包含`x-custom-version: pre-release`就匹配，路由到集群。下面是负载均衡子集的元数据匹配

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
```

# 路由配置

## 路由定义概览 `routes`

```yaml
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
          http_filters:
          - name: envoy.router
          route_config:
            name: local_route
            virtual_hosts:
            - name: vh_001
              domains: ["ilinux.io", "*.ilinux.io", "ilinux.*"]
              routes:
              - match:
                  path: "/service/blue"
                route:
                  cluster: blue
                redirect: {}
                direct_response: {}
                metadata: {}
                decorator: {}
                per_filter_config: {}
                typed_per_filter_config: {}
                request_headers_to_add: []
                request_headers_to_remove: []
                response_headers_to_add: []
                response_headers_to_remove: []
                tracing: {}
```



## 路由匹配概览 `routes.match`

路由匹配, 过滤符合条件的请求，作处理

```yaml
prefix: ""
path: ""
regex: ""
case_sensitive: {}
runtime_fraction: {}
headers: []
query_parameters: []
grpc: {}
```

> 必须定义基础匹配条件，`prefix`, `path`, `regex`三者之一
>
> 可选匹配：
>
> 1. 区分大小写
> 2. 匹配指定的运行键值表示的比例进行流量迁移（runtime_fraction），不断修改运行时键值完成流量迁移
> 3. **基于标头路由**
> 4. **基于参数路由** ?开头的key=value&key=value&...
> 5. 仅匹配`grpc`流量
>
> 匹配必须是与关系

### 基础匹配 `route.match.path|regex|prefix`

必须3选1，可选结合后面的高级匹配

```yaml
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
          http_filters:
          - name: envoy.router
          route_config:
            name: local_route
            virtual_hosts:
            - name: vh_001
              domains: ["ilinux.io", "*.ilinux.io", "ilinux.*"]
              routes:
              - match:
                  path:
                  regex:
                  prefix:
```

### 基于标头匹配 `route.match.headers`

```yaml
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
          http_filters:
          - name: envoy.router
          route_config:
            name: local_route
            virtual_hosts:
            - name: vh_001
              domains: ["ilinux.io", "*.ilinux.io", "ilinux.*"]
              routes:
              - match:
                  headers:
                  - name: ""
                    exact_match: '...'
                    safe_regex_match: '{...}'
                    range_match: '{...}'
                    present_match: '...'
                    prefix_match: '...'
                    suffix_match: '...'
                    contains_match: '...'
                    string_match: '{...}'
                    invert_match: '...'
```

> name： 请求报文的标头名
>
> exact_match 精确匹配标头的值
>
> 范围内
>
> present_match 标头存在性匹配
>
> 前缀、后缀、包含
>
> 将匹配结果取反. 可以与以上各种match取反, 默认flase 是匹配到为真, true表示上述条件获取到为假，不匹配为真。

### 基于查询参数的路由匹配 `route.match.query_parameters`

```yaml
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
          http_filters:
          - name: envoy.router
          route_config:
            name: local_route
            virtual_hosts:
            - name: vh_001
              domains: ["ilinux.io", "*.ilinux.io", "ilinux.*"]
              routes:
              - match:
                  query_parameters:
                  - name: '...'
					string_match: '{...}'
                        exact: '...'
                        prefix: '...'
                        suffix: '...'
                        safe_regex: '{...}'
                        contains: '...'
                        ignore_case: '...'
					present_match: '...'
```

> `/path?key=value&key=value&...`
>
> name 查询key
>
> 字串匹配，精确、前缀、后缀、模式、包含、忽略大小写
>
> 存在性匹配，只要有key1这个参数，就匹配了



## 路由目标

### 路由目标对比 `routes.redirect|route|direct_response`

| 路由目标        | 是否到上游主机 |
| --------------- | -------------- |
| redirect        | 否             |
| direct_response | 否             |
| route           | 是             |



### 重定向概览  `routes.redirect`

```yaml
static_resources:
  listeners:
  - name: listener_http
    address:
      socket_address: { address: 0.0.0.0, port_value: 80 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                redirect:
                    https_redirect: '...'
                    scheme_redirect: '...'
                    host_redirect: '...'
                    port_redirect: '...'
                    path_redirect: '...'
                    prefix_rewrite: '...'
                    regex_rewrite: '{...}'
                    response_code: '...'
                    strip_query: '...'
```

> 协议重定向 https_redirect/scheme_redirect, 通常把http->https
>
> 主机重定向: host_redirect
>
> 端点、路由、前缀
>
> 正则重写
>
> 响应码,默认301
>
> strip_query是否重定向时删除查询参数，默认flase

`scheme://host:port/path?query_parameters;parameters#fragments`

### 直接响应请求概览 `routes.direct_response`

```yaml
static_resources:
  listeners:
  - name: listener_http
    address:
      socket_address: { address: 0.0.0.0, port_value: 80 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                direct_response:
                  status: '...'
				  body: '{...}'
                    filename: '...'
                    inline_bytes: '...'
                    inline_string: '...'

```

响应码、响应内容

响应内容是什么？

1. 加载本地文件，作为内容
2. 直接给出字节
3. 直接给出字符



### 路由到指定集群概览 `routes.route`

```yaml
static_resources:
  listeners:
  - name: listener_http
    address:
      socket_address: { address: 0.0.0.0, port_value: 80 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match:
                    prefix: "/"
                route:
                    cluster: '...'
                    cluster_header: '...'
                    weighted_clusters: '{...}'
                    cluster_not_found_response_code: '...'
                    metadata_match: '{...}'
                    prefix_rewrite: '...'
                    regex_rewrite: '{...}'
                    host_rewrite_literal: '...'
                    auto_host_rewrite: '{...}'
                    host_rewrite_header: '...'
                    host_rewrite_path_regex: '{...}'
                    timeout: '{...}'
                    idle_timeout: '{...}'
                    retry_policy: '{...}'
                    request_mirror_policies: []
                    priority: '...'
                    rate_limits: []
                    include_vh_rate_limits: '{...}'
                    hash_policy: []
                    cors: '{...}'
                    max_grpc_timeout: '{...}'
                    grpc_timeout_offset: '{...}'
                    upgrade_configs: []
                    internal_redirect_policy: '{...}'
                    internal_redirect_action: '...'
                    max_internal_redirects: '{...}'
                    hedge_policy: '{...}'
                    max_stream_duration: '{...}'
```

cluster: 路由到指定上游集群。事先存在集群

cluster_header：据请求首部cluster_header值来指定哪个集群处理。事先存在集群

weighted_clusters：路由多个集群，按权重比例分割到多个上游集群。事先存在集群

metadata_match： [负载均衡器子集使用](http://blog.mykernel.cn/2021/09/03/Envoy%E9%9B%86%E7%BE%A4%E7%AE%A1%E7%90%86/#%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E5%99%A8%E5%AD%90%E9%9B%86%E8%B0%83%E5%BA%A6)

prefix_rewrite: 前缀重定向

host_rewrite_literal: 请求首部的(host)重写

timeout 超时

retry_policy 重试策略

cors 跨站资源引用

request_mirror_policies 流量镜像

## 常用路由策略 示例

### 概述

1. 基础路由

   match: prefix, path, regex

   route, direct_response, redirect

2. 高级策略
   - match: prefix, path, regex 匹配，并结合高级匹配机制
     1. runtime_fraction 按比例切割流量
     2. headers 指定标头路由，基于cookie进行，user-agent
     3. 结合query_parameters指定参数路由，基于参数group进行，将其值分组后路由到不同目标。
   - 复杂路由
     1. 结合请求报文头中cluster_header的值进行路由
     2. weighted_clusters: 请求根据目标集群权重进行流量分割
     3. 高级路由属性：重试 `retry_policy`、`timeout`、CORS、限速、request_mirror_policies

### 基础路由 `routes.match.path|regex|prefix`

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
            - name: vh_001
              domains: ["ilinux.io", "*.ilinux.io", "ilinux.*"]
              routes:
              - match:
                  path: "/service/blue"
                route:
                  cluster: blue
              - match:
                  regex: "^/service/.*blue$"
                redirect:
                  path_redirect: "/service/blue"
              - match:
                  prefix: "/service/yellow"
                direct_response:
                  status: 200
                  body:
                    inline_string: "This page will be provided soon later.\n"
              - match:
                  prefix: "/"
                route:
                  cluster: red
            - name: vh_002
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: gray
          http_filters:
          - name: envoy.router
```

启动

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]#  tree http-connection-manager/httproute-simple-match/
http-connection-manager/httproute-simple-match/
├── docker-compose.yaml
└── front-envoy.yaml
```

> `front-envoy.yaml` 用户请求host报文被vh_001任意的domains匹配，就到达虚拟主机1， 其他vh_002. green集群并没有被引用。

```bash
$ docker-compose up
```

查看状态

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# docker exec httproute-simple-match_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:1C:00:09  
          inet addr:172.28.0.9  Bcast:172.28.255.255  Mask:255.255.0.0
```

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -s 172.28.0.9:9901/clusters | grep flag
red::172.28.0.5:80::health_flags::healthy
red::172.28.0.8:80::health_flags::healthy
green::172.28.0.6:80::health_flags::healthy
green::172.28.0.4:80::health_flags::healthy
blue::172.28.0.2:80::health_flags::healthy
blue::172.28.0.3:80::health_flags::healthy
gray::172.28.0.7:80::health_flags::healthy
```

到第1个虚拟主机的blue集群

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -H 'host: ilinux.io' 172.28.0.9/service/blue
<body bgcolor="dark_blue"><span style="color:white;font-size:4em;">
Hello from dark_blue (hostname: 3b1725a165c3 resolvedhostname:172.28.0.2)
</span></body>
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -H 'host: ilinux.io' 172.28.0.9/service/blue
<body bgcolor="light_blue"><span style="color:white;font-size:4em;">
Hello from light_blue (hostname: 42f751f8ee28 resolvedhostname:172.28.0.3)
</span></body>

```

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -s -H 'host: ilinux.io' 172.28.0.9/service/blue/
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>404 Not Found</title>
```

> 注意，当多了/结尾，就到了 到第1个虚拟主机的red集群, 而我们的浏览器通常可能会带/过来，所以这里就需要优化。

到第1个虚拟主机的默认url的red集群

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -H 'host: ilinux.io' 172.28.0.9/service/colored
<body bgcolor="light_red"><span style="color:white;font-size:4em;">
Hello from light_red (hostname: 1842289ad36d resolvedhostname:172.28.0.8)
</span></body>
```

到第1个虚拟主机的正则匹配，就会重定向，envoy可以配置，直接envoy自己重定向后，返回给你内容。

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -s -v -H 'host: ilinux.io' 172.28.0.9/service/ablue
* About to connect() to 172.28.0.9 port 80 (#0)
*   Trying 172.28.0.9...
* Connected to 172.28.0.9 (172.28.0.9) port 80 (#0)
> GET /service/ablue HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> host: ilinux.io
> 
< HTTP/1.1 301 Moved Permanently
< location: http://ilinux.io/service/blue
< date: Wed, 08 Sep 2021 08:09:15 GMT
< server: envoy
< content-length: 0
< 
* Connection #0 to host 172.28.0.9 left intact
```

> 观察` location: http://ilinux.io/service/blue`

到第2个虚拟主机, 不给域名，把ip作为域名，就匹配了`*`

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl 172.28.0.9/service/colored
<body bgcolor="gray"><span style="color:white;font-size:4em;">
Hello from gray (hostname: 8c17d085596d resolvedhostname:172.28.0.7)
</span></body>
```

到第1个虚拟主机，直接响应

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -s  -H 'host: ilinux.io' 172.28.0.9/service/yellow
This page will be provided soon later.
```

以yellow为前缀

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -s -v  -H 'host: ilinux.io' 172.28.0.9/service/yellow123123
* About to connect() to 172.28.0.9 port 80 (#0)
*   Trying 172.28.0.9...
* Connected to 172.28.0.9 (172.28.0.9) port 80 (#0)
> GET /service/yellow123123 HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> host: ilinux.io
> 
< HTTP/1.1 200 OK
< content-length: 39
< content-type: text/plain
< location: http://ilinux.io/service/yellow123123
< date: Wed, 08 Sep 2021 08:12:56 GMT
< server: envoy
< 
This page will be provided soon later.
* Connection #0 to host 172.28.0.9 left intact
```

> 请求：About to connect() to 172.28.0.9 port 80 (#0)
>
> ```bash
> > GET /service/yellow123123 HTTP/1.1
> > User-Agent: curl/7.29.0
> > Accept: */*
> > host: ilinux.io
> > 
> ```

### 标头及查询参数路由 `routes.match.headers|routes.match.query_parameters`

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
            - name: vh_001
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                  headers:
                  - name: X-Canary
                    exact_match: "true"
                route:
                  cluster: ver-1.7-pre
              - match:
                  prefix: "/"
                  query_parameters:
                  - name: "username"
                    string_match:
                      prefix: "vip_"
                route:
                  cluster: ver-1.6
              - match:
                  prefix: "/"
                route:
                  cluster: ver-1.5
          http_filters:
          - name: envoy.router
```

用户请求/开始，且首部有`X-Canary`精确值是`true` 就路由到金丝雀集群 `ver-1.7-pre`: version 1.7版本 预发

用户请求/开始，且查询条件 `?username`, 以`vip_`为前缀时，就路由到 `ver-1.6`集群： version 1.6版本

默认路由，请求/开始，路由到`ver-1.5`集群：version 1.5版本



```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# tree http-connection-manager/httproute-header-match/
http-connection-manager/httproute-header-match/
├── docker-compose.yaml
└── front-envoy.yaml
```

> `front-envoy.yaml` 描述如上, 集群定义，引用docker-compose的名称 ver-x.x

```bash
docker-compose up
```

信息

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# docker exec httproute-header-match_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:1D:00:03  
          inet addr:172.29.0.3  Bcast:172.29.255.255  Mask:255.255.0.0
```

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -s 172.29.0.3:9901/clusters | grep flag
ver-1.7-pre::172.29.0.2:80::health_flags::healthy
ver-1.5::172.29.0.5:80::health_flags::healthy
ver-1.5::172.29.0.6:80::health_flags::healthy
ver-1.6::172.29.0.7:80::health_flags::healthy
ver-1.6::172.29.0.4:80::health_flags::healthy
```

到达第1个虚拟主机的version 1.5版本

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -s 172.29.0.3/service/colored
<body bgcolor="version-1.5-1"><span style="color:white;font-size:4em;">
Hello from version-1.5-1 (hostname: 76cd158e9e4c resolvedhostname:172.29.0.6)
</span></body>
```

到达第1个虚拟主机的`vip`的version 1.6版本。前缀匹配

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -s 172.29.0.3/service/colored?username=vip_123
<body bgcolor="version-1.6-2"><span style="color:white;font-size:4em;">
Hello from version-1.6-2 (hostname: 91ceae7e0d87 resolvedhostname:172.29.0.4)
</span></body>

```

到达第1个虚拟主机的`canary` `version 1.7-pre`版本

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -s -H 'x-canary: true' 172.29.0.3/service/colored?username=vip_123
<body bgcolor="version-1.7-pre"><span style="color:white;font-size:4em;">
Hello from version-1.7-pre (hostname: 673aab048105 resolvedhostname:172.29.0.2)
</span></body>
[root@ip-10-0-0-91 servicemesh_in_practise-master]# curl -s -H 'x-Canary: true' 172.29.0.3/service/colored?username=vip_123
<body bgcolor="version-1.7-pre"><span style="color:white;font-size:4em;">
Hello from version-1.7-pre (hostname: 673aab048105 resolvedhostname:172.29.0.2)
</span></body>

```

> 首部并不区分大小写

### 灰度发布

#### 基本概念

互联网产品的特点：用户规模大、版本更新快

新版本的上线，运维和产品都需要很大的压力，灰度发布就是规避风险的工具。

为什么讲灰度？黑（未上线）到白（上线）平滑过渡。灰度就是黑到白的中间值。

**灰度发布**：如果更新过程中，让新老版本同时在线，运行对应的版本的承载形态（容器、裸金属）。如果新版本和老版本同时在线，一开始，仅分出较小比例的流量到新版本，待确认新版本没有问题后，再逐级将老版本（待老版本连接排空，才停应用程序）的承载流量切换到新版本上。

金丝雀发布等价叫金丝雀部署。蓝绿发布等价叫蓝绿部署 blue/green deploy。

![image-20210908163655288](http://myapp.img.mykernel.cn/image-20210908163655288.png)

#### 灰度发布场景

生产环境发布一个新的待上线的版本，而后将原生产环境的默认版本的流量 引流到此新版本，**配置的引流机制**叫灰度策略。

验证稳定后，新版本将承载所有流量。

一般叫金丝雀为灰度发布，上线新版本前，要有足够充分的测试。

测试环境中，使用流量镜像测试。就可以测试到真实场景的效果。



灰度策略：

- 金丝雀部署

  引入部分流量后，**立即暂停**，持续测试新版本的稳定性。

  缺点：

  - 认为稳定后，就滚动更新了，如果出问题，**需要回滚回来，时间漫长**。

  特点：

  - 原实例之上，就地更新。

  ![mage金丝雀发布](http://myapp.img.mykernel.cn/mage金丝雀发布.png)

- 蓝绿部署

  一模一样的备用实例，完成新旧版本的切换。

  特点：

  - 互为热备。
  - 流量100%切换到新版本，切换路由权重（非0即100）。
  - **出现问题，可以快速回滚。**

  缺点：

  - 浪费资源
  - 如果有bug非常影响线上用户

  ![蓝绿发布](http://myapp.img.mykernel.cn/蓝绿发布.png)

- A/B测试场景

  10%和90%的蓝绿部署，评估新版本有好的反馈后，再逐渐将流量引入到新版本，再评估哪个版本有较好的转化率。

#### 常见的灰度策略

基于请求内容发布和基于流量比例发布

- 基于请求内容发布：配置相应请求内容规则(高级路由匹配机制)，路由到灰度版本。
  - cookie
  - 自定义header
- 流量比例：灰度版本配置期望流量权重，将服务流量以指定权重比例引入到灰度版本。
  - 所有版本权重之和为100
  - 这种灰度策略也叫A/B测试

#### 灰度发布的实施方式

- LB灰度
  - LB上配置流量分布机制
  - 仅支持入口服务进行灰度，无法支持后端服务需求
- 基于k8s灰度
  - 新旧版本应用所在的pod数量比例进行流量分配
  - 滚动更新
  - 服务入口service/ingress
- 基于服务网格进行灰度发布
  - `envoy/istio`，灰度发布仅是流量治理机制的一种典型应用。
  - 通过控制平台，将流量配置策略分发到目标服务的请求发起方的envoy sidecar上即可。
  - 支持基于请求内容分配机制：user-agent/cookie
  - 服务访问入口是单独部署的envoy gateway.

1. 构筑承载实例

   蓝绿：需要一模一样的实例数，配置。投入数是双倍的。 快速回滚

   金丝雀：一批批更新节点。   缓慢回滚

2. 配置流量策略

   下线排空旧版本实例的流量；

   上线并分配流量到新版本实例；



### envoy流量迁移 `routes.match.runtime_fraction`

```yaml
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
          http_filters:
          - name: envoy.router
          route_config:
            name: local_route
            virtual_hosts:
            - name: vh_001
              domains: ["ilinux.io", "*.ilinux.io", "ilinux.*"]
              routes:
              - match:
                  prefix|path|regex:
                  runtime_fraction:
                     default_value: # 默认值 比例
                       numerator: 50          # 分子 默认0
                       denominator: HUNDRED   # 分母 分母小于分子，最终百分比为1. 默认HUNDRED（分母100->1/100）、TEN_THOUSAND（分母1万->1/10000）、MILLION(百万->1/1000000)
                     runtime_key: routing.traffic_shift.KEY   # 此key可以调整比例。此KEY表示分子
                  route: app1_v1
              - match:
                  prefix|path|regex:
                  cluster: app1_v2
```

>- 需要定义2条路由，一模一样的匹配条件 
>
>- 但是路由的目标，是不同的
>
>- 第1个匹配，需要定义流量切分比例的配置，而且**支持运行时调整**
>
>```bash
>                  runtime_fraction:
>                     default_value:
>                       numerator: 50
>                       denominator: HUNDRED
>                     runtime_key: routing.traffic_shift.KEY  
>```
>
>- 工作逻辑是断路机制，流量全部到第1个上，由`runtime_fraction`挑选部分流量到第2个上。
>
>动态修改流量迁移
>
>```bash
>curl -XPOST enovy_ip:admin_port/runtime_modify?KEY1=val1&KEY2=val2
>```
>
>如果分母为HUNDRED, 分子为100，则所有流量在app1_v1
>
>动态修改为分子为90，则表示旧版本app1_v1承载90%流量 ，新版本app1_v2承载10%流量。

实现金丝雀：

- 新版本+1个pod, 就旧版本流量切10%， 并下一个旧pod. 
- 新版本+1个pod, 就旧版本流量切10%， 并下一个旧pod. 
- 直到旧版本流量全部到新版本

实现蓝绿发布

- 新版本和旧版本有相同数量的pod, 就旧版本流量切100%，旧版pod不动。



**示例**

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# tree http-connection-manager/http-traffic-shifting/
http-connection-manager/http-traffic-shifting/
├── docker-compose.yaml
├── front-envoy.yaml
└── send-request.sh
```

> `docker-compose.yaml` envoy: 1.5版本：1.5.1，1.5.2、 1.6版本：1.6.1，1.6.2；生产环境中，一般使用新版本为老版本的1/10, 很小比例新版本正常后，就使用新版本，老版本滚动更新指定数量的pod, 再向下更新。
>
> `front-envoy.yaml ` 所有到当前请求为一个虚拟主机处理。2个match为相同的prefix(/), 路由目标是1.5和1.6集群。在1.5版本路由条目上，添加100/100=100%流量的比例，可以动态调整对比的key: `myapp`
>
> 随后通过调整此值，90/100, 指缝间漏走的就到第2个集群。

启动

```bash
docker-compose up
```

查看集群状态

```bash
[root@ip-10-0-0-91 http-traffic-shifting]# curl -s 172.30.0.6:9901/clusters | grep flag
myapp-v1.6::172.30.0.4:80::health_flags::healthy
myapp-v1.6::172.30.0.3:80::health_flags::healthy
myapp-v1.5::172.30.0.5:80::health_flags::healthy
myapp-v1.5::172.30.0.2:80::health_flags::healthy
```

请求

```bash
[root@ip-10-0-0-91 http-traffic-shifting]# cat send-request.sh 
#!/bin/bash
declare -i ver15=0
declare -i ver16=0

interval="0.2"

while true; do
        if curl -s http://$1/service/myapp | grep "v1.5" &> /dev/null; then
                # $1 is the host address of the front-envoy.
                ver15=$[$ver15+1]
        else
                ver16=$[$ver16+1]
        fi
        echo "myapp-v1.5:myapp-v1.6 = $ver15:$ver16"
        sleep $interval
done

```

> 请求每一次请求到达哪个版本，并输出各版本占比

```bash
[root@ip-10-0-0-91 http-traffic-shifting]# ./send-request.sh 172.30.0.6
myapp-v1.5:myapp-v1.6 = 1:0

```

> 现在所有请求的流量到达1.5

切分部分流量到新集群，在envoy集群不停机的情况下



模拟蓝绿部署

```bash
[root@ip-10-0-0-91 http-traffic-shifting]# curl -XPOST 172.30.0.6:9901/runtime_modify?routing.traffic_shift.myapp=0
OK
```

> 可以观察到上面的流量，全部到v1.6
>
> 这就是蓝绿部署

模拟金丝雀

```bash
# 先恢复成所有流量到老版本, 并重试测试
curl -XPOST 172.30.0.6:9901/runtime_modify?routing.traffic_shift.myapp=100

# 给10%新版本
[root@ip-10-0-0-91 http-traffic-shifting]# curl -XPOST 172.30.0.6:9901/runtime_modify?routing.traffic_shift.myapp=90
OK

# 根据新版本流量的反馈没有问题时，旧版本下线部分实例




# 上线新版本一定数量的pod, 给10%新版本，
[root@ip-10-0-0-91 http-traffic-shifting]# curl -XPOST 172.30.0.6:9901/runtime_modify?routing.traffic_shift.myapp=80
OK
# 根据新版本流量的反馈没有问题时，旧版本下线部分实例

...

# 最终，新版本所有数量pod与老版本一致，给100%新版本
[root@ip-10-0-0-91 http-traffic-shifting]# curl -XPOST 172.30.0.6:9901/runtime_modify?routing.traffic_shift.myapp=0
OK
# 根据新版本流量的反馈没有问题时，旧版本下线部分实例
```

模拟A/B测试

```bash
[root@ip-10-0-0-91 http-traffic-shifting]# curl -XPOST 172.30.0.6:9901/runtime_modify?routing.traffic_shift.myapp=50
OK
# 线上同时运行2套版本， 收集用户的转化率，看哪个版本转化率高，用户更喜欢。
```



### envoy流量分割 `routes.route.weighted_clusters`

由于在match 相同时，而route不同，指定不同cluster集群仅对2个集群可以进行流量迁移，而是不能精确控制流量。

所以在match对应的route中不再使用cluster指定集群，而是使用weighted_cluster.  将路由到一组带权重的目标上，可以配置多个集群，每一组分配的流量按权重计算即可. 多个集群的权重和为100即可。同时也支持runtime_fractor.

```yaml
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
          http_filters:
          - name: envoy.router
          route_config:
            name: local_route
            virtual_hosts:
            - name: vh_001
              domains: ["ilinux.io", "*.ilinux.io", "ilinux.*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  metadata_match: {}
                  weighted_clusters:
                    total_weight: 100 # 默认
                    runtime_key_prefix: "" # 运行时键前缀
                    clusters:
                    - name: webcluster1
                      weight: 80
                      metadata_match: 
                        filter_metadata:
                          envoy.lb:
                            version: "1.0"
                    - name: webcluster2
                      weight: 10
                      metadata_match:
                        filter_metadata:
                          envoy.lb:
                            version: "1.1"
                    - name: webcluster3
                      weight: 10
                      metadata_match:
                        filter_metadata:
                          envoy.lb:
                            version: "1.2"
```

> `total_weight`  `weighted_clusters.clusters`中每一个集群权重`[0-totalweight]`
>
> `runtime_key_prefix`  调整多个集群的权重时，因此同时调整2个集群起：每个集群使用`prefix.集群名`
>
> 调用时，最简单 `curl -XPOST enovy_ip:admin_port/runtime_modify?KEY1=val1&KEY2=val2`

- match匹配到之后，直接到weighted之后，再按权重分流量 



**示例**

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# tree http-connection-manager/http-traffic-splitting/
http-connection-manager/http-traffic-splitting/
├── docker-compose.yaml
├── front-envoy.yaml
└── send-request.sh
```

> `docker-compose.yaml` 定义了2个服务，每个服务有2个补丁版本
>
> `front-envoy.yaml` 只是2个集群，模拟蓝绿发布

启动

```bash
docker-compose up
```

详情

```bash
[root@ip-10-0-0-91 ~]# docker exec  http-traffic-splitting_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:1F:00:05  
          inet addr:172.31.0.5  Bcast:172.31.255.255  Mask:255.255.0.0
```

```bash
[root@ip-10-0-0-91 ~]# curl -s 172.31.0.5:9901/clusters | grep flag
myapp-v1.6::172.31.0.3:80::health_flags::healthy
myapp-v1.6::172.31.0.2:80::health_flags::healthy
myapp-v1.5::172.31.0.7:80::health_flags::healthy
myapp-v1.5::172.31.0.4:80::health_flags::healthy
myapp-v1.5::172.31.0.6:80::health_flags::healthy
```

现在动态修改v1.5 和v1.6的权重

模拟蓝绿流量策略

```bash
[root@ip-10-0-0-91 ~]# curl -XPOST -s '172.31.0.5:9901/runtime_modify?routing.traffic_split.myapp-v1.5=0&routing.traffic_split.myapp-v1.6=100'
OK
[root@ip-10-0-0-91 ~]# curl -s 172.31.0.5/service/version | grep -i 'hello'
Hello from myapp-v1.6-1 (hostname: 061d906a6ed3 resolvedhostname:172.31.0.2)
[root@ip-10-0-0-91 ~]# curl -s 172.31.0.5/service/version | grep -i 'hello'
Hello from myapp-v1.6-2 (hostname: 3246f1e79a8c resolvedhostname:172.31.0.3)
[root@ip-10-0-0-91 ~]# curl -s 172.31.0.5/service/version | grep -i 'hello'
Hello from myapp-v1.6-1 (hostname: 061d906a6ed3 resolvedhostname:172.31.0.2)
[root@ip-10-0-0-91 ~]# curl -s 172.31.0.5/service/version | grep -i 'hello'
Hello from myapp-v1.6-2 (hostname: 3246f1e79a8c resolvedhostname:172.31.0.3)
[root@ip-10-0-0-91 ~]# curl -s 172.31.0.5/service/version | grep -i 'hello'
Hello from myapp-v1.6-1 (hostname: 061d906a6ed3 resolvedhostname:172.31.0.2)
[root@ip-10-0-0-91 ~]# curl -s 172.31.0.5/service/version | grep -i 'hello'
Hello from myapp-v1.6-2 (hostname: 3246f1e79a8c resolvedhostname:172.31.0.3)
[root@ip-10-0-0-91 ~]# 
```

> 蓝绿，应该有相同配置数量的新版本

```bash
[root@ip-10-0-0-91 http-traffic-splitting]# cat send-request.sh 
#!/bin/bash
declare -i ver15=0
declare -i ver16=0

interval="0.2"

while true; do
        if curl -s http://$1/service/myapp | grep "1.5" &> /dev/null; then
                # $1 is the host address of the front-envoy.
                ver15=$[$ver15+1]
        else
                ver16=$[$ver16+1]
        fi
        echo "Version 1.5:Version 1.6 = $ver15:$ver16"
        sleep $interval
done

```

```bash
[root@ip-10-0-0-91 http-traffic-splitting]# ./send-request.sh 172.31.0.5
Version 1.5:Version 1.6 = 0:1
Version 1.5:Version 1.6 = 0:2
Version 1.5:Version 1.6 = 0:3
```

恢复

```bash
[root@ip-10-0-0-91 ~]# curl -XPOST -s '172.31.0.5:9901/runtime_modify?routing.traffic_split.myapp-v1.5=100&routing.traffic_split.myapp-v1.6=0'
OK

```



模拟金丝雀：v1.5滚动到v1.6

```bash
[root@ip-10-0-0-91 ~]# curl -XPOST -s '172.31.0.5:9901/runtime_modify?routing.traffic_split.myapp-v1.5=90&routing.traffic_split.myapp-v1.6=10'
OK
[root@ip-10-0-0-91 ~]# curl -XPOST -s '172.31.0.5:9901/runtime_modify?routing.traffic_split.myapp-v1.5=80&routing.traffic_split.myapp-v1.6=20'
OK

```

> 边调整边观察到比例实时生效的比例。

### envoy流量镜像  `routes.route.request_mirror_policies`

通过将生产流量拷贝到**测试集群**的新版本，实现新版本接近真实环境的测试，验证承载正常流量的工作状况，旨在有效地降低新版本上线的风险。

- 验证新版本
- 做测试
- 需要新旧版本间需要使用不同的数据库，所以就把流量引入到测试环境即可。

```yaml
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
          http_filters:
          - name: envoy.router
          route_config:
            name: local_route
            virtual_hosts:
            - name: vh_001
              domains: ["ilinux.io", "*.ilinux.io", "ilinux.*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster|weighed_clusters:
                  ...
				  request_mirror_policies: {}
                    cluster: '...'     # 镜像的流量 引入到哪个集群。测试集群
                    runtime_fraction: '{...}' # 多大比例引入到测试集群，如果测试集群规模小了，默认是100%比例过去，肯定测试集群就压垮
                       default_value: # 默认值 比例
                         numerator: 50          # 分子 默认0
                         denominator: HUNDRED   # 分母 分母小于分子，最终百分比为1. 默认HUNDRED（分母100->1/100）、TEN_THOUSAND（分母1万->1/10000）、MILLION(百万->1/1000000)
                       runtime_key: routing.traffic_shift.KEY   # 此key可以调整比例。此KEY表示分子
                    trace_sampled: '{...}' # 默认true, trace span 应该采样
```



**示例**

```bash
[root@ip-10-0-0-91 http-request-mirror]# tree
.
├── docker-compose.yaml
├── front-envoy.yaml
└── send-request.sh
```

默认引入到测试10/100的流量 ，可以定义为0，按需引流量

```bash
docker-compose up
```

详情

```bash
[root@ip-10-0-0-91 http-traffic-splitting]# docker exec http-request-mirror_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:00:04  
          inet addr:192.168.0.4  Bcast:192.168.15.255  Mask:255.255.240.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:24 errors:0 dropped:0 overruns:0 frame:0
          TX packets:14 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:2266 (2.2 KiB)  TX bytes:1308 (1.2 KiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:72 errors:0 dropped:0 overruns:0 frame:0
          TX packets:72 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:5826 (5.6 KiB)  TX bytes:5826 (5.6 KiB)
```

```bash
[root@ip-10-0-0-91 http-traffic-splitting]# curl -s 192.168.0.4:9901/clusters | grep flag
myapp-v1.6::192.168.0.6:80::health_flags::healthy
myapp-v1.6::192.168.0.7:80::health_flags::healthy
myapp-v1.5::192.168.0.3:80::health_flags::healthy
myapp-v1.5::192.168.0.2:80::health_flags::healthy
myapp-v1.5::192.168.0.5:80::health_flags::healthy
```

请求多次，然后观察到1.6版本有响应，但是请求的结果并没有1.6版本，所以不是流量迁移、分割。只是镜像，而不影响用户。

```bash
[root@ip-10-0-0-91 http-traffic-splitting]# curl 192.168.0.4/service/version
```

脚本

```bash
[root@ip-10-0-0-91 http-traffic-splitting]# cat send-request.sh 
#!/bin/bash
declare -i ver15=0
declare -i ver16=0

interval="0.2"

while true; do
        if curl -s http://$1/service/myapp | grep "1.5" &> /dev/null; then
                # $1 is the host address of the front-envoy.
                ver15=$[$ver15+1]
        else
                ver16=$[$ver16+1]
        fi
        echo "Version 1.5:Version 1.6 = $ver15:$ver16"
        sleep $interval
done

```

```bash
[root@ip-10-0-0-91 http-traffic-splitting]# ./send-request.sh 192.168.0.4
Version 1.5:Version 1.6 = 1:0
Version 1.5:Version 1.6 = 2:0
```

> 可以发现请求N次，都在1.5上，但是控制台日志，确有10%的流量到达测试实例。

调整镜像100流量，再次请求同样用户只看到1.5，但是日志显示1.5的一次请求，必须有1.6的一次请求，这就非常依赖测试环境的配置和承载能力要和生产一样了。

```bash
[root@ip-10-0-0-91 http-traffic-splitting]# curl -XPOST -s '192.168.0.4:9901/runtime_modify?routing.request_mirror.myapp=100'
OK

```

```bash
[root@ip-10-0-0-91 http-traffic-splitting]# ./send-request.sh 192.168.0.4
```

### envoy 混沌工程 

#### 概念

https://www.infoq.com/articles/devops-and-cloud-trends-2021/

![2021](https://imgopt.infoq.com/fit-in/625x1000/filters:quality(80)/filters:no_upscale()/articles/devops-and-cloud-trends-2021/en/resources/1devops-cloud-trend-report-2021-graph-1626085907084.jpg)

> 此图是在devops领域

混沌工程：通过工具(Chaos Monkey：随机kill实例)找出系统的脆弱点。减轻it人员工作量。

2014： 故障注入测试（Fail Inject Test)

2015：混沌金刚 (Chaos Kong), 将monkey故障级别从实例到地域。

2017：golang v2版本，集成spinnaker使用

2017：混沌工程在网上出版



给工作人员带来什么样的帮助 

- 架构师：验证系统架构的容错能力
- 开发和运维：提高故障应急的效率，实现故障告警、定位、恢复的有效和高效性。
- 对于测试：弥补传统测试方法留下的空白，混沌工程从系统角度进行测试，降低故障复发率，这有别于传统测试方法从用户角度的进行方式。
- 产品和设计：通过混沌事件，提升用户体验

#### 故障注入 `typed_config.http_filters`

envoy的故障注入，主要是通过用户指定参数(延迟、请求中止)，让envoy呈现不同的故障状态。仅支持网络故障，不支持本地的CPU满载、磁盘...问题。

```yaml
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
            - name: service
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: local_service
          http_filters:
          - name: envoy.fault
            config:
              delay:
                type: fixed
                fixed_delay: 10s
                percentage:
                  numerator: 10
                  denominator: HUNDRED
          - name: envoy.router
            config: {}
```

> 主要关注`typed_config.http_filters`此以下的配置。

使用故障注入功能，得先启用过滤器`envoy.fault`然后在路由配置上，添加配置

```yaml
deploy: {} #延迟
abort:  {} # 特定请求中止，返回503
upstream_cluster: # 过滤器生效的集群
header: []  # 请求头
downstream_nodes: [] # 默认所有主机
max_active_faults: {} # 可以多个故障，同一个时间最大活动故障数,
response_rate_limit: {} # 响应速率限制
```

> `http_filters.config`下的一级字段

示例，注入请求终止，5%请求注入503

```yaml
          http_filters:
          - name: envoy.fault
            config:
              abort:
                http_status: 503
                percentage:
                  numerator: 5
                  denominator: HUNDRED
          - name: envoy.router
            config: {}
```

示例，注入请求延迟，50的请求注入10s的延迟

```yaml
          http_filters:
          - name: envoy.fault
            config:
              delay:
                type: fixed
                fixed_delay: 10s
                percentage:
                  numerator: 10
                  denominator: HUNDRED
          - name: envoy.router
            config: {}

```



示例：

```bash
ls -l http-connection-manager/fault-injection
total 20
-rw-r--r--. 1 root root 1165 Apr 23  2020 docker-compose.yaml
-rw-r--r--. 1 root root 2553 Apr 23  2020 front-envoy.yaml
-rw-r--r--. 1 root root 1426 Apr 23  2020 service-envoy-fault-injection-abort.yaml
-rw-r--r--. 1 root root 1454 Apr 23  2020 service-envoy-fault-injection-delay.yaml
-rw-r--r--. 1 root root 1229 Apr 23  2020 service-envoy.yaml
```

> `docker-compose.yaml` front, 3个容器
>
> ```bash
>   service_red:
>     image: ikubernetes/servicemesh-app:latest
>     volumes:
>       - ./service-envoy-fault-injection-delay.yaml:/etc/envoy/envoy.yaml
> 
> ```
>
> > blue在sidecar上注入故障, 终止
> >
> > red在sidecar上注入故障，延迟
>
> `front-envoy.yaml`  不同url到达不同的集群。
>
> ```yaml
>           http_filters:
>           - name: envoy.router
>             typed_config: {}
> ```
>
> red 延迟， 集群对应的red服务有挂载延迟故障的envoy配置。
>
> blue终止
>
> green 无故障
>
> `service-envoy.yaml` 没有使用
>
> `service-envoy-fault-injection-abort.yaml`
>
> ```yaml
>           http_filters:
>           - name: envoy.fault
>             config:
>               abort:
>                 http_status: 503
>                 percentage:
>                   numerator: 10
>                   denominator: HUNDRED
>           - name: envoy.router
>             config: {}
> ```
>
> > 添加一个故障的envoy.fault, 原来的envoy.router也同时存在 
>
> 请求流量的10%比例注入故障。改到50%可以看的更清晰，生产中，就类似于自杀
>
> `service-envoy-fault-injection-delay.yaml`
>
> ```yaml
>           http_filters:
>           - name: envoy.fault
>             config:
>               delay:
>                 type: fixed
>                 fixed_delay: 10s
>                 percentage:
>                   numerator: 10
>                   denominator: HUNDRED
>           - name: envoy.router
>             config: {}
> ```
>
> > 延迟10s, 比率10%

启动

```bash
[root@ip-10-0-0-91 fault-injection]# docker-compose up
```

了解集群状态

```bash
[root@ip-10-0-0-91 ~]# docker exec fault-injection_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:10:02  
          inet addr:192.168.16.2  Bcast:192.168.31.255  Mask:255.255.240.0

```

```bash
[root@ip-10-0-0-91 ~]# curl -s 192.168.16.2:9901/clusters | grep -i flag
red_delay::192.168.16.3:80::health_flags::healthy
mycluster::192.168.16.3:80::health_flags::healthy
mycluster::192.168.16.4:80::health_flags::healthy
mycluster::192.168.16.5:80::health_flags::healthy
blue_abort::192.168.16.5:80::health_flags::healthy
green::192.168.16.4:80::health_flags::healthy
```

> 全部正常，mycluster不用过于关注

测试 red的终止故障

```bash
[root@ip-10-0-0-91 ~]# curl -s 192.168.16.2/service/red
```

> 注意，是10%的请求有故障。你到了10%的那部分请求时，就会等待10s才会响应

测试 blud终止故障

```bash
[root@ip-10-0-0-91 ~]# while true; do curl -s -I 192.168.16.2/service/blue | grep 'HTTP/1.1'; sleep 0.1; done
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 503 Service Unavailable
HTTP/1.1 503 Service Unavailable
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 503 Service Unavailable
HTTP/1.1 200 OK
HTTP/1.1 200 OK
```

> 注意，是10%的请求有故障。你到了10%的那部分请求时, 才是故障

验证请求终止、延迟了，服务网格自动解决问题。

- 超时
- 重试

这个配置来自于RDS， 更新RDS配置及版本，对应节点就达到了故障注入了。

#### 超时和重试, `前端的front-end: routes.route.retry_policy|routes.route.timeout`

真实系统中，用来自动对以上故障生产韧性

分布式场景中，几种故障处理机制：

- retry: 出现问题后，envoy就透明处理故障，用户只会看到结果。     abort

  - 重试
  - 延迟后重试
  - 指数式延迟后重试
  - 非关键性操作，最好快速失败
  - 批处理应用，增加重试次数更合适，但是重试次数之间的延迟就成倍或指数级增加。
  - 大量重试，还失败，应该需要避免后面的请求再次到相同服务。熔断器半打开。
  - **还要考虑操作是否幂等，非幂等会导致数据库出问题**
  - 请求可能会由于多种原因而失败，它们可能分别引发不同的异常，一些异常可以迅速完成，一些异常可能会持续很长时间，重试间隔需要考虑。
  - 确保所有重试代码已针对各种故障情况进行全面测试，以检查它们不会严重影响应用的性能及可靠性。

  > 为避免数据库多次请求，一般不要使用重试。
  >
  > **默认envoy不会进行任何类型的重试，除非明确定义** 
  >
  > retry_back_off 默认是退避算法，一次重试失败，就会指数级延后再重试。第1次重试失败等5ms, 第2次失败等75ms, 第3次失败等.., 所有时间累积加之和不能超过下面定义的超时时长。
  >
  > 哪种响应状态码才重试
  >
  > retry_host_predicate 重试时，调度时不会再发到之前失败的那个主机。默认不会排除之前失败的主机。
  >
  > retriable_headers 哪些响应标头才重试
  >
  > retriable_request_headers 哪些请求标头出现时才有可能重试。

- timeout: 避免故障长时间等待，反复重试给系统造成压力，就有超时。 delay

- circuit breaker：避免级联故障，熔断器、连接池 

示例

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
                  prefix: "/service/blue"
                route:
                  cluster: blue_abort
                  retry_policy:
                    retry_on: "5xx"
                    num_retries: 3
              - match:
                  prefix: "/service/red"
                route:
                  cluster: red_delay
                  timeout: 1s
```

> abort就会重试：仅在5xx响应码时，就退避算法最多重试3次
>
> 延迟就超时。默认注入故障是10s, 所以front在5s时就返回。
>
> 重试时，可以重试相同主机。或者优先级负载到新的集群



```bash
[root@ip-10-0-0-91 http-connection-manager]# ls -l timeout-retries/
total 20
-rw-r--r--. 1 root root 1165 Apr 23  2020 docker-compose.yaml
-rw-r--r--. 1 root root 2686 Apr 23  2020 front-envoy.yaml
-rw-r--r--. 1 root root 1426 Apr 23  2020 service-envoy-fault-injection-abort.yaml
-rw-r--r--. 1 root root 1454 Apr 23  2020 service-envoy-fault-injection-delay.yaml
-rw-r--r--. 1 root root 1229 Apr 23  2020 service-envoy.yaml
-rw-r--r--. 1 root root    0 Apr 23  2020 vim
```

> `docker-compose.yaml `
>
> `front-envoy.yaml`文件中 定义超时和重试

为了演示请求终止和delay为50%比率，并且延迟10s

启动

```bash
docker-compose up
```

```bash
[root@ip-10-0-0-91 ~]# docker exec timeout-retries_front-envoy_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:20:04  
          inet addr:192.168.32.4  Bcast:192.168.47.255  Mask:255.255.240.0

```

```bash
[root@ip-10-0-0-91 ~]# curl -s 192.168.32.3:9901/clusters | grep flag
red_delay::192.168.32.4:80::health_flags::healthy
mycluster::192.168.32.2:80::health_flags::healthy
mycluster::192.168.32.5:80::health_flags::healthy
mycluster::192.168.32.4:80::health_flags::healthy
blue_abort::192.168.32.5:80::health_flags::healthy
green::192.168.32.2:80::health_flags::healthy
```

现在请求blue, 有重试策略

```bash
[root@ip-10-0-0-91 ~]# while true; do curl -s 192.168.32.3/service/blue | grep -E '^(Hello|fault)'; sleep 0.1; done
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
fault filter abort
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
Hello from blue (hostname: 0ced883b8111 resolvedhostname:192.168.32.5)
fault filter abort

```

> 虽然我们注入了50%的故障，但是我们有了重试策略，所以用户得到重试后的结果。如果提升重试次数，成功率就更高了。重试次数更多的话，对服务压力更大

请求red集群

- 准备验证脚本

  创建curl-format.txt， 粘贴以下内容

  ```
  time_namelookup:  %{time_namelookup}s\n
          time_connect:  %{time_connect}s\n
       time_appconnect:  %{time_appconnect}s\n
      time_pretransfer:  %{time_pretransfer}s\n
         time_redirect:  %{time_redirect}s\n
    time_starttransfer:  %{time_starttransfer}s\n
                       ----------\n
            time_total:  %{time_total}s\n
  
  ```

现在请求

```bash
[root@ip-10-0-0-91 ~]# curl -w "@curl-format.txt"  -s 192.168.32.3/service/red 
<body bgcolor="red"><span style="color:white;font-size:4em;">
Hello from red (hostname: 9fdacc771620 resolvedhostname:192.168.32.4)
</span></body>
time_namelookup:  0.000s
        time_connect:  0.000s
     time_appconnect:  0.000s
    time_pretransfer:  0.000s
       time_redirect:  0.000s
  time_starttransfer:  0.002s
                     ----------
          time_total:  0.002s
[root@ip-10-0-0-91 ~]# curl -w "@curl-format.txt"  -s 192.168.32.3/service/red 
upstream request timeouttime_namelookup:  0.000s
        time_connect:  0.000s
     time_appconnect:  0.000s
    time_pretransfer:  0.000s
       time_redirect:  0.000s
  time_starttransfer:  1.001s
                     ----------
          time_total:  1.001s

```

> 第1次成功了，第2次超时，显示为1s, 说明front返回了超时。



##### 重试插件

重试时，有可能会调用到失败的相同主机。避免再到相同主机

```yaml
                  retry_policy:
                    retry_on: "5xx"
                    num_retries: 3
                    retry_host_predicate: # 符合条件的就不作为重试的对象
                    - name: envoy.retry_host_predicate.previous_hosts # 以前尝试过的主机将不作为重试请求
                    - name: envoy.retry_host_predicate.omit_canary_hosts # 
                    retry_priority:
                      name: envoy.retry_priority.previous_priorities # 将失败的主机优先级调低。
                      config:
                        update_frequecy: 2
```

> 常见的配置，如果基于优先级



### CORS `typed_config.http_filters|routes.route.cors`

跨域资源共享是http的访问控制机制 通过http头部告诉浏览器允许访问其他站点的资源。

XMLHttpRequest和Fetch API只能加载同一个域请求HTTP资源，除非响应报文正确CORS响应头。

envoy支持cors配置，就是在响应头部添加标头

虚拟主机或路由级别

```yaml
allow_origin_string_match: []
allow_methods: '...'
allow_headers: '...'
expose_headers: '...'
max_age: '...'
allow_credentials: '{...}'
filter_enabled: '{...}'
shadow_enabled: '{...}'
```

> allow_origin_string_match 允许哪些源，支持正则
>
> 其他域名访问当前域名，可以使用哪些方法，header, 
>
> header白名单
>
> 请求缓存时长
>
> 是否允许凭据发起实际请求
>
> filter_enabled 多大请求比例上跨域共享，默认是100%请求启动共享。强制生效。
>
> shadow_enabled 不是强制生效的。filter_enabled优先于shadow

```bash
[root@ip-10-0-0-91 cors]# ls
backend  frontend  README.md
```

要支持cors, 需要http_filter中引入envoy.cors

```yaml
          http_filters:
          - name: envoy.cors
            typed_config: {}
          - name: envoy.router
            typed_config: {}
```

然后在路由中定义

```yaml
static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 80
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          codec_type: auto
          stat_prefix: ingress_http
          access_log:
            - name: envoy.file_access_log
              typed_config:
                "@type": type.googleapis.com/envoy.config.accesslog.v2.FileAccessLog
                path: "/var/log/access.log"
          route_config:
            name: local_route
            virtual_hosts:
            - name: www
              domains:
              - "*"
              cors:
                allow_origin:
                - "*"
                allow_methods: "GET"
                filter_enabled:
                  default_value:
                    numerator: 100
                    denominator: HUNDRED
                  runtime_key: cors.www.enabled
                shadow_enabled:
                  default_value:
                    numerator: 0
                    denominator: HUNDRED
                  runtime_key: cors.www.shadow_enabled
              routes:
              - match:
                  prefix: "/cors/open"
                route:
                  cluster: backend_service
              - match:
                  prefix: "/cors/disabled"
                route:
                  cluster: backend_service
                  cors:
                    filter_enabled:
                      default_value:
                        numerator: 0
                        denominator: HUNDRED
              - match:
                  prefix: "/cors/restricted"
                route:
                  cluster: backend_service
                  cors:
                    allow_origin:
                    - "envoyproxy.io"
                    allow_methods: "GET"
              - match:
                  prefix: "/"
                route:
                  cluster: backend_service

```

> 与routes同级别的cors时，就是虚拟主机级别，每个路由都会继承。
>
> routes下一级的route.cors定义，就是路由级别。

front中，是一个flask网页运行在后面集群中的容器中，响应一个网页， 网页中有跨域请求

```yaml
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          codec_type: auto
          stat_prefix: ingress_http
          access_log:
            - name: envoy.file_access_log
              typed_config:
                "@type": type.googleapis.com/envoy.config.accesslog.v2.FileAccessLog
                path: "/var/log/access.log"
          route_config:
            name: local_route
            virtual_hosts:
            - name: services
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: frontend_service
          http_filters:
          - name: envoy.cors
            typed_config: {}
          - name: envoy.router
            typed_config: {}
```

> 前端模拟网页，所以是客户端，不需要cors配置

后端提供了flask, 暴露了/cors/status. 服务自动不拒绝

启动前端

```bash
[root@ip-10-0-0-91 cors]#  cd backend/
# 启动前映射 8081:80
[root@ip-10-0-0-91 backend]# docker-compose up
ec2-161-189-24-154.cn-northwest-1.compute.amazonaws.com.cn
```

> 8081, 9901

启动前端

```bash
[root@ip-10-0-0-91 ~]# cd servicemesh_in_practise-master/http-connection-manager/cors/
[root@ip-10-0-0-91 cors]# cd frontend/
# 启动前映射 8082:80
[root@ip-10-0-0-91 frontend]# docker-compose up

```

> 为了浏览器访问frontend envoy，所以暴露了2个envoy的port
>
> 要访问8082, 9902

现在请求前端的页面，而页面中的数据是在浏览器中执行的。

![image-20210918164718377](http://myapp.img.mykernel.cn/image-20210918164718377.png)

这里的地址应该填后端的地址，即`ec2-161-189-24-154.cn-northwest-1.compute.amazonaws.com.cn`

如果选择disabled, 就是获取值为disabled, 然后就会请求`ec2-161-189-24-154.cn-northwest-1.compute.amazonaws.com.cn:8081/cors/disabled`路径, 后端对应了此路由之后, 则跨域的请求0%的请求能到达后端

```yaml
              - match:
                  prefix: "/cors/disabled"
                route:
                  cluster: backend_service
                  cors:
                    filter_enabled:
                      default_value:
                        numerator: 0
                        denominator: HUNDRED
```

```html
       function invokeRemoteDomain() {
            var remoteIP = document.getElementById("remoteip").value;
            var enforcement = document.querySelector('input[name="cors"]:checked').value;
            if(client) {
                var url = `http://${remoteIP}:8081/cors/${enforcement}`;
                client.open('GET', url, true);
                client.onreadystatechange = handler;
                client.send();
            <input type="radio" name="cors" value="disabled" checked="checked"/> Disabled<br/>
            <input type="radio" name="cors" value="open"/> Open<br/>
            <input type="radio" name="cors" value="restricted"/> Restricted<br/>
            <br/>
```

![image-20210918165745979](http://myapp.img.mykernel.cn/image-20210918165745979.png)

出现cors error, 控制台打印了跨域。但是从请求network处发现是正常的，而且上面的路由是0/100的流量到达后端，为什么通了呢？

> 因为在路由同级别是全局设定，这里是所有域放行，如果注释后，再请求open, 都会报错，默认就拒绝跨域了。



跨域不是 浏览器拒绝的，而是代理：只有浏览器请求才会有跨域。所以代理在服务器上，去请求后端，无论请求啥都不存在跨域，所以跨域的请求在代理上，默认会拒绝。



