---
title: 资源指标与HPA控制器
date: 2021-02-03 10:26:23
tags:
---



# 前言

真正构建k8s集群时，至少多个节点多个pod资源，管理员必须能够随时得知每个节点的利用率、POD利用率、POD是否正常？....

不应该让没有被监控的任何组件投入生产使用。

k8s 之上监控指标：系统级、业务级

- Deployment，StatefulSet  评估pod数量多还是少
- 自动弹性缩放器：Autoscale
  - HPA
    - hpa v1:  cpu核心指标
    - hpa v2: cumstom metric
  - Cluster Autoscaler
    - 自动添加节点，基于云计算环境API
  - Vertical Pod AutoScaler: 垂直弹性伸缩
    - 增大单个Pod资源能力，实验
  - Addon Resizer
    - sidecar pod, 根据pod资源状态，弹性扩展或缩减, 通过修改sts/deploy的副本性。

<!--more-->

# 为什么使用资源指标？

k8s集群完整功能，需要以下功能

1. kubectl top 命令

```bash
[root@master ~]# kubectl top pod
error: Metrics API not available
```

> cAdvisor采集资源指标，并聚合运算，不现实

2. hpa
3. scheduler 根据资源情况调度

kubelet内建了cAdvisor(仅核心指标：cpu/ram/disk)，但是不能聚合集群核心指标(cpu/ram/disk)。kubectl top 命令通过内建的api完成获取聚合指标。 所以聚合这些指标通过扩展api 实现

> kubectl top - https -> aggregator代理(APIServer路由) - https ->  service -> pod
>
> ```bash
> # APIServer路由
> # 创建
> apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io unchanged
> [root@master ~]# kubectl api-versions | grep metrics.k8s.io/v1beta1
> metrics.k8s.io/v1beta1
> 
> 
> apiVersion: apiregistration.k8s.io/v1
> kind: APIService
> metadata:
>   labels:
>     k8s-app: metrics-server
>   name: v1beta1.metrics.k8s.io
> spec:
>   group: metrics.k8s.io
>   groupPriorityMinimum: 100
>   insecureSkipTLSVerify: true
>   service:
>     name: metrics-server
>     namespace: kube-system
>   version: v1beta1
>   versionPriority: 100
> 
> ```

这个api通过metric群组输出，但是群组默认不存在，本身k8s不提供，用户需要部署pod, 并注册在api server之上，生成一个群组。

**监控程序需要更多的指标，**所以需要在k8s集群之上在部署一个heapster程序，收集cAdvisor的指标，也能响应自定义api server的查询。这里实时的。但是要查询过去的指标，过去一个月中每个节点/pod指标运行状态，需要记录过去采集结果。

**存储过去数据**，需要data sipk, heapster支持influxdb，....

**展示界面**，grafana, 可以整合在elasticsearch上，kibana即grafana的前身。



heapster架构体系有问题，后来废弃了。heapster被prometheus组件取代（cncf已经毕业），非常著名监控系统。高性能的时间序列数据存储查询系统，内建UI非常丑陋，可以使用grafana展示。内建查询接口promql，完成查询展示报警。

prometheus采集数据：

- cadvisor: 核心指标
- prometheus专用agent：不仅仅只有cadvisor指标

prometheus是完整的监控系统：

- 采集：exporter
- 存储：tsdb
- 报警：alertmanager
- 展示：grafana







# 自定义资源

## api server <-> 自定义api server

### aggregator -> 自定义api

