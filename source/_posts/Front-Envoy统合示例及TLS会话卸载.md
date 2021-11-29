---
title: Front Envoy统合示例及TLS会话卸载
date: 2021-03-17 11:01:12
tags:
- istio
---



# front envoy

在官方文档中的[sandbox](https://www.envoyproxy.io/docs/envoy/v1.17.1/start/sandboxes/)提供大量示例

图中表示在envoymesh network网络**内部，有1到多个微服务**，每个服务当作kubernetes中的Pod来看待，即**一个sidecar和一个main container**. 为了**引入网格(mesh)之外的流量**，所以在**前端添加了一个front-envoy container**

front-envoy的envoy监听在 80，对任意域，请求prefix为/service/1，代理至 service1 container的envoy listener 8000, 然后内部代理至本地的8080端口。

 front-envoy的envoy监听在 80，对任意域，请求prefix为/service/2，代理至 service2 container的envoy listener 8000, 然后内部代理至本地的8080端口。

有一个示例的yaml文件

![image-20210317191045508](http://myapp.img.mykernel.cn/image-20210317191045508.png)

> 老版本中，这里是github.com处的地址，直接在[github.com中切换至最新版本](https://github.com/envoyproxy/envoy/tree/main/examples/front-proxy)，可以看到如下内容。
>
> 3个dockerfile制作3个容器
>
> 前端，容器只有1个服务，直接dockerfile启动
>
> ```bash
> FROM envoyproxy/envoy-dev:latest
> 
> RUN apt-get update && apt-get -q install -y \
>     curl
> COPY ./front-envoy.yaml /etc/front-envoy.yaml
> RUN chmod go+r /etc/front-envoy.yaml
> CMD ["/usr/local/bin/envoy", "-c", "/etc/front-envoy.yaml", "--service-cluster", "front-proxy"]
> ```
>
> 后端，由于一个容器中是2个服务，并没有使用kubernetes的pod形式，而是基于start_service.sh脚本，里面启动进程完成 
>
> ```bash
> FROM envoyproxy/envoy-alpine-dev:latest
> 
> RUN apk update && apk add py3-pip bash curl
> RUN pip3 install -q Flask==0.11.1 requests==2.18.4
> RUN mkdir /code
> ADD ./service.py /code
> ADD ./start_service.sh /usr/local/bin/start_service.sh
> RUN chmod u+x /usr/local/bin/start_service.sh
> ENTRYPOINT ["/bin/sh", "/usr/local/bin/start_service.sh"]
> ```
>
> ```bash
> #!/bin/sh
> python3 /code/service.py &       # 模拟，后端主容器.
> envoy -c /etc/service-envoy.yaml --service-cluster "service${SERVICE_NAME}" # envoy集群名会覆盖cluster的local_service集群名
> ```
>
> docker-compose 编排
>
> ```bash
> version: "3.7"
> services:
> 
>   front-envoy:
>     build:
>       context: .
>       dockerfile: Dockerfile-frontenvoy # 直接启动一个进程
>     networks: 
>       - envoymesh
>     ports: # 暴露端口
>       - "8080:8080" # http
>       - "8443:8443" # tls
>       - "8001:8001" # admin
> 
>   service1:
>     build:
>       context: .
>       dockerfile: Dockerfile-service # 启动一个脚本，2个进程
>     volumes:
>       - ./service-envoy.yaml:/etc/service-envoy.yaml
>     networks:
>       - envoymesh
>     environment:
>       - SERVICE_NAME=1 # 环境变量，指定集群 名
> 
>   service2:
>     build:
>       context: .
>       dockerfile: Dockerfile-service
>     volumes:
>       - ./service-envoy.yaml:/etc/service-envoy.yaml
>     networks:
>       - envoymesh
>     environment:
>       - SERVICE_NAME=2
> 
> networks:
>   envoymesh: {}
> 
> ```
>
> front-envoy.yaml
>
> ```bash
> # listen 引入集群外部http流量
> 	socket_address:
>         address: 0.0.0.0
>         port_value: 8080
>         
>       - name: envoy.filters.network.http_connection_manager
>         typed_config:
>           "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
>           codec_type: AUTO
>           stat_prefix: ingress_http # http流量
> # route_config
>             name: local_route
>             virtual_hosts:
>             - name: backend
>               domains:
>               - "*" # 请求任意主机或域名。只要到达这个 8080端口
>               routes:
>               - match: 
>                   prefix: "/service/1" # 匹配
>                 route:
>                   cluster: service1 # 转发至service1集群名
>               - match:
>                   prefix: "/service/2"
>                 route:
>                   cluster: service2
>                   
>                   
> # cluster
> clusters:
>   - name: service1 # service1集群 
>     connect_timeout: 0.25s
>     type: STRICT_DNS # STRICT_DNS 动态生成endpoint
>     lb_policy: ROUND_ROBIN # 如果多个endpoint 将会RR
>     http2_protocol_options: {}
>     load_assignment:
>       cluster_name: service1 # 集群名
>       endpoints:
>       - lb_endpoints:
>         - endpoint:
>             address:
>               socket_address:
>                 address: service1 # 对应docker-compose 的service1一个服务
>                 port_value: 8000
>   - name: service2
>     connect_timeout: 0.25s
>     type: STRICT_DNS
>     lb_policy: ROUND_ROBIN
>     http2_protocol_options: {}
>     load_assignment:
>       cluster_name: service2
>       endpoints:
>       - lb_endpoints:
>         - endpoint:
>             address:
>               socket_address:
>                 address: service2
>                 port_value: 8000
>             
> ```
>
> service-envoy.yaml
>
> ```bash
>   - address:
>       socket_address:
>         address: 0.0.0.0
>         port_value: 8000
>         
> 	- name: envoy.filters.network.http_connection_manager
>         typed_config:
>           "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
>           codec_type: AUTO
>           stat_prefix: ingress_http
>           route_config:
>             name: local_route
>             virtual_hosts:
>             - name: service
>               domains:
>               - "*"
>               routes:
>               - match:
>                   prefix: "/service"
>                 route:
>                   cluster: local_service
>           http_filters:
>           
>           
> # http 8080 -> 任意域 -> /service -> local_service
> 
> 
>   clusters:
>   - name: local_service
>     connect_timeout: 0.25s
>     type: STRICT_DNS
>     lb_policy: ROUND_ROBIN
>     load_assignment:
>       cluster_name: local_service
>       endpoints:
>       - lb_endpoints:
>         - endpoint:
>             address:
>               socket_address:
>                 address: 127.0.0.1
>                 port_value: 8080
> 
> # -> local-service -> 127.0.0.1:8080
> ```





![image-20210317191317518](http://myapp.img.mykernel.cn/image-20210317191317518.png)



![image-20210317190538470](http://myapp.img.mykernel.cn/image-20210317190538470.png)



# 配置http front proxy

```bash
git clone https://github.com/envoyproxy/envoy.git
cd envoy/examples/front-proxy
```

## 准备front-proxy

### dockerfile

```bash
root@harbor01:~/envoy/examples/front-proxy# cat Dockerfile-frontenvoy
FROM envoyproxy/envoy-dev:latest

RUN apt-get update && apt-get -q install -y \
    curl
COPY ./front-envoy.yaml /etc/front-envoy.yaml
RUN chmod go+r /etc/front-envoy.yaml
CMD ["/usr/local/bin/envoy", "-c", "/etc/front-envoy.yaml", "--service-cluster", "front-proxy"]
```



### envoy.yaml

```bash
static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 8080
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/service/1"
                route:
                  cluster: service1
              - match:
                  prefix: "/service/2"
                route:
                  cluster: service2
          http_filters:
          - name: envoy.filters.http.router
            typed_config: {}

  clusters:
  - name: service1
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      cluster_name: service1
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: service1
                port_value: 8000
  - name: service2
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      cluster_name: service2
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: service2
                port_value: 8000
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8001
```



## 准备后端

### dockerfile

```bash
root@harbor01:~/envoy/examples/front-proxy# cat Dockerfile-service 
FROM envoyproxy/envoy-alpine-dev:latest

RUN apk update && apk add py3-pip bash curl
RUN pip3 install -q Flask==0.11.1 requests==2.18.4
RUN mkdir /code
ADD ./service.py /code
ADD ./start_service.sh /usr/local/bin/start_service.sh
RUN chmod u+x /usr/local/bin/start_service.sh
ENTRYPOINT ["/bin/sh", "/usr/local/bin/start_service.sh"]
```



### envoy.yaml

```bash
root@harbor01:~/envoy/examples/front-proxy# cat service-envoy.yaml 
static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 8000
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: service
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/service"
                route:
                  cluster: local_service
          http_filters:
          - name: envoy.filters.http.router
            typed_config: {}
  clusters:
  - name: local_service
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: local_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 8080
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8081
```

### service.py

```bash
from flask import Flask
from flask import request
import os
import requests
import socket
import sys

app = Flask(__name__)

TRACE_HEADERS_TO_PROPAGATE = [
    'X-Ot-Span-Context',
    'X-Request-Id',

    # Zipkin headers
    'X-B3-TraceId',
    'X-B3-SpanId',
    'X-B3-ParentSpanId',
    'X-B3-Sampled',
    'X-B3-Flags',

    # Jaeger header (for native client)
    "uber-trace-id",

    # SkyWalking headers.
    "sw8"
]


@app.route('/service/<service_number>')
def hello(service_number):
  return ('Hello from behind Envoy (service {})! hostname: {} resolved'
          'hostname: {}\n'.format(os.environ['SERVICE_NAME'], socket.gethostname(),
                                  socket.gethostbyname(socket.gethostname())))


@app.route('/trace/<service_number>')
def trace(service_number):
  headers = {}
  # call service 2 from service 1
  if int(os.environ['SERVICE_NAME']) == 1:
    for header in TRACE_HEADERS_TO_PROPAGATE:
      if header in request.headers:
        headers[header] = request.headers[header]
    requests.get("http://localhost:9000/trace/2", headers=headers)
  return ('Hello from behind Envoy (service {})! hostname: {} resolved'
          'hostname: {}\n'.format(os.environ['SERVICE_NAME'], socket.gethostname(),
                                  socket.gethostbyname(socket.gethostname())))


if __name__ == "__main__":
  app.run(host='127.0.0.1', port=8080, debug=True)
```

来自`/service/<service_number>`的请求，返回 环境变量 主机名 ..

监听在127.0.0.1:8080

### start_service.sh

```bash
#!/bin/sh
python3 /code/service.py &
envoy -c /etc/service-envoy.yaml --service-cluster "service${SERVICE_NAME}"
```

覆盖local_service集群名



## 准备docker-compose.yaml

```bash
version: "3.3"
services:

  front-envoy:
    build:
      context: .
      dockerfile: Dockerfile-frontenvoy
    networks:
      - envoymesh
    ports:
      - "8080:8080"
      - "8443:8443"
      - "8001:8001"

  service1:
    build:
      context: .
      dockerfile: Dockerfile-service
    volumes:
      - ./service-envoy.yaml:/etc/service-envoy.yaml
    networks:
      - envoymesh
    environment:
      - SERVICE_NAME=1

  service2:
    build:
      context: .
      dockerfile: Dockerfile-service
    volumes:
      - ./service-envoy.yaml:/etc/service-envoy.yaml
    networks:
      - envoymesh
    environment:
      - SERVICE_NAME=2

networks:
  envoymesh: {}
```

变量用于`Dockerfile-service`引用脚本`start_service.sh ` 中引入的变量

## 启动

```bash
root@harbor01:~/envoy/examples/front-proxy# docker-compose up
service1_1     | [2021-03-17 11:52:05.127][8][info][main] [source/server/server.cc:497] admin address: 0.0.0.0:8081
service1_1     | [2021-03-17 11:52:05.129][8][info][config] [source/server/configuration_impl.cc:91] loading 1 cluster(s)
service1_1     | [2021-03-17 11:52:05.131][8][info][config] [source/server/configuration_impl.cc:95] loading 1 listener(s)
service1_1     | [2021-03-17 11:52:05.141][8][info][upstream] [source/common/upstream/cluster_manager_impl.cc:188] cm init: all clusters initialized

```

## 了解运行特性

```bash
root@harbor01:~# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                                                                               NAMES
c9ee0de8305a   frontproxy_front-envoy   "/docker-entrypoint.…"   12 minutes ago   Up 12 minutes   0.0.0.0:8001->8001/tcp, 0.0.0.0:8080->8080/tcp, 0.0.0.0:8443->8443/tcp, 10000/tcp   frontproxy_front-envoy_1
c34be7c593a8   frontproxy_service2      "/bin/sh /usr/local/…"   12 minutes ago   Up 12 minutes   10000/tcp                                                                           frontproxy_service2_1
abce1b73cb94   frontproxy_service1      "/bin/sh /usr/local/…"   12 minutes ago   Up 12 minutes   10000/tcp                                                                           frontproxy_service1_1
```

### http请求 /

无结果，原因只定义了prefix `/service/1`或`/service/2`

```diff
        port_value: 8080
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/service/1"
                route:
                  cluster: service1
              - match:
                  prefix: "/service/2"
                route:
                  cluster: service2
```



![image-20210317200800691](http://myapp.img.mykernel.cn/image-20210317200800691.png)



### http请求/service/1

![image-20210317200851070](http://myapp.img.mykernel.cn/image-20210317200851070.png)

service 1即环境变量传递1，则是docker-compose的service1

```bash
  service1:
    build:
      context: .
      dockerfile: Dockerfile-service
    volumes:
      - ./service-envoy.yaml:/etc/service-envoy.yaml
    networks:
      - envoymesh
    environment:
      - SERVICE_NAME=1
```



hostname即是service1对应的containerid

```bash
abce1b73cb94   frontproxy_service1      "/bin/sh /usr/local/…"   12 minutes ago   Up 12 minutes   10000/tcp                                                                           frontproxy_service1_1
```

ip就是的ip

日志

```bash
service1_1     | 127.0.0.1 - - [17/Mar/2021 12:07:53] "GET /service/1 HTTP/1.1" 200 -
```



### 请求/service/2

![image-20210317200909654](http://myapp.img.mykernel.cn/image-20210317200909654.png)

与/service/1同理

日志

```bash
service2_1     | 127.0.0.1 - - [17/Mar/2021 12:09:04] "GET /service/2 HTTP/1.1" 200 -
```



# 配置https front proxy

基于以上的http front proxy, 所以仅修改即可

## 准备front-envoy.yaml

```diff
static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
+        port_value: 80
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
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
                redirect:
+                  path_redirect: "/"
+                  https_redirect: true
+                  port_redirect: 8443
          http_filters:
          - name: envoy.filters.http.router
            typed_config: {}

  - address:
      socket_address:
        address: 0.0.0.0
+        port_value: 443
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
              - "*"
              routes:
              - match:
+                  prefix: "/service/1"
                route:
+                  cluster: service1
              - match:
+                  prefix: "/service/2"
                route:
+                  cluster: service2
          http_filters:
          - name: envoy.filters.http.router
            typed_config: {}

+      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
+          common_tls_context: # 常规tls证书设置
+            tls_certificates: # cert/key配置
            # The following self-signed certificate pair is generated using: 自签证书的方式
+            # $ openssl req -x509 -newkey rsa:2048 -keyout a/front-proxy-key.pem -out  a/front-proxy-crt.pem -days 3650 -nodes -subj '/CN=front-envoy'
            #
            # Instead of feeding it as an inline_string, certificate pair can also be fed to Envoy
            # via filename. Reference: https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#config-core-v3-datasource.
            #
            # Or in a dynamic configuration scenario, certificate pair can be fetched remotely via
            # Secret Discovery Service (SDS). Reference: https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret.
+            - certificate_chain:
+                filename: /etc/envoy/certs/front-proxy-crt.pem #证书路径, 后期tls过期，需要envoy hot restart

+              private_key:
+                filename: /etc/envoy/certs/front-proxy-key.pem #key路径

+              #password:  # key的密码
+                #filename: # key密码文件路径 
+            #require_client_certificate: # 是否验证client证书
+            #tls_certificate_sds_secret_configs: [] # SDS API获取tls信息, 

  clusters:
+  - name: service1
    connect_timeout: 0.25s
+    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
+      cluster_name: service1
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
+                address: service1
+                port_value: 8000
  - name: service2
    connect_timeout: 0.25s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      cluster_name: service2
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: service2
                port_value: 8000
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8001
```

容器监听80/443, docker-compose映射过来就是80/443

用户请求80就重定向到443

tls证书要么SDS动态，要么静态配置，静态配置

- 写密钥
- 引用文件

strict_dns 可以动态识别多个endpoint, 此处引用docker-compose的服务名

## 准备docker-compose.yaml

自签证书

```bash
root@harbor01:~/envoy/examples/front-proxy# install -dv certs
install: 正在创建目录'certs'
root@harbor01:~/envoy/examples/front-proxy# cd certs
root@harbor01:~/envoy/examples/front-proxy/certs# openssl req -x509 -newkey rsa:2048 -keyout front-proxy-key.pem -out  front-proxy-crt.pem -days 3650 -nodes -subj '/CN=front-envoy'
Can't load /root/.rnd into RNG
140354475307456:error:2406F079:random number generator:RAND_load_file:Cannot open file:../crypto/rand/randfile.c:88:Filename=/root/.rnd
Generating a RSA private key
.............................+++++
................................................+++++
writing new private key to 'front-proxy-key.pem'
-----

#/etc/envoy/certs/front-proxy-crt.pem
#/etc/envoy/certs/front-proxy-key.pem
```

添加权限

```bash
chmod 644 certs/*
```



```diff
version: "3.3"
services:

  front-envoy:
    build:
      context: .
      dockerfile: Dockerfile-frontenvoy
    networks:
      - envoymesh
    ports:
+      - "8080:80"
+      - "8443:443"
      - "8001:8001"
    volumes:
+ 	- ./front-envoy.yaml:/etc/front-envoy.yaml                            
+   - ./certs:/etc/envoy/certs/
  service1:
    build:
      context: .
      dockerfile: Dockerfile-service
    volumes:
      - ./service-envoy.yaml:/etc/service-envoy.yaml
    networks:
      - envoymesh
    environment:
      - SERVICE_NAME=1

  service2:
    build:
      context: .
      dockerfile: Dockerfile-service
    volumes:
      - ./service-envoy.yaml:/etc/service-envoy.yaml
    networks:
      - envoymesh
    environment:
      - SERVICE_NAME=2

networks:
  envoymesh: {}
```

## 启动

```bash
root@harbor01:~/envoy/examples/front-proxy# docker-compose up
admin address: 0.0.0.0:8001

front-envoy_1  | [2021-03-17 12:26:52.275][1][info][config] [source/server/configuration_impl.cc:91] loading 2 cluster(s)
front-envoy_1  | [2021-03-17 12:26:52.278][1][info][config] [source/server/configuration_impl.cc:95] loading 1 listener(s)
front-envoy_1  | [2021-03-17 12:26:52.287][1][info][upstream] [source/common/upstream/cluster_manager_impl.cc:188] cm init: all clusters initialized

```

## 了解运行特性

### 请求8080

```bash
root@harbor01:~# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED              STATUS              PORTS                                                                            NAMES
2df78cefaff4   frontproxy_front-envoy   "/docker-entrypoint.…"   About a minute ago   Up About a minute   0.0.0.0:8001->8001/tcp, 10000/tcp, 0.0.0.0:8080->80/tcp, 0.0.0.0:8443->443/tcp   frontproxy_front-envoy_1
c34be7c593a8   frontproxy_service2      "/bin/sh /usr/local/…"   36 minutes ago       Up About a minute   10000/tcp                                                                        frontproxy_service2_1
abce1b73cb94   frontproxy_service1      "/bin/sh /usr/local/…"   36 minutes ago       Up About a minute   10000/tcp                                                                        frontproxy_service1_1
```

![image-20210317204452723](http://myapp.img.mykernel.cn/image-20210317204452723.png)

发生301重定向，8443 / 但是 没有prefix为/所以失败。生产中应该直接重写443

![image-20210317204510543](http://myapp.img.mykernel.cn/image-20210317204510543.png)

### 请求8443

![image-20210317204546703](http://myapp.img.mykernel.cn/image-20210317204546703.png)

![image-20210317204601153](http://myapp.img.mykernel.cn/image-20210317204601153.png)

```bash
service1_1     | 127.0.0.1 - - [17/Mar/2021 12:43:34] "GET /service/1 HTTP/1.1" 200 -
service1_1     | 127.0.0.1 - - [17/Mar/2021 12:45:42] "GET /service/1 HTTP/1.1" 200 -
service2_1     | 127.0.0.1 - - [17/Mar/2021 12:45:55] "GET /service/2 HTTP/1.1" 200 -
```



### admin接口

```bash
root@harbor01:~/envoy/examples/front-proxy# curl -s localhost:8001/server_info | grep service_cluster
  "service_cluster": "front-proxy",

root@harbor01:~/envoy/examples/front-proxy# docker exec -it frontproxy_service2_1 wget -O - -q localhost:8081/server_info | grep service_cluster
  "service_cluster": "service2",
root@harbor01:~/envoy/examples/front-proxy# docker exec -it frontproxy_service1_1 wget -O - -q localhost:8081/server_info | grep service_cluster
  "service_cluster": "service1",
```

测试去掉加入--service-cluster，# 默认为空

```bash
root@harbor01:~/envoy/examples/front-proxy# curl -s localhost:8001/server_info | grep service_cluster
  "service_cluster": "",
root@harbor01:~/envoy/examples/front-proxy# docker exec -it frontproxy_service2_1 wget -O - -q localhost:8081/server_info | grep service_cluster
  "service_cluster": "",
root@harbor01:~/envoy/examples/front-proxy# docker exec -it frontproxy_service1_1 wget -O - -q localhost:8081/server_info | grep service_cluster
  "service_cluster": "",
```



certs信息

```bash
curl -s localhost:8001/certs
{
 "certificates": [
  {
   "ca_cert": [],
   "cert_chain": [
    {
     "path": "/etc/envoy/certs/front-proxy-crt.pem", # 路径 
     "serial_number": "7d6d5bdf430762049f53980908a16367255627ca",
     "subject_alt_names": [],
     "days_until_expiration": "3649",
     "valid_from": "2021-03-17T12:32:14Z",  # 有效
     "expiration_time": "2031-03-15T12:32:14Z" # 失效
    }
   ]
  }
 ]
}

```

