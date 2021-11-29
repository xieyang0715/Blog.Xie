---
title: "Istio Sidecar流量拦截"
date: 2021-09-26 09:35:22
tags:
- istio
---



# 概览

Sidecar CRD: 特定sidecar, 定义入站流量和出站流量

EnvoyFilter:  自定义envoy的配置文件，添加侦听器、过滤器、自定义字段的值、...。



# sidecar代理方式

## sidecar流量拦截方式

注入的sidecar实现应该交给application入站流量和经由application出站流量，都应该交给sidecar代理完成。

因此在Pod内的网络协议栈上(同享NETWORK, UTS, IPC网络名称空间)，sidecar会拦截流量。

入站或出站，iptables拦截流量，转给envoy, envoy的代理机制来代理。

## 拦截模式

- REDIRECT: 重定向模式，使用最多，稳定。
- TPROXY： 透明代理模式, 稳定性不好。

envoy 1.4之后就转换成go脚本，之前使用bash脚本。



## 流量拦截

- 入站流量

  application监听在 *:port,  而envoy监听在另一个套接字`15001`（同协议栈监听相同套接字，会冲突。），iptables拦截后给envoy, envoy再按代理规则代理到applicaton监听的套接字上。

  > envoy 支持绑定了这个套接字，但是没有监听。

- 出站的流量

  1. 7层代理：代理到后端走的127.0.0.1，并使用**原始目标集群**, sidecar外发的流量iptables不会拦截。

  2. application请求别的服务，application目标地址是外部的地址。正常的情况是 `app` 流量直接到 `iptables`， `iptables`仍会将此类流量拦截下来转交给`envoy`, `istio proxy` 经由内部的路由处理，路由给cluster, cluster作为客户端与外部服务通信。

iptables的流量拦截在`nat`表，定义`REDIRECT` `target`规则， 就可以完成流量代理



## istio的envoy sidecar

- istio-init: 隶属于init containers, 初始化容器，生成iptables规则 。
- istio-proxy: envoy代理

```bash
root@ip-10-0-0-245:~# kubectl get  pod productpage-v1-596598f447-kkqn7  -o yaml
```

```bash
  initContainers:
  - command:
    - istio-iptables
    - -p
    - "15001"
    - -z
    - "15006"
    - -u
    - "1337"
    - -m
    - REDIRECT
    - -i
    - '*'
    - -x
    - ""
    - -b
    - '*'
    - -d
    - "15020"
    image: docker.io/istio/proxyv2:1.4.0
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
      runAsNonRoot: false
      runAsUser: 0
```

> 初始化容器，在pod启动前会运行，此容器正常运行完，才会运行其他容器
>
> 依赖NET_ADMIN特性，并且需要root用户运行

> `-z 15006` REDIRECT拦截 到pod的入站流量，并转发给envoy的listener.
>
> `-p 15001` application发出的流量，被拦截转发给envoy的listener `15001`.
>
> `-m ` 流量拦截模式
>
> `b` 目标端口是哪些的这种流量就拦截，`*`目的端口是任何的相关流量都会拦截, 但是可以使用`-d`会排除某些端口不拦截。
>
> `-i` 拦截哪些目标地址，`-x`排除某些地址，空表示不排除。



在容器运行的节点之上，进入容器的网格名称空间

```bash
root@ip-10-0-0-245:~# ps -ef | grep productpage | grep envoy
1337     21193 21018  0 Sep24 ?        00:01:53 /usr/local/bin/pilot-agent proxy sidecar --domain default.svc.cluster.local --configPath /etc/istio/proxy --binaryPath /usr/local/bin/envoy --serviceCluster productpage.default --drainDuration 45s --parentShutdownDuration 1m0s --discoveryAddress istio-pilot.istio-system:15011 --zipkinAddress zipkin.istio-system:9411 --proxyLogLevel=warning --proxyComponentLogLevel=misc:error --connectTimeout 10s --proxyAdminPort 15000 --concurrency 2 --controlPlaneAuthPolicy MUTUAL_TLS --dnsRefreshRate 300s --statusPort 15020 --applicationPorts 9080 --trust-domain=cluster.local
1337     21285 21193  0 Sep24 ?        00:06:20 /usr/local/bin/envoy -c /etc/istio/proxy/envoy-rev0.json --restart-epoch 0 --drain-time-s 45 --parent-shutdown-time-s 60 --service-cluster productpage.default --service-node sidecar~10.244.0.34~productpage-v1-596598f447-kkqn7.default~default.svc.cluster.local --max-obj-name-len 189 --local-address-ip-version v4 --log-format [Envoy (Epoch 0)] [%Y-%m-%d %T.%e][%t][%l][%n] %v -l warning --component-log-level misc:error --concurrency 2
```

