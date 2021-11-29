---
title: "Mixer-Policy-Telemetry-配置模型"
date: 2021-09-26 14:17:09
tags:
- istio
---



# 话题

istio Mixer的策略和可观测性(policy和Telemetry)

- 策略：速率限制、黑白名单、标头重写及重定向
- 可观测性：指标、日志和分布式跟踪

Mixer

- template和Adapter
- Atrribute和Attrubue Expression
- Mixer的配置模型：Handler、Instance和Rule

Handler配置格式及示例

Instance配置格式及常用的Template

- Metric
- LogEntry
- Authorization
- Quota

Rule配置格式及示例





# istio的服务访问控制和可观测性

- policy: 
  - 指用户让用户配置自定义策略，用于限制对服务的访问。
  - mixer中间层可以让用户自定义除了速率限制、黑白名单、标头重写及重定向这些策略之外的策略。

- 可观测性

  - 传统：监控系统
  - 分布式：
    - metric: istio中有4个黄金指标：延迟、流量、错误和饱和度
    - distributed Traces: 支持为每种服务生成分布式跟踪记录
    - access Log: 可于workload级别生成服务请求的完整记录

  > 第3方插件：`zipkin`/`jaeger`/...， 而mixer就是一个中间层，用户统一配置，可以适配不同的组件。

# Mixer工作拓扑