![图片发布](https://miro.medium.com/max/893/1*mkaLbGPNL3YBV3fOzGEN7w.png)

```bash
    # 将用户信息反代给后端api server
    - --requestheader-allowed-names=front-proxy-client,system:metrics-server,metrics-server
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra- # ...
    - --requestheader-group-headers=X-Remote-Group   # 用户组
    - --requestheader-username-headers=X-Remote-User # 用户信息
	# 这些信息保存在	kube-system的configmap中extension-apiserver-authentication和metric-server自定义api共享
    # 由于k8s工作于rbac, 需要授权自定义api server的pod对这个配置可读extension-apiserver-authentication-reader
    
        apiVersion: rbac.authorization.k8s.io/v1
        kind: RoleBinding
        metadata:
          labels:
            k8s-app: metrics-server
          name: metrics-server-auth-reader
          namespace: kube-system
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: extension-apiserver-authentication-reader #kube-system名称空间中的role
        subjects:
        - kind: ServiceAccount
          name: metrics-server
          namespace: kube-system
          

# 角色
[root@master ~]# kubectl describe role -n kube-system extension-apiserver-authentication-reader
  configmaps  []                 [extension-apiserver-authentication]  [get list watch]
# 可以对 与role同名称空间的extension-apiserver-authentication的configmap读


# metric-server可以读client-ca, front-ca, 允许的公钥
[root@master ~]# kubectl get  cm extension-apiserver-authentication -n kube-system -o yaml
apiVersion: v1
data:
  client-ca-file: |
  requestheader-allowed-names: '["front-proxy-client","system:metrics-server","metrics-server"]'
  requestheader-client-ca-file: |
  requestheader-extra-headers-prefix: '["X-Remote-Extra-"]'
  requestheader-group-headers: '["X-Remote-Group"]'
  requestheader-username-headers: '["X-Remote-User"]'

```

使用的协议 HTTP

```bash
---
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    k8s-app: metrics-server
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true # api server -- http --> 自定义api server
  service:
    name: metrics-server
    namespace: kube-system
  version: v1beta1
  versionPriority: 100
```

### 自定义api -> aggregator 

```bash
- --cert-dir=/tmp # If --tls-cert-file and --tls-private-key-file are provided, this flag will be ignored. 
```

不指定证书，自动生成

指定必须是api server信任的ca颁发，且证书名必须是serving.crt, serving.key



## 自定义api server Pod连接kubelet

```bash
# metric-server可以读client-ca, front-ca, 允许的公钥
[root@master ~]# kubectl get  cm extension-apiserver-authentication -n kube-system -o yaml
apiVersion: v1
data:
  client-ca-file: |                       # kubelet的ca
  requestheader-allowed-names: '["front-proxy-client","system:metrics-server","metrics-server"]'
  requestheader-client-ca-file: |         # 代理的ca 
  requestheader-extra-headers-prefix: '["X-Remote-Extra-"]'
  requestheader-group-headers: '["X-Remote-Group"]'
  requestheader-username-headers: '["X-Remote-User"]'
```

默认走https, 通过调试metric-server可以看出请求kubelet使用 Bearer token认证，但是要提交证书

```bash
curl -k -v -XGET  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IjE2UndYZzJOMWY1NWlFWHFVY0xXMkFlbEtmejZLYUdWM0RvTzdzaURRQmcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJtZXRyaWNzLXNlcnZlci10b2tlbi1scG5rcyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJtZXRyaWNzLXNlcnZlciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImIxYWQxMTNhLWY1MzctNGZhYi05NWY2LTZjZDViZDMxODQwNCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTptZXRyaWNzLXNlcnZlciJ9.Grpl9UtKWFZs-ioTrjWT6bZF0bMxIXkLFWNY7RF2Oh_Y95l_RbMIpaVuGfHST7FPOPW6EPz-acPPHx5-E40S9vsRgm9WHGd2Fic059AMw3pjK3XiQ-S-6ffRsduWj9NMMMtkjsnieYMancUOtNIJeAYO-1hBLWbgvvYWnRdO8JvrQ4AX2zkFvwLNi4dmtW7Mlloxhcm5kO2c2xXAQ6cka_Pz2-vbxD1vpTwDDKSeLhgAA5i--sK86bgBAM3qEj8eNAq7t8pNgWkmJFjJkaWV-aSyHDOci-RSQNBYdzFchNz5yTjdnc8xOVLjOrdL9szY3L93UVHPZEhCcFaqOYxJJA" 'https://node03.magedu.com:10250/stats/summary?only_cpu_and_memory=true'
```

> https://www.brightbox.com/blog/2020/09/15/secure-kubernetes-metrics/
>
> 与kubelet建立安全连接更具挑战性。默认情况下，由kubeadm安装的所有kubelet都会提供一个自签名证书，显然，此证书无法由metrics-server进行验证。因此`--kubelet-insecure-tls`，解决方法中的标志。



## 安装metrics-server

https://github.com/kubernetes-sigs/metrics-server
kubectl top 命令需要

hpa组件也需要metric server

scheduler也需要

| Metrics Server | Metrics API group/version | Supported Kubernetes version |
| -------------- | ------------------------- | ---------------------------- |
| 0.4.x          | `metrics.k8s.io/v1beta1`  | 1.8+                         |
| 0.3.x          | `metrics.k8s.io/v1beta1`  | 1.8+                         |

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created # 角色
  name: system:aggregated-metrics-reader
rules: # 对自已组的读。仅需要
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch

clusterrole.rbac.authorization.k8s.io/system:metrics-server created # 角色
rules: # 对核心群组的读
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - nodes/stats
  - namespaces
  - configmaps
  verbs:
  - get
  - list
  - watch

rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
	kube-system 中的rolebinding, 所以用户的权限只对这个名称空间生效
	
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created # 用户绑定角色
	tokenreviews、subjectaccessreviews 创建权限
	
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created


deployment.apps/metrics-server created # k8s之上的pod
      serviceAccountName: metrics-server # 指定这个pod内进程是这个用户


apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created # 创建自定义资源，
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    k8s-app: metrics-server
  name: v1beta1.metrics.k8s.io # 对这个版本的访问
spec:
  group: metrics.k8s.io # 组
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true # 非tls, 只是不强制证书认证
  service: # 这个服务
    name: metrics-server
    namespace: kube-system
  version: v1beta1 # 这个版本
  versionPriority: 100



service/metrics-server created
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server # 到这个service之后
  namespace: kube-system 
spec:
  ports:
  - name: https
    port: 443 # 443是https
    protocol: TCP
    targetPort: https
  selector:
    k8s-app: metrics-server # 选择pod



# apiserver定义
[root@master ~]# kubectl api-versions | grep metrics
metrics.k8s.io/v1beta1

# service
[root@master ~]# kubectl get svc -n kube-system
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
metrics-server   ClusterIP   10.100.162.94   <none>        443/TCP                  11m

```

```bash
      - command:
        - /metrics-server
        - --cert-dir=/tmp # If --tls-cert-file and --tls-private-key-file are provided, this flag will be ignored. 
        - --secure-port=4443
        # kubelet不走https
        - --kubelet-insecure-tls 
        - --kubelet-preferred-address-types=InternalIP
        #- --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt          # 这里是kube aggregator连接当前pod, 当前pod验证aggregator提供的证书,  extension-apiserver-authentication api定义的配置提供，当前pod共享
        image: k8s.gcr.io/metrics-server-amd64:v0.3.6

```

```	bash
[root@master ~]# kubectl apply -f components.yaml 
```



# hpa

一个控制器(Deployment/StatefulSet)下多个pod, 有的pod负载时高时低，人为发现资源承载变化，再去扩容pod, 响应慢。比如半夜来了一个请求尖峰，无法及时响应。

hpa就是deployment更高级控制器，通过kubectl top 命令监视deploy系统资源占用比例，如果资源占用80%(举例),高出比例时，就自动添加pod，确保所有pod平均资源占用量低于我们期望的水平线之下 。 

版本

- **hpav1** 基于cpu自动扩容. 系统核心指标, metric-server 自定义APIServer

- **hpav2**   基于自定义指标自动扩容，prometheus实现`custom metrics prometheus adapter, ` 自定义指标。 

  > 自定义APIService: k8s不能直接提供进程指标，借助自定义APIService，请求->aggregator -> 自定义指标服务器(custom metrics prometheus adapter) -> promQL -> exporter --> 返回给k8s Restful风格的接口。
  >
  > 示例：以nginx并发上限实现自动扩缩容



Pod内进程监控，通过restful接口监控

- 程序内建restful接口
- 辅助的程序采集，输出restful接口



prometheus查询API server, 获取pod，这么多pod, 需要annotation定义允许prometheus动态加载监控pod指标。

HPA V2 通过`custom metrics prometheus adapter` 收集prometheus自定义指标。



## 安装prometheus

有状态应用operator

- coreos operator 建议
- 测试 kubernetes官方代码树的addons/prometheus定义

> 参考https://www.servicemesher.com/blog/prometheus-operator-manual/

```bash
git clone https://github.com/prometheus-operator/kube-prometheus.git
cd manifests/

# 准备目录
mkdir -p operator node-exporter alertmanager grafana kube-state-metrics prometheus serviceMonitor adapter
mv *-serviceMonitor* serviceMonitor/
mv grafana-* grafana/
mv kube-state-metrics-* kube-state-metrics/
mv alertmanager-* alertmanager/
mv node-exporter-* node-exporter/
mv prometheus-adapter* adapter/
mv prometheus-* prometheus/

# 报警规则
mv kube-prometheus-prometheusRule.yaml prometheus/
mv kubernetes-prometheusRule.yaml  prometheus/
 
kube-state-metric 用于输出聚合计数器类数据
k8s-prometheus-adapter 自定义API

[root@master manifests]# cat ./adapter/prometheus-adapter-apiService.yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    app.kubernetes.io/component: metrics-adapter
    app.kubernetes.io/name: prometheus-adapter
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 0.8.3
  name: v1beta1.metrics.k8s.io 
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: prometheus-adapter
    namespace: monitoring
  version: v1beta1
  versionPriority: 100

```

> 定义的APIServer会覆盖metrics-server的APIService所以不需要metrics-server也可以

```diff
[root@master ~]# kubectl get APIService
NAME                                   SERVICE                         AVAILABLE   AGE
...
+v1beta1.metrics.k8s.io                 monitoring/prometheus-adapter   True        85m # 注意走的prometheus APIService
```

删除metrics-server

```bash
# 清理metricsserver
[root@master ~]# kubectl delete -f components.yaml 

# 将prometheus的APIServer重新应用
[root@master ~]# kubectl apply -f kube-prometheus/manifests/adapter/
# top命令正常
[root@master ~]# kubectl top nodes
NAME                CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
master.magedu.com   1990m        99%    1396Mi          80%       
node01.magedu.com   343m         17%    1424Mi          81%       
node02.magedu.com   265m         13%    825Mi           47%       
node03.magedu.com   337m         16%    1166Mi          67%      
```



配置operator

```bash
manifests/setup/prometheus-operator-deployment.yaml
		resources:
          limits:
            cpu: 1
            memory: 1024Mi
          requests:
            cpu: 300m
            memory: 100Mi
```



```bash
# CRD 和prometheus operator
kubectl apply -f setup/
kubectl apply -f adapter/
[root@master adapter]# ll
total 56
-rw-r--r--. 1 root root  482 Feb  4 14:43 prometheus-adapter-apiService.yaml # 暴露服务
-rw-r--r--. 1 root root  576 Feb  4 14:43 prometheus-adapter-clusterRoleAggregatedMetricsReader.yaml # 核心资源访问获取
-rw-r--r--. 1 root root  494 Feb  4 14:43 prometheus-adapter-clusterRoleBindingDelegator.yaml # 认证api的token
-rw-r--r--. 1 root root  471 Feb  4 14:43 prometheus-adapter-clusterRoleBinding.yaml 
-rw-r--r--. 1 root root  378 Feb  4 14:43 prometheus-adapter-clusterRoleServerResources.yaml # 对自定义资源metrics.k8s.io 有所有权限
-rw-r--r--. 1 root root  409 Feb  4 14:43 prometheus-adapter-clusterRole.yaml # 对核心群组node/pod/svc/ns有读权限
-rw-r--r--. 1 root root 1568 Feb  4 14:43 prometheus-adapter-configMap.yaml
-rw-r--r--. 1 root root 1981 Feb  4 17:15 prometheus-adapter-deployment.yaml
-rw-r--r--. 1 root root  515 Feb  4 14:43 prometheus-adapter-roleBindingAuthReader.yaml # api读extension-apiserver-authentication] configmap文件, 自定义APIServer可以获取 aggregator传递过来的用户信息，CA信息，允许的CN信息。
-rw-r--r--. 1 root root  287 Feb  4 14:43 prometheus-adapter-serviceAccount.yaml # prometheus-adapter
-rw-r--r--. 1 root root  501 Feb  4 14:43 prometheus-adapter-service.yaml


kubectl apply -f alertmanager/
kubectl apply -f node-exporter/
kubectl apply -f kube-state-metrics/
kubectl apply -f grafana/
kubectl apply -f prometheus/
kubectl apply -f serviceMonitor/
```



## 暴露prometheus

```bash
kubectl edit svc -n monitoring prometheus-k8s
```



```diff

  ports:
  - name: web
    port: 9090
    protocol: TCP
    targetPort: web
+    nodePort: 30090
  selector:
    app: prometheus
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    prometheus: k8s
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
+  type: NodePort

```

网页访问http://172.16.1.100:30090/targets



## 部署 custom metrics-server

prometheus过于重量，又要kubectl top命令，指标分类

1. 核心指标(core metrics): cpu/mem/disk/io:  

   - prometheus-adaptor -> node-exporter 有存储

   - metrics-server -> cAdvisor 不存储
     - metrics.k8s.io/
     - Pod: metric-server

2. 扩展指标/自定义指标：监控系统 prometheus， 指标有： 应用程序指标 nginx连接数

   - prometheus 可存储

     - custom-metrics.k8s.io
     - Pod: prometheus

项目：

https://github.com/kubernetes-sigs/prometheus-adapter.git

> 这个才是自定义资源指标，APIService指向一个专用的Pod来提供自定义的资源指标。

以上部署的`prometheus operator`默认会提供APIService, 名称和metric-server提供的APIService名一样，这样`系统指标`来自metric-server 和prometheus.

- top核心指标 -> aggregator -> v1beta1.metrics.k8s.io -> **metrics-server pod** 或 prometheus operator提供的adaptor pod
- hpa 自定义指标 -> aggregator ->  custom.metrics.k8s.io -> `custom metrics-server`

基于自定义指标，完成pod自动伸缩

```bash
git clone https://github.com/kubernetes-sigs/prometheus-adapter.git
[root@master ~]# kubectl apply -f prometheus-adapter/deploy/manifests/
[root@master manifests]# sed -i 's@namespace: custom-metrics@namespace: monitoring@g' prometheus-adapter/deploy/manifests/*


```

> 去掉中文

### 为aggregated 生成证书连接api server

aggregator连接api server的证书。签署证书的机构是ApiServer信任的证书，证书名必须是serving.crt, serving.key

要求必须serving命名

```bash
[root@master ~]# cd kube-prometheus/manifests/adapter/
openssl genrsa -out serving.key 2048
openssl req -new -key serving.key -out serving.csr -subj "/CN=serving"

openssl x509 -req -in serving.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out serving.crt -days 3650

```

创建generic的secret

```bash
#monitoring名称空间中的volume-serving-cert
[root@master adapter]# kubectl create secret generic cm-adapter-serving-certs -n monitoring --from-file=./serving.crt  --from-file=./serving.key 
# 保原名generic
# 不保原名 tls.crt/tls.key, tls类型
```



### 配置自定义api server

adapter/prometheus-adapter-deployment.yaml

```diff
# cat prometheus-adapter/deploy/manifests/custom-metrics-apiserver-deployment.yaml 
    spec:
      serviceAccountName: custom-metrics-apiserver
      containers:
      - name: custom-metrics-apiserver
        image: directxman12/k8s-prometheus-adapter-amd64
        args:
+        # 连接apiserver的证书和端口。
        - --secure-port=6443
        - --tls-cert-file=/var/run/serving-cert/serving.crt 
        - --tls-private-key-file=/var/run/serving-cert/serving.key
        
        - --logtostderr=true
+        # 连接prometheus的地址
+        - --prometheus-url=http://prometheus-k8s.monitoring.svc:9090/
        - --metrics-relist-interval=1m
        # 日志级别
+        - --v=10 
        - --config=/etc/adapter/config.yaml
       ports:
        - containerPort: 6443
        volumeMounts:
+        - mountPath: /var/run/serving-cert
          name: volume-serving-cert1
          readOnly: true
        - mountPath: /etc/adapter/
          name: config
          readOnly: true
        - mountPath: /tmp
          name: tmp-vol
      volumes:
+      - name: volume-serving-cert1
+        secret:
+          secretName: cm-adapter-serving-certs

```

```bash
[root@master ~]# kubectl apply -f prometheus-adapter/deploy/manifests/
```

获取自定义资源

```bash
[root@master m1]# kubectl api-versions
custom.metrics.k8s.io/v1beta1
custom.metrics.k8s.io/v1beta2

[root@master ~]# kubectl get APIService
																			 # 正常
v1beta1.custom.metrics.k8s.io          monitoring/custom-metrics-apiserver   True        55m
v1beta1.external.metrics.k8s.io        monitoring/custom-metrics-apiserver   True        55m
v1beta2.custom.metrics.k8s.io          monitoring/custom-metrics-apiserver   True        55m


[root@master m1]# kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta2" | jq .
    {
      "name": "namespaces/node_network_device_id",
      "singularName": "",
      "namespaced": false,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
    {
      "name": "namespaces/node_memory_Active_file_bytes",
      "singularName": "",
      "namespaced": false,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },

# 接口可以查询prometheus所有数据

```



## grafana + promQL

https://github.com/kubernetes-retired/heapster/blob/master/deploy/kube-config/influxdb/grafana.yaml

> 以上是独立的grafana, 但是prometheus已经有grafana了





```bash
[root@master ~]# cat kube-prometheus/manifests/grafana/grafana-deployment.yaml 
	volumes:
      - emptyDir: {}
        name: grafana-storage

# 可以配置存储
```

获取grafana

```bash
kubectl get svc -A | grep gra
```

网页访问 http://172.16.1.100:30256/login

### 配置prometheus数据源

![image-20210204181804067](http://myapp.img.mykernel.cn/image-20210204181804067.png)

### 选择内建模板

![image-20210204181839985](http://myapp.img.mykernel.cn/image-20210204181839985.png)



![image-20210204181902926](http://myapp.img.mykernel.cn/image-20210204181902926.png)

### 添加面板

kubernetes deployment statefulset daemonset



k8s cluster smmary





## 利用核心指标和自定义指标完成容器弹性伸缩

### hpa v1版本(metric-server/prometheus adapter)

kube-prometheus提供的adapter定义APIService会覆盖metrics-server定义的APIService

```bash
[root@node01 ~]# kubectl explain hpa
KIND:     HorizontalPodAutoscaler
VERSION:  autoscaling/v1          # HPA版本
   apiVersion	<string>
   kind	<string>
   metadata	<Object>
   spec	<Object>
   status	<Object>
   
   
# 
[root@node01 ~]# kubectl explain hpa.spec | grep '>'
RESOURCE: spec <Object>
   maxReplicas	<integer> -required- # 最大副本数
   minReplicas	<integer>
   scaleTargetRef	<Object> -required- # 监控哪个控制器   
   targetCPUUtilizationPercentage	<integer> # cpu利用率
```

#### 测试deploy伸缩

启动deploy控制的pod, 资源利用率一定要限制，这样方便实验

```diff
[root@master chapter14] myapp.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
  labels:
    app: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
+        resources:
+          requests:
+            cpu: 50m
+            memory: 64Mi
+          limits:                    
+            cpu: 50m
+            memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  labels:
    app: myapp
  namespace: default
spec:
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: NodePort

```

```bash
[root@master chapter14]# kubectl apply -f myapp.yaml

# 获取服务
[root@master chapter14]#  kubectl get svc 
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
myapp-svc    NodePort    10.109.128.68   <none>        80:31466/TCP   93s
```

查看pod运行状态，查看运行

```bash
[root@master chapter14]# kubectl top pods
NAME                     CPU(cores)   MEMORY(bytes)   
myapp-779867bcfc-652pr   0m           1Mi             
myapp-779867bcfc-7qlr9   0m           2Mi         
```

安装压测工具`httpd-tools`, `apache2-utils`

```bash
# 测试, ipvs调度
[root@master chapter14]# curl 172.16.1.101:31466
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>

[root@master chapter14]# ab -c 100 -n 10000 http://172.16.1.102:31466/index.html

# 观察CPU
[root@node01 ~]# kubectl top pods 
NAME                     CPU(cores)   MEMORY(bytes)   
myapp-779867bcfc-652pr   13m          1Mi             
myapp-779867bcfc-7qlr9   27m          2Mi 

# cpu已经超载，但是pod个数无变化
[root@node01 ~]# kubectl top pods 
NAME                     CPU(cores)   MEMORY(bytes)   
myapp-779867bcfc-652pr   61m          1Mi             
myapp-779867bcfc-7qlr9   22m          2Mi     
```

创建自动伸缩

```bash
kubectl autoscale -h

# 先停止压测
# 实验目的定义50%扩展
kubectl autoscale deployment myapp -n default --min=2 --max=5 --cpu-percent=50

# 描述
[root@master chapter14]# kubectl get hpa
NAME    REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
myapp   Deployment/myapp   17%/50%   2         5         4          2m33s

```

> --cpu-percent=60 高于扩展，低于缩减，不能超界[2,5]

```bash
# 终端1
[root@node01 ~]# kubectl get pod -w

# 终端2
ab -c 100 -n 10000 http://172.16.1.102:31466/index.html 


# 观察到终端1自动扩展， 计算：当前占用m之和求平均 占 每个pod最低需求之和求平均  计算的值和--cpu-percent=60 比较
[root@node01 ~]# kubectl get pod -w
NAME                     READY   STATUS    RESTARTS   AGE
myapp-779867bcfc-652pr   1/1     Running   0          16m
myapp-779867bcfc-7qlr9   1/1     Running   0          16m
myapp-779867bcfc-vlqg8   1/1     Running   0          20s
myapp-779867bcfc-wxcjk   1/1     Running   0          20s


# 压测完了
[root@master chapter14]# kubectl top pod
NAME                     CPU(cores)   MEMORY(bytes)   
myapp-779867bcfc-652pr   0m           1Mi             
myapp-779867bcfc-7qlr9   0m           2Mi             
myapp-779867bcfc-vlqg8   0m           1Mi             
myapp-779867bcfc-wxcjk   0m           1Mi      

...
自动缩容
```

> 指标采集，有滞后性。太实时了，可能反复跳动
>
> 扩容和缩容也有滞后性



### hpa v2版本(custom metric-server)

```bash
[root@node01 ~]# kubectl get APIService | grep auto
v1.autoscaling                         Local                           True        21d
v2beta1.autoscaling                    Local                           True        21d # v2版本beta1
v2beta2.autoscaling                    Local                           True        21d # v2版本beta2
```

查看官方文档 `kubernetes hpav2`

https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#autoscaling-on-multiple-metrics-and-custom-metrics

#### Resource: cpu

基于cpu资源利用率

```bash
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment # 资源控制
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics: # 不是HPA V1的cpupercentage. 而是metrics.  
  - type: Resource # 有多种类型
    resource:
      name: cpu
      target:
        type: Utilization # 使用率
        averageUtilization: 50
status:
  observedGeneration: 1
  lastScaleTime: <some-time>
  currentReplicas: 1
  desiredReplicas: 1
  currentMetrics:
  - type: Resource
    resource:
      name: cpu
      current:
        averageUtilization: 0
        averageValue: 0
```

> - type: Resource # 有多种类型; pods/objects/resource ，用的多是resource

> 不要使用2个hpa控制一个控制器

#### Pods: metric

Pod内部输出的资源指标：prometheus定义annotation，定义路径 

- 外部sidecar
- 内部的输出：http/https /metrics

```bash
type: Pods
pods:
  metric:
    name: packets-per-second # pod自已的指标
  target: 
    type: AverageValue  # 平均值
    averageValue: 1k   # 小于1K
```

#### Object

```bash
type: Object
object:
  metric:
    name: requests-per-second
  describedObject:
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    name: main-route
  target:
    type: Value
    value: 2k
```

#### 混合

如果你的监控系统能够提供网络流量数据

```bash
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: AverageUtilization
        averageUtilization: 50
  - type: Pods
    pods:
      metric:
        name: packets-per-second # 每s报文
      target:
        type: AverageValue
        averageValue: 1k
  - type: Object
    object:
      metric:
        name: requests-per-second # 每s请求
      describedObject: # 期望的是
        apiVersion: networking.k8s.io/v1beta1
        kind: Ingress
        name: main-route # 对象
      target:
        kind: Value # 指标的值
        value: 10k # 10k以内
status:
  observedGeneration: 1
  lastScaleTime: <some-time>
  currentReplicas: 1
  desiredReplicas: 1
  currentMetrics:
  - type: Resource
    resource:
      name: cpu
    current:
      averageUtilization: 0
      averageValue: 0
  - type: Object
    object:
      metric:
        name: requests-per-second
      describedObject:
        apiVersion: networking.k8s.io/v1beta1
        kind: Ingress
        name: main-route
      current:
        value: 10k
```

#### 示例自定义指标

##### 部署metrics-app deploy

```bash
[root@master chapter14]# cat metrics-app.yaml 
apiVersion: apps/v1
kind: Deployment # deploy
metadata:
  labels:
    app: metrics-app
  name: metrics-app # deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: metrics-app
  template:
    metadata:
      labels:
        app: metrics-app
      annotations: # 内部输出url
        # based on your Prometheus config above, this tells prometheus
        # to scrape this pod for metrics on port 80 at "/metrics"
        prometheus.io/scrape: "true" # 允许prom抓数据
        prometheus.io/port: "80"      
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - image: ikubernetes/metrics-app # 镜像
        name: metrics-app
        ports:
        - name: web
          containerPort: 80 # 这个url
        resources: # 限制资源
          requests:
            cpu: 200m
            memory: 256Mi
        readinessProbe: # 就绪
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
        livenessProbe: # 存活
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
---
apiVersion: v1
kind: Service # 接入pod流量
metadata:
  name: metrics-app
  labels:
    app: metrics-app
spec:
  ports:
  - name: web
    port: 80
    targetPort: 80
  selector:
    app: metrics-app

```



```bash
# 部署
[root@master chapter14]# kubectl apply -f metrics-app.yaml
```

通过80端口获取指标

```bash
[root@master chapter14]# kubectl get po -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE                NOMINATED NODE   READINESS GATES
metrics-app-65b7b55bdb-k8sc6   1/1     Running   0          75s   10.244.2.44   node02.magedu.com   <none>           <none>
metrics-app-65b7b55bdb-nps4g   1/1     Running   0          76s   10.244.1.43   node01.magedu.com   <none>           <none>


[root@master chapter14]# curl 10.244.2.44:80/metrics
# HELP http_requests_total The amount of requests in total
# TYPE http_requests_total counter
http_requests_total 13
# HELP http_requests_per_second The amount of requests per second the latest ten seconds
# TYPE http_requests_per_second gauge
http_requests_per_second 0.5



# 不断请求不断增加
[root@master chapter14]# curl 10.244.2.44:80/metrics
# HELP http_requests_total The amount of requests in total
# TYPE http_requests_total counter
http_requests_total 55
# HELP http_requests_per_second The amount of requests per second the latest ten seconds
# TYPE http_requests_per_second gauge
http_requests_per_second 1.3

```

> http_requests_total 13              # 服务器接收请求总数。 counter累加
> http_requests_per_second 0.5 # 每s平均接收请求。    上面(total2 - total1)/2

##### 验证prometheus抓取到指标

在prometheus界面获取http_requests_per_second 

![image-20210205103607112](http://myapp.img.mykernel.cn/image-20210205103607112.png)

###### annotation自动发现

K8S github.com官方仓库的1.16版本的cluster/addons/prometheus中定义

通过pod的annotation自动发现https://github.com/kubernetes/kubernetes/blob/release-1.16/cluster/addons/prometheus/prometheus-configmap.yaml#L125

```bash
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs: # 修改target标签集
      - action: keep # 删除指标，不匹配source_labels各标签串连值，删除target
        regex: true
        source_labels:  #引用哪些已有标签: 将值连接起来
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape # https://www.cnblogs.com/skyflask/p/11498834.html prometheus配置 自动发现每个annotations
      - action: replace # 匹配的值放在target_label
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: ([^:]+)(?::\d+)?;(\d+) #  对值匹配，匹配结果赋值给target_label
        replacement: $1:$2 # $1: 不是:任意长度，主机名。 端口
        source_labels: # 连接
        - __address__
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        target_label: __address__ # 结果
      - action: labelmap # 创建标签 匹配标签名，匹配到的标签的值赋值给replacement指定的标签名之上
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: kubernetes_namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: kubernetes_pod_name
```

查看operator中prometheus配置文件

```bash
# 配置文件
 kubectl get pods -n monitoring prometheus-k8s-0  -o yaml
# secrect中
  volumes:
  - name: config
    secret:
      defaultMode: 420
      secretName: prometheus-k8s

#kubectl get secret -n monitoring prometheus-k8s -o yaml

[root@master chapter14]# echo 'H4sIAAAAAAAA/+xdzXLb...bwAAA==' | base64 -d | gunzip  > prometheus.cfg

```

结果发现没有定义抓取annotation, 可以修改配置后，重新应用prometheus.

但是prometheus operator提供了自已监控的方法



###### 基于operator 的serviceMonitor

prometheus operator提供了自已监控的方法

https://yunlzheng.gitbook.io/prometheus-book/part-iii-prometheus-shi-zhan/operator/use-operator-manage-prometheus

为了能够自动化的管理Prometheus的配置，Prometheus Operator使用了自定义资源类型ServiceMonitor来描述监控对象的信息。

并不是你定义serviceMonitor, prometheus就可以使用，需要定义serviceMonitorSelector

```bash
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  namespace: monitoring 
  labels:
    team: frontend
spec:
  namespaceSelector: # 省略时监控同名称空间的svc；否则监控匹配名称空间的svc
    matchNames:
    - default
  selector: # 选择service
    matchLabels:
      app: example-app
  endpoints: # 指定 svc输出的metrics的端口
  - port: web
```

如果希望ServiceMonitor可以关联任意命名空间下的标签，则通过以下方式定义：

```bash
spec:
  namespaceSelector:
    any: true
```

如果监控的Target对象启用了BasicAuth认证，那在定义ServiceMonitor对象时，可以使用endpoints配置中定义basicAuth如下所示：

```diff
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  namespace: monitoring
  labels:
    team: frontend
spec:
  namespaceSelector:
    matchNames:
    - default
  selector:
    matchLabels:
      app: example-app
  endpoints:
+  - basicAuth:
+      password:
+        name: basic-auth
+        key: password
+      username:
+        name: basic-auth
+        key: user
+    port: web
```

其中basicAuth中关联了名为basic-auth的Secret对象，用户需要手动将认证信息保存到Secret中:

```bash
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
data:
  password: dG9vcg== # base64编码后的密码
  user: YWRtaW4= # base64编码后的用户名
type: Opaque
```

定义prometheus选择serviceMonitor

```diff
apiVersion: monitoring.coreos.com/v1
+kind: Prometheus # operator的自定义资源类型
metadata:
  name: inst
  namespace: monitoring
spec:
+  serviceMonitorSelector:
+    matchLabels:
+      team: frontend
  resources:
    requests:
      memory: 400Mi
```

应用后，prometheus配置文件自动生成scrape_configs.monitoring/example-app/0 

```bash
scrape_configs:
- job_name: monitoring/example-app/0
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  kubernetes_sd_configs:
  - role: endpoints
    namespaces:
      names:
      - default
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_label_app]
    separator: ;
    regex: example-app
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_endpoint_port_name]
    separator: ;
    regex: web
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
    separator: ;
    regex: Node;(.*)
    target_label: node
    replacement: ${1}
    action: replace
  - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
    separator: ;
    regex: Pod;(.*)
    target_label: pod
    replacement: ${1}
    action: replace
  - source_labels: [__meta_kubernetes_namespace]
    separator: ;
    regex: (.*)
    target_label: namespace
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_name]
    separator: ;
    regex: (.*)
    target_label: service
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_name]
    separator: ;
    regex: (.*)
    target_label: pod
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_name]
    separator: ;
    regex: (.*)
    target_label: job
    replacement: ${1}
    action: replace
  - separator: ;
    regex: (.*)
    target_label: endpoint
    replacement: web
    action: replace