```bash
root@ip-10-0-0-245:~# nsenter  -t 21285 -n  iptables -t nat -S
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N ISTIO_INBOUND
-N ISTIO_IN_REDIRECT
-N ISTIO_OUTPUT
-N ISTIO_REDIRECT
-A PREROUTING -p tcp -j ISTIO_INBOUND # 在PREROUTING链添加，TCP协议就转给 ISTIO_INBOUND 链。

-A ISTIO_INBOUND -p tcp -m tcp --dport 22 -j RETURN     # 入站流量22端口, 就回到主链PREROUTING
-A ISTIO_INBOUND -p tcp -m tcp --dport 15020 -j RETURN  # 入站流量15020端口, 就回到主链PREROUTING
-A ISTIO_INBOUND -p tcp -j ISTIO_IN_REDIRECT  #入站访问非22/15020端口的请求就重定向ISTIO_IN_REDIRECT链
-A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-ports 15006 # tcp的由REDIRECT target处理，转发给15006端口
-A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -j ISTIO_IN_REDIRECT  # 如果目标地址不是127.0.0.1/32，但是也是走lo接口，这种流量也交给ISTIO_IN_REDIRECT链。 envoy通过源地址集群转发请求到后端时，收到的目标地址和端口，依然是自己。源ip:port, 也没有做变动。 直接给istio_in_redirect, 就转发给15006. 即进入envoy。说明走了lo接口，不离开当前pod, 就给了envoy.


-A OUTPUT -p tcp -j ISTIO_OUTPUT       # 出站流量，先经由OUTPUT链, tcp协议的所有请求 转给 ISTIO_OUTPUT
-A ISTIO_OUTPUT -s 127.0.0.6/32 -o lo -j RETURN               # 如果源地址是127.0.0.6/32, 且流出接口是lo就不管。意味着不重定向
-A ISTIO_OUTPUT -m owner --uid-owner 1337 -j RETURN   # 流量的属主就是1337, 就是上面envoy进程的id, 并且是OUTPUT的, 就是外发, 就不管。
-A ISTIO_OUTPUT -m owner --gid-owner 1337 -j RETURN   # 属组
-A ISTIO_OUTPUT -d 127.0.0.1/32 -j RETURN             # 到本地，也不管
-A ISTIO_OUTPUT -j ISTIO_REDIRECT                     # 其他流出流量，转向另一个自定义链。ISTIO_REDIRECT
-A ISTIO_REDIRECT -p tcp -j REDIRECT --to-ports 15001 # 出站重定向监听在15001，即envoy.
```

> productpage 内部iptables规则列表
>
> 出站：
>
> 1. 不是envoy进程(1337 uid/gid)生成的流量
> 2. 不走lo接口
> 3. 目标不是127.0.0.1
> 4. 转发给envoy的15001, 新版本使用内建的filterchain处理。
>
> 入站：
>
> 1. 请求目标不是22/15020
> 2. 出站不是127.0.0.1/32, 且走lo接口
> 3. 均会发给envoy的15006



## istio-proxy容器

- pilot-agent
  - 提供envoy配置
  - 重载envoy
- envoy
  - 基于pilot-agent的bootstrap信息启动，指定pilot地址，通过XDS API配置
  - sidecar形式，拦截流量为应用实现入站和出站代理。

### listener

envoy大量的配置，来自pilot的XDS API， 这些资源来自`virtualservice`, `destinationrule`, `gateway`和`serviceentry`提供配置，会被polit转换成envoy支持的配置。

每个envoy sidecar会创建众多的listeners资源， `15000` admin接口

```bash
root@ip-10-0-0-245:~# kubectl exec productpage-v1-596598f447-kkqn7  -c istio-proxy -- curl -s localhost:15000/listeners   
```