![image-20210926160635054](http://myapp.img.mykernel.cn/image-20210926160635054.png)

- 当service A, 要请求service B时，service b请求<b>mixer</b>发起check的请求，检查是否有权限访问service B.
- 请求过程中，会有多次的report自己的指标给mixer.
- envoy会向<strong>telemetry</strong>报告数据。
- 结束通信时，A/B都需要向telemetry报告数据。

考虑到性能问题，一般策略问题是关闭的，如果打开对性能影响非常大。遥测功能会打开。

> <div class="admonition note">
>  <p class="admonition-title">提示</p>
>  <p>已经在使用istio的项目，都弃用了Policy. 甚至会重新改写dataplane, 甚至替换Mixer实现，以便此功能的性能提升</p>
> </div>

# Mixer

运维定义config, 定义什么时间将哪些数据，通过**Adapter**发给哪个后端。

## 指标

记录的指标

- 代理envoy
- 服务级别指标，黄金指标
- 控制平面指标，监控istio自身
  - pilot
  - galley
  - mixer
  - citadel

## 分布式跟踪

## 日志

## mixer client library (面向envoy)

在每个envoy内部启用此filter, 启用后，一次请求，envoy在服务服务间进行请求代理时将会对Mixer发起Check和Report两次rpc请求。

Check: 于服务请求转发之前进行，用于向Mixer请求访问策略与配额(rate limiting)

Report: 于服务请求转发之后进行，用于向Mixer上报遥测数据。



## Template和Adapter

不同adapter适配不同后端，都是基于属性配置，template就是将传递来的属性配置，转化成适配不同adapter的配置。

运维不需要关心template和adapter, 只需要定义**instance**定义收集哪些数据、通过**handler**将数据发到何处，基于**rule**定义何时发送。

https://istio.io/latest/blog/2017/adapter-model/

![mixer-template-adapter](http://myapp.img.mykernel.cn/mixer-template-adapter.png)

> <div class="admonition note">
>  	<p class="admonition-title">了解Adapter开发流程？</p>
> 	<ul>
>         <li>确定Adapter类型：check, quota, report</li>
>         <li>adapter如何获取数据：从template</li>
>         <li>如何配置adapter, 通过handler完成</li>
>     </ul> 
> </div>



## 运维定义的字段

![operater的handler_instance_rule字段定义](http://myapp.img.mykernel.cn/operater的handler_instance_rule字段定义.png)

<div class="admonition warning">
	<p class="admonition-title">warning</p>
	<p>对我们而言，我们更多使用reporter的遥测功能，对于check功能很多时候都是禁用的，此功能一般不会使用</p>
</div>

下节课基于bookinfo，了解mixer的policy和telemetry的应用

总结

mixer是外部存储的统一接口



# mixer的policy和telemetry的应用

## 可观测性

### metric

#### collecting metric

https://istio.io/latest/docs/tasks/observability/

<div class="admonition note">
	<p>上课讲的是istio 1.4版本: <a target="_blank" rel="noopener" href="https://istio.io/v1.4/docs/tasks/observability/metrics/collecting-metrics/">Istio 1.4</a></p>
</div>

```bash
kubectl apply -f samples/bookinfo/telemetry/metrics.yaml
```

> ```yaml
> # Configuration for metric instances
> apiVersion: config.istio.io/v1alpha2
> kind: instance # 内建模板实例化
> metadata:
>   name: doublerequestcount
>   namespace: istio-system
> spec:
>   compiledTemplate: metric
>   params:
>     value: "2" # count each request twice
>     dimensions:
>       reporter: conditional((context.reporter.kind | "inbound") == "outbound", "client", "server")
>       source: source.workload.name | "unknown"
>       destination: destination.workload.name | "unknown"
>       message: '"twice the fun!"'
>     monitored_resource_type: '"UNSPECIFIED"'
> ---
> # Configuration for a Prometheus handler
> apiVersion: config.istio.io/v1alpha2
> kind: handler  #adapter实例化
> metadata:
>   name: doublehandler
>   namespace: istio-system
> spec:
>   compiledAdapter: prometheus # 对接prometheus 未定义prometheus位置，依赖于portforward
>   params:
>     metrics:
>     - name: double_request_count # Prometheus metric name
>       instance_name: doublerequestcount.instance.istio-system # Mixer instance name (fully-qualified)
>       kind: COUNTER
>       label_names: # 过滤这几个标签 
>       - reporter
>       - source
>       - destination
>       - message
> ---
> # Rule to send metric instances to a Prometheus handler
> apiVersion: config.istio.io/v1alpha2
> kind: rule # 规则
> metadata:
>   name: doubleprom
>   namespace: istio-system
> spec: # 应用在所有流量上
>   actions:
>   - handler: doublehandler
>     instances: [ doublerequestcount ]
> ```

```bash
root@ip-10-0-0-245:/usr/local/istio# kubectl get instance -n istio-system
NAME                   AGE
attributes             15m
doublerequestcount     5m9s
requestcount           15m
requestduration        15m
requestsize            15m
responsesize           15m
tcpbytereceived        15m
tcpbytesent            15m
tcpconnectionsclosed   15m
tcpconnectionsopened   15m
```

访问productpage页面, 访问几次：http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage

```bash
root@ip-10-0-0-245:/usr/local/istio# while true; do curl http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage &> /dev/null; echo ok; sleep 0.$[$RANDOM%10]; done
```

需要打开prometheus端口转发

```bash
 kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &
```



prometheus控制台，验证指标已经收到

> 暴露prometheus, 参考：http://blog.mykernel.cn/2021/09/22/istio%E6%9E%B6%E6%9E%84%E4%B8%8E%E5%AE%9E%E8%B7%B5/#%E9%80%9A%E8%BF%87kali%E7%9B%91%E6%8E%A7istio

```bash
root@ip-10-0-0-245:/usr/local/istio# kubectl get svc -A
istio-system   istio-ingressgateway     NodePort    10.96.38.54      <none>        15020:30162/TCP,80:30080/TCP,443:30443/TCP,15029:30531/TCP,15030:31806/
```

http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:31806/graph

`istio_double_request_count` 指标

![image-20210927150557070](http://myapp.img.mykernel.cn/image-20210927150557070.png)

> reporter 谁报告
>
> destination 请求的目标
>
> message 自定义数据

- Remove the new metrics configuration:

  ```
  $ kubectl delete -f samples/bookinfo/telemetry/metrics.yaml
  ```

  If you are using Istio 1.1.2 or prior:

  ```
  $ kubectl delete -f samples/bookinfo/telemetry/metrics-crd.yaml
  ```

- Remove any `kubectl port-forward` processes that may still be running:

  ```
  $ killall kubectl
  ```

- If you are not planning to explore any follow-on tasks, refer to the [Bookinfo cleanup](https://istio.io/v1.4/docs/examples/bookinfo/#cleanup) instructions to shutdown the application.

#### collecting metric for tcp services

https://istio.io/v1.4/docs/tasks/observability/metrics/tcp-metrics/

```bash
 kubectl apply -f samples/bookinfo/telemetry/tcp-metrics.yaml
```

> ```yaml
> # Configuration for a metric measuring bytes sent from a server
> # to a client
> apiVersion: config.istio.io/v1alpha2
> kind: instance
> metadata:
>   name: mongosentbytes # mongodb 发出来的数据
>   namespace: default
> spec:
>   compiledTemplate: metric
>   params:
>     value: connection.sent.bytes | 0 # uses a TCP-specific attribute
>     dimensions:
>       source_service: source.workload.name | "unknown"
>       source_version: source.labels["version"] | "unknown"
>       destination_version: destination.labels["version"] | "unknown"
>     monitoredResourceType: '"UNSPECIFIED"'
> ---
> # Configuration for a metric measuring bytes sent from a client
> # to a server
> apiVersion: config.istio.io/v1alpha2
> kind: instance
> metadata:
>   name: mongoreceivedbytes # 接收
>   namespace: default
> spec:
>   compiledTemplate: metric
>   params:
>     value: connection.received.bytes | 0 # uses a TCP-specific attribute
>     dimensions:
>       source_service: source.workload.name | "unknown"
>       source_version: source.labels["version"] | "unknown"
>       destination_version: destination.labels["version"] | "unknown"
>     monitoredResourceType: '"UNSPECIFIED"'
> ---
> # Configuration for a Prometheus handler
> apiVersion: config.istio.io/v1alpha2
> kind: handler
> metadata:
>   name: mongohandler # 2个指标，counter
>   namespace: default
> spec:
>   compiledAdapter: prometheus
>   params:
>     metrics:
>     - name: mongo_sent_bytes # Prometheus metric name
>       instance_name: mongosentbytes.instance.default # Mixer instance name (fully-qualified)
>       kind: COUNTER
>       label_names:
>       - source_service
>       - source_version
>       - destination_version
>     - name: mongo_received_bytes # Prometheus metric name
>       instance_name: mongoreceivedbytes.instance.default # Mixer instance name (fully-qualified)
>       kind: COUNTER
>       label_names:
>       - source_service
>       - source_version
>       - destination_version
> ---
> # Rule to send metric instances to a Prometheus handler
> apiVersion: config.istio.io/v1alpha2
> kind: rule # 规则 
> metadata:
>   name: mongoprom
>   namespace: default
> spec:
>   match: context.protocol == "tcp"
>          && destination.service.host == "mongodb.default.svc.cluster.local"
>   actions:
>   - handler: mongohandler
>     instances:
>     - mongoreceivedbytes
>     - mongosentbytes
> ```

启动reviews v2使用mongodb

```yaml
kubectl apply -f samples/bookinfo/platform/kube/bookinfo-ratings-v2.yaml
```

安装mongodb

```yaml
 kubectl apply -f samples/bookinfo/platform/kube/bookinfo-db.yaml
```

bookinfo的多个版本归组

```bash
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```

> 起码 流量 到mongo

定义服务的路由

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-db.yaml
```

访问productpage页面, 访问几次：http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage

```bash
root@ip-10-0-0-245:/usr/local/istio# while true; do curl http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage &> /dev/null; echo ok; sleep 0.$[$RANDOM%10]; done
```

需要打开prometheus端口转发

```bash
 kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &
```



prometheus控制台，验证指标已经收到

> 暴露prometheus, 参考：http://blog.mykernel.cn/2021/09/22/istio%E6%9E%B6%E6%9E%84%E4%B8%8E%E5%AE%9E%E8%B7%B5/#%E9%80%9A%E8%BF%87kali%E7%9B%91%E6%8E%A7istio

```bash
root@ip-10-0-0-245:/usr/local/istio# kubectl get svc -A
istio-system   istio-ingressgateway     NodePort    10.96.38.54      <none>        15020:30162/TCP,80:30080/TCP,443:30443/TCP,15029:30531/TCP,15030:31806/
```

http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:31806/graph

`mongo_received_bytes` 指标

![image-20210927151813133](http://myapp.img.mykernel.cn/image-20210927151813133.png)

不清理数据

#### 通过prometheus查询数据

https://istio.io/v1.4/docs/tasks/observability/metrics/querying-metrics/

指标1: `istio_requests_total`

- Total count of all requests to the `productpage` service:

  ```plain
  istio_requests_total{destination_service="productpage.default.svc.cluster.local"}
  ```

- Total count of all requests to `v3` of the `reviews` service:

  ```plain
  istio_requests_total{destination_service="reviews.default.svc.cluster.local", destination_version="v3"}
  ```

  This query returns the current total count of all requests to the v3 of the `reviews` service.

- Rate of requests over the past 5 minutes to all instances of the `productpage` service:

  ```plain
  rate(istio_requests_total{destination_service=~"productpage.*", response_code="200"}[5m])
  ```

#### 通过grafana查询

https://istio.io/v1.4/docs/tasks/observability/metrics/using-istio-dashboard/	

暴露grafana, 参考：http://blog.mykernel.cn/2021/09/22/istio%E6%9E%B6%E6%9E%84%E4%B8%8E%E5%AE%9E%E8%B7%B5/#%E9%80%9A%E8%BF%87kali%E7%9B%91%E6%8E%A7istio

面板

http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:31791

<div class="admonition note">
	<p class="admonition-title">注意</p>
	<p>需要使用此脚本不断请求，才会在istio dashboard workload的ratings-v2负载看到数据</p>
	<p>
        while true; do curl http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage &> /dev/null; echo ok; sleep 0.$[$RANDOM%10]; done
    </p>
</div>

![image-20210927152442379](http://myapp.img.mykernel.cn/image-20210927152442379.png)

清理tcp service

- Remove the new telemetry configuration:

  ```
  $ kubectl delete -f samples/bookinfo/telemetry/tcp-metrics.yaml
  ```

  If you are using Istio 1.1.2 or prior:

  ```
  $ kubectl delete -f samples/bookinfo/telemetry/tcp-metrics-crd.yaml
  ```

- Remove the `port-forward` process:

  ```
  $ killall kubectl
  ```

- If you are not planning to explore any follow-on tasks, refer to the [Bookinfo cleanup](https://istio.io/v1.4/docs/examples/bookinfo/#cleanup) instructions to shutdown the application.

### log

https://istio.io/v1.4/docs/tasks/observability/logs/

#### 收集日志

https://istio.io/v1.4/docs/tasks/observability/logs/collecting-logs/

```bash
kubectl apply -f samples/bookinfo/telemetry/log-entry.yaml
```

> ```yaml
> # Configuration for logentry instances
> apiVersion: config.istio.io/v1alpha2
> kind: instance
> metadata:
>   name: newlog
>   namespace: istio-system
> spec:
>   compiledTemplate: logentry
>   params:
>     severity: '"warning"'
>     timestamp: request.time
>     variables:
>       source: source.labels["app"] | source.workload.name | "unknown"
>       user: source.user | "unknown"
>       destination: destination.labels["app"] | destination.workload.name | "unknown"
>       responseCode: response.code | 0
>       responseSize: response.size | 0
>       latency: response.duration | "0ms"
>     monitored_resource_type: '"UNSPECIFIED"'
> ---
> # Configuration for a stdio handler
> apiVersion: config.istio.io/v1alpha2
> kind: handler
> metadata:
>   name: newloghandler
>   namespace: istio-system
> spec:
>   compiledAdapter: stdio
>   params:
>     severity_levels: # 日志级别
>       warning: 1 # Params.Level.WARNING
>     outputAsJson: true # 输出为json
> ---
> # Rule to send logentry instances to a stdio handler
> apiVersion: config.istio.io/v1alpha2
> kind: rule
> metadata:
>   name: newlogstdio
>   namespace: istio-system
> spec:
>   match: "true" # match for all requests 相当于不写，所有流量收集日志
>   actions:
>    - handler: newloghandler
>      instances:
>      - newlog
> ```

访问productpage页面, 访问几次：http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage

```bash
root@ip-10-0-0-245:/usr/local/istio# while true; do curl http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage &> /dev/null; echo ok; sleep 0.$[$RANDOM%10]; done
```

查看日志

```bash
root@ip-10-0-0-245:/usr/local/istio# kubectl logs -n istio-system -l istio-mixer-type=telemetry -c mixer | grep "newlog" | grep -v '"destination":"telemetry"' | grep -v '"destination":"pilot"' | grep -v '"destination":"policy"' | grep -v '"destination":"unknown"'
{"level":"warn","time":"2021-09-27T07:35:45.407834Z","instance":"newlog.instance.istio-system","destination":"ratings","latency":"4.214777ms","responseCode":200,"responseSize":48,"source":"reviews","user":"unknown"}
```

> 日志格式正常好instance定义的`variables`

对访问流程收集日志

- Remove the new logs configuration:

  ```
  $ kubectl delete -f samples/bookinfo/telemetry/log-entry.yaml
  ```

  If you are using Istio 1.1.2 or prior:

  ```
  $ kubectl delete -f samples/bookinfo/telemetry/log-entry-crd.yaml
  ```

- If you are not planning to explore any follow-on tasks, refer to the [Bookinfo cleanup](https://istio.io/v1.4/docs/examples/bookinfo/#cleanup) instructions to shutdown the application.

#### 访问日志

https://istio.io/v1.4/docs/tasks/observability/logs/access-log/

```bash
 kubectl apply -f samples/sleep/sleep.yaml
```

```bash
export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

```bash
 kubectl apply -f samples/httpbin/httpbin.yaml
```

启用envoy日志, 默认部署的服务网格，envoy日志并不打开

```bash
istioctl manifest apply --set values.global.proxy.accessLogFile="/dev/stdout"  --set profile=default \
	--set values.kiali.enabled=true --set values.grafana.enabled=true --set values.tracing.enabled=true \
	--set "values.kiali.dashboard.jaegerURL=http://jaeger-query:16686" \
	--set "values.kiali.dashboard.grafanaURL=http://grafana:3000" 
```

You can also choose between JSON and text by setting `accessLogEncoding` to `JSON` or `TEXT`.

You may also want to customize the [format](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log#format-rules) of the access log by editing `accessLogFormat`.



访问httpbin

```bash
kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items[0].metadata.name}') -c sleep -- curl -v httpbin:8000/status/418
```

检查sleep日志

```bash
 kubectl logs -l app=sleep -c istio-proxy
 [2021-09-27T07:44:25.662Z] "GET /status/418 HTTP/1.1" 418 - "-" "-" 0 135 3 2 "-" "curl/7.77.0" "378926d2-670b-40f0-9564-6cd736c483d3" "httpbin:8000" "10.244.0.24:80" outbound|8000||httpbin.default.svc.cluster.local - 10.105.145.234:8000 10.244.0.23:44180 - default
```

> 在sleep中访问httpbin, 在sleep的envoy中生成了日志

检查httpbin日志

```bash
kubectl logs -l app=httpbin -c istio-proxy
```

> 被访问时，生成日志

Shutdown the [sleep](https://github.com/istio/istio/tree/release-1.4/samples/sleep) and [httpbin](https://github.com/istio/istio/tree/release-1.4/samples/httpbin) services:

```
$ kubectl delete -f samples/sleep/sleep.yaml
$ kubectl delete -f samples/httpbin/httpbin.yaml
```

Edit the `istio` configuration map and set `accessLogFile` to `""`.

```
istioctl manifest apply --set values.global.proxy.accessLogFile=""  --set profile=default \
	--set values.kiali.enabled=true --set values.grafana.enabled=true --set values.tracing.enabled=true \
	--set "values.kiali.dashboard.jaegerURL=http://jaeger-query:16686" \
	--set "values.kiali.dashboard.grafanaURL=http://grafana:3000" 
```

#### fluentd收集日志

https://istio.io/v1.4/docs/tasks/observability/logs/fluentd/

部署fluented的agent,  及elasticserch, kibana

```yaml
kubectl <<EOF apply -f -
# Logging Namespace. All below are a part of this namespace.
apiVersion: v1
kind: Namespace
metadata:
  name: logging
---
# Elasticsearch Service
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  ports:
  - port: 9200
    protocol: TCP
    targetPort: db
  selector:
    app: elasticsearch
---
# Elasticsearch Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
      annotations:
        sidecar.istio.io/inject: "false"
    spec:
      containers:
      - image: docker.elastic.co/elasticsearch/elasticsearch-oss:6.1.1
        name: elasticsearch
        resources:
          # need more cpu upon initialization, therefore burstable class
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: discovery.type
            value: single-node
        ports:
        - containerPort: 9200
          name: db
          protocol: TCP
        - containerPort: 9300
          name: transport
          protocol: TCP
        volumeMounts:
        - name: elasticsearch
          mountPath: /data
      volumes:
      - name: elasticsearch
        emptyDir: {}
---
# Fluentd Service
apiVersion: v1
kind: Service
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    app: fluentd-es
spec:
  ports:
  - name: fluentd-tcp
    port: 24224
    protocol: TCP
    targetPort: 24224
  - name: fluentd-udp
    port: 24224
    protocol: UDP
    targetPort: 24224
  selector:
    app: fluentd-es
---
# Fluentd Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    app: fluentd-es
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fluentd-es
  template:
    metadata:
      labels:
        app: fluentd-es
      annotations:
        sidecar.istio.io/inject: "false"
    spec:
      containers:
      - name: fluentd-es
        image: gcr.io/google-containers/fluentd-elasticsearch:v2.0.1
        env:
        - name: FLUENTD_ARGS
          value: --no-supervisor -q
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: config-volume
          mountPath: /etc/fluent/config.d
      terminationGracePeriodSeconds: 30
      volumes:
      - name: config-volume
        configMap:
          name: fluentd-es-config
---
# Fluentd ConfigMap, contains config files.
kind: ConfigMap
apiVersion: v1
data:
  forward.input.conf: |-
    # Takes the messages sent over TCP
    <source>
      type forward
    </source>
  output.conf: |-
    <match **>
       type elasticsearch
       log_level info
       include_tag_key true
       host elasticsearch
       port 9200
       logstash_format true
       # Set the chunk limits.
       buffer_chunk_limit 2M
       buffer_queue_limit 8
       flush_interval 5s
       # Never wait longer than 5 minutes between retries.
       max_retry_wait 30
       # Disable the limit on the number of retries (retry forever).
       disable_retry_limit
       # Use multiple threads for processing.
       num_threads 2
    </match>
metadata:
  name: fluentd-es-config
  namespace: logging
---
# Kibana Service
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
    protocol: TCP
    targetPort: ui
  selector:
    app: kibana
---
# Kibana Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
      annotations:
        sidecar.istio.io/inject: "false"
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana-oss:6.1.1
        resources:
          # need more cpu upon initialization, therefore burstable class
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
          name: ui
          protocol: TCP
---
EOF
```

> docker.io/ist0ne/fluentd-elasticsearch:v2.0.1
>
> kibana与elasticsearch相同 6.1.1
>
> ```yaml
> # Fluentd ConfigMap, contains config files.
> kind: ConfigMap
> apiVersion: v1
> data:
>   forward.input.conf: |-
>     # Takes the messages sent over TCP
>     <source>
>       type forward
>     </source>
>   output.conf: |-
>     <match **>
>        type elasticsearch
>        log_level info
>        include_tag_key true
>        host elasticsearch
>        port 9200
>        logstash_format true
>        # Set the chunk limits.
>        buffer_chunk_limit 2M
>        buffer_queue_limit 8
>        flush_interval 5s
>        # Never wait longer than 5 minutes between retries.
>        max_retry_wait 30
>        # Disable the limit on the number of retries (retry forever).
>        disable_retry_limit
>        # Use multiple threads for processing.
>        num_threads 2
>     </match>
> ```

配置istio, 收集请求流量的日志，adapter对接这个flutend. 

```bash
kubectl apply -f samples/bookinfo/telemetry/fluentd-istio.yaml
```

> ```yaml
> # Configuration for logentry instances
> apiVersion: config.istio.io/v1alpha2
> kind: instance
> metadata:
>   name: newlog
>   namespace: istio-system
> spec:
>   compiledTemplate: logentry
>   params:
>     severity: '"info"'
>     timestamp: request.time
>     variables:
>       source: source.labels["app"] | source.workload.name | "unknown"
>       user: source.user | "unknown"
>       destination: destination.labels["app"] | destination.workload.name | "unknown"
>       responseCode: response.code | 0
>       responseSize: response.size | 0
>       latency: response.duration | "0ms"
>     monitored_resource_type: '"UNSPECIFIED"'
> ---
> # Configuration for a Fluentd handler
> apiVersion: config.istio.io/v1alpha2
> kind: handler
> metadata:
>   name: handler
>   namespace: istio-system
> spec:
>   compiledAdapter: fluentd # 应用的fluentd的模板
>   params:
>     address: "fluentd-es.logging:24224" # fluentd的服务名.名称空间
> ---
> # Rule to send logentry instances to the Fluentd handler
> apiVersion: config.istio.io/v1alpha2
> kind: rule
> metadata:
>   name: newlogtofluentd
>   namespace: istio-system
> spec:
>   match: "true" # match for all requests 收集所有流量的日志
>   actions:
>    - handler: handler
>      instances:
>      - newlog
> ---
> ```

查看

```bash
root@ip-10-0-0-245:/usr/local/istio# kubectl get svc -n logging
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)               AGE
elasticsearch   ClusterIP   10.111.17.15     <none>        9200/TCP              17m
fluentd-es      ClusterIP   10.111.161.218   <none>        24224/TCP,24224/UDP   17m
kibana          ClusterIP   10.105.4.52      <none>        5601/TCP              17m
```

把kibana暴露为nodeport `kubectl edit svc -n logging kibana`

<div class="admonition note">
	<p class="admonition-title">注意</p>
	<p>需要使用此脚本不断请求，才会生成流量，rule匹配流量后，instance获取fluentd相同字段写入handler</p>
	<p>
        while true; do curl http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage &> /dev/null; echo ok; sleep 0.$[$RANDOM%10]; done
    </p>
</div>

访问kibana

http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30834

![image-20210927161249123](http://myapp.img.mykernel.cn/image-20210927161249123.png)

![image-20210927161326258](http://myapp.img.mykernel.cn/image-20210927161326258.png)

- Remove the new telemetry configuration:

  ```
  $ kubectl delete -f samples/bookinfo/telemetry/fluentd-istio.yaml
  ```

  If you are using Istio 1.1.2 or prior:

  ```
  $ kubectl delete -f samples/bookinfo/telemetry/fluentd-istio-crd.yaml
  ```

- Remove the example Fluentd, Elasticsearch, Kibana stack:

  ```
  $ kubectl delete -f logging-stack.yaml
  ```

- Remove any `kubectl port-forward` processes that may still be running:

  ```
  $ killall kubectl
  ```

### tracing

https://istio.io/v1.4/docs/tasks/observability/distributed-tracing/

只要我们打开了tracing，就默认会启用，不需要额外配置

```bash
 --set values.tracing.enabled=true 
```

#### 依赖

需要应用程序传播一些header

bookinfo的程序已经内置了, 更详细，可以参考：http://blog.mykernel.cn/2021/09/20/%E5%8F%AF%E8%A7%82%E6%B5%8B%E6%80%A7%E5%BA%94%E7%94%A8istio/

#### jaeger

```bash
kubectl edit svc -n istio-system   jaeger-query  
```

http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:31032

![image-20210927162218357](http://myapp.img.mykernel.cn/image-20210927162218357.png)

> 此处可以详细看到问题：
>
> error是因为之前删除了ratings v2, 而上面我们测试mongodb时，添加了v2, 并定义了ratings服务仅到达v2
>
> ```bash
> root@ip-10-0-0-245:/usr/local/istio# kubectl describe vs ratings
> Name:         ratings
> Namespace:    default
> Labels:       <none>
> Annotations:  <none>
> API Version:  networking.istio.io/v1alpha3
> Kind:         VirtualService
> Metadata:
>   Creation Timestamp:  2021-09-27T07:15:29Z
>   Generation:          1
>   Resource Version:    4188
>   Self Link:           /apis/networking.istio.io/v1alpha3/namespaces/default/virtualservices/ratings
>   UID:                 cc0fa954-07ee-498a-8ebe-162377c02c64
> Spec:
>   Hosts:
>     ratings
>   Http:
>     Route:
>       Destination:
>         Host:    ratings
>         Subset:  v2
> 
> ```

### kiali

https://istio.io/v1.4/docs/tasks/observability/kiali/

### grafana/prometheus/tracing/kiali

https://istio.io/v1.4/docs/tasks/observability/gateways/

## 策略

https://istio.io/latest/docs/tasks/policy-enforcement/

<div class="admonition warning">
	<p class="admonition-title">注意</p>
	<p>一旦启用Mixer功能，网格内的每一次请求, 被请求方都会请求Mixer进行一次Check, 请求结束前请求方和被请求会向Mixer reporter(指标、日志)</p>
    <p>
        这对性能非常影响! 所以将来用到不大，以至于后续版本的Mixer都会改动。
    </p>
    <p>
        <strong>尤其是Policy相关功能(黑白名单、配额、速率限制、标头重写、重定向)</strong>
    </p>
</div>

### 限速

https://istio.io/v1.4/docs/tasks/policy-enforcement/rate-limiting/

<a src="https://istio.io/v1.4/docs/tasks/policy-enforcement/enabling-policy/" >启动policy</a>

```bash
kubectl -n istio-system get cm istio -o jsonpath="{@.data.mesh}" | grep disablePolicyChecks
```

> - true 禁用策略检查
> - false 启用策略检查

```bash
# 编辑configmap
kubectl -n istio-system edit cm istio
```

或者 

```bash
istioctl manifest apply --set values.global.disablePolicyChecks=false
```

限速

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

> 恢复所有服务路由

```bash
kubectl apply -f samples/bookinfo/policy/mixer-rule-productpage-ratelimit.yaml
```

> - Client Side
>   - `QuotaSpec` defines quota name and amount that the client should request.
>   - `QuotaSpecBinding` conditionally associates `QuotaSpec` with one or more services.
> - Mixer Side
>   - `quota instance` defines how quota is dimensioned by Mixer.
>   - `memquota handler` defines `memquota` adapter configuration.
>   - `quota rule` defines when quota instance is dispatched to the `memquota` adapter.
>
> 修改mix filter的配置
>
> ```yaml
> apiVersion: config.istio.io/v1alpha2
> kind: handler
> metadata:
>   name: quotahandler
>   namespace: istio-system
> spec:
>   compiledAdapter: memquota
>   params:
>     quotas:
>     - name: requestcountquota.instance.istio-system
>       maxAmount: 500
>       validDuration: 1s
>       # The first matching override is applied.
>       # A requestcount instance is checked against override dimensions.
>       overrides:
>       # The following override applies to 'reviews' regardless
>       # of the source.
>       - dimensions:
>           destination: reviews
>         maxAmount: 1 # 1s访问1次
>         validDuration: 5s
>       # The following override applies to 'productpage' when
>       # the source is a specific ip address.
>       - dimensions:
>           destination: productpage
>           source: "10.28.11.20"
>         maxAmount: 500
>         validDuration: 1s
>       # The following override applies to 'productpage' regardless
>       # of the source.
>       - dimensions:
>           destination: productpage
>         maxAmount: 2
>         validDuration: 5s
> ---
> apiVersion: config.istio.io/v1alpha2
> kind: instance
> metadata:
>   name: requestcountquota
>   namespace: istio-system
> spec:
>   compiledTemplate: quota
>   params:
>     dimensions:
>       source: request.headers["x-forwarded-for"] | "unknown"
>       destination: destination.labels["app"] | destination.service.name | "unknown"
>       destinationVersion: destination.labels["version"] | "unknown"
> ---
> apiVersion: config.istio.io/v1alpha2
> kind: QuotaSpec
> metadata:
>   name: request-count
>   namespace: istio-system
> spec:
>   rules:
>   - quotas:
>     - charge: 1
>       quota: requestcountquota
> ---
> apiVersion: config.istio.io/v1alpha2
> kind: QuotaSpecBinding # 绑定哪个服务
> metadata:
>   name: request-count
>   namespace: istio-system
> spec:
>   quotaSpecs:
>   - name: request-count
>     namespace: istio-system
>   services:
>   - name: productpage
>     namespace: default
>     #  - service: '*'  # Uncomment this to bind *all* services to request-count
> ---
> apiVersion: config.istio.io/v1alpha2
> kind: rule
> metadata:
>   name: quota
>   namespace: istio-system
> spec:
>   # quota only applies if you are not logged in. 非登陆用户就限制
>   # match: match(request.headers["cookie"], "user=*") == false 
>   actions:
>   - handler: quotahandler
>     instances:
>     - requestcountquota
> ```

```bash
$ kubectl -n istio-system get instance requestcountquota -o yaml
$ kubectl -n istio-system get rule quota -o yaml
$ kubectl -n istio-system get QuotaSpec request-count -o yaml
$ kubectl -n istio-system get QuotaSpecBinding request-count -o yaml
```

限速

```bash
$ kubectl -n istio-system edit rules quota

...
spec:
  match: match(request.headers["user-agent"], "curl") == true
  actions:
...
```

> 仅对curl命令限制

浏览器访问

http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage

正常, 登陆jason后正常

修改回来

```bash
$ kubectl -n istio-system edit rules quota

...
spec:
  match: match(request.headers["cookie"], "session=*") == false
  actions:
...
```

浏览器访问

http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage

正常

登出后，不正常

将策略关闭

```bash
# 编辑configmap
kubectl -n istio-system edit cm istio
```

> ```yaml
>     disablePolicyChecks: true
> ```