```

由于默认创建的Prometheus实例使用的是monitoring命名空间下的default账号，该账号并没有权限能够获取default命名空间下的任何资源信息。

授权prometheus实例的serviceaccount，集群读。clusterrolebinding + sa + 集群读

修改prometheus的serviceaccount

```diff
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: inst
  namespace: monitoring
spec:
+  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  resources:
    requests:
      memory: 400Mi
```

等待Prometheus Operator完成相关配置变更后，此时查看Prometheus，我们就能看到当前Prometheus已经能够正常的采集实例应用的相关监控数据了。

##### 配置自动发现metric-app

prometheus配置发现

```bash
[root@master ~]# cat kube-prometheus/manifests/prometheus/prometheus-prometheus.yaml
  podMonitorNamespaceSelector: {} # 空匹配自已的名称空间 kubectl explain Prometheus
  podMonitorSelector: {}
  probeNamespaceSelector: {}
  probeSelector: {}
  serviceMonitorNamespaceSelector: {} # 空选择自已的名称空间的serviceMonitor
  serviceMonitorSelector: {}
```

由于serviceMonitorNamespaceSelector为空，所以prometheus只会选择自已名称空间中的serviceMonitor.

查看自已名称空间中的Monitor定义

在monitoring名称空间定义 serviceMonitor

```bash
kubectl explain  servicemonitor.spec