> https://istio.io/v1.5/pt-br/docs/ops/deployment/requirements/#ports-used-by-istio
>
> | Port  | Protocol | Used by                                                      | Description                                |
> | ----- | -------- | ------------------------------------------------------------ | ------------------------------------------ |
> | 8060  | HTTP     | Citadel                                                      | GRPC server                                |
> | 8080  | HTTP     | Citadel agent                                                | SDS service monitoring                     |
> | 9090  | HTTP     | Prometheus                                                   | Prometheus                                 |
> | 9091  | HTTP     | Mixer                                                        | Policy/Telemetry                           |
> | 9876  | HTTP     | Citadel, Citadel agent                                       | ControlZ user interface                    |
> | 9901  | GRPC     | Galley                                                       | Mesh Configuration Protocol                |
> | 15000 | TCP      | Envoy                                                        | Envoy admin port (commands/diagnostics)    |
> | 15001 | TCP      | Envoy                                                        | Envoy Outbound                             |
> | 15006 | TCP      | Envoy                                                        | Envoy Inbound                              |
> | 15004 | HTTP     | Mixer, Pilot                                                 | Policy/Telemetry - `mTLS`                  |
> | 15010 | HTTP     | Pilot                                                        | Pilot service - XDS pilot - discovery      |
> | 15011 | TCP      | Pilot                                                        | Pilot service - `mTLS` - Proxy - discovery |
> | 15014 | HTTP     | Citadel, Citadel agent, Galley, Mixer, Pilot, Sidecar Injector | Control plane monitoring                   |
> | 15020 | HTTP     | Ingress Gateway                                              | Pilot health checks                        |
> | 15029 | HTTP     | Kiali                                                        | Kiali User Interface                       |
> | 15030 | HTTP     | Prometheus                                                   | Prometheus User Interface                  |
> | 15031 | HTTP     | Grafana                                                      | Grafana User Interface                     |
> | 15032 | HTTP     | Tracing                                                      | Tracing User Interface                     |
> | 15443 | TLS      | Ingress and Egress Gateways                                  | SNI                                        |
> | 15090 | HTTP     | Mixer                                                        | Proxy                                      |
> | 42422 | TCP      | Mixer                                                        | Telemetry - Prometheus                     |

> https://istio.io/latest/docs/ops/deployment/requirements/
>
> | Port  | Protocol | Description                                                  | Pod-internal only |
> | ----- | -------- | ------------------------------------------------------------ | ----------------- |
> | 15000 | TCP      | Envoy admin port (commands/diagnostics)                      | Yes               |
> | 15001 | TCP      | Envoy outbound                                               | No                |
> | 15004 | HTTP     | Debug port                                                   | Yes               |
> | 15006 | TCP      | Envoy inbound                                                | No                |
> | 15008 | H2       | HBONE mTLS tunnel port                                       | No                |
> | 15009 | H2C      | HBONE port for secure networks                               | No                |
> | 15020 | HTTP     | Merged Prometheus telemetry from Istio agent, Envoy, and application | No                |
> | 15021 | HTTP     | Health checks                                                | No                |
> | 15053 | DNS      | DNS port, if capture is enabled                              | Yes               |
> | 15090 | HTTP     | Envoy Prometheus telemetry                                   | No                |
>
> | Port  | Protocol | Description                                                  | Local host only |
> | ----- | -------- | ------------------------------------------------------------ | --------------- |
> | 443   | HTTPS    | Webhooks service port                                        | No              |
> | 8080  | HTTP     | Debug interface (deprecated, container port only)            | No              |
> | 15010 | GRPC     | XDS and CA services (Plaintext, only for secure networks)    | No              |
> | 15012 | GRPC     | XDS and CA services (TLS and mTLS, recommended for production use) | No              |
> | 15014 | HTTP     | Control plane monitoring                                     | No              |
> | 15017 | HTTPS    | Webhook container port, forwarded from 443                   | No              |

### istio-proxy listener

- virtual outbound listener

  app外发的流量会被15001的listener拦截，会配置`use_origin_dest=true`, envoy将请求转发给同请求原目标地址关联的listener之上；

  不存在listener时，会看Istio全局配置， outboundTrafficPolicy参数的值决定

  - allow_any: 允许外发到任何服务的请求，无论目标是否在pilot注册表中，此时，没有匹配的目标Listtener的流量将由该侦听器上的tcp_proxy过滤器指向的PassthroughCluster进行透传。
  - registry_only：仅允许外发请求到注册于pilot注册表中中的服务。没有匹配的目标Listtener的流量将由该侦听器上的tcp_proxy过滤器指向 blackHoleCluster将流量直接丢弃（/dev/null)



envoy会按需为外部服务创建一个listener.

