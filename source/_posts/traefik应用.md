---
title: traefik应用
date: 2021-08-19 08:44:29
tags:
- 个人日记
---

# 前言
<!--more-->

在kubernetes中直接使用nginx作为代理时，configmap不能动态加载。使用nginx-ingress-controller就可以基于ingress动态生成代理配置。

官方文档: https://doc.traefik.io/



<!--more-->
# 安装

在v1.7版本文档中，说明了daemonset的安装方式: https://doc.traefik.io/traefik/v1.7/user-guide/kubernetes/

```bash
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v1.7/examples/k8s/traefik-ds.yaml
```

> daemonset每新加一个满足selector节点，就自动创建一个守护应用。
>
> 并且支持滚动更新



# traefik

traefik是开源的边缘路由，其实就是一个代理，它接管外部的请求，并判断由后端哪个服务来处理这个请求。如果学过nginx的代理

```nginx
location /abc {
    proxy_pass http://upstream_servers;
}
```

> 同样是对指定请求进先走转发到后端，但是nginx只能基于port检测后端是否正常。
>
> traefik同样支持nginx的权重路由、负载均衡、认证，另外也支持一些更高级的：指定向后端添加路径、权重路由、镜像路径、header路径、熔断器、链路追踪、自动重载配置、基于架构自动发现配置（Kubernetes, Docker, Docker Swarm, AWS, Mesos, Marathon, and [the list goes on](https://doc.traefik.io/traefik/providers/overview/); ）。

traefik对应用本身的配置entrypoint, 全局ssl, 日志， tracing, 通过--configFile指定的文件中的配置，就算挂载了，也需要重启。

traefik对应用的动态配置，路由规则、服务、定义的选项，可以动态配置

## based on the [path](https://doc.traefik.io/traefik/routing/routers/#rule), the [host](https://doc.traefik.io/traefik/routing/routers/#rule), [headers](https://doc.traefik.io/traefik/routing/routers/#rule), [and so on](https://doc.traefik.io/traefik/routing/routers/#rule) ...

![The Door to Your Infrastructure](http://myapp.img.mykernel.cn/traefik-concepts-1.png)

## 同时支持多个provider

代理也需要配置，默认是文件配置（即file provider，文件提供配置），只要你修改文件，traefik将自动重载配置。

也可以使用其他的配置提供: Kubernetes, Docker, Docker Swarm, AWS, Mesos, Marathon, and [the list goes on](https://doc.traefik.io/traefik/providers/overview/);

每种自动发现配置的provider,怎么配置，参考：https://doc.traefik.io/traefik/providers/overview/

![Providers](http://myapp.img.mykernel.cn/providers.png)

## 一次请求的过程

![img](http://myapp.img.mykernel.cn/architecture-overview.png)

- entrypoints: 监听端口，接受入站的流量。通常，你在启动traefik时，定义多个port number.
- routes: 基于(host, path, headers, ssl,...)解析请求，看看请求匹配哪组rules. 如果匹配了请求，在转发到services前，先使用中间件处理请求(authentication, rate limiting, headers, ...)。
- services: 真正将请求转发到你的服务(load balancing,  based weight  balancing, mirroring balancing, ...)

## entrypoints

定义到达traefik的网络端点，此端口会接收packets, 可以监听TCP/UDP.

配置方法: https://doc.traefik.io/traefik/routing/entrypoints/#configuration-examples

![entryPoints](http://myapp.img.mykernel.cn/entrypoints.png)

### 配置

```yaml
## Static configuration
entryPoints:
  web:
    address: ":80" # [host]:port[/tcp|/udp]
    http: # 仅适用于http路由
      redirections: # permanent redirect. 全栈SSL
        entryPoint:
          to: websecure # 端口或entrypoint name
          scheme: https # 重定向目标协议
          permanent: true # 默认true
          priority: 10 # 生成路由的优先级，默认1

  websecure:
    address: ":443"
    http:  # 仅适用于http路由
      middlewares: # 所有路由的中间件
        - auth@file
        - strip@file
      tls: # 所有关联此端点的默认tls配置。通配证书可以使用。
        options: foobar
        certResolver: leresolver
        domains:
          - main: example.com
            sans:
              - foo.example.com
              - bar.example.com
          - main: test.com
            sans:
              - foo.test.com
              - bar.test.com
       
  streaming:
    address: ":1704/udp"
  
  name:
    address: ":8888" # same as ":8888/tcp"
    transport: # 
      lifeCycle: # 控制traefik关闭阶段的行为。
        requestAcceptGraceTimeout: 42 # 保持正在访问的请求，优雅终止前的的时间。默认0s，traefik会立即终止。滚动更新traefik时，关闭请求。
        graceTimeOut: 42              # 停止时，来了新请求的，给新处理请求的时间。默认10s.
      respondingTimeouts: # 入站请求到traefik后，经过多久时间不响应，将超时。UDP的entrypoints不生效。
        readTimeout: 42 # 读entire请求（header & body), 超时。
        writeTimeout: 42 # 发送响应超时
        idleTimeout: 42 # 客户端保持连接超时
    proxyProtocol: # traefik支持v1, v2版本的协议。如果代理协议header传递，version自动确定。
      insecure: true
      trustedIPs: # 对信任的ip启用代理协议。会透传客户源地址。建议将负载均衡地址放在这儿。
        - "127.0.0.1"
        - "192.168.0.1"
    forwardedHeaders: # 通常在代理器后时，traefik定义哪些主机转发过来的 X-Forwarded-*, 被traefik信任。
      insecure: true # 定义此时，下面的trustedIPs,将不生效，所有转发来的首部都生效。
      trustedIPs:
        - "127.0.0.1"
        - "192.168.0.1"
    
```



## routes

负责将入站请求与能处理请求的services关联。关联过程中，会使用middleware更新请求，再转发到service.

https://doc.traefik.io/traefik/routing/routers/

![routers](http://myapp.img.mykernel.cn/routers.png)

### http路由

```yaml
## Dynamic configuration
http:
  routers:
    Router-1:
      entryPoints: #  不指定时，将接受所有entrypoints的请求。
        - "websecure"
        - "other"
      rule: "Host(`example.com`)"  # 请求匹配时，路由生效。并调用middleware，再转发到service.
      service: "service-1" # 每一个请求实际由一个service处理。通常这个服务必须事先定义，除了，参考下面的说明
      priority: 1 # 默认没有此选项时或为0时，基于rules的长度逆序order，越长的先匹配。如果有此选项，此数值就是路径的长度。
      middlewares: # 路由匹配时生效。更多中间件参考: https://doc.traefik.io/traefik/middlewares/overview/
      - authentication
      tls: {} # 此选项指定时，traefik会把此路由专用于https请求。
           # 更多tls选项的子选项，参考：https://doc.traefik.io/traefik/routing/routers/#tls
```

> **service** exceptions for label-based providers. See the specific [docker](https://doc.traefik.io/traefik/routing/providers/docker/#service-definition), [rancher](https://doc.traefik.io/traefik/routing/providers/rancher/#service-definition), or [marathon](https://doc.traefik.io/traefik/routing/providers/marathon/#service-definition) documentation.

如果同时需要http&https

```yaml
## Dynamic configuration
http:
  routers:
    my-https-router:
      rule: "Host(`foo-domain`) && Path(`/foo-path/`)"
      service: service-id
      # will terminate the TLS request
      tls: {}

    my-http-router:
      rule: "Host(`foo-domain`) && Path(`/foo-path/`)"
      service: service-id
```

[http跳转https](https://doc.traefik.io/traefik/migration/v1-to-v2/#http-to-https-redirection-is-now-configured-on-routers)

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: http-redirect-ingressroute

spec:
  entryPoints:
    - web
  routes:
    - match: Host(`example.net`)
      kind: Rule
      services:
        - name: whoami
          port: 80
      middlewares:
        - name: https-redirect

---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: https-ingressroute

spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`foo`)
      kind: Rule
      services:
        - name: whoami
          port: 80
  tls: {}

---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: https-redirect
spec:
  redirectScheme:
    scheme: https
    permanent: true
```





### tcp路由

如果http路由和tcp路由同时监听在相同的entrypoints, tcp路由优先于http路由，如果tcp路由不匹配，再由http路由接管。

> nginx中，相同端口要么只能监听tcp,要么只能监听http.

```yaml
## Dynamic configuration

tcp:
  routers:
    Router-1:
      entryPoints: #  不指定时，将接受所有entrypoints的请求。
      rule: "HostSNI(`example.com`)" # 仅tls路由才需要写，非tls路由可以写 *表示每一个域名
      service: "service-1" # 同上面的http路由的service定义。
      tls: 
      	passthrough: true # 表明是否透传tls
      	options:  # 仅HostSNI定义时，才生效。https://doc.traefik.io/traefik/routing/routers/#options_1
      	certResolver: foo # 参考下面的说明
      	domains: # 参考下面的说明
          - main: "snitest.com"
            sans:
              - "*.snitest.com"
       # 同http的tls定义 。
```

> certResolver： See [`certResolver` for HTTP router](https://doc.traefik.io/traefik/routing/routers/#certresolver) for more information.
>
> domains： See [`domains` for HTTP router](https://doc.traefik.io/traefik/routing/routers/#domains) for more information.

### udp路由

```yaml
## Dynamic configuration
udp:
  routers:
    Router-1:
      entryPoints: #  不指定时，将接受所有entrypoints的请求。
        - "streaming"
      service: "service-1" # 必须是one (and only one) UDP service 
```

## services: 

参考: https://doc.traefik.io/traefik/routing/services/

service负责配置，入站请求怎么到达真实的真正处理请求的服务。

![services](http://myapp.img.mykernel.cn/services.png)

### http 服务

负载均衡、session粘性、健康检测、向后端传递首部

权重路由和负载均衡

```yaml
## Dynamic configuration
http:
  services:
    app:
      weighted:
        services:
        - name: appv1
          weight: 3
        - name: appv2
          weight: 1

    appv1:
      loadBalancer:
        servers:
        - url: "http://private-ip-server-1/"

    appv2:
      loadBalancer:
        servers:
        - url: "http://private-ip-server-2/"
```

镜像路由

```yaml
## Dynamic configuration
http:
  services:
    mirrored-api:
      mirroring:
        service: appv1
        # maxBodySize is the maximum size allowed for the body of the request.
        # If the body is larger, the request is not mirrored.
        # Default value is -1, which means unlimited size.
        maxBodySize: 1024
        mirrors:
        - name: appv2
          percent: 10

    appv1:
      loadBalancer:
        servers:
        - url: "http://private-ip-server-1/"

    appv2:
      loadBalancer:
        servers:
        - url: "http://private-ip-server-2/"
```



#### transport: traefik与真正services通信过程中的配置

> 类似于nginx的: 转发超时, ...

使用[Transport](https://doc.traefik.io/traefik/routing/overview/#transport-configuration)配置, 启动时传递 `--serversTransport.` 是全局配置

定义transport及其特性时，参考: https://doc.traefik.io/traefik/routing/services/#serverstransport



### tcp服务

权重路由

```yaml
## Dynamic configuration
tcp:
  services:
    app:
      weighted:
        services:
        - name: appv1
          weight: 3
        - name: appv2
          weight: 1

    appv1:
      loadBalancer:
        servers:
        - address: "xxx.xxx.xxx.xxx:8080"

    appv2:
      loadBalancer:
        servers:
        - address: "xxx.xxx.xxx.xxx:8080"
```

## 日志

默认写在标准输出

(startup, configuration, events, shutdown, and so on).： https://doc.traefik.io/traefik/observability/logs/#logs

访问日志： https://doc.traefik.io/traefik/observability/access-logs/#access-logs

## metric

prometheus

https://doc.traefik.io/traefik/observability/metrics/overview/

- 可以获取http/https请求相关的统计指标，entrypoint的请求过程时间的直方图

- 后端server是否在线，以上http/htpts相关指标

## tracing

绘制请求调用链路图，traefik默认支持[jaeger](https://doc.traefik.io/traefik/observability/tracing/jaeger/).

```bash
--tracing=true
# 再配置一个collector即可
--tracing.jaeger.collector.endpoint=http://127.0.0.1:14268/api/traces?format=jaeger.thrift
--tracing.jaeger.samplingServerURL=http://127.0.0.1:5778/sampling
```

