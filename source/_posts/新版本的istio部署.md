---
title: 新版本的istio部署
date: 2021-10-09 15:39:51
tags:
- istio
---



# 前言

生产需要使用istio了，现在购买了AWS主机，需要上istio.



<!--more-->
# 部署

## 获取最新版本

https://github.com/istio/istio/tags

此处我选择1.11.3



放在/usr/local中，使用链接，方便升级

```bash
tar xvf istio-1.11.3-linux-amd64.tar.gz -C /usr/local
ln -svfT /usr/local/istio-1.11.3/ /usr/local/istio
```

导出二进制文件

```bash
cat > /etc/profile.d/istio.sh  <<EOF
export PATH=/usr/local/istio/bin:$PATH
EOF
```

查看版本

```bash
istioctl version
client version: 1.11.3
control plane version: 1.11.3
data plane version: 1.11.3 (8 proxies)
```

添加自动补全, 当 `bash-completion` 包被安装到您的 Linux 系统以后，添加下行内容到您的 `~/.bash_profile` 中：

```bash
[[ -r "/usr/local/etc/profile.d/bash_completion.sh" ]] && . "/usr/local/etc/profile.d/bash_completion.sh"
```

然后在导出istioctl自动补全

```bash
echo 'source /usr/local/istio/tools/istioctl.bash' >> /etc/profile.d/istio.sh 
```



## 部署

查看可用的配置文件

```bash
# istioctl profile list
Istio configuration profiles:
    default
    demo
    empty
    external
    minimal
    openshift
    preview
    remote
```

> 不同的配置对应不同的组件功能：https://istio.io/latest/zh/docs/setup/additional-setup/config-profiles/ 

| default                | demo | minimal | remote | empty | preview |      |
| ---------------------- | ---- | ------- | ------ | ----- | ------- | ---- |
| 核心组件               |      |         |        |       |         |      |
| `istio-egressgateway`  |      | ✔       |        |       |         |      |
| `istio-ingressgateway` | ✔    | ✔       |        |       |         | ✔    |
| `istiod`               | ✔    | ✔       | ✔      |       |         | ✔    |