- 16686, 发给jaeger的流量
- 9080 
- 9411 zipkin
- 3000 grafana
- 15004  mixer: policy, telemetry
- 9090 发往prometheus的流量 

> 9080 业务 `bind_to_port=false` 只是监听，不会绑定。
>
> 15020 组件间检测

```yaml
root@ip-10-0-0-245:~# kubectl exec productpage-v1-596598f447-kkqn7  -c istio-proxy -- curl -s localhost:15000/config_dump | yq -y . > /tmp/productpage.yaml
```

较早版本envoy, 只有15001, 有死循环的问题

```yaml
      - version_info: 2021-09-24T01:14:18Z/31
        listener:
          name: virtualInbound
          address:
            socket_address:
              address: 0.0.0.0
              port_value: 15006
          filter_chains:
            - filter_chain_match:
                prefix_ranges: 
                  - address_prefix: 0.0.0.0
                    prefix_len: 0
              filters:
                - name: envoy.tcp_proxy
                  typed_config:
                    '@type': type.googleapis.com/envoy.config.filter.network.tcp_proxy.v2.TcpProxy
                    stat_prefix: InboundPassthroughClusterIpv4
                    cluster: InboundPassthroughClusterIpv4
              metadata:
                filter_metadata:
                  pilot_meta:
                    original_listener_name: virtualInbound
            - filter_chain_match:
                prefix_ranges: # 如果请求地址
                  - address_prefix: 10.244.0.34
                    prefix_len: 32
                destination_port: 15020 # 这个端口，就匹配这个filter chain.
              filters:
                - name: envoy.tcp_proxy # 由这个处理
                  typed_config:
                    '@type': type.googleapis.com/envoy.config.filter.network.tcp_proxy.v2.TcpProxy
                    stat_prefix: inbound|15020|mgmt-15020|mgmtCluster
                    cluster: inbound|15020|mgmt-15020|mgmtCluster
              metadata:
                filter_metadata:
                  pilot_meta:
                    original_listener_name: 10.244.0.34_15020
            - filter_chain_match:
                prefix_ranges:
                  - address_prefix: 10.244.0.34
                    prefix_len: 32
                destination_port: 9080
                application_protocols:
                  - istio
              tls_context:
                common_tls_context:
                  tls_certificates:
                    - certificate_chain:
                        filename: /etc/certs/cert-chain.pem
                      private_key:
                        filename: /etc/certs/key.pem
                  validation_context:
                    trusted_ca:
                      filename: /etc/certs/root-cert.pem
                  alpn_protocols:
                    - h2
                    - http/1.1
                require_client_certificate: true
              filters:
                - name: envoy.http_connection_manager # 9080处理
                  typed_config:
                    '@type': type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
                    stat_prefix: inbound_10.244.0.34_9080 # 入站流量
                    http_filters:
                      - name: istio_authn
                        typed_config:
                          '@type': type.googleapis.com/istio.envoy.config.filter.http.authn.v2alpha1.FilterConfig
                          policy:
                            peers:
                              - mtls:
                                  mode: PERMISSIVE
                      - name: mixer
                        typed_config:
                          '@type': type.googleapis.com/istio.mixer.v1.config.client.HttpClientConfig
                          transport:
                            network_fail_policy:
                              policy: FAIL_CLOSE
                              base_retry_wait: 0.080s
                              max_retry_wait: 1s
                            check_cluster: outbound|15004||istio-policy.istio-system.svc.cluster.local
                            report_cluster: outbound|15004||istio-telemetry.istio-system.svc.cluster.local
                            report_batch_max_entries: 100
                            report_batch_max_time: 1s
                          service_configs:
                            default:
                              disable_check_calls: true
                          default_destination_service: default
                          mixer_attributes:
                            attributes:
                              context.proxy_version:
                                string_value: 1.4.0
                              context.reporter.kind:
                                string_value: inbound
                              context.reporter.uid:
                                string_value: kubernetes://productpage-v1-596598f447-kkqn7.default
                              destination.ip:
                                bytes_value: AAAAAAAAAAAAAP//CvQAIg==
                              destination.mesh.id:
                                string_value: cluster.local
                              destination.namespace:
                                string_value: default
                              destination.port:
                                int64_value: '9080'
                              destination.uid:
                                string_value: kubernetes://productpage-v1-596598f447-kkqn7.default
                      - name: envoy.cors
                      - name: envoy.fault
                      - name: envoy.router
                    tracing:
                      client_sampling:
                        value: 100
                      random_sampling:
                        value: 1
                      overall_sampling:
                        value: 100
                    server_name: istio-envoy
                    use_remote_address: false
                    generate_request_id: true
                    forward_client_cert_details: APPEND_FORWARD
                    set_current_client_cert_details:
                      subject: true
                      dns: true
                      uri: true
                    upgrade_configs:
                      - upgrade_type: websocket
                    stream_idle_timeout: 0s
                    normalize_path: true
                    route_config:
                      name: inbound|9080|http|productpage.default.svc.cluster.local
                      virtual_hosts:
                        - name: inbound|http|9080
                          domains:
                            - '*'
                          routes:
                            - match:
                                prefix: /
                              decorator:
                                operation: productpage.default.svc.cluster.local:9080/*
                              typed_per_filter_config:
                                mixer:
                                  '@type': type.googleapis.com/istio.mixer.v1.config.client.ServiceConfig
                                  disable_check_calls: true
                                  mixer_attributes:
                                    attributes:
                                      destination.service.host:
                                        string_value: productpage.default.svc.cluster.local
                                      destination.service.name:
                                        string_value: productpage
                                      destination.service.namespace:
                                        string_value: default
                                      destination.service.uid:
                                        string_value: istio://default/services/productpage
                              name: default
                              route:
                                timeout: 0s
                                max_grpc_timeout: 0s
                                cluster: inbound|9080|http|productpage.default.svc.cluster.local
                      validate_clusters: false
              metadata:
                filter_metadata:
                  pilot_meta:
                    original_listener_name: 10.244.0.34_9080
```