"metrics-app-serviceMonitor.yaml" 14L, 417C                                                                                    
kind: ServiceMonitor
apiVersion: monitoring.coreos.com/v1
metadata:
  name: metrics-app
  namespace: monitoring # prometheus定义只有这个名称空间中的servicemonitor才发现
spec:
  endpoints:  # target的端口，默认/metrics
  - port: web
  namespaceSelector:
    matchNames: # metrics-app的名称空间
    - default
  selector:
    matchLabels: # metrics-app的svc标签
      app: metrics-app

```

prometheus的configure文件发现自动重载

http://172.16.1.100:30090/config

![image-20210205112210654](http://myapp.img.mykernel.cn/image-20210205112210654.png)

prometheus发现target，即获取指标的端点

![image-20210205112418773](http://myapp.img.mykernel.cn/image-20210205112418773.png)

prometheus查看指标

![image-20210205112303484](http://myapp.img.mykernel.cn/image-20210205112303484.png)



##### 配置metric hpa

下面基于请求速率调整pod规模

```diff
[root@master chapter14]# cat metrics-app-hpa.yaml 
kind: HorizontalPodAutoscaler
+apiVersion: autoscaling/v2beta1 # 版本
metadata:
  name: metrics-app-hpa
spec:
  scaleTargetRef:
    # point the HPA at the sample application
    # you created above
    apiVersion: apps/v1