> 为了进一步自定义 Istio，还可以安装一些附加组件。详情请参阅 [集成](https://istio.io/latest/zh/docs/ops/integrations)。

从此图中，可以看出egress/ingress/istiod



此处仅需要集群入口及核心组件

```bash
istioctl manifest generate  --set profile=default > /tmp/istio-default.yaml
kubectl create ns istio-system
kubectl apply -f /tmp/istio-default.yaml
```



## bookinfo示例

默认名称空间自动注入

```bash
kubectl label namespace default istio-injection=enabled
```

启动示例

```bash
cd /usr/local/istio
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

验证是否OK

```bash
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -s productpage:9080/productpage | grep -o "<title>.*</title>"
```

> 当出现输出，即OK

## 启动集群附件

 [集群附件集成](https://istio.io/latest/zh/docs/ops/integrations)

### 启动

```bash
kubectl apply -f samples/addons
```

> 1. 由于kiali默认是匿名，所以需要配置，认证
>
>    编辑 `samples/addons/kiali.yaml`
>
>    ```diff
>    data:
>      config.yaml: |
>        auth:
>    +      strategy: token
>    ```
>
>    > 此token为 类似kubernetes dashboard登陆的token

### 集群日志 stdout + 阿里云SLS

#### 接入SLS

> 1. 安装ds组件
>    https://help.aliyun.com/document_detail/28982.htm?spm=a2c4g.11186623.0.0.2c6d5b78jZ7O7Q#title-1vi-sf5-c29
>
>    > 对接有问题，原因是当sls controller检测到有docker.sock时，默认走docker
>    >
>    > ```yaml
>    > image: registry.cn-hangzhou.aliyuncs.com/log-service/logtail:latest
>    > env:
>    > - name: USE_CONTAINERD
>    >   value: "true"
>    > ```
>
> 2. 定义数据源
>
>    https://help.aliyun.com/document_detail/28979.html
>
>    https://help.aliyun.com/document_detail/104042.html
>
>    https://help.aliyun.com/document_detail/196159.html
>
>    nginx ingress controller配置: https://help.aliyun.com/document_detail/196154.html
>    
>    ```yaml
>    {
>        "inputs": [
>            {
>                "detail": {
>                    "Stderr": true,
>                    "IncludeLabel": {
>                        "io.kubernetes.container.name": "istio-proxy",
>                        "io.kubernetes.pod.namespace": "istio-system"
>                    },
>                    "IncludeEnv": {
>                    	"ISTIO_META_WORKLOAD_NAME": "istio-ingressgateway"
>                    },
>                    "Stdout": true
>                },
>                "type": "service_docker_stdout"
>            }
>        ],
>        "processors": [
>            {
>                "detail": {
>                    "SourceKey": "content",
>                    "NoKeyError":true,
>                    "KeepSource": false,
>    				"IgnoreFirstConnector": true
>                },
>                "type": "processor_json"
>            }
>        ]
>    }
>    ```
>    
>    > https://help.aliyun.com/document_detail/66658.htm?spm=a2c4g.11186623.0.0.677265a7MaXD8i#title-e2b-oee-org
>    >
>    > 比较核心的是南北流量日志，这块详细记录比较好，可以分析用户的行为和报错，kiali只记录了聚合的Metrics指标，做不到这一点。 东西向流量，如果有比较强的分析和审计需求，可以保存，大部分场景是不太需要的
>    >
>    > 如果记录东西流量时，只需要简单修改为
>    >
>    > ```yaml
>    > {
>    >     "inputs": [
>    >         {
>    >             "detail": {
>    >                 "Stderr": true,
>    >                 "IncludeLabel": {
>    >                     "io.kubernetes.container.name": "istio-proxy"
>    >                 },
>    >                 "Stdout": true
>    >             },
>    >             "type": "service_docker_stdout"
>    >         }
>    >     ],
>    >     "processors": [
>    >         {
>    >             "detail": {
>    >                 "SourceKey": "content",
>    >                 "NoKeyError":true,
>    >                 "KeepSource": false,
>    > 				"IgnoreFirstConnector": true
>    >             },
>    >             "type": "processor_json"
>    >         }
>    >     ]
>    > }
>    > ```

采集常见问题: https://help.aliyun.com/document_detail/272567.html

查看日志

- 插件、运行、记录

> 日志定义：https://help.aliyun.com/apsara/enterprise/v_3_15_0_20210816/sls/enterprise-ascm-user-guide/logtail-configuration-and-record-files.html#title-0dp-58z-jib

#### istio日志简述

> https://istio.io/latest/docs/tasks/observability/logs/access-log/

格式1

```bash
  mesh: |-
    accessLogEncoding: TEXT
    accessLogFile: /dev/stdout
    accessLogFormat: |
      [%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%" %RESPONSE_CODE% %RESPONSE_FLAGS% %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% "%REQ(X-FORWARDED-FOR)%" "%REQ(USER-AGENT)%" "%REQ(X-REQUEST-ID)%" "%REQ(:AUTHORITY)%" "%UPSTREAM_HOST%" "%DOWNSTREAM_REMOTE_ADDRESS_WITHOUT_PORT%"
```

> ```nginx
> [2021-10-19T02:24:12.504Z] "GET / HTTP/1.1" 200 - 0 130 2 2 "10.244.0.1" "curl/7.58.0" "266300ac-230b-4893-b202-221426abfa11" "test-pos.graspishop.com" "10.244.0.84:7001" "10.244.0.1"
> ```

格式2

```yaml
data:
  mesh: |-
    accessLogEncoding: JSON
    accessLogFile: /dev/stdout
    accessLogFormat: |-
      '{  
        "authority": "%REQ(:AUTHORITY)%",
        "bytes_received": "%BYTES_RECEIVED%",
        "bytes_sent": "%BYTES_SENT%",
        "downstream_local_address": "%DOWNSTREAM_LOCAL_ADDRESS%",
        "downstream_remote_address": "%DOWNSTREAM_REMOTE_ADDRESS%",
        "duration": "%DURATION%",
        "istio_policy_status": "%DYNAMIC_METADATA(istio.mixer:status)%",
        "method": "%REQ(:METHOD)%",
        "path": "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%",
        "protocol": "%PROTOCOL%",
        "request_id": "%REQ(X-REQUEST-ID)%",
        "requested_server_name": "%REQUESTED_SERVER_NAME%",
        "response_code": "%RESPONSE_CODE%",
        "response_flags": "%RESPONSE_FLAGS%",
        "route_name": "%ROUTE_NAME%",                                                                                                                      
        "start_time": "%START_TIME%",
        "upstream_cluster": "%UPSTREAM_CLUSTER%",
        "upstream_host": "%UPSTREAM_HOST%",
        "upstream_local_address": "%UPSTREAM_LOCAL_ADDRESS%",
        "upstream_service_time": "%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%",
        "upstream_transport_failure_reason": "%UPSTREAM_TRANSPORT_FAILURE_REASON%",
        "user_agent": "%REQ(USER-AGENT)%",
        "x_forwarded_for": "%REQ(X-FORWARDED-FOR)%"
      }'
```

> ```json
> {"method":"GET","bytes_received":0,"bytes_sent":130,"x_forwarded_for":"182.148.48.147","upstream_cluster":"outbound|7001|v1-0-65|ishop-pos-service.default.svc.cluster.local","path":"/","route_name":null,"downstream_remote_address":"182.148.48.147:42262","downstream_local_address":"10.244.0.5:8080","requested_server_name":null,"connection_termination_details":null,"response_code":200,"duration":2,"user_agent":"curl/7.58.0","request_id":"4f29b224-dfd6-47c8-9eac-f3226dbd227d","response_code_details":"via_upstream","response_flags":"-","upstream_local_address":"10.244.0.5:45184","protocol":"HTTP/1.1","upstream_transport_failure_reason":null,"authority":"test-pos.graspishop.com","start_time":"2021-10-19T03:59:21.556Z","upstream_host":"10.244.0.84:7001","upstream_service_time":"2"}
> ```

收集除了envoy之外的应用日志, 写到其他位置 
env通过这个： https://help.aliyun.com/document_detail/87540.html
crd的通过这个： https://help.aliyun.com/document_detail/74878.htm

#### istio配置接入SLS的日志

> https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage

```xml
# xpath
//div[@id="command-operators"]/dl/dt
```

测试

参考aliyun的nginx controller日志

```bash
        log_format upstreaminfo '$remote_addr - [$remote_addr] - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_use
r_agent" $request_length $request_time [$proxy_upstream_name] $upstream_addr $upstream_response_length $upstream_response_time $upstream_status $req_id $host [$proxy_alternative_upstream_name]';
```

> nginx日志字段说明: https://help.aliyun.com/document_detail/197675.html

```bash
istioctl apply  --set profile=default --set meshConfig.accessLogFile=/dev/stdout --set meshConfig.accessLogEncoding=JSON --set meshConfig.accessLogFormat='
{
    "remote-ip": "%DOWNSTREAM_REMOTE_ADDRESS_WITHOUT_PORT%",
    "authority": "%REQ(:AUTHORITY)%",
    "bytes_received": "%BYTES_RECEIVED%",
    "bytes_sent": "%BYTES_SENT%",
    "downstream_local_address": "%DOWNSTREAM_LOCAL_ADDRESS%",
    "downstream_remote_address": "%DOWNSTREAM_REMOTE_ADDRESS%",
    "duration": "%DURATION%",
    "istio_policy_status": "%DYNAMIC_METADATA(istio.mixer:status)%",
    "method": "%REQ(:METHOD)%",
    "path": "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%",
    "protocol": "%PROTOCOL%",
    "request_id": "%REQ(X-REQUEST-ID)%",
    "requested_server_name": "%REQUESTED_SERVER_NAME%",
    "response_code": "%RESPONSE_CODE%",
    "response_flags": "%RESPONSE_FLAGS%",
    "route_name": "%ROUTE_NAME%",
    "start_time": "%START_TIME%",
    "upstream_cluster": "%UPSTREAM_CLUSTER%",
    "upstream_host": "%UPSTREAM_HOST%",
    "upstream_local_address": "%UPSTREAM_LOCAL_ADDRESS%",
    "upstream_service_time": "%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%",
    "upstream_transport_failure_reason": "%UPSTREAM_TRANSPORT_FAILURE_REASON%",
    "user_agent": "%REQ(USER-AGENT)%",
    "referer": "%REQ(referer)%",
    "x_forwarded_for": "%REQ(X-FORWARDED-FOR)%"
  }
'
```

>  `kubectl edit cm -n istio-system istio`
>
>  ```yaml
>  accessLogEncoding: JSON
>  accessLogFile: /dev/stdout
>  accessLogFormat:  '{"remote-ip":"%DOWNSTREAM_REMOTE_ADDRESS_WITHOUT_PORT%","authority":"%REQ(:AUTHORITY)%","bytes_received":"%BYTES_RECEIVED%","bytes_sent":"%BYTES_SENT%","downstream_local_address":"%DOWNSTREAM_LOCAL_ADDRESS%","downstream_remote_address":"%DOWNSTREAM_REMOTE_ADDRESS%","duration":"%DURATION%","istio_policy_status":"%DYNAMIC_METADATA(istio.mixer:status)%","method":"%REQ(:METHOD)%","path":"%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%","protocol":"%PROTOCOL%","request_id":"%REQ(X-REQUEST-ID)%","requested_server_name":"%REQUESTED_SERVER_NAME%","response_code":"%RESPONSE_CODE%","response_flags":"%RESPONSE_FLAGS%","route_name":"%ROUTE_NAME%","start_time":"%START_TIME%","upstream_cluster":"%UPSTREAM_CLUSTER%","upstream_host":"%UPSTREAM_HOST%","upstream_local_address":"%UPSTREAM_LOCAL_ADDRESS%","upstream_service_time":"%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%","upstream_transport_failure_reason":"%UPSTREAM_TRANSPORT_FAILURE_REASON%","user_agent":"%REQ(USER-AGENT)%","referer":"%REQ(referer)%","x_forwarded_for":"%REQ(X-FORWARDED-FOR)%"}'
>  ```
>
>  > 查看日志
>  >
>    > ```bash
>    > kubectl logs --tail 20 -f  -n istio-system istio-ingressgateway-7776dd578-k6sx7 istio-proxy  | jq -r .
>    > ```

要求ingressgateway的svc配置

```bash
spec.externalTrafficPolicy: Local
```

> https://kubernetes.io/zh/docs/tutorials/services/source-ip/#type-nodeport-%E7%B1%BB%E5%9E%8B-services-%E7%9A%84-source-ip

> 为了获取源IP地址的:
>
> - `spec.externalTrafficPolicy:  Cluster` 跨多层TCP代理时，nginx的x-forward-for首部记录真实用户的IP哈，自动识别。这个有一定**开销**
> - `spec.externalTrafficPolicy: Local`  这样不需要xff了，如果关闭的话就需要ingress的xff最开销，这样会增大一定负载
>   - 集群中，通过SLB的域名访问**到相同集群内的pod**就会被 reset。仅可以通过coredns服务自动发现功能访问域名，或者使用clusterIP

示例图

![image-20211021105628097](http://myapp.img.mykernel.cn/image-20211021105628097.png)

### 暴露

https://istio.io/latest/zh/docs/tasks/observability/gateways/#option-two-insecure-access-HTTP

#### grafana

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: grafana-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 15031
      name: http-grafana
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grafana-vs
  namespace: istio-system
spec:
  hosts:
  - "*"
  gateways:
  - grafana-gateway
  http:
  - match:
    - port: 15031
    route:
    - destination:
        host: grafana
        port:
          number: 3000
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: grafana
  namespace: istio-system
spec:
  host: grafana
  trafficPolicy:
    tls:
      mode: DISABLE
---
EOF

```

#### Kiali

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: kiali-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 15029
      name: http-kiali
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: kiali-vs
  namespace: istio-system
spec:
  hosts:
  - "*"
  gateways:
  - kiali-gateway
  http:
  - match:
    - port: 15029
    route:
    - destination:
        host: kiali
        port:
          number: 20001
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: kiali
  namespace: istio-system
spec:
  host: kiali
  trafficPolicy:
    tls:
      mode: DISABLE
---
EOF

```

#### Prometheus

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: prometheus-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 15030
      name: http-prom
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: prometheus-vs
  namespace: istio-system
spec:
  hosts:
  - "*"
  gateways:
  - prometheus-gateway
  http:
  - match:
    - port: 15030
    route:
    - destination:
        host: prometheus
        port:
          number: 9090
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: prometheus
  namespace: istio-system
spec:
  host: prometheus
  trafficPolicy:
    tls:
      mode: DISABLE
---
EOF

```

#### 跟踪

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: tracing-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 15032
      name: http-tracing
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: tracing-vs
  namespace: istio-system
spec:
  hosts:
  - "*"
  gateways:
  - tracing-gateway
  http:
  - match:
    - port: 15032
    route:
    - destination:
        host: tracing
        port:
          number: 80
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: tracing
  namespace: istio-system
spec:
  host: tracing
  trafficPolicy:
    tls:
      mode: DISABLE
---
EOF

```

### 访问

通过浏览器访问这些遥测插件。

- Kiali: `http://<IP ADDRESS OF CLUSTER INGRESS>:15029/`
- Prometheus: `http://<IP ADDRESS OF CLUSTER INGRESS>:15030/`
- Grafana: `http://<IP ADDRESS OF CLUSTER INGRESS>:15031/`
- Tracing: `http://<IP ADDRESS OF CLUSTER INGRESS>:15032/`

## 灰度发布

### 流量迁移

v2版本的补丁版本发布时，先启动一个v3，然后把流量从v2慢慢迁移到v3. 最后v2提升到v3.

> 此处应该结合 flagger发版

```bash
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: bookinfo
  namespace: default
spec:
  hosts:
    - '*'
  gateways:
    - bookinfo-gateway
  http:
    - match:
        - headers:
            version:
              exact: v1
      route:
        - destination:
            host: reviews
            subset: v1
    - match:
        - headers:
            version:
              exact: v2
      route:
        - destination:
            host: reviews
            subset: v2
          weight: 50
        - destination:
            host: reviews
            subset: v3
          weight: 50
    - match:
        - uri:
            exact: /productpage
        - uri:
            prefix: /static
        - uri:
            exact: /login
        - uri:
            exact: /logout
        - uri:
            prefix: /api/v1/products
      route:
        - destination:
            host: productpage
            port:
              number: 9080
```

### 流量镜像

将v2版本的流量镜像一份到测试环境, 方便测试

```yaml
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: bookinfo
  namespace: default
spec:
  hosts:
    - '*'
  gateways:
    - bookinfo-gateway
  http:
    - match:
        - headers:
            version:
              exact: v1
      route:
        - destination:
            host: reviews
            subset: v1
    - match:
        - headers:
            version:
              exact: v2
      route:
        - destination:
            host: reviews
            subset: v2
          weight: 50
        - destination:
            host: reviews
            subset: v3
          weight: 50
      mirror:
        host: test-reviews
        port:
          number: 10000
      mirrorPercent: 100
    - match:
        - uri:
            exact: /productpage
        - uri:
            prefix: /static
        - uri:
            exact: /login
        - uri:
            exact: /logout
        - uri:
            prefix: /api/v1/products
      route:
        - destination:
            host: productpage
            port:
              number: 9080
```