出站配置

```yaml

```

clusters配置

```bash
root@ip-10-0-0-245:~# kubectl exec productpage-v1-596598f447-kkqn7  -c istio-proxy -- curl -s localhost:15000/clusters  | awk -F: '{print $1}' | sort | uniq
BlackHoleCluster # 出站请求的目标和端口，没有目标listener, 定义选项为
InboundPassthroughClusterIpv4  # 出站请求的目标和端口，没有目标listener, 定义选项为
PassthroughCluster
inbound|15020|mgmt-15020|mgmtCluster
inbound|9080|http|productpage.default.svc.cluster.local
outbound|14250||jaeger-collector.istio-system.svc.cluster.local
outbound|14267||jaeger-collector.istio-system.svc.cluster.local
outbound|14268||jaeger-collector.istio-system.svc.cluster.local
outbound|15004||istio-policy.istio-system.svc.cluster.local
outbound|15004||istio-telemetry.istio-system.svc.cluster.local
outbound|15010||istio-pilot.istio-system.svc.cluster.local
outbound|15011||istio-pilot.istio-system.svc.cluster.local
outbound|15014||istio-citadel.istio-system.svc.cluster.local
outbound|15014||istio-galley.istio-system.svc.cluster.local
outbound|15014||istio-pilot.istio-system.svc.cluster.local
outbound|15014||istio-policy.istio-system.svc.cluster.local
outbound|15014||istio-telemetry.istio-system.svc.cluster.local
outbound|15019||istio-galley.istio-system.svc.cluster.local
outbound|15020||istio-ingressgateway.istio-system.svc.cluster.local
outbound|15029||istio-ingressgateway.istio-system.svc.cluster.local
outbound|15030||istio-ingressgateway.istio-system.svc.cluster.local
outbound|15031||istio-ingressgateway.istio-system.svc.cluster.local
outbound|15032||istio-ingressgateway.istio-system.svc.cluster.local
outbound|15443||istio-ingressgateway.istio-system.svc.cluster.local
outbound|16686||jaeger-query.istio-system.svc.cluster.local
outbound|20001||kiali.istio-system.svc.cluster.local
outbound|3000||grafana.istio-system.svc.cluster.local
outbound|42422||istio-telemetry.istio-system.svc.cluster.local
outbound|443||istio-galley.istio-system.svc.cluster.local
outbound|443||istio-ingressgateway.istio-system.svc.cluster.local
outbound|443||istio-sidecar-injector.istio-system.svc.cluster.local
outbound|443||kubernetes.default.svc.cluster.local
outbound|53||kube-dns.kube-system.svc.cluster.local
outbound|8000||httpbin.default.svc.cluster.local
outbound|8060||istio-citadel.istio-system.svc.cluster.local
outbound|8080||fortio.default.svc.cluster.local
outbound|8080||istio-pilot.istio-system.svc.cluster.local
outbound|80||istio-ingressgateway.istio-system.svc.cluster.local
outbound|9080|v1|details.default.svc.cluster.local
outbound|9080|v1|productpage.default.svc.cluster.local
outbound|9080|v1|ratings.default.svc.cluster.local
outbound|9080|v1|reviews.default.svc.cluster.local
outbound|9080|v2-mysql-vm|ratings.default.svc.cluster.local
outbound|9080|v2-mysql|ratings.default.svc.cluster.local
outbound|9080|v2|details.default.svc.cluster.local
outbound|9080|v2|ratings.default.svc.cluster.local
outbound|9080|v2|reviews.default.svc.cluster.local
outbound|9080|v3|reviews.default.svc.cluster.local
outbound|9080||details.default.svc.cluster.local
outbound|9080||productpage.default.svc.cluster.local
outbound|9080||ratings.default.svc.cluster.local
outbound|9080||reviews.default.svc.cluster.local
outbound|9090||prometheus.istio-system.svc.cluster.local
outbound|9091||istio-policy.istio-system.svc.cluster.local
outbound|9091||istio-telemetry.istio-system.svc.cluster.local
outbound|9153||kube-dns.kube-system.svc.cluster.local
outbound|9411||tracing.istio-system.svc.cluster.local
outbound|9411||zipkin.istio-system.svc.cluster.local
outbound|9901||istio-galley.istio-system.svc.cluster.local
prometheus_stats
xds-grpc
zipkin

```