+    kind: Deployment     # 控制器
    name: metrics-app
  # autoscale between 2 and 10 replicas
+  minReplicas: 2 # 最低副本
  maxReplicas: 10
  metrics:
  # use a "Pods" metric, which takes the average of the
  # given metric across all pods controlled by the autoscaling target
  - type: Pods
+    pods:  # pod自已的指标
      # use the metric that you used above: pods/http_requests
+      metricName: http_requests       # 自动计算(total - total) / 时间
      # target 500 milli-requests per second,
      # which is 1 request every two seconds
+      targetAverageValue: 500m              # 1s 0.5个请求

```

```bash
[root@master chapter14]# kubectl apply -f metrics-app-hpa.yaml
horizontalpodautoscaler.autoscaling/metrics-app-hpa created
[root@master chapter14]# kubectl get hpa
NAME              REFERENCE                TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
																				# 当前pod
metrics-app-hpa   Deployment/metrics-app   <unknown>/500m   2         10        2          2m35s

```

##### 测试伸缩

```bash
[root@node01 ~]# kubectl get svc 
metrics-app   ClusterIP   10.97.76.29     <none>        80/TCP         7m57s

[root@node01 ~]# while true; do curl 10.97.76.29:80; sleep 2; done
Hello! My name is metrics-app-65b7b55bdb-nps4g. The last 10 seconds, the average QPS has been 0.6. Total requests served: 195
Hello! My name is metrics-app-65b7b55bdb-k8sc6. The last 10 seconds, the average QPS has been 0.6. Total requests served: 207
Hello! My name is metrics-app-65b7b55bdb-nps4g. The last 10 seconds, the average QPS has been 0.6. Total requests served: 198
[root@node01 ~]# while true; do curl 10.97.76.29:80; sleep .2; done
Hello! My name is metrics-app-65b7b55bdb-k8sc6. The last 10 seconds, the average QPS has been 2.4. Total requests served: 293
Hello! My name is metrics-app-65b7b55bdb-nps4g. The last 10 seconds, the average QPS has been 2.4. Total requests served: 283
Hello! My name is metrics-app-65b7b55bdb-k8sc6. The last 10 seconds, the average QPS has been 2.5. Total requests served: 294
Hello! My name is metrics-app-65b7b55bdb-nps4g. The last 10 seconds, the average QPS has been 2.5. Total requests served: 284
Hello! My name is metrics-app-65b7b55bdb-k8sc6. The last 10 seconds, the average QPS has been 2.6. Total requests served: 295
# 注意过去10s的平均值，一直在上升

