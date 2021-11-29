---
title: diary-flagger渐进式的金丝雀发版
date: 2021-09-16 15:59:04
tags:
- 个人日记
- kubernetes
---



官方: https://github.com/fluxcd/flagger

![FLAGGER](https://raw.githubusercontent.com/fluxcd/flagger/main/docs/diagrams/flagger-overview.png)

flagger是一个工具，通过逐渐迁移流量到新版本并评估指标和一致性测试来减少新版本发布到生产环境的风险。

flagger使用  a service mesh (App Mesh, Istio, Linkerd, Open Service Mesh) or an ingress controller (Contour, Gloo, NGINX, Skipper, Traefik) 实现了几种发布策略（Canary releases, A/B testing, Blue/Green mirroring。对于发布分析，Flagger can query Prometheus, Datadog, New Relic or CloudWatch。对于报警，it uses Slack, MS Teams, Discord, Rocket and Google Chat.

Flagger is a [Cloud Native Computing Foundation](https://cncf.io/) project and part of [Flux](https://fluxcd.io/) family of GitOps tools.

flagger会生成一系列的k8s资源，并时刻监控 ConfigMaps and Secrets ，其变化时就触发分析。当分析OK时，所有(container images) and configuration (config maps and secrets) 将会同步到旧的版本。



# 工作逻辑
<!--more-->

![flagger](http://myapp.img.mykernel.cn/flagger.png)

# yaml文件

`deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: podinfo
    tier: backend
  name: podinfo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: podinfo
      tier: backend
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      annotations:
        prometheus.io/port: "7001"
        prometheus.io/scrape: "true"
      labels:
        app: podinfo
        tier: backend
    spec:
      containers:
      - env:
        - name: cpus
          valueFrom:
            resourceFieldRef:
              containerName: deploy-container
              resource: limits.cpu
        image: busybox
        imagePullPolicy: Always
        name: deploy-container
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

`ingress.yml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
  labels:
    app: podinfo
    tier: backend
  name: podinfo
  namespace: default
spec:
  rules:
  - host: www.mykernel.cn
    http:
      paths:
      - backend:
          service:
            name: podinfo
            port:
              number: 8080
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - www.mykernel.cn
    secretName: mykernel.cn
```

`canary.yml`

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  labels:
    app: podinfo
    tier: backend
  name: podinfo
  namespace: default
spec:
  analysis:
    interval: 10s
    maxWeight: 50 # 最大权重50
    metrics:
    - interval: 10s
      name: sucess-percentage-rate
      templateRef:
        name: sucess-percentage-rate
        namespace: default
      thresholdRange: # 最低成功率 99%
        min: 99
    - interval: 10s
      name: falgger-request-duration
      templateRef:
        name: falgger-request-duration
        namespace: default
      thresholdRange: # 最多延迟 500ms
        max: 500
    stepWeight: 5 # 权重步进
    threshold: 3  # 失败3次就回滚
    webhooks:
    - metadata:
        cmd: hey -z 10m -q 10 -c 2 -host podinfo.grasp.com  http://podinfo-canary.test.svc:9898
        logCmdOutput: "true"
        type: cmd
      name: load-test
      timeout: 5s
      type: rollout
      url: http://flagger-loadtester.test.svc/
#  此为可选，上面没有定义，所以不需要
#  autoscalerRef:
#    apiVersion: autoscaling/v2beta2
#    kind: HorizontalPodAutoscaler
#    name: podinfo
  ingressRef: # 引用的ingress
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    name: podinfo
  progressDeadlineSeconds: 60
  provider: nginx
  service: # 自动生成svc
    port: 8080
    portName: http
    targetPort: 8080
  targetRef: # 自动引用deploy
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
```

`rule.yml`

```yaml
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: sucess-percentage-rate
  namespace: default
spec:
  provider:
    type: prometheus # can be prometheus, datadog, etc
    address: http://flagger-prometheus.kube-system.svc:9090 # API URL
    insecureSkipVerify: true # if set to true, disables the TLS cert validation
  query: |
     sum(
        irate(
            http_request_duration_milliseconds_bucket{
              kubernetes_namespace="{{ namespace }}",
              kubernetes_pod_name=~"{{ target }}-[^-]+-[^-]+$",
              status!~"5.*"
              }[{{ interval }}] 
        )   
              )   
              /   
      sum(
        irate(http_request_duration_milliseconds_bucket{
            kubernetes_namespace=~"{{ namespace }}",
            kubernetes_pod_name=~"{{ target }}-[^-]+-[^-]+$",
        }[{{ interval }}])) * 100 

---
# {"level":"info","ts":"2021-08-25T01:37:59.234Z","caller":"controller/events.go:45","msg":"Halt podinfo.test advancement falgger-request-duration 2482.00 > 500","canary":"podinfo.test"}
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: falgger-request-duration
  namespace: default
spec:
  provider:
    type: prometheus # can be prometheus, datadog, etc
    address: http://flagger-prometheus.kube-system.svc:9090 # API URL
    insecureSkipVerify: true # if set to true, disables the TLS cert validation
  query: |
    histogram_quantile(0.99, sum(irate(http_request_duration_milliseconds_bucket{
        kubernetes_namespace="{{ namespace }}",
        kubernetes_pod_name=~"{{ target }}-[^-]+-[^-]+$",
      }[{{ interval }}])) by (le)) 
```

安装flagger

https://docs.flagger.app/tutorials/nginx-progressive-delivery

```bash
helm repo add flagger https://flagger.app

helm upgrade -i flagger flagger/flagger \
--namespace kube-system \
--set prometheus.install=true \
--set meshProvider=nginx

kubectl apply -f https://raw.githubusercontent.com/fluxcd/flagger/main/artifacts/flagger/crd.yaml
```

准备负载测试

```bash
kubectl create ns test
helm upgrade -i flagger-loadtester flagger/loadtester --namespace=test
```



# 更新新版本

```bash
# update the container image
kubectl set image deployment/podinfo podinfod=stefanprodan/podinfo:3.0.1

# wait for Flagger to detect the change
count=1
ok=false
until ${ok}; do
    kubectl get canary/podinfo | grep 'Progressing' && ok=true || ok=false
    sleep 5
    let $(( count++ ))
    if [ $count -ge 10 ]; then
    	echo '更新失败';
    	exit 1
    fi
done
# wait for the canary analysis to finish
kubectl wait canary/podinfo --for=condition=promoted --timeout=5m

# check if the deployment was successful 
kubectl get canary/podinfo | grep Succeeded
```