> 所有pod中envoy均会有

passthroughcluster 集群

```yaml
      - version_info: 2021-09-24T01:14:18Z/31
        cluster:
          name: InboundPassthroughClusterIpv4
          type: ORIGINAL_DST # 原始目标集群
          connect_timeout: 1s
          lb_policy: CLUSTER_PROVIDED
          circuit_breakers: # istio叫连接池，envoy中叫熔断器。
            thresholds:
              - max_connections: 4294967295
                max_pending_requests: 4294967295
                max_requests: 4294967295
                max_retries: 4294967295
          upstream_bind_config:
            source_address:
              address: 127.0.0.6
              port_value: 0
        last_updated: '2021-09-24T01:14:22.511Z'
      - version_info: 2021-09-24T01:14:18Z/31
        cluster:
          name: PassthroughCluster
          type: ORIGINAL_DST
          connect_timeout: 1s
          lb_policy: CLUSTER_PROVIDED
          circuit_breakers:
            thresholds:
              - max_connections: 4294967295
                max_pending_requests: 4294967295
                max_requests: 4294967295
                max_retries: 4294967295
        last_updated: '2021-09-24T01:14:22.510Z'
```

product page集群

```yaml
      - version_info: 2021-09-24T01:14:18Z/31
        cluster:
          name: inbound|9080|http|productpage.default.svc.cluster.local
          type: STATIC
          connect_timeout: 1s
          circuit_breakers:
            thresholds:
              - max_connections: 4294967295
                max_pending_requests: 4294967295
                max_requests: 4294967295
                max_retries: 4294967295
          load_assignment: # 静态集群
            cluster_name: inbound|9080|http|productpage.default.svc.cluster.local
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: 127.0.0.1 # 本地的9080
                          port_value: 9080
        last_updated: '2021-09-24T01:14:22.510Z'
```



# 配置sidecar

网格不知道哪个服务到哪个服务需要通信，所以每一个sidecar, 通信与本地，通信与所有外部服务，实际通信时，不需要与所有服务通信。

可以发现网格内的所有pod中的envoy，均有所有pod集群定义。

所以需要配置 

| 名称         | 作用                    | 方法                                                         |
| ------------ | ----------------------- | ------------------------------------------------------------ |
| sidecar crd  | ingress/egress          | <p>sidecar，默认不提供workloadselector, 此sidecar将应用同一名称空间中所有workload(带了sidecar的服务网格中的服务).</p> <br/><p>而提供workloadselector时, 此envoy只会应用匹配了的workload.</p> |
| envoy filter | 细粒度调整envoy的配置。 |                                                              |

![image-20210926140907015](http://myapp.img.mykernel.cn/image-20210926140907015.png)



# EnvoyFilter CRD

修改envoy具体配置

envoyfilter, 配置复杂，用的不多