0.2s一次，1s就5次请求。2个pod承载1半，就是1s2.5个请求。
```

查看 prometheus状态

![image-20210205125828423](http://myapp.img.mykernel.cn/image-20210205125828423.png)

获取hpa

```diff
[root@master ~]#  kubectl get hpa
NAME              REFERENCE                TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
										  # 当前1s一个请求
metrics-app-hpa   Deployment/metrics-app   977m/500m   2         10        10         15h
myapp             Deployment/myapp         0%/50%      2         5         2          16h
[root@master ~]# 
```

#### 解读发现规则

##### [annotation](http://blog.mykernel.cn/2021/02/03/%E8%B5%84%E6%BA%90%E6%8C%87%E6%A0%87%E4%B8%8EHPA%E6%8E%A7%E5%88%B6%E5%99%A8/#%E9%AA%8C%E8%AF%81prometheus%E6%8A%93%E5%8F%96%E5%88%B0%E6%8C%87%E6%A0%87)

现在已经自动发现每个pod的target了

现在看一下之前 annotation自动发现

- [prometheus定义抓取annotaion](https://github.com/kubernetes/kubernetes/blob/release-1.16/cluster/addons/prometheus/prometheus-configmap.yaml#L125)

![image-20210205113216866](http://myapp.img.mykernel.cn/image-20210205113216866.png)

```bash
      - action: replace
        regex: ([^:]+)(?::\d+)?;(\d+) # 匹配这个字符串 10.244.1.43:80;80
        replacement: $1:$2 10.244.1.43
        source_labels: # 会将列表标签基于separator连接, 10.244.1.43:80;80
        - __address__ # 10.244.1.43:80
        - __meta_kubernetes_pod_annotation_prometheus_io_port # 80
        target_label: __address__
