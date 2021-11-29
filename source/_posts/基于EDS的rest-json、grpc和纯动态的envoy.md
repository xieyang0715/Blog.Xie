---
title: 基于EDS的rest-json、grpc和纯动态的envoy
date: 2021-03-23 06:33:33
tags:
- istio
---



# 前言
基于[动态配置源](http://blog.mykernel.cn/2021/03/18/Envoy%E5%9F%BA%E4%BA%8E%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E7%9A%84EDS%E5%92%8CCDS%E8%AE%A2%E9%98%85%E7%A4%BA%E4%BE%8B/#Envoy%E5%8A%A8%E6%80%81%E9%85%8D%E7%BD%AE%E5%8F%8A%E9%85%8D%E7%BD%AE%E6%BA%90)的应用场景，filesystem 仅合适小规模集群，大规模集群应该使用discovery request和discovery response 与集中的配置中心交互（这个配置中心可以使用envoy的SDK开发），istio就提供了piolt就提供了配置中心。

接下来演示test-json, grpc。不使用istio提供的piolt这样重量的配置中心，而是使用envoy SDK开发的微型服务器。dockerhub.com/ikubernetes下的

- `eds-rest-server` 提供eds配置，用户可以基于api提交动态的配置，此配置信息可以分发给订阅了此eds的客户端。
- `sxds`                       仅提供`lds/cds/rds/eds` 这几个应用。`grpc server` 解决方案之一。 sxds在更新时，流量会丢失，使用ads时将会正常。

> envoy的EDS/CDS/RDS可以静态或动态。但是一旦使用这些XDS时，就需要指定XDS的地址，这个地址一定是静态配置的集群，其中将不能再次使用EDS。


<!--more-->
# rest-json的eds

```yaml
  clusters:
  - name: webcluster1
    type: EDS # 动态获取endpoints
    connect_timeout: 0.25s
    eds_cluster_config:
      service_name: myservice # 提交json的末尾
      eds_config:
        api_config_source:  #rest-json/grpc定义
          api_type: REST  # REST/GRPC/DELTA_GRPC
          cluster_names: [edscluster] 
          refresh_delay: 5s
          request_timeout: 1s
```

>  如果filesystem eds自动发现 定义就是 `clusters.eds_cluster_config.eds_config.path`
>
> - cluster_names： 多个提供xds协议的eds服务器，冗余。
>
> - refresh_delay：轮询REST API的时间间隔
> - request_timeout: 请求REST API超时时长

指定XDS协议的EDS服务，必须静态

```bash
  clusters:
  - name: edscluster
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: edscluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: edsserver, port_value: 8080 } # docker-compose定义的edsserver的别名"edsserver"
```

**如果使用动态的CDS, 那么配置的cds server(grpc/delta_grpc/rest-json)，必须静态指定。server上传递的配置将是cluster及其端点**

## 示例架构

![image-20210901214510013](http://myapp.img.mykernel.cn/image-20210901214510013.png)

1. envoy连接rest-json的eds获取配置，完成集群初始化。
2. 获取eds失败就初始化没有endpoints.
3. 手工提交server1的信息，envoy就轮询到信息，envoy就可以反代到第1个服务器。
4. 手工提交server1, server2的信息。同上

> 由于是外部的动态配置获取，不依赖文件系统的inotify, 可以直接挂载配置文件，启动。

## 准备配置

```bash
git clone https://github.com/slcnx/servicemesh_in_practise.git
```

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# tree eds-rest/
eds-rest/
├── docker-compose.yaml
├── Dockerfile-envoy
├── envoy.yaml
└── resources
    ├── endpoints-2.json
    └── endpoints.json

1 directory, 5 files
```

> `envoy.yaml` 定义静态定义的cluster获取动态rest EDS获取endpoints端点。而xds eds集群必须是静态配置的。
>
> `docker-compose.yaml` 编排结果和[示例架构](#示例架构) 相同架构。其中的edsserver，将引用`ikubernetes/eds-rest-server`
>
> `resources` 可以提交给 rest-server, 让envoy轮询获取新配置
>
> `endpoints.json` 可以发现提交给eds上的内容并不遵守 discovery protocol. 这个是rest server自己完成的事情。
>
> ```yaml
> [root@ip-10-0-0-91 eds-rest]# cat resources/endpoints.json  | yq -y
> hosts:
>   - ip_address: 172.18.0.4
>     port: 8081
>     tags:
>       az: cn-north1-a
>       canary: false
>       load_balancing_weight: 50
> ```
>
> - endpoint地址、端口、标签

## 启动

```bash
docker-compose up
loading 2 cluster(s)
loading 1 listener(s)
webserver2_1 |Listening on http://0.0.0.0:8081
webserver1_1 |Listening on http://0.0.0.0:8081

edsserver_1  | * Serving Flask app "app" (lazy loading)
edsserver_1  | * Environment: production
edsserver_1  |   WARNING: This is a development server. Do not use it in a production deployment.
edsserver_1  |   Use a production WSGI server instead.
edsserver_1  | * Debug mode: on
edsserver_1  | * Running on http://0.0.0.0:8080/ (Press CTRL+C to quit)
edsserver_1  | * Restarting with stat
edsserver_1  | * Debugger is active!
edsserver_1  | * Debugger PIN: 118-291-983
edsserver_1  |10.4.1.5 - - [01/Sep/2021 14:26:41] "POST /v2/discovery:endpoints HTTP/1.1" 200 -
edsserver_1  |10.4.1.5 - - [01/Sep/2021 14:26:49] "POST /v2/discovery:endpoints HTTP/1.1" 200 -

```

> eds的server会收到envoy的轮询请求，原因是`envoy.yaml`定义动态从eds rest集群获取，并指定轮询的时间为5s.

获取集群情况, 可以看到2个集群：1个是动态获取endpoint（没有endpoints）. 1个是必须静态的eds.

```bash
[root@ip-10-0-0-91 kubernetes]# curl 10.4.1.5:9901/clusters
```

现在将webserver1配置提供给eds

```bash
[root@ip-10-0-0-91 eds-rest]# docker inspect eds-rest_webserver1_1  | grep IPADDRESS -i
            "IPAddress": "10.4.1.2",
                    "IPAddress": "10.4.1.2",
```

更新地址到endpoints.json

```diff
[root@ip-10-0-0-91 eds-rest]# cat resources/endpoints.json
{
  "hosts": [
    {
+      "ip_address": "10.4.1.2",
      "port": 8081,
      "tags": {
        "az": "cn-north1-a",
        "canary": false,
        "load_balancing_weight": 50
      }
    }
  ]
}
```

把内容提交给rest-json

```bash
[root@ip-10-0-0-91 eds-rest]# docker exec eds-rest_edsserver_1 ifconfig
eth0      Link encap:Ethernet  HWaddr 92:66:DF:78:D5:EF  
          inet addr:10.4.1.4  Bcast:10.4.1.255  Mask:255.255.255.0
[root@ip-10-0-0-91 eds-rest]# docker exec eds-rest_edsserver_1 netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN   
```

```bash
[root@ip-10-0-0-91 eds-rest]# curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d @resources/endpoints.json http://10.4.1.4:8080/edsservice/myservice
{
    "hosts": [
        {
            "ip_address": "10.4.1.2",
            "port": 8081,
            "tags": {
                "az": "cn-north1-a",
                "canary": false,
                "load_balancing_weight": 50
            }
        }
    ]
}
```

再次验证cluster的endpoints, 可以发现已经加载到webserver1 endpoints.

```bash
curl 10.4.1.5:9901/clusters

[root@ip-10-0-0-91 eds-rest]# curl 10.4.1.5/hostname
Hostname: webserver1.
```

使用相同的方法，上传endpoints2, 需要先删除之前的配置

```bash
 curl -X DELETE http://10.4.1.4:8080/edsservice/myservice
 curl 10.4.1.5:9901/clusters
```

现在检验时，就没有了原来的端点

> 生产环境中，应该是追加而不是替换

现在应用endpoints-2.json, 就可以发现2个endpoints了，而且请求的结果就是轮询的web服务。

```bash
[root@ip-10-0-0-91 eds-rest]# curl 10.4.1.5/hostname
Hostname: webserver2.
[root@ip-10-0-0-91 eds-rest]# curl 10.4.1.5/hostname
Hostname: webserver1.
[root@ip-10-0-0-91 eds-rest]# curl 10.4.1.5/hostname
Hostname: webserver2.
[root@ip-10-0-0-91 eds-rest]# 
```

# grpc的eds

```diff

  clusters:
  - name: web-cluster-1
    connect_timeout: 0.25s
    type: EDS
    lb_policy: ROUND_ROBIN
    eds_cluster_config:
      service_name: web-cluster-1
      eds_config:
        api_config_source:
          api_type: GRPC
          rate_limit_settings:
          grpc_services:
            timeout: 
            google_grpc: 
            envoy_grpc:
              cluster_name: xds_cluster
```

> `api_type`将使用grpc
>
> `rate_limit_settings` 与grpc连接时，配置的速率限制
>
> `grpc_services` 指定grpc servers地址
>
> envoy支持2种连接grpc server的客户端，
>
> - envoy_grpc, envoy内建的，更多使用。 `envoy_grpc.cluster_name` grpc cluster地址
> - google_grpc, google的grpc客户端
>
> `timeout` 连接grpc server超时。grpc始终保持在线，连接不上需要超时。

```bash
  - name: xds_cluster
    type: STRICT_DNS
    connect_timeout: 0.25s
    http2_protocol_options: {}
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: xds_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: xds-service
                port_value: 8081
```

> grpc server
>
> 第三方可以提供grpc server，非常多。

## 架构

与rest架构相同[示例架构](#示例架构)

## 准备配置

```bash
git clone https://github.com/slcnx/servicemesh_in_practise.git
```

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# tree eds-grpc/
eds-grpc/
├── docker-compose.yaml
├── Dockerfile-envoy
├── envoy.yaml
└── resources
    ├── endpoints-2.json
    └── endpoints.json
```

> `envoy.yaml` 
>
> - 配置2个集群，1个是 静态配置的web-cluster-1, 其中的endpoints采用grpc (http2)动态获取, 另一个集群就是grpc cluster, 只能基于非api_type的方式。此处采用STRICT_DNS。
> - `node.id` 前缀非常重要。要么frontproxy(router)/sidecar  , **sxds必须使用sidecar.** 当前docker-compose编排的envoy只是一个前端代理。
>
> `docker-compose.yaml` 中
>
> - 对xds server(3 rd支持的grpc server)配置了别名: `xds-service`, 即上面strict_dns指向的名称，envoy将自动发现grpc server端点。xds对应的github.com地址[ikubernetes/sxds - Docker Image | Docker Hub](https://hub.docker.com/r/ikubernetes/sxds) ， 
> - `8082` 接收用户提交的配置。`8081` envoy http2双向通信的接口。
>
> `endpoints.json` 仍然不支持 discovery response格式， 是支持sxds上传的资源格式。
>
> ```yaml
> [root@ip-10-0-0-91 eds-grpc]# cat resources/endpoints.json  | yq -y
> version: '1.0'
> endpoints:
>   - cluster_name: web-cluster-1
>     endpoints:
>       - lb_endpoints:
>           - endpoint:
>               address:
>                 socket_address:
>                   address: 172.19.0.4
>                   port_value: 8081
> ```

## 启动

```bash
[root@ip-10-0-0-91 eds-grpc]# docker-compose up
loading 2 cluster(s)
loading 1 listener(s)
```

了解集群状态, 已经发现sxds(grpc server)，也发现了web-cluster-1，但没有endpoints

```bash
[root@ip-10-0-0-91 eds-grpc]# docker exec  eds-grpc_envoy_1 ifconfig eth0
eth0      Link encap:Ethernet  HWaddr A6:FE:BB:8C:0B:BC  
          inet addr:10.4.3.3  Bcast:10.4.3.255  Mask:255.255.255.0
```

```bash
[root@ip-10-0-0-91 eds-grpc]# curl 10.4.3.3:9901/clusters
```

提交webserver1, 先获取server1的ip, 然后把resource.json文件的ip更新为最新的ip, 通过`PUT`方法8082端口上传文件

```bash
[root@ip-10-0-0-91 ~]# docker exec eds-grpc_xdsserver_1 ifconfig
eth0      Link encap:Ethernet  HWaddr AA:3E:81:CB:47:73  
          inet addr:10.4.3.9  Bcast:10.4.3.255  Mask:255.255.255.0
```

```bash
[root@ip-10-0-0-91 eds-grpc]# curl -X PUT -d @resources/endpoints.json  http://10.4.3.9:8082/resources/sidecar
true
```

> 一旦true之后，xds将缓存资源，见日志`xdsserver_1  |2021-09-02T11:25:48.847Z	INFO	cacher/server.go:91	Successfully cache resources	{"node type": "sidecar"}` 
>
> 并发现新版本`xdsserver_1   | SnapshotCache: respond open watch 1[web-cluster-1] with new version "1.0"`

现在通过集群状态接口可以看到webserver1端点

```bash
[root@ip-10-0-0-91 ~]# curl 10.4.3.7:9901/clusters

[root@ip-10-0-0-91 eds-grpc]# curl 10.4.3.7/hostname
Hostname: 56a66b34504d.
```

同理，获取webserver2 ip, 更新ip于endpoints-v2.json, 提交后，查看hostname, envoy代理是roundrobin的。

```bash
[root@ip-10-0-0-91 eds-grpc]# curl -X PUT -d @resources/endpoints-v2.json  http://10.4.3.9:8082/resources/sidecar
true
```

> PUT方法，不需要像rest-json的eds先DELETE后POST，可以直接更新。

```bash
[root@ip-10-0-0-91 eds-grpc]# curl 10.4.3.7/hostname
```



# lds + rds +  cds + eds 均采用api_type

先加载lds-> rds. 而后cds->eds

```diff
node:
  id: sidecar-001
dynamic_resources:
  lds_config:
    api_config_source:
      api_type: GRPC
      grpc_services:
        envoy_grpc:
          cluster_name: xds_cluster
  cds_config:
    api_config_source:
      api_type: GRPC
      grpc_services:
        envoy_grpc:
          cluster_name: xds_cluster

```

> **`node.id`  在api_type为grpc时，必须是sidecar**
>
> lds使用grpc, cds使用grpc
>
> lds获取的配置中包含rds定义。rds也使用api_type，也会引用xds_clsuter
>
> cds获取的配置中包含eds定义。

```diff
static_resources:
  clusters:
  - name: xds_cluster
    connect_timeout: 10s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      cluster_name: xds_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
+                address: xds-service # 可以获取LDS, CDS,EDS
                port_value: 8081
```

> grpc必须静态配置，不可以再使用api_type



## 架构

与rest架构相同[示例架构](#示例架构)

## 准备配置

```bash
git clone https://github.com/slcnx/servicemesh_in_practise.git
```

```bash
[root@ip-10-0-0-91 servicemesh_in_practise-master]# tree lds-cds-grpc/
lds-cds-grpc/
├── docker-compose.yaml
├── Dockerfile-envoy
├── envoy.yaml
└── resources
    ├── all-dynamic.json
    └── all-dynamic-v2.json

```

> `all-dynamic.json` 定义了listeners, rds(grpc), clusters, eds(grpc), 同时也定义了从xdsserver获取的rds配置及endpoints配置。
>
> ```bash
> [root@ip-10-0-0-91 lds-cds-grpc]# yq -y < resources/all-dynamic.json 
> version: '1.0'
> listeners:
>   - name: listener-http
>     address:
>       socket_address:
>         address: 0.0.0.0
>         port_value: 80
>     filter_chains:
>       - filters:
>           - name: envoy.http_connection_manager
>             config:
>               codec_type: auto
>               stat_prefix: egress_http
>               idle_timeout: 60s
>               drain_timeout: 5s
>               rds:
>                 config_source:
>                   api_config_source:
>                     api_type: GRPC
>                     grpc_services:
>                       - envoy_grpc:
>                           cluster_name: xds_cluster
>                 route_config_name: local_route
>               http_filters:
>                 - name: envoy.router
>                   config: {}
> clusters:
>   - name: local_service
>     connect_timeout: 0.25s
>     lb_policy: ROUND_ROBIN
>     type: EDS
>     eds_cluster_config:
>       eds_config:
>         api_config_source:
>           api_type: GRPC
>           grpc_services:
>             - envoy_grpc:
>                 cluster_name: xds_cluster
> endpoints:
>   - cluster_name: local_service
>     endpoints:
>       - lb_endpoints:
>           - endpoint:
>               address:
>                 socket_address:
>                   address: 172.20.0.4
>                   port_value: 8081
> routes:
>   - name: local_route
>     virtual_hosts:
>       - name: service
>         domains:
>           - '*'
>         routes:
>           - match:
>               prefix: /
>             route:
>               cluster: local_service
> ```

> **如果新加了listeners，会先把旧的连接全部释放完，才会添加新的listener.**

## 启动

```bash
[root@ip-10-0-0-91 lds-cds-grpc]# docker-compose up
loading 1 cluster(s)
loading 0 listener(s)
```

> 这次边监听都没有了，仅提供一个xds集群



查看当前集群状态， only xds server在线，不能发现其他集群了

```bash
root@servicemesh:~# curl 172.20.0.3:9901/clusters
```

也没有监听器

```bash
curl 172.20.0.3:9901/listeners
```

添加

```bash
root@servicemesh:~/servicemesh_in_practise/lds-cds-grpc/resources# curl -X PUT -d @all-dynamic.json http://172.20.0.2:8082/resources/sidecar
true

```

添加整个envoy配置，可以看到clusters, 2个。一个local_service和sxds,     local_service通过grpc获取xds的endpoints.

```bash
root@servicemesh:~/servicemesh_in_practise/lds-cds-grpc/resources# curl 172.20.0.3:9901/clusters
```

查看反代成功了, 到的id正好是web server1的容器id

```bash
root@servicemesh:~/servicemesh_in_practise/lds-cds-grpc/resources# curl 172.20.0.3/hostname
Hostname: d0445005c7c3.

```

使用相同的方式，可以添加另一个enpoints. 结果也获取到了

```bash
curl -X PUT -d @resources/all-dynamic-v2.json http://172.19.0.2:8082/resources/sidecar

[root@ip-10-0-0-91 lds-cds-grpc]# curl 172.19.0.3/hostname
Hostname: 8d01012237c2.
[root@ip-10-0-0-91 lds-cds-grpc]# curl 172.19.0.3/hostname
Hostname: 4a7cb00fe9a3.
```

# ads配置

避免流量丢弃

```yaml
node:
  id: ...
dynamic_resources:
  lds_config: {ads: {}}
  cds_config: {ads: {}}
  ads_config:
    api_type: GRPC
    grpc_services:
      envoy_grpc:
        cluster_name: ads_cluster
        
static_resources:
  clusters:
  - name: ads_cluster
    connect_timeout: {seconds: 5}
    type: STATIC
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    hosts:
    - socket_address:
      address:
      port_value:
    upstream_connection_options:
      tcp_keepalive:
        ...
admin: 
```

> LDS配置中，之前引用`lds_config.api_config_source`, 现在使用ads时，仅需要`lds_config: {ads: {}}` 这里就是留空。它将会自动加载ads_config的配置定义。

lds/.../...和ads可以混合使用，建议全部使用ads, 方便冗余，而且有防流量丢弃的**聚合功能**。

