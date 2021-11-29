---
title: prometheus监控kubernetes
date: 2021-03-15 05:36:25
tags:
- 个人日记
- kubernetes
- prometheus
---





# 前言
<!--more-->

在当下这个微服务与容器化的时代，很多企业的监控系统中，所有组件及配置均实现了容器化并由Kubernetes编排。如果需要在任意Kubernetes集群里都实现一键部署，且需要变更系统时仅需修改相关编排文件，那么Prometheus就是不二的选择了。Prometheus的动态发现机制，不仅支持swarm原生集群，还支持Kubernetes容器集群的监控，它是目前容器监控最好的解决方案。

<!--more-->
# 3种监控系统

## nagios

- nagios daemon, 组织协调组件,负责监控信息的组织与展示.
- nagios plugins,  Nagios核心组件自带的组件，还包括一些用户自开发的插件。这些组件监控各项指标，并将采集到的数据回送给Nagios服务器。
- NRPE（Nagios Remote Plugin Executor）是安装在监控主机客户端上的代理程序（Linux系统是NRPE软件）。通过NRPE可获取监控数据，这些数据会被再回送给Nagios服务器。默认使用的端口是5666

![image-20210315134805621](http://myapp.img.mykernel.cn/image-20210315134805621.png)

Nagios出现于20世纪90年代，它的思想仍然是通过复杂的交错文本文件、脚本和手动程序进行管理，基于文本文件的配置每次进行更改时都需要进行重置，这也使得在文件分布复杂的情况下必须借助第三方工具（例如Chef或Puppet）进行部署，因此Nagios如何进行有效配置管理是一个瓶颈[2]。为了使用Nagios监控，有必要熟练掌握处理数百个自定义脚本的方法，若这些脚本由不同的人采用不同的风格编写，那么处理脚本的过程几乎会变成某种“黑魔法”。对很多人来说，管理Nagios是非常复杂的，这最终导致Nagios成为软件与定制开发之间的奇怪组合。



## zabbix

适用于绝大多数IT基础架构、服务、应用程序、云、资源等的解决方案

Zabbix可以通过Agent(JVM Agent、IPMI Agent、SNMP Agent等)及Proxy的形式，采集主机性能、网络设备性能、数据库性能的相关数据，以及FTP、SNMP、IPMI、JMX、Telnet、SSH等通用协议的相关信息，采集信息会上传到Zabbix的服务端并存储在数据库中，相关人员通过Web管理界面就可查看报表、实时图形化数据、进行7×24小时集中监控、定制告警等。

![image-20210315135206473](http://myapp.img.mykernel.cn/image-20210315135206473.png)

## open-falcon

小米开源的监控系统，是为了解决Zabbix的不足而出现的产物，它由Go语言开发而成，小米、滴滴、美团等超过200家公司都在不同程度上使用Open-Falcon。小米同样经历过Zabbix的时代，但是当机器数量、业务量上来以后（尤其是达到上万上报节点），Zabbix在性能、管理成本、易用性、监控统一等方面就有些力不从心了。Open-Falcon具有数据采集免配置、容量水平扩展、告警策略自动发现、告警设置人性化、历史数据高效查询、Dashboard人性化、架构设计高可用等特点。

Falcon的文档目前分为0.1和0.2版本，且每个版本都有中、英两个版本[10]，社区贡献了MySQL、Redis、Windows、交换机、LVS、MongoDB、Memcache、Docker、Mesos、URL监控等多种插件支持

![image-20210315135327927](http://myapp.img.mykernel.cn/image-20210315135327927.png)

Open-Falcon需要在Linux服务器上安装**Falcon-agent**，这是一款用Go语言开发的Daemon程序，用于自发现且采集主机上的各种指标数据，主要包括CPU、磁盘、I/O、Load、内存、网络、端口存活、进程存活、ntpoffset（插件）、某个进程资源消耗（插件）、netstat和ss等相关统计项，以及机器内核配置参数等指标，指标数据采集完成后主动上报。

Falcon-agent配备了一个**proxy-gateway**，用户可以使用HTTP接口将数据推送到本机的Gateway，从而将数据转发到服务端。Transfer模块接收到Falcon-agent的数据以后，对数据进行过滤梳理以后通过一致性Hash算法发送到Judge告警判定模块和Graph数据存储归档模块。

**Heartbeat server**（心跳服务）简称HBS，每个Agent都会周期性地通过RPC方式将自己的状态上报给HBS，主要包括主机名、主机IP、Agent版本和插件版本，Agent还会从HBS获取需要执行的采集任务和自定义插件。



# kubernetes部署prometheus监控

## prometheus

### 准备dockerfile

```bash
root@master01:/data/weizhixiu/dockerfile/prometheus# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/alpine:base

ADD prometheus-2.25.0.linux-amd64.tar.gz /usr/local/

ENTRYPOINT ["/usr/local/prometheus-2.25.0.linux-amd64/prometheus"]
```

```bash
# docker build -t harbor.youwoyouqu.io/pub/prometheus:2.25.0 ./
root@master01:/data/weizhixiu/dockerfile/prometheus# docker run -it --rm harbor.youwoyouqu.io/pub/prometheus:2.25.0 --version
prometheus, version 2.25.0 (branch: HEAD, revision: a6be548dbc17780d562a39c0e4bd0bd4c00ad6e2)
  build user:       root@615f028225c9
  build date:       20210217-14:17:24
  go version:       go1.15.8
  platform:         linux/amd64
  
  
# docker push harbor.youwoyouqu.io/pub/prometheus:2.25.0
```

### 准备yaml

参考：https://github.com/kubernetes/kubernetes/tree/release-1.12/cluster/addons/prometheus

```diff
root@master01:/data/weizhixiu/yaml/prometheus# cat prometheus-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus 
  namespace: monitoring
spec:
  replicas: 1
  strategy:
   rollingUpdate:  
+     maxSurge: 1 #确保先创建pod, 后停止
+     maxUnavailable: 0
   type: "RollingUpdate"
  selector:
    matchLabels:
      k8s-app: prometheus
  template:
    metadata:
      labels:
        k8s-app: prometheus
    spec:
+      serviceAccountName: prometheus     # 此pod需要获取k8s api权限
      initContainers:
      - name: "init-chown-data"
        image: harbor.youwoyouqu.io/pub/busybox:latest
        imagePullPolicy: "IfNotPresent"
        command: ["chown", "-R", "65534:65534", "/data"]
        volumeMounts:
        - name: prometheus-data
          mountPath: /data
      containers:
        - name: prometheus-server
          image:  harbor.youwoyouqu.io/pub/prometheus:2.25.0
          imagePullPolicy: "IfNotPresent"
          args:
            - --config.file=/etc/config/prometheus.yml
            - --storage.tsdb.path=/data
            - --web.console.libraries=/usr/local/prometheus-2.25.0.linux-amd64/console_libraries
            - --web.console.templates=/usr/local/prometheus-2.25.0.linux-amd64/consoles
            - --web.enable-lifecycle
          ports:
            - containerPort: 9090
          readinessProbe:
            httpGet:
              path: /-/ready
              port: 9090
            initialDelaySeconds: 30
            timeoutSeconds: 30
          livenessProbe:
            httpGet:
              path: /-/healthy
              port: 9090
            initialDelaySeconds: 30
            timeoutSeconds: 30
          resources:
            limits:
              cpu: 200m
              memory: 1000Mi
            requests:
              cpu: 200m
              memory: 1000Mi
            
          volumeMounts:
            # config
            - name: config-volume
+              mountPath: /etc/config
              
            # rules
            - name:  rules-volume
+              mountPath: /etc/rules/

            # node exporter rules
            - name:  node-exporter-config-volume
+              mountPath: /etc/node-exporter-rules/

            - name: prometheus-data
              mountPath: /data
              subPath: ""
      terminationGracePeriodSeconds: 300
      volumes:
        - name: node-exporter-config-volume
          configMap:
            name: node-exporter-configmap
        - name: config-volume
          configMap:
            name: prometheus-config
        - name: rules-volume
          configMap:
            name: prometheus-operator-rules
        - name: prometheus-data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: prometheus-service
  name: prometheus-service
  namespace: monitoring
spec:
  ports:
  - name: 9090-9090
    port: 9090
    protocol: TCP
    targetPort: 9090
    nodePort: 32094
  selector:
    k8s-app: prometheus
  type: NodePort
```

#### 准备prometheus账号权限

```diff
root@master01:/data/weizhixiu/yaml/prometheus# cat rbac-setup.yml 
# To have Prometheus retrieve metrics from Kubelets with authentication and
# authorization enabled (which is highly recommended and included in security
# benchmarks) the following flags must be set on the kubelet(s):
#
# --authentication-token-webhook
# --authorization-mode=Webhook
#
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  - networking.k8s.io
  - networking.k8s.io/v1       #level=warn ts=2021-03-11T04:10:43.102Z caller=klog.go:76 component=k8s_client_runtime func=Warning msg="networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress"
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics", "/metrics/cadvisor"]
  verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
+  namespace: monitoring # 同deployment
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
+  namespace: monitoring #level=error ts=2021-03-11T04:08:51.395Z caller=klog.go:96 component=k8s_client_runtime func=ErrorDepth msg="pkg/mod/k8s.io/client-go@v0.20.2/tools/cache/reflector.go:167: Failed to watch *v1.Pod: failed to list *v1.Pod: pods is forbidden: User \"system:serviceaccount:monitoring:prometheus\" cannot list resource \"pods\" in API group \"\" at the cluster scope"
```

即monitoring的prometheus有prometheus集群角色的权限



#### 准备prometheus.yml配置文件

https://github.com/kubernetes/kubernetes/blob/release-1.12/cluster/addons/prometheus/prometheus-configmap.yaml

https://github.com/prometheus/prometheus/blob/release-2.25/documentation/examples/prometheus-kubernetes.yml

https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config



```diff
root@master01:/data/weizhixiu/yaml/prometheus# cat prometheus-configmap.yaml 
apiVersion: v1
data:
  prometheus.yml: |
    global:
      scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
      evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
    # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
    rule_files:
    - "/etc/rules/*.yml"    # 由于deploy挂载configmap在此路径 
+    - "/etc/node-exporter-rules/*.yml" # 加载node相关的通用报警  https://www.cnblogs.com/zhaojiedi1992/p/zhaojiedi_liunx_65_prometheus_alertmanager_rule.html
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
     
    - job_name: 'elasticsearch-cluster' 
      static_configs:
      - targets: ['192.168.0.171:9114']
      relabel_configs:
      - target_label: address
        replacement: 192.168.0.171:9200

    - job_name: 'mysql' 
      static_configs:
      - targets: ['192.168.0.171:9104']
      relabel_configs:
+      - target_label: address     # 添加label
        replacement: 192.168.0.171:3306

    - job_name: 'redis' 
      static_configs:
      - targets: ['192.168.0.171:9121']
      relabel_configs:
      - target_label: address
        replacement: 192.168.0.171:6379


    # 监控kubernetes , reference: 
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names: []
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    ########################
    #    # 监控pod, 此方式可以监控所有pod.
    #    # Example scrape config for pods
    #    #
    #    # The relabeling allows the actual pod scrape to be configured
    #    # for all the declared ports (or port-free target if none is declared)
    #    # or only some ports.
    #    - job_name: 'kubernetes-pods'
    #
    #      kubernetes_sd_configs:
    #      - role: pod
    #
    #      relabel_configs:
    #      # Example relabel to scrape only pods that have
    #      # "example.io/should_be_scraped = true" annotation. # 启用此项时，只要明确=true才抓取
    #      #  - source_labels: [__meta_kubernetes_pod_annotation_example_io_should_be_scraped]
    #      #    action: keep
    #      #    regex: true
    #      #
    #      # Example relabel to customize metric path based on pod
    #      # "example.io/metric_path = <metric path>" annotation.
    #      #  - source_labels: [__meta_kubernetes_pod_annotation_example_io_metric_path]
    #      #    action: replace
    #      #    target_label: __metrics_path__
    #      #    regex: (.+)
    #      #
    #      # Example relabel to scrape only single, desired port for the pod
    #      # based on pod "example.io/scrape_port = <port>" annotation.
    #      #  - source_labels: [__address__, __meta_kubernetes_pod_annotation_example_io_scrape_port]
    #      #    action: replace
    #      #    regex: ([^:]+)(?::\d+)?;(\d+)
    #      #    replacement: $1:$2
    #      #    target_label: __address__
    #      - action: labelmap
    #        regex: __meta_kubernetes_pod_label_(.+)
    #      - source_labels: [__meta_kubernetes_namespace]
    #        action: replace
    #        target_label: kubernetes_namespace
    #      - source_labels: [__meta_kubernetes_pod_name]
    #        action: replace
    #        target_label: kubernetes_pod_name

    # 抓取方式2, 获取metadata.labels来获取
    #    # 1. COREDNS
    #    - job_name: 'kubernetes-coredns-pod'
    #      scheme: http
    #      kubernetes_sd_configs:
    #      - role: pod
    #        namespaces:
    #          names: []
    #      relabel_configs:
    #      # 过滤coredns 标签是k8s-app=kube-dns
    #      - source_labels: [__meta_kubernetes_pod_label_k8s_app]
    #        regex: kube-dns
    #        action: keep
    #      # 过滤coredns 标签是k8s-app=kube-dns 并且:       pod的端口名是metrics的pod
    #      - source_labels: [__meta_kubernetes_pod_container_port_name]
    #        regex: metrics
    #        action: keep
    #      # 主机
    #      - source_labels: [__meta_kubernetes_pod_node_name]
    #        action: replace
    #        target_label: kubernetes_WeiZhiXiu_node_name
    #      # 名称空间
    #      - source_labels: [__meta_kubernetes_namespace]
    #        action: replace
    #        target_label: kubernetes_WeiZhiXiu_namespace
    #      # pod名
    #      - source_labels: [__meta_kubernetes_pod_name]
    #        action: replace
    #        target_label: kubernetes_WeiZhiXiu_pod_name
    - job_name: 'kubernetes-dns-pod2'
      scheme: http
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names: []
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: kube-system;kube-dns;metrics
    # 2.1 kubelet
    - job_name: 'kubernetes-kubelet-daemon-on-every-node-by-role-node'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node       # 此角色默认监控每个节点的10250端口
        namespaces:
          names: []
      relabel_configs:
      #- action: labelmap
      #  regex: __meta_kubernetes_node_label_(.+)
      - source_labels: [__metrics_path__]
        separator: ;
        regex: (.*)
        target_label: metrics_path
        replacement: $1
        action: replace
    # 2.2 kubelet cadvisor
    - job_name: 'kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node'
      metrics_path: /metrics/cadvisor
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node       # 此角色默认监控每个节点的10250端口
        namespaces:
          names: []
      #relabel_configs:
      #- action: labelmap
      #  regex: __meta_kubernetes_node_label_(.+)
      relabel_configs:
      #- action: labelmap
      #  regex: __meta_kubernetes_node_label_(.+)
      - source_labels: [__metrics_path__]
        separator: ;
        regex: (.*)
        target_label: metrics_path
        replacement: $1
        action: replace
    # 2.3. kubelet cadvisor
    - job_name: 'kubernetes-kubelet-probes-daemon-on-every-node-by-role-node'
      metrics_path: /metrics/probes
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node       # 此角色默认监控每个节点的10250端口
        namespaces:
          names: []
      #relabel_configs:
      #- action: labelmap
      #  regex: __meta_kubernetes_node_label_(.+)
      relabel_configs:
      #- action: labelmap
      #  regex: __meta_kubernetes_node_label_(.+)
      - source_labels: [__metrics_path__]
        separator: ;
        regex: (.*)
        target_label: metrics_path
        replacement: $1
        action: replace
    # 3. kube-controller-manager
    - job_name: 'kube-controller-manager'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: []
      relabel_configs:
      # 过滤coredns 标签是k8s-app=kube-dns
      - source_labels: [__meta_kubernetes_pod_label_component]
        regex: kube-controller-manager
        action: keep
    # 4. kube-scheduler
    - job_name: 'kube-scheduler'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: []
      relabel_configs:
      # 过滤coredns 标签是k8s-app=kube-dns
      - source_labels: [__meta_kubernetes_pod_label_component]
        regex: kube-scheduler
        action: keep
     #
     #

    # 5. node-exporter
    - job_name: 'kubernetes-node-exporter-daemonset-pod'
      scheme: http
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: []
      relabel_configs:
      # 过滤coredns 标签是k8s-app=kube-dns
      - source_labels: [__meta_kubernetes_pod_label_k8s_app]
        regex: node-exporter
        action: keep
      # 过滤coredns 标签是k8s-app=node-exporter 并且:       pod的端口名是metrics的pod
      - source_labels: [__meta_kubernetes_pod_container_port_name]
        regex: metrics
        action: keep
      # 主机
      - source_labels: [__meta_kubernetes_pod_node_name]
        action: replace
        target_label: kubernetes_WeiZhiXiu_node_name
      # 名称空间
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_WeiZhiXiu_namespace
      # pod名
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_WeiZhiXiu_pod_name
    # 6. kube-state-metric
    - job_name: 'kube-state-metric-for-kube_node_info-metric'
      scheme: http
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: []
      relabel_configs:
      # 过滤coredns 标签是k8s-app=kube-dns
      #- source_labels: [__meta_kubernetes_pod_label_k8s_app]
      #  regex: kube-state-metrics
      #  action: keep
      # 过滤coredns 标签是k8s-app=node-exporter 并且:       pod的端口名是metrics的pod
      - source_labels: [__meta_kubernetes_pod_container_port_name]
        regex: http-metrics
        action: keep
      # 主机
      - source_labels: [__meta_kubernetes_pod_node_name]
        action: replace
        target_label: kubernetes_WeiZhiXiu_node_name
      # 名称空间
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_WeiZhiXiu_namespace
      # pod名
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_WeiZhiXiu_pod_name
    # 7. etcd
    - job_name: 'etcd-cluster'
      scheme: http
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: []
      relabel_configs:
      # 过滤coredns 标签是k8s-app=kube-dns
      - source_labels: [__meta_kubernetes_pod_label_component]
        regex: etcd
        action: keep
      # 过滤coredns 标签是k8s-app=etcd 并且:       pod的端口名是metrics的pod
      - source_labels: [__meta_kubernetes_pod_container_port_name]
        regex: metrics
        action: keep
    # 8. kube-proxy
    - job_name: 'kube-proxy'
      scheme: http
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: []
      relabel_configs:
      # 过滤coredns 标签是k8s-app=kube-dns
      - source_labels: [__meta_kubernetes_pod_label_k8s_app]
        regex: kube-proxy
        action: keep

    # Alerting specifies settings related to the Alertmanager.
    alerting:
      #alert_relabel_configs:
      # [ - <relabel_config> ... ]
      alertmanagers:
      - kubernetes_sd_configs:
        - role: pod
          namespaces:
            names: []
        relabel_configs:
        # 过滤coredns 标签是k8s-app=alertmanager
        - source_labels: [__meta_kubernetes_pod_label_k8s_app]
          regex: alertmanager
          action: keep
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
```

> ![image-20210315145840549](http://myapp.img.mykernel.cn/image-20210315145840549.png)
>
> prometheus 自动发现编写探寻
>
> 

#### 准备规则配置文件

https://github.com/prometheus-operator/kube-prometheus/blob/main/manifests/prometheus-prometheusRule.yaml

https://github.com/kubernetes/kubernetes/blob/release-1.12/cluster/addons/prometheus/prometheus-configmap.yaml

https://www.cnblogs.com/zhaojiedi1992/p/zhaojiedi_liunx_65_prometheus_alertmanager_rule.html

##### k8s, prometheus, mysql, redis, es相关规则

规则来源：https://awesome-prometheus-alerts.grep.to/rules

`root@master01:/data/weizhixiu/yaml/prometheus# cat prometheus-configmap.yaml `

```diff
apiVersion: v1
data:
  monitoring-prometheus-k8s-rules.yml: |
    groups:
    - name: kubernetes-node-exporter-daemonset-pod.rules
      rules:
      - expr: |
          count without (cpu) (
            count without (mode) (
              node_cpu_seconds_total{job="kubernetes-node-exporter-daemonset-pod"}
            )
          )
        record: instance:node_num_cpu:sum
      - expr: |
          1 - avg without (cpu, mode) (
            rate(node_cpu_seconds_total{job="kubernetes-node-exporter-daemonset-pod", mode="idle"}[1m])
          )
        record: instance:node_cpu_utilisation:rate1m
      - expr: |
          (
            node_load1{job="kubernetes-node-exporter-daemonset-pod"}
          /
            instance:node_num_cpu:sum{job="kubernetes-node-exporter-daemonset-pod"}
          )
        record: instance:node_load1_per_cpu:ratio
      - expr: |
          1 - (
            node_memory_MemAvailable_bytes{job="kubernetes-node-exporter-daemonset-pod"}
          /
            node_memory_MemTotal_bytes{job="kubernetes-node-exporter-daemonset-pod"}
          )
        record: instance:node_memory_utilisation:ratio
      - expr: |
          rate(node_vmstat_pgmajfault{job="kubernetes-node-exporter-daemonset-pod"}[1m])
        record: instance:node_vmstat_pgmajfault:rate1m
      - expr: |
          rate(node_disk_io_time_seconds_total{job="kubernetes-node-exporter-daemonset-pod", device=~"nvme.+|rbd.+|sd.+|vd.+|xvd.+|dm-.+"}[1m])
        record: instance_device:node_disk_io_time_seconds:rate1m
      - expr: |
          rate(node_disk_io_time_weighted_seconds_total{job="kubernetes-node-exporter-daemonset-pod", device=~"nvme.+|rbd.+|sd.+|vd.+|xvd.+|dm-.+"}[1m])
        record: instance_device:node_disk_io_time_weighted_seconds:rate1m
      - expr: |
          sum without (device) (
            rate(node_network_receive_bytes_total{job="kubernetes-node-exporter-daemonset-pod", device!="lo"}[1m])
          )
        record: instance:node_network_receive_bytes_excluding_lo:rate1m
      - expr: |
          sum without (device) (
            rate(node_network_transmit_bytes_total{job="kubernetes-node-exporter-daemonset-pod", device!="lo"}[1m])
          )
        record: instance:node_network_transmit_bytes_excluding_lo:rate1m
      - expr: |
          sum without (device) (
            rate(node_network_receive_drop_total{job="kubernetes-node-exporter-daemonset-pod", device!="lo"}[1m])
          )
        record: instance:node_network_receive_drop_excluding_lo:rate1m
      - expr: |
          sum without (device) (
            rate(node_network_transmit_drop_total{job="kubernetes-node-exporter-daemonset-pod", device!="lo"}[1m])
          )
        record: instance:node_network_transmit_drop_excluding_lo:rate1m
    - name: kube-apiserver.rules
      rules:
      - expr: |
          histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket{job="kubernetes-apiservers"}[5m])) without(instance, pod))
        labels:
          quantile: "0.99"
        record: cluster_quantile:apiserver_request_duration_seconds:histogram_quantile
      - expr: |
          histogram_quantile(0.9, sum(rate(apiserver_request_duration_seconds_bucket{job="kubernetes-apiservers"}[5m])) without(instance, pod))
        labels:
          quantile: "0.9"
        record: cluster_quantile:apiserver_request_duration_seconds:histogram_quantile
      - expr: |
          histogram_quantile(0.5, sum(rate(apiserver_request_duration_seconds_bucket{job="kubernetes-apiservers"}[5m])) without(instance, pod))
        labels:
          quantile: "0.5"
        record: cluster_quantile:apiserver_request_duration_seconds:histogram_quantile
    - name: k8s.rules
      rules:
      - expr: |
          sum(rate(container_cpu_usage_seconds_total{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node", image!="", container!="POD"}[5m])) by (namespace)
        record: namespace:container_cpu_usage_seconds_total:sum_rate
      - expr: |
          sum by (namespace, pod, container) (
            rate(container_cpu_usage_seconds_total{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node", image!="", container!="POD"}[5m])
          ) * on (namespace, pod) group_left(node) max by(namespace, pod, node) (kube_pod_info)
        record: node_namespace_pod_container:container_cpu_usage_seconds_total:sum_rate
      - expr: |
          container_memory_working_set_bytes{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node", image!=""}
          * on (namespace, pod) group_left(node) max by(namespace, pod, node) (kube_pod_info)
        record: node_namespace_pod_container:container_memory_working_set_bytes
      - expr: |
          container_memory_rss{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node", image!=""}
          * on (namespace, pod) group_left(node) max by(namespace, pod, node) (kube_pod_info)
        record: node_namespace_pod_container:container_memory_rss
      - expr: |
          container_memory_cache{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node", image!=""}
          * on (namespace, pod) group_left(node) max by(namespace, pod, node) (kube_pod_info)
        record: node_namespace_pod_container:container_memory_cache
      - expr: |
          container_memory_swap{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node", image!=""}
          * on (namespace, pod) group_left(node) max by(namespace, pod, node) (kube_pod_info)
        record: node_namespace_pod_container:container_memory_swap
      - expr: |
          sum(container_memory_usage_bytes{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node", image!="", container!="POD"}) by (namespace)
        record: namespace:container_memory_usage_bytes:sum
      - expr: |
          sum by (namespace, label_name) (
              sum(kube_pod_container_resource_requests_memory_bytes{job="kube-state-metric-for-kube_node_info-metric"} * on (endpoint, instance, job, namespace, pod, service) group_left(phase) (kube_pod_status_phase{phase=~"Pending|Running"} == 1)) by (namespace, pod)
            * on (namespace, pod)
              group_left(label_name) kube_pod_labels{job="kube-state-metric-for-kube_node_info-metric"}
          )
        record: namespace:kube_pod_container_resource_requests_memory_bytes:sum
      - expr: |
          sum by (namespace, label_name) (
              sum(kube_pod_container_resource_requests_cpu_cores{job="kube-state-metric-for-kube_node_info-metric"} * on (endpoint, instance, job, namespace, pod, service) group_left(phase) (kube_pod_status_phase{phase=~"Pending|Running"} == 1)) by (namespace, pod)
            * on (namespace, pod)
              group_left(label_name) kube_pod_labels{job="kube-state-metric-for-kube_node_info-metric"}
          )
        record: namespace:kube_pod_container_resource_requests_cpu_cores:sum
      - expr: |
          sum(
            label_replace(
              label_replace(
                kube_pod_owner{job="kube-state-metric-for-kube_node_info-metric", owner_kind="ReplicaSet"},
                "replicaset", "$1", "owner_name", "(.*)"
              ) * on(replicaset, namespace) group_left(owner_name) kube_replicaset_owner{job="kube-state-metric-for-kube_node_info-metric"},
              "workload", "$1", "owner_name", "(.*)"
            )
          ) by (namespace, workload, pod)
        labels:
          workload_type: deployment
        record: mixin_pod_workload
      - expr: |
          sum(
            label_replace(
              kube_pod_owner{job="kube-state-metric-for-kube_node_info-metric", owner_kind="DaemonSet"},
              "workload", "$1", "owner_name", "(.*)"
            )
          ) by (namespace, workload, pod)
        labels:
          workload_type: daemonset
        record: mixin_pod_workload
      - expr: |
          sum(
            label_replace(
              kube_pod_owner{job="kube-state-metric-for-kube_node_info-metric", owner_kind="StatefulSet"},
              "workload", "$1", "owner_name", "(.*)"
            )
          ) by (namespace, workload, pod)
        labels:
          workload_type: statefulset
        record: mixin_pod_workload
    - name: kube-scheduler.rules
      rules:
      - expr: |
          histogram_quantile(0.99, sum(rate(scheduler_e2e_scheduling_duration_seconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod))
        labels:
          quantile: "0.99"
        record: cluster_quantile:scheduler_e2e_scheduling_duration_seconds:histogram_quantile
      - expr: |
          histogram_quantile(0.99, sum(rate(scheduler_scheduling_algorithm_duration_seconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod))
        labels:
          quantile: "0.99"
        record: cluster_quantile:scheduler_scheduling_algorithm_duration_seconds:histogram_quantile
      - expr: |
          histogram_quantile(0.99, sum(rate(scheduler_binding_duration_seconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod))
        labels:
          quantile: "0.99"
        record: cluster_quantile:scheduler_binding_duration_seconds:histogram_quantile
      - expr: |
          histogram_quantile(0.9, sum(rate(scheduler_e2e_scheduling_duration_seconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod))
        labels:
          quantile: "0.9"
        record: cluster_quantile:scheduler_e2e_scheduling_duration_seconds:histogram_quantile
      - expr: |
          histogram_quantile(0.9, sum(rate(scheduler_scheduling_algorithm_duration_seconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod))
        labels:
          quantile: "0.9"
        record: cluster_quantile:scheduler_scheduling_algorithm_duration_seconds:histogram_quantile
      - expr: |
          histogram_quantile(0.9, sum(rate(scheduler_binding_duration_seconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod))
        labels:
          quantile: "0.9"
        record: cluster_quantile:scheduler_binding_duration_seconds:histogram_quantile
      - expr: |
          histogram_quantile(0.5, sum(rate(scheduler_e2e_scheduling_duration_seconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod))
        labels:
          quantile: "0.5"
        record: cluster_quantile:scheduler_e2e_scheduling_duration_seconds:histogram_quantile
      - expr: |
          histogram_quantile(0.5, sum(rate(scheduler_scheduling_algorithm_duration_seconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod))
        labels:
          quantile: "0.5"
        record: cluster_quantile:scheduler_scheduling_algorithm_duration_seconds:histogram_quantile
      - expr: |
          histogram_quantile(0.5, sum(rate(scheduler_binding_duration_seconds_bucket{job="kube-scheduler"}[5m])) without(instance, pod))
        labels:
          quantile: "0.5"
        record: cluster_quantile:scheduler_binding_duration_seconds:histogram_quantile
    - name: node.rules
      rules:
      - expr: sum(min(kube_pod_info) by (node))
        record: ':kube_pod_info_node_count:'
      - expr: |
          max(label_replace(kube_pod_info{job="kube-state-metric-for-kube_node_info-metric"}, "pod", "$1", "pod", "(.*)")) by (node, namespace, pod)
        record: 'node_namespace_pod:kube_pod_info:'
      - expr: |
          count by (node) (sum by (node, cpu) (
            node_cpu_seconds_total{job="kubernetes-node-exporter-daemonset-pod"}
          * on (namespace, pod) group_left(node)
            node_namespace_pod:kube_pod_info:
          ))
        record: node:node_num_cpu:sum
      - expr: |
          sum(
            node_memory_MemAvailable_bytes{job="kubernetes-node-exporter-daemonset-pod"} or
            (
              node_memory_Buffers_bytes{job="kubernetes-node-exporter-daemonset-pod"} +
              node_memory_Cached_bytes{job="kubernetes-node-exporter-daemonset-pod"} +
              node_memory_MemFree_bytes{job="kubernetes-node-exporter-daemonset-pod"} +
              node_memory_Slab_bytes{job="kubernetes-node-exporter-daemonset-pod"}
            )
          )
        record: :node_memory_MemAvailable_bytes:sum
    - name: kube-prometheus-node-recording.rules
      rules:
      - expr: sum(rate(node_cpu_seconds_total{mode!="idle",mode!="iowait"}[3m])) BY (instance)
        record: instance:node_cpu:rate:sum
      - expr: sum((node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_free_bytes{mountpoint="/"}))
          BY (instance)
        record: instance:node_filesystem_usage:sum
      - expr: sum(rate(node_network_receive_bytes_total[3m])) BY (instance)
        record: instance:node_network_receive_bytes:rate:sum
      - expr: sum(rate(node_network_transmit_bytes_total[3m])) BY (instance)
        record: instance:node_network_transmit_bytes:rate:sum
      - expr: sum(rate(node_cpu_seconds_total{mode!="idle",mode!="iowait"}[5m])) WITHOUT
          (cpu, mode) / ON(instance) GROUP_LEFT() count(sum(node_cpu_seconds_total) BY
          (instance, cpu)) BY (instance)
        record: instance:node_cpu:ratio
      - expr: sum(rate(node_cpu_seconds_total{mode!="idle",mode!="iowait"}[5m]))
        record: cluster:node_cpu:sum_rate5m
      - expr: cluster:node_cpu_seconds_total:rate5m / count(sum(node_cpu_seconds_total)
          BY (instance, cpu))
        record: cluster:node_cpu:ratio
    - name: kubernetes-node-exporter-daemonset-pod
      rules:
      - alert: NodeFilesystemSpaceFillingUp
        annotations:
          description: Filesystem on {{ $labels.device }} at {{ $labels.instance }} has
            only {{ printf "%.2f" $value }}% available space left and is filling up.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-nodefilesystemspacefillingup
          summary: Filesystem is predicted to run out of space within the next 24 hours.
        expr: |
          (
            node_filesystem_avail_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} / node_filesystem_size_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} * 100 < 40
          and
            predict_linear(node_filesystem_avail_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype!=""}[6h], 24*60*60) < 0
          and
            node_filesystem_readonly{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} == 0
          )
        for: 1h
        labels:
          severity: warning
      - alert: NodeFilesystemSpaceFillingUp
        annotations:
          description: Filesystem on {{ $labels.device }} at {{ $labels.instance }} has
            only {{ printf "%.2f" $value }}% available space left and is filling up fast.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-nodefilesystemspacefillingup
          summary: Filesystem is predicted to run out of space within the next 4 hours.
        expr: |
          (
            node_filesystem_avail_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} / node_filesystem_size_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} * 100 < 20
          and
            predict_linear(node_filesystem_avail_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype!=""}[6h], 4*60*60) < 0
          and
            node_filesystem_readonly{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} == 0
          )
        for: 1h
        labels:
          severity: critical
      - alert: NodeFilesystemAlmostOutOfSpace
        annotations:
          description: Filesystem on {{ $labels.device }} at {{ $labels.instance }} has
            only {{ printf "%.2f" $value }}% available space left.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-nodefilesystemalmostoutofspace
          summary: Filesystem has less than 5% space left.
        expr: |
          (
            node_filesystem_avail_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} / node_filesystem_size_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} * 100 < 5
          and
            node_filesystem_readonly{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} == 0
          )
        for: 1h
        labels:
          severity: warning
      - alert: NodeFilesystemAlmostOutOfSpace
        annotations:
          description: Filesystem on {{ $labels.device }} at {{ $labels.instance }} has
            only {{ printf "%.2f" $value }}% available space left.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-nodefilesystemalmostoutofspace
          summary: Filesystem has less than 3% space left.
        expr: |
          (
            node_filesystem_avail_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} / node_filesystem_size_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} * 100 < 3
          and
            node_filesystem_readonly{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} == 0
          )
        for: 1h
        labels:
          severity: critical
      - alert: NodeFilesystemFilesFillingUp
        annotations:
          description: Filesystem on {{ $labels.device }} at {{ $labels.instance }} has
            only {{ printf "%.2f" $value }}% available inodes left and is filling up.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-nodefilesystemfilesfillingup
          summary: Filesystem is predicted to run out of inodes within the next 24 hours.
        expr: |
          (
            node_filesystem_files_free{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} / node_filesystem_files{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} * 100 < 40
          and
            predict_linear(node_filesystem_files_free{job="kubernetes-node-exporter-daemonset-pod",fstype!=""}[6h], 24*60*60) < 0
          and
            node_filesystem_readonly{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} == 0
          )
        for: 1h
        labels:
          severity: warning
      - alert: NodeFilesystemFilesFillingUp
        annotations:
          description: Filesystem on {{ $labels.device }} at {{ $labels.instance }} has
            only {{ printf "%.2f" $value }}% available inodes left and is filling up fast.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-nodefilesystemfilesfillingup
          summary: Filesystem is predicted to run out of inodes within the next 4 hours.
        expr: |
          (
            node_filesystem_files_free{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} / node_filesystem_files{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} * 100 < 20
          and
            predict_linear(node_filesystem_files_free{job="kubernetes-node-exporter-daemonset-pod",fstype!=""}[6h], 4*60*60) < 0
          and
            node_filesystem_readonly{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} == 0
          )
        for: 1h
        labels:
          severity: critical
      - alert: NodeFilesystemAlmostOutOfFiles
        annotations:
          description: Filesystem on {{ $labels.device }} at {{ $labels.instance }} has
            only {{ printf "%.2f" $value }}% available inodes left.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-nodefilesystemalmostoutoffiles
          summary: Filesystem has less than 5% inodes left.
        expr: |
          (
            node_filesystem_files_free{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} / node_filesystem_files{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} * 100 < 5
          and
            node_filesystem_readonly{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} == 0
          )
        for: 1h
        labels:
          severity: warning
      - alert: NodeFilesystemAlmostOutOfFiles
        annotations:
          description: Filesystem on {{ $labels.device }} at {{ $labels.instance }} has
            only {{ printf "%.2f" $value }}% available inodes left.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-nodefilesystemalmostoutoffiles
          summary: Filesystem has less than 3% inodes left.
        expr: |
          (
            node_filesystem_files_free{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} / node_filesystem_files{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} * 100 < 3
          and
            node_filesystem_readonly{job="kubernetes-node-exporter-daemonset-pod",fstype!=""} == 0
          )
        for: 1h
        labels:
          severity: critical
      - alert: NodeNetworkReceiveErrs
        annotations:
          description: '{{ $labels.instance }} interface {{ $labels.device }} has encountered
            {{ printf "%.0f" $value }} receive errors in the last two minutes.'
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-nodenetworkreceiveerrs
          summary: Network interface is reporting many receive errors.
        expr: |
          increase(node_network_receive_errs_total[2m]) > 10
        for: 1h
        labels:
          severity: warning
      - alert: NodeNetworkTransmitErrs
        annotations:
          description: '{{ $labels.instance }} interface {{ $labels.device }} has encountered
            {{ printf "%.0f" $value }} transmit errors in the last two minutes.'
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-nodenetworktransmiterrs
          summary: Network interface is reporting many transmit errors.
        expr: |
          increase(node_network_transmit_errs_total[2m]) > 10
        for: 1h
        labels:
          severity: warning
    - name: kubernetes-apps
      rules:
      - alert: KubePodCrashLooping
        annotations:
          message: Pod {{ $labels.namespace }}/{{ $labels.pod }} ({{ $labels.container
            }}) is restarting {{ printf "%.2f" $value }} times / 5 minutes.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubepodcrashlooping
        expr: |
          rate(kube_pod_container_status_restarts_total{job="kube-state-metric-for-kube_node_info-metric"}[15m]) * 60 * 5 > 0
        for: 15m
        labels:
          severity: critical
      - alert: KubePodNotReady
        annotations:
          message: Pod {{ $labels.namespace }}/{{ $labels.pod }} has been in a non-ready
            state for longer than 15 minutes.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubepodnotready
        expr: |
          sum by (namespace, pod) (kube_pod_status_phase{job="kube-state-metric-for-kube_node_info-metric", phase=~"Failed|Pending|Unknown"} * on(namespace, pod) group_left(owner_kind) kube_pod_owner{owner_kind!="Job"}) > 0
        for: 15m
        labels:
          severity: critical
      - alert: KubeDeploymentGenerationMismatch
        annotations:
          message: Deployment generation for {{ $labels.namespace }}/{{ $labels.deployment
            }} does not match, this indicates that the Deployment has failed but has not
            been rolled back.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubedeploymentgenerationmismatch
        expr: |
          kube_deployment_status_observed_generation{job="kube-state-metric-for-kube_node_info-metric"}
            !=
          kube_deployment_metadata_generation{job="kube-state-metric-for-kube_node_info-metric"}
        for: 15m
        labels:
          severity: critical
      - alert: KubeDeploymentReplicasMismatch
        annotations:
          message: Deployment {{ $labels.namespace }}/{{ $labels.deployment }} has not
            matched the expected number of replicas for longer than 15 minutes.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubedeploymentreplicasmismatch
        expr: |
          kube_deployment_spec_replicas{job="kube-state-metric-for-kube_node_info-metric"}
            !=
          kube_deployment_status_replicas_available{job="kube-state-metric-for-kube_node_info-metric"}
        for: 15m
        labels:
          severity: critical
      - alert: KubeStatefulSetReplicasMismatch
        annotations:
          message: StatefulSet {{ $labels.namespace }}/{{ $labels.statefulset }} has not
            matched the expected number of replicas for longer than 15 minutes.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubestatefulsetreplicasmismatch
        expr: |
          kube_statefulset_status_replicas_ready{job="kube-state-metric-for-kube_node_info-metric"}
            !=
          kube_statefulset_status_replicas{job="kube-state-metric-for-kube_node_info-metric"}
        for: 15m
        labels:
          severity: critical
      - alert: KubeStatefulSetGenerationMismatch
        annotations:
          message: StatefulSet generation for {{ $labels.namespace }}/{{ $labels.statefulset
            }} does not match, this indicates that the StatefulSet has failed but has
            not been rolled back.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubestatefulsetgenerationmismatch
        expr: |
          kube_statefulset_status_observed_generation{job="kube-state-metric-for-kube_node_info-metric"}
            !=
          kube_statefulset_metadata_generation{job="kube-state-metric-for-kube_node_info-metric"}
        for: 15m
        labels:
          severity: critical
      - alert: KubeStatefulSetUpdateNotRolledOut
        annotations:
          message: StatefulSet {{ $labels.namespace }}/{{ $labels.statefulset }} update
            has not been rolled out.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubestatefulsetupdatenotrolledout
        expr: |
          max without (revision) (
            kube_statefulset_status_current_revision{job="kube-state-metric-for-kube_node_info-metric"}
              unless
            kube_statefulset_status_update_revision{job="kube-state-metric-for-kube_node_info-metric"}
          )
            *
          (
            kube_statefulset_replicas{job="kube-state-metric-for-kube_node_info-metric"}
              !=
            kube_statefulset_status_replicas_updated{job="kube-state-metric-for-kube_node_info-metric"}
          )
        for: 15m
        labels:
          severity: critical
      - alert: KubeDaemonSetRolloutStuck
        annotations:
          message: Only {{ $value | humanizePercentage }} of the desired Pods of DaemonSet
            {{ $labels.namespace }}/{{ $labels.daemonset }} are scheduled and ready.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubedaemonsetrolloutstuck
        expr: |
          kube_daemonset_status_number_ready{job="kube-state-metric-for-kube_node_info-metric"}
            /
          kube_daemonset_status_desired_number_scheduled{job="kube-state-metric-for-kube_node_info-metric"} < 1.00
        for: 15m
        labels:
          severity: critical
      - alert: KubeContainerWaiting
        annotations:
          message: Pod {{ $labels.namespace }}/{{ $labels.pod }} container {{ $labels.container}}
            has been in waiting state for longer than 1 hour.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubecontainerwaiting
        expr: |
          sum by (namespace, pod, container) (kube_pod_container_status_waiting_reason{job="kube-state-metric-for-kube_node_info-metric"}) > 0
        for: 1h
        labels:
          severity: warning
      - alert: KubeDaemonSetNotScheduled
        annotations:
          message: '{{ $value }} Pods of DaemonSet {{ $labels.namespace }}/{{ $labels.daemonset
            }} are not scheduled.'
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubedaemonsetnotscheduled
        expr: |
          kube_daemonset_status_desired_number_scheduled{job="kube-state-metric-for-kube_node_info-metric"}
            -
          kube_daemonset_status_current_number_scheduled{job="kube-state-metric-for-kube_node_info-metric"} > 0
        for: 10m
        labels:
          severity: warning
      - alert: KubeDaemonSetMisScheduled
        annotations:
          message: '{{ $value }} Pods of DaemonSet {{ $labels.namespace }}/{{ $labels.daemonset
            }} are running where they are not supposed to run.'
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubedaemonsetmisscheduled
        expr: |
          kube_daemonset_status_number_misscheduled{job="kube-state-metric-for-kube_node_info-metric"} > 0
        for: 10m
        labels:
          severity: warning
      - alert: KubeCronJobRunning
        annotations:
          message: CronJob {{ $labels.namespace }}/{{ $labels.cronjob }} is taking more
            than 1h to complete.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubecronjobrunning
        expr: |
          time() - kube_cronjob_next_schedule_time{job="kube-state-metric-for-kube_node_info-metric"} > 3600
        for: 1h
        labels:
          severity: warning
      - alert: KubeJobCompletion
        annotations:
          message: Job {{ $labels.namespace }}/{{ $labels.job_name }} is taking more than
            one hour to complete.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubejobcompletion
        expr: |
          kube_job_spec_completions{job="kube-state-metric-for-kube_node_info-metric"} - kube_job_status_succeeded{job="kube-state-metric-for-kube_node_info-metric"}  > 0
        for: 1h
        labels:
          severity: warning
      - alert: KubeJobFailed
        annotations:
          message: Job {{ $labels.namespace }}/{{ $labels.job_name }} failed to complete.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubejobfailed
        expr: |
          kube_job_failed{job="kube-state-metric-for-kube_node_info-metric"}  > 0
        for: 15m
        labels:
          severity: warning
      - alert: KubeHpaReplicasMismatch
        annotations:
          message: HPA {{ $labels.namespace }}/{{ $labels.hpa }} has not matched the desired
            number of replicas for longer than 15 minutes.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubehpareplicasmismatch
        expr: |
          (kube_hpa_status_desired_replicas{job="kube-state-metric-for-kube_node_info-metric"}
            !=
          kube_hpa_status_current_replicas{job="kube-state-metric-for-kube_node_info-metric"})
            and
          changes(kube_hpa_status_current_replicas[15m]) == 0
        for: 15m
        labels:
          severity: warning
      - alert: KubeHpaMaxedOut
        annotations:
          message: HPA {{ $labels.namespace }}/{{ $labels.hpa }} has been running at max
            replicas for longer than 15 minutes.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubehpamaxedout
        expr: |
          kube_hpa_status_current_replicas{job="kube-state-metric-for-kube_node_info-metric"}
            ==
          kube_hpa_spec_max_replicas{job="kube-state-metric-for-kube_node_info-metric"}
        for: 15m
        labels:
          severity: warning
    - name: kubernetes-resources
      rules:
      - alert: KubeCPUOvercommit
        annotations:
          message: Cluster has overcommitted CPU resource requests for Pods and cannot
            tolerate node failure.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubecpuovercommit
        expr: |
          sum(namespace:kube_pod_container_resource_requests_cpu_cores:sum)
            /
          sum(kube_node_status_allocatable_cpu_cores)
            >
          (count(kube_node_status_allocatable_cpu_cores)-1) / count(kube_node_status_allocatable_cpu_cores)
        for: 5m
        labels:
          severity: warning
      - alert: KubeMemOvercommit
        annotations:
          message: Cluster has overcommitted memory resource requests for Pods and cannot
            tolerate node failure.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubememovercommit
        expr: |
          sum(namespace:kube_pod_container_resource_requests_memory_bytes:sum)
            /
          sum(kube_node_status_allocatable_memory_bytes)
            >
          (count(kube_node_status_allocatable_memory_bytes)-1)
            /
          count(kube_node_status_allocatable_memory_bytes)
        for: 5m
        labels:
          severity: warning
      - alert: KubeCPUOvercommit
        annotations:
          message: Cluster has overcommitted CPU resource requests for Namespaces.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubecpuovercommit
        expr: |
          sum(kube_resourcequota{job="kube-state-metric-for-kube_node_info-metric", type="hard", resource="cpu"})
            /
          sum(kube_node_status_allocatable_cpu_cores)
            > 1.5
        for: 5m
        labels:
          severity: warning
      - alert: KubeMemOvercommit
        annotations:
          message: Cluster has overcommitted memory resource requests for Namespaces.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubememovercommit
        expr: |
          sum(kube_resourcequota{job="kube-state-metric-for-kube_node_info-metric", type="hard", resource="memory"})
            /
          sum(kube_node_status_allocatable_memory_bytes{job="kubernetes-node-exporter-daemonset-pod"})
            > 1.5
        for: 5m
        labels:
          severity: warning
      - alert: KubeQuotaExceeded
        annotations:
          message: Namespace {{ $labels.namespace }} is using {{ $value | humanizePercentage
            }} of its {{ $labels.resource }} quota.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubequotaexceeded
        expr: |
          kube_resourcequota{job="kube-state-metric-for-kube_node_info-metric", type="used"}
            / ignoring(instance, job, type)
          (kube_resourcequota{job="kube-state-metric-for-kube_node_info-metric", type="hard"} > 0)
            > 0.90
        for: 15m
        labels:
          severity: warning
      - alert: CPUThrottlingHigh
        annotations:
          message: '{{ $value | humanizePercentage }} throttling of CPU in namespace {{
            $labels.namespace }} for container {{ $labels.container }} in pod {{ $labels.pod
            }}.'
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-cputhrottlinghigh
        expr: |
          sum(increase(container_cpu_cfs_throttled_periods_total{container!="", }[5m])) by (container, pod, namespace)
            /
          sum(increase(container_cpu_cfs_periods_total{}[5m])) by (container, pod, namespace)
            > ( 25 / 100 )
        for: 15m
        labels:
          severity: warning
    - name: kubernetes-storage
      rules:
      - alert: KubePersistentVolumeUsageCritical
        annotations:
          message: The PersistentVolume claimed by {{ $labels.persistentvolumeclaim }}
            in Namespace {{ $labels.namespace }} is only {{ $value | humanizePercentage
            }} free.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubepersistentvolumeusagecritical
        expr: |
          kubelet_volume_stats_available_bytes{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node"}
            /
          kubelet_volume_stats_capacity_bytes{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node"}
            < 0.03
        for: 1m
        labels:
          severity: critical
      - alert: KubePersistentVolumeFullInFourDays
        annotations:
          message: Based on recent sampling, the PersistentVolume claimed by {{ $labels.persistentvolumeclaim
            }} in Namespace {{ $labels.namespace }} is expected to fill up within four
            days. Currently {{ $value | humanizePercentage }} is available.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubepersistentvolumefullinfourdays
        expr: |
          (
            kubelet_volume_stats_available_bytes{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node"}
              /
            kubelet_volume_stats_capacity_bytes{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node"}
          ) < 0.15
          and
          predict_linear(kubelet_volume_stats_available_bytes{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node"}[6h], 4 * 24 * 3600) < 0
        for: 1h
        labels:
          severity: critical
      - alert: KubePersistentVolumeErrors
        annotations:
          message: The persistent volume {{ $labels.persistentvolume }} has status {{
            $labels.phase }}.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubepersistentvolumeerrors
        expr: |
          kube_persistentvolume_status_phase{phase=~"Failed|Pending",job="kube-state-metric-for-kube_node_info-metric"} > 0
        for: 5m
        labels:
          severity: critical
    - name: kubernetes-system
      rules:
      - alert: KubeVersionMismatch
        annotations:
          message: There are {{ $value }} different semantic versions of Kubernetes components
            running.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeversionmismatch
        expr: |
          count(count by (gitVersion) (label_replace(kubernetes_build_info{job!~"kube-dns|coredns"},"gitVersion","$1","gitVersion","(v[0-9]*.[0-9]*.[0-9]*).*"))) > 1
        for: 15m
        labels:
          severity: warning
      - alert: KubeClientErrors
        annotations:
          message: Kubernetes API server client '{{ $labels.job }}/{{ $labels.instance
            }}' is experiencing {{ $value | humanizePercentage }} errors.'
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeclienterrors
        expr: |
          (sum(rate(rest_client_requests_total{code=~"5.."}[5m])) by (instance, job)
            /
          sum(rate(rest_client_requests_total[5m])) by (instance, job))
          > 0.01
        for: 15m
        labels:
          severity: warning
    - name: kubernetes-system-apiserver
      rules:
      - alert: KubeAPILatencyHigh
        annotations:
          message: The API server has a 99th percentile latency of {{ $value }} seconds
            for {{ $labels.verb }} {{ $labels.resource }}.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapilatencyhigh
        expr: |
          cluster_quantile:apiserver_request_duration_seconds:histogram_quantile{job="kubernetes-apiservers",quantile="0.99",subresource!="log",verb!~"LIST|WATCH|WATCHLIST|PROXY|CONNECT"} > 1
        for: 10m
        labels:
          severity: warning
      - alert: KubeAPILatencyHigh
        annotations:
          message: The API server has a 99th percentile latency of {{ $value }} seconds
            for {{ $labels.verb }} {{ $labels.resource }}.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapilatencyhigh
        expr: |
          cluster_quantile:apiserver_request_duration_seconds:histogram_quantile{job="kubernetes-apiservers",quantile="0.99",subresource!="log",verb!~"LIST|WATCH|WATCHLIST|PROXY|CONNECT"} > 4
        for: 10m
        labels:
          severity: critical
      - alert: KubeAPIErrorsHigh
        annotations:
          message: API server is returning errors for {{ $value | humanizePercentage }}
            of requests.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapierrorshigh
        expr: |
          sum(rate(apiserver_request_total{job="kubernetes-apiservers",code=~"5.."}[5m]))
            /
          sum(rate(apiserver_request_total{job="kubernetes-apiservers"}[5m])) > 0.03
        for: 10m
        labels:
          severity: critical
      - alert: KubeAPIErrorsHigh
        annotations:
          message: API server is returning errors for {{ $value | humanizePercentage }}
            of requests.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapierrorshigh
        expr: |
          sum(rate(apiserver_request_total{job="kubernetes-apiservers",code=~"5.."}[5m]))
            /
          sum(rate(apiserver_request_total{job="kubernetes-apiservers"}[5m])) > 0.01
        for: 10m
        labels:
          severity: warning
      - alert: KubeAPIErrorsHigh
        annotations:
          message: API server is returning errors for {{ $value | humanizePercentage }}
            of requests for {{ $labels.verb }} {{ $labels.resource }} {{ $labels.subresource
            }}.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapierrorshigh
        expr: |
          sum(rate(apiserver_request_total{job="kubernetes-apiservers",code=~"5.."}[5m])) by (resource,subresource,verb)
            /
          sum(rate(apiserver_request_total{job="kubernetes-apiservers"}[5m])) by (resource,subresource,verb) > 0.10
        for: 10m
        labels:
          severity: critical
      - alert: KubeAPIErrorsHigh
        annotations:
          message: API server is returning errors for {{ $value | humanizePercentage }}
            of requests for {{ $labels.verb }} {{ $labels.resource }} {{ $labels.subresource
            }}.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapierrorshigh
        expr: |
          sum(rate(apiserver_request_total{job="kubernetes-apiservers",code=~"5.."}[5m])) by (resource,subresource,verb)
            /
          sum(rate(apiserver_request_total{job="kubernetes-apiservers"}[5m])) by (resource,subresource,verb) > 0.05
        for: 10m
        labels:
          severity: warning
      - alert: KubeClientCertificateExpiration
        annotations:
          message: A client certificate used to authenticate to the apiserver is expiring
            in less than 7.0 days.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeclientcertificateexpiration
        expr: |
          apiserver_client_certificate_expiration_seconds_count{job="kubernetes-apiservers"} > 0 and histogram_quantile(0.01, sum by (job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="kubernetes-apiservers"}[5m]))) < 604800
        labels:
          severity: warning
      - alert: KubeClientCertificateExpiration
        annotations:
          message: A client certificate used to authenticate to the apiserver is expiring
            in less than 24.0 hours.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeclientcertificateexpiration
        expr: |
          apiserver_client_certificate_expiration_seconds_count{job="kubernetes-apiservers"} > 0 and histogram_quantile(0.01, sum by (job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="kubernetes-apiservers"}[5m]))) < 86400
        labels:
          severity: critical
      - alert: KubeAPIDown
        annotations:
          message: KubeAPI has disappeared from Prometheus target discovery.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapidown
        expr: |
          absent(up{job="kubernetes-apiservers"} == 1)
        for: 15m
        labels:
          severity: critical
    - name: kubernetes-system-kubelet
      rules:
      - alert: KubeNodeNotReady
        annotations:
          message: '{{ $labels.node }} has been unready for more than 15 minutes.'
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubenodenotready
        expr: |
          kube_node_status_condition{job="kube-state-metric-for-kube_node_info-metric",condition="Ready",status="true"} == 0
        for: 15m
        labels:
          severity: warning
      - alert: KubeNodeUnreachable
        annotations:
          message: '{{ $labels.node }} is unreachable and some workloads may be rescheduled.'
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubenodeunreachable
        expr: |
          kube_node_spec_taint{job="kube-state-metric-for-kube_node_info-metric",key="node.kubernetes.io/unreachable",effect="NoSchedule"} == 1
        labels:
          severity: warning
      - alert: KubeletTooManyPods
        annotations:
          message: Kubelet '{{ $labels.node }}' is running at {{ $value | humanizePercentage
            }} of its Pod capacity.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubelettoomanypods
        expr: |
          max(max(kubelet_running_pod_count{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node"}) by(instance) * on(instance) group_left(node) kubelet_node_name{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node"}) by(node) / max(kube_node_status_capacity_pods{job="kube-state-metric-for-kube_node_info-metric"}) by(node) > 0.95
        for: 15m
        labels:
          severity: warning
      - alert: KubeletDown
        annotations:
          message: Kubelet has disappeared from Prometheus target discovery.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeletdown
        expr: |
          absent(up{job="kubernetes-kubelet-cadvisor-daemon-on-every-node-by-role-node"} == 1)
        for: 15m
        labels:
          severity: critical
    - name: kubernetes-system-scheduler
      rules:
      - alert: KubeSchedulerDown
        annotations:
          message: KubeScheduler has disappeared from Prometheus target discovery.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeschedulerdown
        expr: |
          absent(up{job="kube-scheduler"} == 1)
        for: 15m
        labels:
          severity: critical
    - name: kubernetes-system-controller-manager
      rules:
      - alert: KubeControllerManagerDown
        annotations:
          message: KubeControllerManager has disappeared from Prometheus target discovery.
          runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubecontrollermanagerdown
        expr: |
          absent(up{job="kube-controller-manager"} == 1)
        for: 15m
        labels:
          severity: critical
    - name: general.rules
      rules:
      - alert: targetDown
        annotations:
          description: A Prometheus target has disappeared. An exporter might be crashed.
        expr: 'up == 0'
        labels:
          severity: critical 
      - alert: Watchdog
        annotations:
          message: |
            这是一个警报，旨在确保整个警报管道均正常运行。 此警报始终处于触发状态，因此应始终在Alertmanager中触发并且始终向接收方触发。 与各种通知机制的集成，这些机制在未触发此警报时发送通知。 例如，PagerDuty中的“ DeadMansSnitch”集成。
        expr: vector(1)
        labels:
          severity: none
  mysql-exporter-configmap.yml: |
    groups:
    #https://awesome-prometheus-alerts.grep.to/rules#mysql
    - name: mysqlAlert 
      rules:
      - alert: MysqlDown
        annotations:
          description: "MySQL instance is down on {{ $labels.instance }}\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"
          summary: MySQL down (instance {{ $labels.instance }})
        expr: mysql_up == 0
        for: 0m
        labels:
          severity: critical

      - alert: MysqlTooManyConnections(>80%)
        expr: avg by (instance) (rate(mysql_global_status_threads_connected[1m])) / avg by (instance) (mysql_global_variables_max_connections) * 100 > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: MySQL too many connections (> 80%) (instance {{ $labels.instance }})
          description: "More than 80% of MySQL connections are in use on {{ $labels.instance }}\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"


      - alert: MysqlHighThreadsRunning
        expr: avg by (instance) (rate(mysql_global_status_threads_running[1m])) / avg by (instance) (mysql_global_variables_max_connections) * 100 > 60
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: MySQL high threads running (instance {{ $labels.instance }})
          description: "More than 60% of MySQL connections are in running state on {{ $labels.instance }}\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"


      - alert: MysqlSlaveIoThreadNotRunning
        expr: mysql_slave_status_master_server_id > 0 and ON (instance) mysql_slave_status_slave_io_running == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: MySQL Slave IO thread not running (instance {{ $labels.instance }})
          description: "MySQL Slave IO thread not running on {{ $labels.instance }}\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"


      - alert: MysqlSlaveSqlThreadNotRunning
        expr: mysql_slave_status_master_server_id > 0 and ON (instance) mysql_slave_status_slave_sql_running == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: MySQL Slave SQL thread not running (instance {{ $labels.instance }})
          description: "MySQL Slave SQL thread not running on {{ $labels.instance }}\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"
      - alert: MysqlSlaveReplicationLag
        expr: mysql_slave_status_master_server_id > 0 and ON (instance) (mysql_slave_status_seconds_behind_master - mysql_slave_status_sql_delay) > 30
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: MySQL Slave replication lag (instance {{ $labels.instance }})
          description: "MySQL replication lag on {{ $labels.instance }}\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"
      - alert: MysqlSlowQueries
        expr: increase(mysql_global_status_slow_queries[1m]) > 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: MySQL slow queries (instance {{ $labels.instance }})
          description: "MySQL server mysql has some new slow query.\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"
      - alert: MysqlInnodbLogWaits
        expr: rate(mysql_global_status_innodb_log_waits[15m]) > 10
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: MySQL InnoDB log waits (instance {{ $labels.instance }})
          description: "MySQL innodb log writes stalling\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"
      - alert: MysqlRestarted
        expr: mysql_global_status_uptime < 60
        for: 0m
        labels:
          severity: info
        annotations:
          summary: MySQL restarted (instance {{ $labels.instance }})
          description: "MySQL has just been restarted, less than one minute ago on {{ $labels.instance }}.\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"
  redis-exporter-configmap.yml: |
    #https://awesome-prometheus-alerts.grep.to/rules#redis
    groups:
    - name: redisAlert
      rules:
      - alert: RedisDown
        expr: redis_up == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Redis down (instance {{ $labels.instance }})
          description: "Redis instance is down\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: RedisMissingMaster
        expr: (count(redis_instance_info{role="master"}) or vector(0)) < 1
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Redis missing master (instance {{ $labels.instance }})
          description: "Redis cluster has no node marked as master.\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: RedisTooManyMasters
        expr: count(redis_instance_info{role="master"}) > 1
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Redis too many masters (instance {{ $labels.instance }})
          description: "Redis cluster has too many nodes marked as master.\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: RedisDisconnectedSlaves
        expr: count without (instance, job) (redis_connected_slaves) - sum without (instance, job) (redis_connected_slaves) - 1 > 1
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Redis disconnected slaves (instance {{ $labels.instance }})
          description: "Redis not replicating for all slaves. Consider reviewing the redis replication status.\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: RedisReplicationBroken
        expr: delta(redis_connected_slaves[1m]) < 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Redis replication broken (instance {{ $labels.instance }})
          description: "Redis instance lost a slave\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: RedisClusterFlapping
        expr: changes(redis_connected_slaves[1m]) > 1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: Redis cluster flapping (instance {{ $labels.instance }})
          description: "Changes have been detected in Redis replica connection. This can occur when replica nodes lose connection to the master and reconnect (a.k.a flapping).\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: RedisMissingBackup
        expr: time() - redis_rdb_last_save_timestamp_seconds > 60 * 60 * 24
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Redis missing backup (instance {{ $labels.instance }})
          description: "Redis has not been backuped for 24 hours\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      # The exporter must be started with --include-system-metrics flag or REDIS_EXPORTER_INCL_SYSTEM_METRICS=true environment variable.
      - alert: RedisOutOfSystemMemory
        expr: redis_memory_used_bytes / redis_total_system_memory_bytes * 100 > 90
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Redis out of system memory (instance {{ $labels.instance }})
          description: "Redis is running out of system memory (> 90%)\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: RedisOutOfConfiguredMaxmemory
        expr: redis_memory_used_bytes / redis_memory_max_bytes * 100 > 90
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Redis out of configured maxmemory (instance {{ $labels.instance }})
          description: "Redis is running out of configured maxmemory (> 90%)\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: RedisTooManyConnections
        expr: redis_connected_clients > 100
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Redis too many connections (instance {{ $labels.instance }})
          description: "Redis instance has too many connections\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: RedisNotEnoughConnections
        expr: redis_connected_clients < 5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Redis not enough connections (instance {{ $labels.instance }})
          description: "Redis instance should have more connections (> 5)\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: RedisRejectedConnections
        expr: increase(redis_rejected_connections_total[1m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Redis rejected connections (instance {{ $labels.instance }})
          description: "Some connections to Redis has been rejected\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"
  elasticsearch-exporter-configmap.yml: |
    #https://awesome-prometheus-alerts.grep.to/rules#elasticsearch
    groups:
    - name: eslasticsearch Alert
      rules:
      - alert: ElasticsearchHeapUsageTooHigh
        expr: (elasticsearch_jvm_memory_used_bytes{area="heap"} / elasticsearch_jvm_memory_max_bytes{area="heap"}) * 100 > 90
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: Elasticsearch Heap Usage Too High (instance {{ $labels.instance }})
          description: "The heap usage is over 90%\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchHeapUsageWarning
        expr: (elasticsearch_jvm_memory_used_bytes{area="heap"} / elasticsearch_jvm_memory_max_bytes{area="heap"}) * 100 > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Elasticsearch Heap Usage warning (instance {{ $labels.instance }})
          description: "The heap usage is over 80%\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchDiskOutOfSpace
        expr: elasticsearch_filesystem_data_available_bytes / elasticsearch_filesystem_data_size_bytes * 100 < 10
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Elasticsearch disk out of space (instance {{ $labels.instance }})
          description: "The disk usage is over 90%\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchDiskSpaceLow
        expr: elasticsearch_filesystem_data_available_bytes / elasticsearch_filesystem_data_size_bytes * 100 < 20
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Elasticsearch disk space low (instance {{ $labels.instance }})
          description: "The disk usage is over 80%\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchClusterRed
        expr: elasticsearch_cluster_health_status{color="red"} == 1
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Elasticsearch Cluster Red (instance {{ $labels.instance }})
          description: "Elastic Cluster Red status\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchClusterYellow
        expr: elasticsearch_cluster_health_status{color="yellow"} == 1
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Elasticsearch Cluster Yellow (instance {{ $labels.instance }})
          description: "Elastic Cluster Yellow status\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchHealthyNodes
        expr: elasticsearch_cluster_health_number_of_nodes < 3
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Elasticsearch Healthy Nodes (instance {{ $labels.instance }})
          description: "Missing node in Elasticsearch cluster\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchHealthyDataNodes
        expr: elasticsearch_cluster_health_number_of_data_nodes < 3
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Elasticsearch Healthy Data Nodes (instance {{ $labels.instance }})
          description: "Missing data node in Elasticsearch cluster\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchRelocatingShards
        expr: elasticsearch_cluster_health_relocating_shards > 0
        for: 0m
        labels:
          severity: info
        annotations:
          summary: Elasticsearch relocating shards (instance {{ $labels.instance }})
          description: "Elasticsearch is relocating shards\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchRelocatingShardsTooLong
        expr: elasticsearch_cluster_health_relocating_shards > 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: Elasticsearch relocating shards too long (instance {{ $labels.instance }})
          description: "Elasticsearch has been relocating shards for 15min\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchInitializingShards
        expr: elasticsearch_cluster_health_initializing_shards > 0
        for: 0m
        labels:
          severity: info
        annotations:
          summary: Elasticsearch initializing shards (instance {{ $labels.instance }})
          description: "Elasticsearch is initializing shards\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchInitializingShardsTooLong
        expr: elasticsearch_cluster_health_initializing_shards > 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: Elasticsearch initializing shards too long (instance {{ $labels.instance }})
          description: "Elasticsearch has been initializing shards for 15 min\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchUnassignedShards
        expr: elasticsearch_cluster_health_unassigned_shards > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Elasticsearch unassigned shards (instance {{ $labels.instance }})
          description: "Elasticsearch has unassigned shards\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchPendingTasks
        expr: elasticsearch_cluster_health_number_of_pending_tasks > 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: Elasticsearch pending tasks (instance {{ $labels.instance }})
          description: "Elasticsearch has pending tasks. Cluster works slowly.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchNoNewDocuments
        expr: increase(elasticsearch_indices_docs{es_data_node="true"}[10m]) < 1
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Elasticsearch no new documents (instance {{ $labels.instance }})
          description: "No new documents for 10 min!\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

  prometheus-configmap.yml: |
    groups:
    - name:  prometheus alert
      rules:
      - alert: PrometheusJobMissing
        expr: absent(up{job="prometheus"})
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Prometheus job missing (instance {{ $labels.instance }})
          description: "A Prometheus job has disappeared\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTargetMissing
        expr: up == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus target missing (instance {{ $labels.instance }})
          description: "A Prometheus target has disappeared. An exporter might be crashed.\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusAllTargetsMissing
        expr: count by (job) (up) == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus all targets missing (instance {{ $labels.instance }})
          description: "A Prometheus job does not have living target anymore.\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusConfigurationReloadFailure
        expr: prometheus_config_last_reload_successful != 1
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Prometheus configuration reload failure (instance {{ $labels.instance }})
          description: "Prometheus configuration reload error\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTooManyRestarts
        expr: changes(process_start_time_seconds{job=~"prometheus|pushgateway|alertmanager"}[15m]) > 2
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Prometheus too many restarts (instance {{ $labels.instance }})
          description: "Prometheus has restarted more than twice in the last 15 minutes. It might be crashlooping.\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusAlertmanagerConfigurationReloadFailure
        expr: alertmanager_config_last_reload_successful != 1
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Prometheus AlertManager configuration reload failure (instance {{ $labels.instance }})
          description: "AlertManager configuration reload error\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusAlertmanagerConfigNotSynced
        expr: count(count_values("config_hash", alertmanager_config_hash)) > 1
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Prometheus AlertManager config not synced (instance {{ $labels.instance }})
          description: "Configurations of AlertManager cluster instances are out of sync\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusAlertmanagerE2eDeadManSwitch
        expr: vector(1)
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus AlertManager E2E dead man switch (instance {{ $labels.instance }})
          description: "Prometheus DeadManSwitch is an always-firing alert. It's used as an end-to-end test of Prometheus through the Alertmanager.\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusNotConnectedToAlertmanager
        expr: prometheus_notifications_alertmanagers_discovered < 1
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus not connected to alertmanager (instance {{ $labels.instance }})
          description: "Prometheus cannot connect the alertmanager\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusRuleEvaluationFailures
        expr: increase(prometheus_rule_evaluation_failures_total[3m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus rule evaluation failures (instance {{ $labels.instance }})
          description: "Prometheus encountered {{ $value }} rule evaluation failures, leading to potentially ignored alerts.\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTemplateTextExpansionFailures
        expr: increase(prometheus_template_text_expansion_failures_total[3m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus template text expansion failures (instance {{ $labels.instance }})
          description: "Prometheus encountered {{ $value }} template text expansion failures\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusRuleEvaluationSlow
        expr: prometheus_rule_group_last_duration_seconds > prometheus_rule_group_interval_seconds
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: Prometheus rule evaluation slow (instance {{ $labels.instance }})
          description: "Prometheus rule evaluation took more time than the scheduled interval. It indicates a slower storage backend access or too complex query.\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusNotificationsBacklog
        expr: min_over_time(prometheus_notifications_queue_length[10m]) > 0
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Prometheus notifications backlog (instance {{ $labels.instance }})
          description: "The Prometheus notification queue has not been empty for 10 minutes\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusAlertmanagerNotificationFailing
        expr: rate(alertmanager_notifications_failed_total[1m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus AlertManager notification failing (instance {{ $labels.instance }})
          description: "Alertmanager is failing sending notifications\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTargetEmpty
        expr: prometheus_sd_discovered_targets == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus target empty (instance {{ $labels.instance }})
          description: "Prometheus has no target in service discovery\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTargetScrapingSlow
        expr: prometheus_target_interval_length_seconds{quantile="0.9"} > 60
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: Prometheus target scraping slow (instance {{ $labels.instance }})
          description: "Prometheus is scraping exporters slowly\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusLargeScrape
        expr: increase(prometheus_target_scrapes_exceeded_sample_limit_total[10m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: Prometheus large scrape (instance {{ $labels.instance }})
          description: "Prometheus has many scrapes that exceed the sample limit\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTargetScrapeDuplicate
        expr: increase(prometheus_target_scrapes_sample_duplicate_timestamp_total[5m]) > 0
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Prometheus target scrape duplicate (instance {{ $labels.instance }})
          description: "Prometheus has many samples rejected due to duplicate timestamps but different values\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTsdbCheckpointCreationFailures
        expr: increase(prometheus_tsdb_checkpoint_creations_failed_total[1m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus TSDB checkpoint creation failures (instance {{ $labels.instance }})
          description: "Prometheus encountered {{ $value }} checkpoint creation failures\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTsdbCheckpointDeletionFailures
        expr: increase(prometheus_tsdb_checkpoint_deletions_failed_total[1m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus TSDB checkpoint deletion failures (instance {{ $labels.instance }})
          description: "Prometheus encountered {{ $value }} checkpoint deletion failures\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTsdbCompactionsFailed
        expr: increase(prometheus_tsdb_compactions_failed_total[1m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus TSDB compactions failed (instance {{ $labels.instance }})
          description: "Prometheus encountered {{ $value }} TSDB compactions failures\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTsdbHeadTruncationsFailed
        expr: increase(prometheus_tsdb_head_truncations_failed_total[1m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus TSDB head truncations failed (instance {{ $labels.instance }})
          description: "Prometheus encountered {{ $value }} TSDB head truncation failures\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTsdbReloadFailures
        expr: increase(prometheus_tsdb_reloads_failures_total[1m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus TSDB reload failures (instance {{ $labels.instance }})
          description: "Prometheus encountered {{ $value }} TSDB reload failures\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTsdbWalCorruptions
        expr: increase(prometheus_tsdb_wal_corruptions_total[1m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus TSDB WAL corruptions (instance {{ $labels.instance }})
          description: "Prometheus encountered {{ $value }} TSDB WAL corruptions\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: PrometheusTsdbWalTruncationsFailed
        expr: increase(prometheus_tsdb_wal_truncations_failed_total[1m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Prometheus TSDB WAL truncations failed (instance {{ $labels.instance }})
          description: "Prometheus encountered {{ $value }} TSDB WAL truncation failures\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"


kind: ConfigMap
metadata:
  creationTimestamp: null
  name: prometheus-operator-rules
  namespace: monitoring

```

参考：prometheus-operator生成的规则

##### 单机es的配置

注释以上内容。因为单机没有多个node, datanode, 也不会分片 

`:'<,'>s@\(^[[:space:]]\+\)\([^[:space:]]\)@\1#\2@g                                            `

```bash
      - alert: ElasticsearchClusterYellow
        expr: elasticsearch_cluster_health_status{color="yellow"} == 1 
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: Elasticsearch Cluster Yellow (instance {{ $labels.instance }})
          description: "Elastic Cluster Yellow status\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchHealthyNodes
        expr: elasticsearch_cluster_health_number_of_nodes < 3
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Elasticsearch Healthy Nodes (instance {{ $labels.instance }})
          description: "Missing node in Elasticsearch cluster\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: ElasticsearchHealthyDataNodes
        expr: elasticsearch_cluster_health_number_of_data_nodes < 3
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Elasticsearch Healthy Data Nodes (instance {{ $labels.instance }})
          description: "Missing data node in Elasticsearch cluster\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"


      - alert: ElasticsearchUnassignedShards
        expr: elasticsearch_cluster_health_unassigned_shards > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Elasticsearch unassigned shards (instance {{ $labels.instance }})
          description: "Elasticsearch has unassigned shards\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

```

##### 去掉APIError

集群bug了，只要不影响集群使用

```bash
      #- alert: KubeAPIErrorsHigh
        #annotations:
          #message: API server is returning errors for {{ $value | humanizePercentage }}
            #of requests.
          #runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapierrorshigh
        #expr: |
          #sum(rate(apiserver_request_total{job="kubernetes-apiservers",code=~"5.."}[5m]))
            #/
          #sum(rate(apiserver_request_total{job="kubernetes-apiservers"}[5m])) > 0.03
        #for: 10m
        #labels:
          #severity: critical
      #- alert: KubeAPIErrorsHigh
        #annotations:
          #message: API server is returning errors for {{ $value | humanizePercentage }}
            #of requests.
          #runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapierrorshigh
        #expr: |
          #sum(rate(apiserver_request_total{job="kubernetes-apiservers",code=~"5.."}[5m]))
            #/
          #sum(rate(apiserver_request_total{job="kubernetes-apiservers"}[5m])) > 0.01
        #for: 10m
        #labels:
          #severity: warning
      #- alert: KubeAPIErrorsHigh
        #annotations:
          #message: API server is returning errors for {{ $value | humanizePercentage }}
            #of requests for {{ $labels.verb }} {{ $labels.resource }} {{ $labels.subresource
            #}}.
          #runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapierrorshigh
        #expr: |
          #sum(rate(apiserver_request_total{job="kubernetes-apiservers",code=~"5.."}[5m])) by (resource,subresource,verb)
            #/
          #sum(rate(apiserver_request_total{job="kubernetes-apiservers"}[5m])) by (resource,subresource,verb) > 0.10
        #for: 10m
        #labels:
          #severity: critical
      #- alert: KubeAPIErrorsHigh
        #annotations:
          #message: API server is returning errors for {{ $value | humanizePercentage }}
            #of requests for {{ $labels.verb }} {{ $labels.resource }} {{ $labels.subresource
            #}}.
          #runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapierrorshigh
        #expr: |
          #sum(rate(apiserver_request_total{job="kubernetes-apiservers",code=~"5.."}[5m])) by (resource,subresource,verb)
            #/
          #sum(rate(apiserver_request_total{job="kubernetes-apiservers"}[5m])) by (resource,subresource,verb) > 0.05
        #for: 10m
        #labels:
          #severity: warning
```

##### node-exporter相关规则

`root@master01:/data/weizhixiu/yaml/prometheus# cat node-exporter-configmap.yaml`

```diff
apiVersion: v1
data:
  node-exporter-record-rules.yml: |
    groups:
      - name: kubernetes-node-exporter-daemonset-pod-record
        rules:
        - expr: up{job=~"kubernetes-node-exporter-daemonset-pod"}
          record: node_exporter:up 
          labels: 
            desc: "节点是否在线, 在线1,不在线0"
            unit: " "
            job: "kubernetes-node-exporter-daemonset-pod"
        - expr: time() - node_boot_time_seconds{}
          record: node_exporter:node_uptime
          labels: 
            desc: "节点的运行时间"
            unit: "s"
            job: "kubernetes-node-exporter-daemonset-pod"
    ##############################################################################################
    #                              cpu                                                           #
        - expr: (1 - avg by (environment,instance) (irate(node_cpu_seconds_total{job="kubernetes-node-exporter-daemonset-pod",mode="idle"}[5m])))  * 100 
          record: node_exporter:cpu:total:percent
          labels: 
            desc: "节点的cpu总消耗百分比"
            unit: "%"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job="kubernetes-node-exporter-daemonset-pod",mode="idle"}[5m])))  * 100 
          record: node_exporter:cpu:idle:percent
          labels: 
            desc: "节点的cpu idle百分比"
            unit: "%"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job="kubernetes-node-exporter-daemonset-pod",mode="iowait"}[5m])))  * 100 
          record: node_exporter:cpu:iowait:percent
          labels: 
            desc: "节点的cpu iowait百分比"
            unit: "%"
            job: "kubernetes-node-exporter-daemonset-pod"


        - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job="kubernetes-node-exporter-daemonset-pod",mode="system"}[5m])))  * 100 
          record: node_exporter:cpu:system:percent
          labels: 
            desc: "节点的cpu system百分比"
            unit: "%"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job="kubernetes-node-exporter-daemonset-pod",mode="user"}[5m])))  * 100 
          record: node_exporter:cpu:user:percent
          labels: 
            desc: "节点的cpu user百分比"
            unit: "%"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job="kubernetes-node-exporter-daemonset-pod",mode=~"softirq|nice|irq|steal"}[5m])))  * 100 
          record: node_exporter:cpu:other:percent
          labels: 
            desc: "节点的cpu 其他的百分比"
            unit: "%"
            job: "kubernetes-node-exporter-daemonset-pod"
    ##############################################################################################


    ##############################################################################################
    #                                    memory                                                  #
        - expr: node_memory_MemTotal_bytes{job="kubernetes-node-exporter-daemonset-pod"}
          record: node_exporter:memory:total
          labels: 
            desc: "节点的内存总量"
            unit: byte
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: node_memory_MemFree_bytes{job="kubernetes-node-exporter-daemonset-pod"}
          record: node_exporter:memory:free
          labels: 
            desc: "节点的剩余内存量"
            unit: byte
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: node_memory_MemTotal_bytes{job="kubernetes-node-exporter-daemonset-pod"} - node_memory_MemFree_bytes{job="kubernetes-node-exporter-daemonset-pod"}
          record: node_exporter:memory:used
          labels: 
            desc: "节点的已使用内存量"
            unit: byte
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: node_memory_MemTotal_bytes{job="kubernetes-node-exporter-daemonset-pod"} - node_memory_MemAvailable_bytes{job="kubernetes-node-exporter-daemonset-pod"}
          record: node_exporter:memory:actualused
          labels: 
            desc: "节点用户实际使用的内存量"
            unit: byte
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: (1-(node_memory_MemAvailable_bytes{job="kubernetes-node-exporter-daemonset-pod"} / (node_memory_MemTotal_bytes{job="kubernetes-node-exporter-daemonset-pod"})))* 100
          record: node_exporter:memory:used:percent
          labels: 
            desc: "节点的内存使用百分比"
            unit: "%"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: ((node_memory_MemAvailable_bytes{job="kubernetes-node-exporter-daemonset-pod"} / (node_memory_MemTotal_bytes{job="kubernetes-node-exporter-daemonset-pod"})))* 100
          record: node_exporter:memory:free:percent
          labels: 
            desc: "节点的内存剩余百分比"
            unit: "%"
            job: "kubernetes-node-exporter-daemonset-pod"
    ##############################################################################################
    #                                   load                                                     #
        - expr: sum by (instance) (node_load1{job="kubernetes-node-exporter-daemonset-pod"})
          record: node_exporter:load:load1
          labels: 
            desc: "系统1分钟负载"
            unit: " "
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: sum by (instance) (node_load5{job="kubernetes-node-exporter-daemonset-pod"})
          record: node_exporter:load:load5
          labels: 
            desc: "系统5分钟负载"
            unit: " "
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: sum by (instance) (node_load15{job="kubernetes-node-exporter-daemonset-pod"})
          record: node_exporter:load:load15
          labels: 
            desc: "系统15分钟负载"
            unit: " "
            job: "kubernetes-node-exporter-daemonset-pod"
       
    ##############################################################################################
    #                                 disk                                                       #
        - expr: node_filesystem_size_bytes{job="kubernetes-node-exporter-daemonset-pod" ,fstype=~"ext4|xfs"}
          record: node_exporter:disk:usage:total
          labels: 
            desc: "节点的磁盘总量"
            unit: byte
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: node_filesystem_avail_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype=~"ext4|xfs"}
          record: node_exporter:disk:usage:free
          labels: 
            desc: "节点的磁盘剩余空间"
            unit: byte
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: node_filesystem_size_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype=~"ext4|xfs"} - node_filesystem_avail_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype=~"ext4|xfs"}
          record: node_exporter:disk:usage:used
          labels: 
            desc: "节点的磁盘使用的空间"
            unit: byte
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr:  (1 - node_filesystem_avail_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype=~"ext4|xfs"} / node_filesystem_size_bytes{job="kubernetes-node-exporter-daemonset-pod",fstype=~"ext4|xfs"}) * 100 
          record: node_exporter:disk:used:percent    
          labels: 
            desc: "节点的磁盘的使用百分比"
            unit: "%"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: irate(node_disk_reads_completed_total{job="kubernetes-node-exporter-daemonset-pod"}[1m])
          record: node_exporter:disk:read:count:rate
          labels: 
            desc: "节点的磁盘读取速率"
            unit: "次/秒"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: irate(node_disk_writes_completed_total{job="kubernetes-node-exporter-daemonset-pod"}[1m])
          record: node_exporter:disk:write:count:rate
          labels: 
            desc: "节点的磁盘写入速率"
            unit: "次/秒"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: (irate(node_disk_written_bytes_total{job="kubernetes-node-exporter-daemonset-pod"}[1m]))/1024/1024
          record: node_exporter:disk:read:mb:rate
          labels: 
            desc: "节点的设备读取MB速率"
            unit: "MB/s"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: (irate(node_disk_read_bytes_total{job="kubernetes-node-exporter-daemonset-pod"}[1m]))/1024/1024
          record: node_exporter:disk:write:mb:rate
          labels: 
            desc: "节点的设备写入MB速率"
            unit: "MB/s"
            job: "kubernetes-node-exporter-daemonset-pod"

    ##############################################################################################
    #                                filesystem                                                  #
        - expr:   (1 -node_filesystem_files_free{job="kubernetes-node-exporter-daemonset-pod",fstype=~"ext4|xfs"} / node_filesystem_files{job="kubernetes-node-exporter-daemonset-pod",fstype=~"ext4|xfs"}) * 100 
          record: node_exporter:filesystem:used:percent    
          labels: 
            desc: "节点的inode的剩余可用的百分比"
            unit: "%"
            job: "kubernetes-node-exporter-daemonset-pod"
    #############################################################################################
    #                                filefd                                                     #
        - expr: node_filefd_allocated{job="kubernetes-node-exporter-daemonset-pod"}
          record: node_exporter:filefd_allocated:count
          labels: 
            desc: "节点的文件描述符打开个数"
            unit: "%"
            job: "kubernetes-node-exporter-daemonset-pod"
     
        - expr: node_filefd_allocated{job="kubernetes-node-exporter-daemonset-pod"}/node_filefd_maximum{job="kubernetes-node-exporter-daemonset-pod"} * 100 
          record: node_exporter:filefd_allocated:percent
          labels: 
            desc: "节点的文件描述符打开百分比"
            unit: "%"
            job: "kubernetes-node-exporter-daemonset-pod"

    #############################################################################################
    #                                network                                                    #
        - expr: avg by (environment,instance,device) (irate(node_network_receive_bytes_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
          record: node_exporter:network:netin:bit:rate
          labels: 
            desc: "节点网卡eth0每秒接收的比特数"
            unit: "bit/s"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: avg by (environment,instance,device) (irate(node_network_transmit_bytes_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
          record: node_exporter:network:netout:bit:rate
          labels: 
            desc: "节点网卡eth0每秒发送的比特数"
            unit: "bit/s"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: avg by (environment,instance,device) (irate(node_network_receive_packets_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
          record: node_exporter:network:netin:packet:rate
          labels: 
            desc: "节点网卡每秒接收的数据包个数"
            unit: "个/秒"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: avg by (environment,instance,device) (irate(node_network_transmit_packets_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
          record: node_exporter:network:netout:packet:rate
          labels: 
            desc: "节点网卡发送的数据包个数"
            unit: "个/秒"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: avg by (environment,instance,device) (irate(node_network_receive_errs_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
          record: node_exporter:network:netin:error:rate
          labels: 
            desc: "节点设备驱动器检测到的接收错误包的数量"
            unit: "个/秒"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: avg by (environment,instance,device) (irate(node_network_transmit_errs_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
          record: node_exporter:network:netout:error:rate
          labels: 
            desc: "节点设备驱动器检测到的发送错误包的数量"
            unit: "个/秒"
            job: "kubernetes-node-exporter-daemonset-pod"
          
        - expr: node_tcp_connection_states{job="kubernetes-node-exporter-daemonset-pod", state="established"}
          record: node_exporter:network:tcp:established:count
          labels: 
            desc: "节点当前established的个数"
            unit: "个"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: node_tcp_connection_states{job="kubernetes-node-exporter-daemonset-pod", state="time_wait"}
          record: node_exporter:network:tcp:timewait:count
          labels: 
            desc: "节点timewait的连接数"
            unit: "个"
            job: "kubernetes-node-exporter-daemonset-pod"

        - expr: sum by (environment,instance) (node_tcp_connection_states{job="kubernetes-node-exporter-daemonset-pod"})
          record: node_exporter:network:tcp:total:count
          labels: 
            desc: "节点tcp连接总数"
            unit: "个"
            job: "kubernetes-node-exporter-daemonset-pod"
       
    #############################################################################################
    #                                process                                                    #
        - expr: node_processes_state{state="Z"}
          record: node_exporter:process:zoom:total:count
          labels: 
            desc: "节点当前状态为zoom的个数"
            unit: "个"
            job: "kubernetes-node-exporter-daemonset-pod"
    #############################################################################################
    #                                other                                                    #
        - expr: abs(node_timex_offset_seconds{job="kubernetes-node-exporter-daemonset-pod"})
          record: node_exporter:time:offset
          labels: 
            desc: "节点的时间偏差"
            unit: "s"
            job: "kubernetes-node-exporter-daemonset-pod"

    #############################################################################################
       
        - expr: count by (instance) ( count by (instance,cpu) (node_cpu_seconds_total{ mode='system'}) ) 
          record: node_exporter:cpu:count
  node-exporter-alert-rules.yml: |
    groups:
      - name: node-exporter-alert
        rules:
        - alert: node-exporter-down
          expr: node_exporter:up == 0 
          for: 1m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} 宕机了"  
            description: "instance: {{ $labels.instance }} \n- job: {{ $labels.job }} 关机了， 时间已经1分钟了。" 
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-cpu-high 
          expr:  node_exporter:cpu:total:percent > 80
          for: 3m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} cpu 使用率高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-cpu-iowait-high 
          expr:  node_exporter:cpu:iowait:percent >= 12
          for: 3m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} cpu iowait 使用率高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-load-load1-high 
          expr:  (node_exporter:load:load1) > (node_exporter:cpu:count) * 1.2
          for: 3m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} load1 使用率高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-memory-high
          expr:  node_exporter:memory:used:percent > 85
          for: 3m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} memory 使用率高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-disk-high
          expr:  node_exporter:disk:used:percent > 88
          for: 10m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} disk 使用率高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-disk-read:count-high
          expr:  node_exporter:disk:read:count:rate > 3000
          for: 2m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} iops read 使用率高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-disk-write-count-high
          expr:  node_exporter:disk:write:count:rate > 3000
          for: 2m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} iops write 使用率高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"




        - alert: node-exporter-disk-read-mb-high
          expr:  node_exporter:disk:read:mb:rate > 60 
          for: 2m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} 读取字节数 高于 {{ $value }}"  
            description: ""    
            instance: "{{ $labels.instance }}"
            value: "{{ $value }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-disk-write-mb-high
          expr:  node_exporter:disk:write:mb:rate > 60
          for: 2m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} 写入字节数 高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-filefd-allocated-percent-high 
          expr:  node_exporter:filefd_allocated:percent > 80
          for: 10m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} 打开文件描述符 高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-network-netin-error-rate-high
          expr:  node_exporter:network:netin:error:rate > 4
          for: 1m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} 包进入的错误速率 高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"
        - alert: node-exporter-network-netin-packet-rate-high
          expr:  node_exporter:network:netin:packet:rate > 35000
          for: 1m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} 包进入速率 高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-network-netout-packet-rate-high
          expr:  node_exporter:network:netout:packet:rate > 35000
          for: 1m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} 包流出速率 高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-network-tcp-total-count-high
          expr:  node_exporter:network:tcp:total:count > 40000
          for: 1m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} tcp连接数量 高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-process-zoom-total-count-high 
          expr:  node_exporter:process:zoom:total:count > 10
          for: 10m
          labels: 
            severity: info
          annotations: 
            summary: "instance: {{ $labels.instance }} 僵死进程数量 高于 {{ $value }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"

        - alert: node-exporter-time-offset-high
          expr:  node_exporter:time:offset > 0.03
          for: 2m
          labels: 
            severity: info
          annotations:
            summary: "instance: {{ $labels.instance }} {{ $labels.desc }}  {{ $value }} {{ $labels.unit }}"  
            description: ""    
            value: "{{ $value }}"
            instance: "{{ $labels.instance }}"
            grafana: "http://xxxxxxxx.com/d/node-exporter/node-exporter?orgId=1&var-instance={{ $labels.instance }} "
            console: "https://ecs.console.aliyun.com/#/server/{{ $labels.instanceid }}/detail?regionId=cn-beijing"
            cloudmonitor: "https://cloudmonitor.console.aliyun.com/#/hostDetail/chart/instanceId={{ $labels.instanceid }}&system=&region=cn-beijing&aliyunhost=true"
            id: "{{ $labels.instanceid }}"
            type: "aliyun_meta_ecs_info"
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: node-exporter-configmap
  namespace: monitoring
```

##### alertmanager是否挂了？

```bash
 - alert: PrometheusAlertmanagerE2eDeadManSwitch
    expr: vector(1)
    for: 0m
    labels:
      severity: critical
    annotations:
      summary: Prometheus AlertManager E2E dead man switch (instance {{ $labels.instance }})
      description: Prometheus DeadManSwitch is an always-firing alert. It's used as an end-to-end test of Prometheus through the Alertmanager.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}
```

报警挂了你怎么知道？这个在group_interval一直发告警就好了

#### 启动命令

```bash
root@master01:/data/weizhixiu/yaml/prometheus# kubectl apply -f prometheus-deployment.yaml -f rbac-setup.yml -f prometheus-rules-configmap.yaml -f prometheus-configmap.yaml 
```

## alertmanager



### 准备Dockerfile

```bash
root@master01:/data/weizhixiu/dockerfile/prometheus# docker build -t harbor.youwoyouqu.io/pub/alertmanager:0.21.0 ./  -f Dockerfile-alertmanager 

root@master01:/data/weizhixiu/dockerfile/prometheus# docker run harbor.youwoyouqu.io/pub/alertmanager:0.21.0 --version
alertmanager, version 0.21.0 (branch: HEAD, revision: 4c6c03ebfe21009c546e4d1e9b92c371d67c021d)
  build user:       root@dee35927357f
  build date:       20200617-08:54:02
  go version:       go1.14.4

root@master01:/data/weizhixiu/dockerfile/prometheus# docker push  harbor.youwoyouqu.io/pub/alertmanager:0.21.0
```

### 准备yaml

`root@master01:/data/weizhixiu/yaml/prometheus# cat alertmanager-deployment.yaml`

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
  labels:
+    k8s-app: alertmanager # 由于以上prometheus配置获取k8s-app=alertmanager, 所以需要加这个标签
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: alertmanager
  template:
    metadata:
      labels:
        k8s-app: alertmanager
    spec:
      containers:
        - name: alertmanager-pod
          image: harbor.youwoyouqu.io/pub/alertmanager:0.21.0
          imagePullPolicy: "IfNotPresent"
          args:
            - --config.file=/etc/config/alertmanager.yml
            - --storage.path=/data
          ports:
            - containerPort: 9093
          readinessProbe:
            httpGet:
              path: /#/status
              port: 9093
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
            - name: config-volume
+              mountPath: /etc/config # 配置文件
            - name: storage-volume
              mountPath: "/data"
              subPath: ""
          resources:
            limits:
              cpu: 10m
              memory: 50Mi
            requests:
              cpu: 10m
              memory: 50Mi
      volumes:
        - name: config-volume
          configMap:
            name: alertmanager-config
        - name: storage-volume
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: alertmanager-service
  namespace: monitoring
spec:
  ports:
  - name: 9093-9093
    port: 9093
    protocol: TCP
    targetPort: 9093
    nodePort: 32095
  selector:
    k8s-app: alertmanager
  type: NodePort
```

### 准备配置文件

`root@master01:/data/weizhixiu/yaml/prometheus# cat alertmanager-deployment.yaml`

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
  labels:
    k8s-app: alertmanager
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: alertmanager
  template:
    metadata:
      labels:
        k8s-app: alertmanager
    spec:
      containers:
        - name: alertmanager-pod
          image: harbor.youwoyouqu.io/pub/alertmanager:0.21.0
          imagePullPolicy: "IfNotPresent"
          args:
            - --config.file=/etc/config/alertmanager.yml
            - --storage.path=/data
          ports:
            - containerPort: 9093
          readinessProbe:
            httpGet:
              path: /#/status
              port: 9093
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
            - name: storage-volume
              mountPath: "/data"
              subPath: ""
          resources:
            limits:
              cpu: 10m
              memory: 50Mi
            requests:
              cpu: 10m
              memory: 50Mi
      volumes:
        - name: config-volume
          configMap:
            name: alertmanager-config
        - name: storage-volume
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: alertmanager-service
  namespace: monitoring
spec:
  ports:
  - name: 9093-9093
    port: 9093
    protocol: TCP
    targetPort: 9093
    nodePort: 32095
  selector:
    k8s-app: alertmanager
  type: NodePort
root@master01:/data/weizhixiu/yaml/prometheus# cat alertmanager-configmap.yml 
apiVersion: v1
data:
  alertmanager.yml: |
    #https://hub.docker.com/r/prom/alertmanager
    global:
      #在没有告警的情况下声明为已解决的时间
      resolve_timeout: 5m
      smtp_smarthost: 'smtp.qq.com:465'
      smtp_from: '1062670898@qq.com'
      smtp_auth_username: '1062670898@qq.com'
      smtp_auth_password: 'scyhdklulyrmbdci'
    #  smtp_hello: 'hello'
      smtp_require_tls: false
    #所有告警信息进入之后的根路由，用于设置告警的分发策略. 根路由之下可以设置子路由，子路由继承根属性，也可以覆盖根路由的属性, 
    route:
      #这里的标签列表是接收到告警信息后的重新分组标签，例如，在接收到的告警信息里有许多具有 cluster=A 和 alertname=LatncyHigh 标签的告警信息会被批量聚合到一个分组里
      #group_by: ['alertname', 'cluster']
      #在一个新的告警分组被创建后，需要等待至少 group_wait 时间来初始化通知，这种方式可以确保有足够的时间为同一分组收获多条告警，然后一起触发这条告警信息
+      group_wait: 1m
      #在第 1 条告警发送后，等待group_interval时间来发送新的一组告警信息
+      group_interval: 1h     
      #如果某条告警信息已经发送成功，则等待repeat_interval时间重新发送他们。这里不启用这个功能~
+      repeat_interval: 3d
      #默认的receiver：如果某条告警没有被一个route匹配，则发送给默认的接收器
      receiver: default 
      #上面的所有属性都由所有子路由继承，并且可以在每个子路由上覆盖
      routes:
      - receiver: email
+        group_interval: 3m    # 重要的告警3分钟一次
+        repeat_interval: 1d  # 1天重复一次，避免相同的一些错误总来
        match:
          severity: critical   # prometheus定义的labels为这个  
    receivers:
    - name: 'default'      # 接收名
      email_configs:
      - to: '1062670898@qq.com'
        send_resolved: true
    #  - to: 'wang*******@163.com'
    #    send_resolved: true
    - name: 'email'      # 接收名
      email_configs:
      - to: '1062670898@qq.com'
        send_resolved: true
    #  - to: 'wang*******@163.com'
    #    send_resolved: true

kind: ConfigMap
metadata:
  creationTimestamp: null
  name: alertmanager-config
  namespace: monitoring
```

### 启动命令

查看报警：

https://github.com/prometheus/alertmanager#high-availability

```bash
root@master01:/data/weizhixiu/yaml/prometheus# kubectl apply -f alertmanager-deployment.yaml -f alertmanager-configmap.yml
```

## grafana

### 准备dockerfile

```bash
root@master01:/data/weizhixiu/dockerfile/prometheus# cat Dockerfile-grafana 
FROM harbor.youwoyouqu.io/baseimage/alpine:base

ADD grafana-7.4.3.linux-amd64.tar.gz /usr/local/

WORKDIR /usr/local/grafana-7.4.3/
ENTRYPOINT ["/usr/local/grafana-7.4.3/bin/grafana-server"]
```

```bash
root@master01:/data/weizhixiu/dockerfile/prometheus# docker build -t harbor.youwoyouqu.io/pub/grafana:7.4.3 . -f Dockerfile-grafana 
root@master01:/data/weizhixiu/dockerfile/prometheus# docker run  --rm  harbor.youwoyouqu.io/pub/grafana:7.4.3  -v
Version 7.4.3 (commit: 010f20c1c8, branch: HEAD)

root@master01:/data/weizhixiu/yaml/prometheus# docker push harbor.youwoyouqu.io/pub/grafana:7.4.3

```

### 准备yaml

`root@master01:/data/weizhixiu/yaml/prometheus# cat grafana.yaml`

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  strategy:
   rollingUpdate:
     maxSurge: 1
     maxUnavailable: 0
   type: "RollingUpdate"
  selector:
    matchLabels:
      k8s-app: grafana
  template:
    metadata:
      labels:
        k8s-app: grafana
    spec:
       # 由于grafana有数据产生，避免重启丢失数据，所以把grafana定在一个空闲节点
+      tolerations:
      - effect: NoSchedule
        operator: Exists
+      nodeName: master04.youwoyouqu.io
      initContainers:
      - name: "init-chown-data"
        image: "busybox:latest"
        imagePullPolicy: "IfNotPresent"
        command: ["/bin/sh","-c","install -dv -o 472 -g 1  /var/lib/grafana/plugins /var/log/grafana"] # https://grafana.com/docs/grafana/latest/installation/docker/#migrate-to-v51-or-later
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
        - name: grafana-storage
          mountPath: /var/log/grafana
      containers:
      - name: grafana-pod
        image:  harbor.youwoyouqu.io/pub/grafana:7.4.3
        ports:
        - containerPort: 3000
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/ssl/certs
          name: ca-certificates
          readOnly: true
        - name: grafana-storage
          mountPath: /var/lib/grafana
        - name: grafana-storage
          mountPath: /var/log/grafana
        env:
        - name: GF_PATHS_PLUGINS
          value: /var/lib/grafana/plugins
        - name: GF_PATHS_DATA
          value: /var/lib/grafana
        - name: GF_SERVER_HTTP_PORT
          value: "3000"
        #- name: GF_INSTALL_PLUGINS
        #  value: grafana-clock-panel,grafana-simple-json-datasource,grafana-piechart-panel
        # 进入容器开代理;
        # 列出插件 ./bin/grafana-cli plugins list-remote 
        # 安装插件 ./bin/grafana-cli plugins install <name>
      volumes:
      - name: ca-certificates
        hostPath:
          path: /etc/ssl/certs
      - name: grafana-storage
        hostPath:
          path: /opt/grafana_data
          type: DirectoryOrCreate
---
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 32093
  selector:
    k8s-app: grafana

```

### 启动

```bash
root@master01:/data/weizhixiu/yaml/prometheus# kubectl apply -f grafana.yaml 
```



## nginx反代prometheus, alertmanager, grafana

```nginx
[root@wzx ~]# cat /etc/nginx/conf.d/monitor.conf 
server {
	server_name grafana.youwoyouqu.io;
	listen 80;


	location / {
		proxy_pass http://192.168.0.27:32093;
		proxy_set_header Host $host;
	}
}
server {
	server_name prometheus.youwoyouqu.io;
	listen 80;

	location / {
		proxy_pass http://192.168.0.27:32094;
		proxy_set_header Host $host;
	}
}
server {
	server_name alertmanager.youwoyouqu.io;
	listen 80;

	location / {
		proxy_pass http://192.168.0.27:32095;
		proxy_set_header Host localhost;
	}
}
```

```bash
[root@wzx ~]# nginx -t
nginxnginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@wzx ~]# nginx -s reload
```

## 常用命令

### 重载脚本

curl命令如下

#### prometheus

```bash
curl --location --request POST 'http://prometheus.youwoyouqu.io/-/reload'
```

#### alertmanager

```bash
curl --location --request POST 'http://alertmanager.youwoyouqu.io/-/reload'
```

### 获取报警

```bash
[root@elk ~]# /usr/local/alertmanager-0.21.0.linux-amd64/amtool  alert query --alertmanager.url=http://alertmanager.youwoyouqu.io
Alertname                               Starts At                Summary                                                         
Watchdog                                2021-03-15 21:31:47 UTC                                                                  
ElasticsearchUnassignedShards           2021-03-15 21:31:48 UTC  Elasticsearch unassigned shards (instance 192.168.0.171:9114)   
ElasticsearchClusterYellow              2021-03-15 21:31:48 UTC  Elasticsearch Cluster Yellow (instance 192.168.0.171:9114)      
ElasticsearchHealthyNodes               2021-03-15 21:31:48 UTC  Elasticsearch Healthy Nodes (instance 192.168.0.171:9114)       
ElasticsearchHealthyDataNodes           2021-03-15 21:31:48 UTC  Elasticsearch Healthy Data Nodes (instance 192.168.0.171:9114)  
PrometheusAlertmanagerE2eDeadManSwitch  2021-03-15 21:31:55 UTC  Prometheus AlertManager E2E dead man switch (instance )         
KubeAPIErrorsHigh                       2021-03-15 21:42:43 UTC                                                                  
KubeAPIErrorsHigh                       2021-03-15 21:42:43 UTC                                                                  
KubeAPIErrorsHigh                       2021-03-15 21:42:43 UTC                                                                  
KubeAPIErrorsHigh                       2021-03-15 21:42:43 UTC                                                                  
KubeAPIErrorsHigh                       2021-03-15 21:42:43 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:21 UTC                                                                  
CPUThrottlingHigh                       2021-03-15 21:47:36 UTC                                                                  
node-exporter-memory-high               2021-03-15 21:48:09 UTC  instance: 192.168.0.91:9100 memory 使用率高于 92.1970064573924       
KubeAPIErrorsHigh                       2021-03-16 01:06:58 UTC                                                                  
targetDown                              2021-03-16 01:20:32 UTC      # 即3-16号9点20的报警。已经发出了。
```

### 清理指标脚本

https://www.qikqiak.com/post/prometheus-delete-metrics/

```bash
# 清理job为..
curl --location -g --request POST 'http://prometheus.youwoyouqu.io/api/v1/admin/tsdb/delete_series?match[]={job="node-exporter-static"}'
```



## grafana配置

### 安装插件

```bash
#  value: grafana-clock-panel,grafana-simple-json-datasource,grafana-piechart-panel
# 进入容器开代理;
# 列出插件 ./bin/grafana-cli plugins list-remote 
# 安装插件 ./bin/grafana-cli plugins install <name>
```

### kubernetes 集群  11802

![image-20210315144419304](http://myapp.img.mykernel.cn/image-20210315144419304.png)

### mysql 13106

![image-20210315144557152](http://myapp.img.mykernel.cn/image-20210315144557152.png)



### redis 11835

![image-20210315144704353](http://myapp.img.mykernel.cn/image-20210315144704353.png)

### elasticsearch 13562

![image-20210315144836123](http://myapp.img.mykernel.cn/image-20210315144836123.png)

### etcd集群  3070

![image-20210315144939361](http://myapp.img.mykernel.cn/image-20210315144939361.png)

### 打包所有json，于下

链接：https://pan.baidu.com/s/1rKqPaVyZZ8f134dS-jTGkQ 
提取码：sfl2 
复制这段内容后打开百度网盘手机App，操作更方便哦