```

> ```
> (?:...)
> ```
>
> > 常规圆括号的非捕获版本。 匹配括号内的任何正则表达式,但匹配的子字符串在执行匹配或稍后引用模式后无法检索。

##### serviceMonitor

prometheus自动获取同名称空间中的servicemonitor,

servicemonitor获取指定名称空间中的指定标签对应的service

获取到target后，进行relabel 标签，进入网页

![image-20210205113216866](http://myapp.img.mykernel.cn/image-20210205113216866.png)



promestatus -> status -> configuration

```bash
- job_name: monitoring/metrics-app/0
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_label_app] 
    separator: ;
    regex: metrics-app
    replacement: $1
    action: keep # 匹配时保留

  - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
    separator: ;
    regex: Node;(.*)
    target_label: node
    replacement: ${1}
    action: replace
  - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
    separator: ;
    regex: Pod;(.*)
    target_label: pod
    replacement: ${1}
    action: replace
  - source_labels: [__meta_kubernetes_namespace]
    separator: ;
    regex: (.*)
    target_label: namespace
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_name]
    separator: ;
    regex: (.*)
    target_label: service
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_name]
    separator: ;
    regex: (.*)
    target_label: pod
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_container_name]
    separator: ;
    regex: (.*)
    target_label: container
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_name]
    separator: ;
    regex: (.*)
    target_label: job
    replacement: ${1}
    action: replace
  - separator: ;
    regex: (.*)
    target_label: endpoint
    replacement: web
    action: replace
  - source_labels: [__address__]
    separator: ;
    regex: (.*)
    modulus: 1
    target_label: __tmp_hash
    replacement: $1
    action: hashmod
  - source_labels: [__tmp_hash]
    separator: ;
    regex: "0"
    replacement: $1
    action: keep
  kubernetes_sd_configs: # 发现k8s
  - role: endpoints # 发现端点
    namespaces:
      names:
      - default
```

> 发现 -> 配置 -> 重新打标(before relabeling, meta开头元标签。__address__当前target地址和端口,  metric_path 路径 scheme 协议) -> 配置target到prometheus之上，抓取 -> 指标重新打标 metric_relable_configs
>
> 
>
> kubernetes_sd_configs
>
> 每个资源独有发现机制role
>
> 	node role，检索节点上的, internalip externalip, hostip, hostname, ... 发现第1个地址作为__address__
> 	
> 	pod role 每个pod的...
> 	
> 	service role 每个service的元数据
> 	
> 	endpoint role 每个endpoint的元数据





