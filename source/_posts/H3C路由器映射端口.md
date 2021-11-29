---
title: H3C路由器映射端口
date: 2021-11-04 10:10:42
tags:
- H3C
- iptables
photos:
- http://myapp.img.mykernel.cn/h3c.jpg
---



# 前言

H3C路由器，虚拟服务服务器的端口映射最多20个条目，每个条目支持任意端口范围到，另一个主机的任意端口范围。

公司中，使用H3C路由器时，由于疫情期间需要办公，太多人映射公网 3389-3389 到 内网某开发IP的 3389-3389，明显不够用 





<!--more-->
# 实现方法

h3c 端口 30000 - 30100 映射到公司内网的一个Linux的IP 30000 - 30100



通过这个IP的30000端口基于DNAT, 完成映射到某个开发的电脑,

> 由于dnat只改变目标，源不会改变。
>
> 因为客户端是给服务器发起的请求，如果我们只是修改了目标（开发的电脑,），目标返回了数据，但是客户端不认识目标啊。
>
> 所以方法是将开发的电脑 的网关修改成服务器，然后做一个此电脑相关网络的SNAT, 这样就可以识别到源了。

通过4层代理，监听端口。不需要iptables规则 。

> 代理着去完成事情



# 华3配置

![image-20211104101941516](http://myapp.img.mykernel.cn/image-20211104101941516.png)

> 1.248就是公司内网常年在线的Linux服务器

# Linux服务器配置

##  iptables

<strong style="color: red">略，需要开发电脑修改网关</strong>



## nginx

`docker-compose.yaml`

```yaml
version: '3.3'
services:
  nginx:
    image:  envoy
    restart: always
    volumes:
    - ./nginx.conf:/etc/nginx/nginx.conf
    ports:
    - 33000-33100:33000-33100
```

`nginx.conf`

```nginx
user  nginx;
worker_processes  2;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}



stream {
    server {
        listen 33000;
        proxy_connect_timeout 1s;
        proxy_timeout 1m;
        proxy_pass 192.168.1.222:3389;
    }
}
```

<strong style="color: red">每次修改配置，需要重启服务</strong>

## envoy

https://www.envoyproxy.io/docs/envoy/latest/start/docker#running-envoy-with-docker-compose

### 静态调试

`docker-compose.yaml`

```yaml
version: '3.3'
services:
  nginx:
    image:  envoyproxy/envoy-dev:c41808220dda74de37af6f21988fae4d6975b60c
    restart: always
    environment:
    - ENVOY_UID='0'
    - ENVOY_GID='0'
    volumes:
    - ./envoy.yaml:/etc/envoy/envoy.yaml
    ports:
    - 33000-33100:33000-33100
    - 33101:33101
```

`envoy.yaml` 静态调试

```yaml
admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 33101
static_resources:
  listeners:
  - name: songliangcheng
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 33000
    filter_chains:
    - filters:
      - name: envoy.filters.network.tcp_proxy
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
          stat_prefix: tcp-proxy
          weighted_clusters:
            clusters:
            - name: songliangcheng_upstream_cluster
              weight: 100
              metadata_match:
                filter_metadata:
                  envoy.lb:
                    songliangcheng: true
          access_log:
          - name: envoy.access_loggers.stream
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
  - name: xieyang
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 33001
    filter_chains:
    - filters:
      - name: envoy.filters.network.tcp_proxy
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
          stat_prefix: tcp-proxy
          weighted_clusters:
            clusters:
            - name: songliangcheng_upstream_cluster
              weight: 100
              metadata_match:
                filter_metadata:
                  envoy.lb:
                    xieyang: true
          access_log:
          - name: envoy.access_loggers.stream
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
  clusters:
  - name: songliangcheng_upstream_cluster
    type: STATIC
    load_assignment:
      cluster_name: songliangcheng_upstream_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 192.168.1.222
                port_value: 3389
          metadata:
            filter_metadata:
              envoy.lb:
                songliangcheng: true
        - endpoint:
            address:
              socket_address:
                address: 192.168.1.67
                port_value: 3389
          metadata:
            filter_metadata:
              envoy.lb:
                xieyang: true
```

### 基于文件系统动态配置

https://github.com/envoyproxy/envoy/blob/main/examples/dynamic-config-fs/

`cds.yaml`

```yaml
resources:
- "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
  name: songliangcheng_upstream_cluster
  load_assignment:
    cluster_name: songliangcheng_upstream_cluster
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: 192.168.1.222
              port_value: 3389
        metadata:
          filter_metadata:
            envoy.lb:
              songliangcheng: true
      - endpoint:
          address:
            socket_address:
              address: 192.168.1.67
              port_value: 3389
        metadata:
          filter_metadata:
            envoy.lb:
              xieyang: true
```

`docker-compose.yaml`

```yaml
version: '3.3'
services:
  envoy:
    image:  envoyproxy/envoy-dev:c41808220dda74de37af6f21988fae4d6975b60c
    restart: always
    environment:
    - ENVOY_UID=0
    - ENVOY_GID=0
    volumes:
    - ./envoy.yaml:/etc/envoy/envoy.yaml
    - ./cds.yaml:/etc/envoy/cds.yaml
    - ./lds.yaml:/etc/envoy/lds.yaml
    ports:
    - 33000-33100:33000-33100
    - 33101:33101 # admin
```

`envoy.yaml`

```yaml
node:
  id: id_1
  cluster: test
admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 33101

dynamic_resources:
  cds_config:
    path: /etc/envoy/cds.yaml
  lds_config:
    path: /etc/envoy/lds.yaml
```

`lds.yaml`

```yaml
resources:
- "@type": type.googleapis.com/envoy.config.listener.v3.Listener
  name: songliangcheng
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 33000
  filter_chains:
  - filters:
    - name: envoy.filters.network.tcp_proxy
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
        stat_prefix: tcp-proxy
        weighted_clusters:
          clusters:
          - name: songliangcheng_upstream_cluster
            weight: 100
            metadata_match:
              filter_metadata:
                envoy.lb:
                  songliangcheng: true
        access_log:
        - name: envoy.access_loggers.stream
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
- "@type": type.googleapis.com/envoy.config.listener.v3.Listener
  name: xieyang
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 33001
  filter_chains:
  - filters:
    - name: envoy.filters.network.tcp_proxy
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
        stat_prefix: tcp-proxy
        weighted_clusters:
          clusters:
          - name: songliangcheng_upstream_cluster
            weight: 100
            metadata_match:
              filter_metadata:
                envoy.lb:
                  xieyang: true
        access_log:
        - name: envoy.access_loggers.stream
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
```

由于直接在宿主机修改配置，并不会动态生成文件，可以通过admin接口验证



通过二进制部署envoy，应该就正常了。

https://www.envoyproxy.io/docs/envoy/latest/start/install#install-envoy-on-rpm-based-distros

```bash
# /etc/systemd/system/envoy.service
[Unit]
Description=Envoy

[Service]
Type=Simple
ExecStart=/usr/bin/envoy -c /root/proxy/envoy.yaml
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```



### 基于ads动态配置

https://github.com/envoyproxy/envoy/blob/main/examples/dynamic-config-cp

`envoy.yaml`

```yaml
node:
  id: id_1
  cluster: test
admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 33101

dynamic_resources:
  cds_config:
    resource_api_version: V3
    ads: {}
  lds_config:
    resource_api_version: V3
    ads: {}
  ads_config:
    api_type: GRPC
    transport_api_version: V3
    grpc_services:
    - envoy_grpc:
        cluster_name: xds_cluster

static_resources:
  clusters:
  - type: STRICT_DNS
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    name: xds_cluster
    load_assignment:
      cluster_name: xds_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: go-control-plane
                port_value: 8081

```

`docker-compose.yaml`

```yaml
version: '3.3'
services:
  envoy:
    image:  envoyproxy/envoy-dev:c41808220dda74de37af6f21988fae4d6975b60c
    restart: always
    environment:
    - ENVOY_UID=0
    - ENVOY_GID=0
    volumes:
    - ./envoy.yaml:/etc/envoy/envoy.yaml
    - ./cds.yaml:/etc/envoy/cds.yaml
    - ./lds.yaml:/etc/envoy/lds.yaml
    ports:
    - 33000-33100:33000-33100
    - 33101:33101 # admin
  go-control-plane:
    #image: registry.cn-hangzhou.aliyuncs.com/slcnx/envoy-control-plane:0.2
    image: ikubernetes/sxds
    ports:
    - 33102:8081
    - 33103:8082
```

由于go-control-plane,需要写resource.go文件，太复杂了，故放弃了。

#　windows配置

一定要不睡眠，不然就断网了

![image-20211104103430112](http://myapp.img.mykernel.cn/image-20211104103430112.png)

打开远程桌面

![image-20211104103517763](http://myapp.img.mykernel.cn/image-20211104103517763.png)



