---
title: "prometheus 自动发现编写探寻"
date: 2021-03-15 10:07:45
tags:
- 个人日记
- prometheus
---



当prometheus和grafana以默认配置启动之后

详细的每个配置参考[prometheus配置篇](http://blog.mykernel.cn/tags/prometheus/)

# grafana配置
<!--more-->

![image-20210315180842085](http://myapp.img.mykernel.cn/image-20210315180842085.png)

默认admin, admin

![image-20210315180956143](http://myapp.img.mykernel.cn/image-20210315180956143.png)

![img](http://myapp.img.mykernel.cn/wps2.jpg) 

![img](http://myapp.img.mykernel.cn/wps3.jpg) 

 

![img](http://myapp.img.mykernel.cn/wps4.jpg) 

 

![img](http://myapp.img.mykernel.cn/wps5.jpg) 

![img](http://myapp.img.mykernel.cn/wps6.jpg) 

刚刚导入

![img](http://myapp.img.mykernel.cn/wps7.jpg) 

![img](http://myapp.img.mykernel.cn/wps8.jpg) 

![image-20210315180920854](http://myapp.img.mykernel.cn/image-20210315180920854.png)

# prometheus配置

```bash
    global:
      scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
      evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
    # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
    rule_files:
    - "/etc/rules/*.yml"    # 由于deploy挂载configmap在此路径 
    - "/etc/node-exporter-rules/*.yml" # 加载node相关的通用报警  https://www.cnblogs.com/zhaojiedi1992/p/zhaojiedi_liunx_65_prometheus_alertmanager_rule.html
      # - "second_rules.yml"
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
      - target_label: address
        replacement: 192.168.0.171:3306

    - job_name: 'redis' 
      static_configs:
      - targets: ['192.168.0.171:9121']
      relabel_configs:
      - target_label: address
        replacement: 192.168.0.171:6379

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

    # 抓取方式2, 既然是pod的指标都走http
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

```



![img](http://myapp.img.mykernel.cn/wps9.jpg) 

## 配置介绍

```bash
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
```

配置job

```bash
静态
动态发现...
```

k8s动态发现

```bash
 kubernetes_sd_configs:
      - role: endpoints
      
      
role:
service
endpoints          # name + ns + port name
pod                # metadata.label + pod port name
```



relabel_configs默认获取以下3个变量

	job=job_name
	__address__= target套接字host:port
	instance = __address__
可以执行的操作

```bash
			relabel_configs:
				source_labels 引用哪些已有标签: 将值连接起来
				separator     标签连接的
				target_label  引用源标签赋值给哪个新标签
				regex         对值匹配，匹配结果赋值给target_label
				modulus       指定hash模块
				replacement   keep时失效, 默认是$1, 即regex 第一个括号的内容
				action 
					替换标签值
						replace 匹配的值放在target_label
						hashmod 由modulus模块hash匹配的值放在target_label 
					删除指标
						keep   不匹配source_labels各标签串连值，删除target
						drop   能匹配source_labels各标签串连值，删除target
					创建或删除标签
						labelmap   匹配标签名，匹配到的标签的值赋值给replacement指定的标签名之上
						labelkeep  regex匹配标签名，不可以匹配，drop 标签
						labeldrop regex匹配标签名，可以匹配，drop 标签
						
# 添加标签
relabel_configs:
- replacement: test-value
  target_label: test-key
# 特殊：添加标签，将会替换target的地址和端口
relabel_configs:
- replacement: 1.1.1.1     
  target_label: __address__
# 保留标签, 联合source_labels的值，regex匹配成功则保留
relabel_configs:
- action: keep
  source_labels: []
  regex:
```

所以以上含义为：

regex匹配到的值，赋值给 replacement上，replacement默认是 regex匹配的$1, 

 

所以结果是， 

![img](http://myapp.img.mykernel.cn/wps13.jpg)

上面只会保留后面部分，所以网页看到的这个标签只有后半部分 

 ![img](http://myapp.img.mykernel.cn/wps12.jpg) 

 

## 现在ELK上有大量的/metrics请求
![img](http://myapp.img.mykernel.cn/wps16.jpg)

filebeat排除

exclude_lines: ['^DBG','kube-probe/1.20','Prometheus/2.25.0']



删除今天的nginx索引，重新请求

[root@elk prometheus-2.25.0.linux-amd64]# /usr/bin/curl -XDELETE http://127.0.0.1:9200/logstash-nginx-accesslog-2021.03.11 

{"acknowledged":true}

 

![img](http://myapp.img.mykernel.cn/wps17.jpg) 

![img](http://myapp.img.mykernel.cn/wps18.jpg) 

 

恢复



## coredns job  role: pod 基于组件 + 端口名

### 先简化配置

通过以下抓不同的角色，发现endpoints出现的结果比pod更少，所以使用pod更全面一些、

![image-20210315182257718](http://myapp.img.mykernel.cn/image-20210315182257718.png)

https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config

没有定义服务发现不了，所以使用pod发现



#### 抓endpoints

![img](http://myapp.img.mykernel.cn/wps19.jpg)

重载prometheus配置 

![img](http://myapp.img.mykernel.cn/wps20.jpg) 

![img](http://myapp.img.mykernel.cn/wps21.jpg) 



#### 抓pod

![img](http://myapp.img.mykernel.cn/wps22.jpg) 

![img](http://myapp.img.mykernel.cn/wps23.jpg) 

![img](http://myapp.img.mykernel.cn/wps24.jpg) 

### 定义指标标签

现在pod只能看Instance和job名，需要看pod 在哪个主机，哪个名称空间，什么pod名

![img](http://myapp.img.mykernel.cn/wps25.jpg) 

热重载

![image-20210315182424497](http://myapp.img.mykernel.cn/image-20210315182424497.png)

![image-20210315182540095](http://myapp.img.mykernel.cn/image-20210315182540095.png)

现在可以看出pod的基本信息了

node指标中会出现定义的label

现在查询的每个dns相关的指标都有这个自定义的Key

过滤dns的target

![image-20210315182504837](http://myapp.img.mykernel.cn/image-20210315182504837.png)

### 以dns的metadata.label过滤dns pod

#### 获取标签

![image-20210315182729114](http://myapp.img.mykernel.cn/image-20210315182729114.png)

![image-20210315182744007](http://myapp.img.mykernel.cn/image-20210315182744007.png)

![image-20210315182804600](http://myapp.img.mykernel.cn/image-20210315182804600.png)

#### 基于标签过滤

![image-20210315182855496](http://myapp.img.mykernel.cn/image-20210315182855496.png)

> 如果标签有 . / 全部会被替换成 _
>
> ![image-20210315182922809](http://myapp.img.mykernel.cn/image-20210315182922809.png)

热加载

![image-20210315183131690](http://myapp.img.mykernel.cn/image-20210315183131690.png)

现在已经出来了

![image-20210315183147967](http://myapp.img.mykernel.cn/image-20210315183147967.png)

并且可以看出replacement=$1是默认值，$1什么都不匹配所以为空

#### 基于端口过滤

以上可以看出是4个target, 实际上，我们只需要9153这个 

在定义时，port如下

![image-20210315183323873](http://myapp.img.mykernel.cn/image-20210315183323873.png)

对应网页上

![image-20210315183351052](http://myapp.img.mykernel.cn/image-20210315183351052.png)

查看这个metrics的端口的指标

![image-20210315183415305](http://myapp.img.mykernel.cn/image-20210315183415305.png)

![image-20210315183429120](http://myapp.img.mykernel.cn/image-20210315183429120.png)

![image-20210315183436249](http://myapp.img.mykernel.cn/image-20210315183436249.png)

##### 添加端口过滤

现在添加过滤这个容器端口名

![image-20210315184309294](http://myapp.img.mykernel.cn/image-20210315184309294.png)

![image-20210315184419625](http://myapp.img.mykernel.cn/image-20210315184419625.png)

![image-20210315184505266](http://myapp.img.mykernel.cn/image-20210315184505266.png)

## coredns job role: endpoints 基于服务名添加

![image-20210315184532544](http://myapp.img.mykernel.cn/image-20210315184532544.png)

![image-20210315184617602](http://myapp.img.mykernel.cn/image-20210315184617602.png)



## kubelet job role: pod

### 端口

![image-20210315184639521](http://myapp.img.mykernel.cn/image-20210315184639521.png)

### 监控配置

![image-20210315184656693](http://myapp.img.mykernel.cn/image-20210315184656693.png)

### pod中加载配置

![image-20210315184716744](http://myapp.img.mykernel.cn/image-20210315184716744.png)

### 热加载

![image-20210315184731958](http://myapp.img.mykernel.cn/image-20210315184731958.png)

### 查看target

![image-20210315184743516](http://myapp.img.mykernel.cn/image-20210315184743516.png)

找kubelet 10250, 并没有, 说明kubelet并没有输出这个



### node角色监控kubelet

![image-20210315185220648](http://myapp.img.mykernel.cn/image-20210315185220648.png)

![image-20210315185230818](http://myapp.img.mykernel.cn/image-20210315185230818.png)



## kube-controller-manager job role: pod

- 端口
- 协议
- 有服务？ 
  - 有：endpoint + ns + port name
- deployment
  - pod + metadata.label + port name
- 更新prometheus配置
- 热重载
- 查看prometheus target

### 二进制

![image-20210315185256507](http://myapp.img.mykernel.cn/image-20210315185256507.png)

### kubeadm

#### metadata.label

![image-20210315185408107](http://myapp.img.mykernel.cn/image-20210315185408107.png)

#### port

![image-20210315185420793](http://myapp.img.mykernel.cn/image-20210315185420793.png)

需要将manifests中的controller监听在0.0.0.0这样才可以获取指标

![image-20210315185430817](http://myapp.img.mykernel.cn/image-20210315185430817.png)

![image-20210315185448160](http://myapp.img.mykernel.cn/image-20210315185448160.png)

定义pod, 这样prometheus才可以自动发现这个端口。Role: pod

3个master一样配置，scp完成

![image-20210315185530265](http://myapp.img.mykernel.cn/image-20210315185530265.png)

#### 查看结果

省略热重载

![image-20210315185619508](http://myapp.img.mykernel.cn/image-20210315185619508.png)

## 监控scheduler role: pod

![image-20210315185653729](http://myapp.img.mykernel.cn/image-20210315185653729.png)

![image-20210315185702906](http://myapp.img.mykernel.cn/image-20210315185702906.png)

![image-20210315185714718](http://myapp.img.mykernel.cn/image-20210315185714718.png)

![image-20210315185722154](http://myapp.img.mykernel.cn/image-20210315185722154.png)

![image-20210315185732593](http://myapp.img.mykernel.cn/image-20210315185732593.png)

Prometheus pod中出现

![image-20210315185741993](http://myapp.img.mykernel.cn/image-20210315185741993.png)

热重载

![image-20210315185759186](http://myapp.img.mykernel.cn/image-20210315185759186.png)

结果

![image-20210315185809827](http://myapp.img.mykernel.cn/image-20210315185809827.png)

## node-exporter job （完整过程）

### 部署

```diff
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter 
  namespace: monitoring
#  labels:
#    k8s-app: node-exporter 
#    kubernetes.io/cluster-service: "true"
#    addonmanager.kubernetes.io/mode: Reconcile
#    version: v1.1.2
spec:
  updateStrategy:
    type: OnDelete
  selector:
    matchLabels:
      k8s-app: node-exporter 
      version: v1.1.2
  template:
    metadata:
      labels:
+        k8s-app: node-exporter        # 监控使用
        version: v1.1.2
#      annotations:
#        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        key: node.kubernetes.io/not-ready
        operator: Exists
      - effect: NoExecute
        key: node.kubernetes.io/unreachable
        operator: Exists
      - effect: NoSchedule
        key: node.kubernetes.io/disk-pressure
        operator: Exists
      - effect: NoSchedule
        key: node.kubernetes.io/memory-pressure
        operator: Exists
      - effect: NoSchedule
        key: node.kubernetes.io/pid-pressure
        operator: Exists
      - effect: NoSchedule
        key: node.kubernetes.io/unschedulable
        operator: Exists
      - effect: NoSchedule
        key: node.kubernetes.io/network-unavailable
        operator: Exists
      containers:
        - name: prometheus-node-exporter
          #image: "prom/node-exporter:v0.15.2"
          image: harbor.youwoyouqu.io/pub/node-exporter:v1.1.2
          imagePullPolicy: "IfNotPresent"
          args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
          ports:
            - name: metrics
              containerPort: 9100
              hostPort: 9100
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly:  true
            - name: sys
              mountPath: /host/sys
              readOnly: true
+          resources: # 避免太慢
            limits:
              cpu: 100m
              memory: 200Mi
+            requests:
              cpu: 100m
              memory: 200Mi
      hostNetwork: true
      hostPID: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys

```





### 定义发现步骤

- [x] 确定metrics的URL
- [x] k8s发现，有service,endpoint,pod
  - [x] service: 需要有service
  - [x] pod: 需要有metadata.label, metric端口名
- [x] 更新prometheus配置的configmap
- [x] kubelet会加载配置，更新到pod中，观察pod中的配置变化
- [x] 热加载配置
- [x] 查看网页prometheus变化



### 判断哪种发现

没有node-exporter服务，所以只能pod发现

![image-20210315185934647](http://myapp.img.mykernel.cn/image-20210315185934647.png)

### pod发现的元数据

获取label ：k8s-app=node-exporter

![image-20210315185958239](http://myapp.img.mykernel.cn/image-20210315185958239.png)

### 协议

确定协议：http

![image-20210315190047352](http://myapp.img.mykernel.cn/image-20210315190047352.png)

### metrics端口

添加支持metrics的端口:9100， 名为metrics

![image-20210315190115057](http://myapp.img.mykernel.cn/image-20210315190115057.png)

### 配置prometheus

因为grafana 的node-exporter的指标需要metrics_path的标签

```bash
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
    #由于后期可能有些节点的node-exporter在宿主机上，所以此处全部使用静态, 画图node-exporter读的是统一job_name.
    - job_name: 'kubernetes-node-exporter-daemonset-pod'
      static_configs:
      - targets: 
        - '192.168.0.85:9100'
        - '192.168.0.84:9100'
        - '192.168.0.83:9100'
        - '192.168.0.82:9100'
        - '192.168.0.81:9100'
        - '192.168.0.79:9100'
        - '192.168.0.171:9100'
        - '192.168.0.92:9100'
        - '192.168.0.27:9100'
        - '192.168.0.87:9100'
        - '192.168.0.91:9100'
        - '192.168.0.12:9100'
        - '192.168.0.78:9100'
```

> scheme：协议
>
> kubernetes_sd_configs 为k8s自动发现
>
> role: pod
>
> namespace: [] 任意名称空间
>
> relabel_configs, 对应默认3个标签进行重新打标
>
> 1. 过滤label
> 2. 过滤端口名
> 3. 添加node name
> 4. 添加ns
> 5. 添加pod name



### 更新配置

![image-20210315190155201](http://myapp.img.mykernel.cn/image-20210315190155201.png)

### kubelet加载到configmap

![image-20210315190210385](http://myapp.img.mykernel.cn/image-20210315190210385.png)

### 热重载配置

![image-20210315190222251](http://myapp.img.mykernel.cn/image-20210315190222251.png)

### 结果

![image-20210315190312664](http://myapp.img.mykernel.cn/image-20210315190312664.png)

### 查看指标

![image-20210315191016123](http://myapp.img.mykernel.cn/image-20210315191016123.png)

### 添加grafana图形

![image-20210315191035758](http://myapp.img.mykernel.cn/image-20210315191035758.png)

图中某些指标还依赖一个外部组件`kube-state-metrics`

#### 安装kube-state-metrics

https://github.com/kubernetes/kube-state-metrics

![image-20210315191404402](http://myapp.img.mykernel.cn/image-20210315191404402.png)

![image-20210315191414120](http://myapp.img.mykernel.cn/image-20210315191414120.png)

```bash
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
```



#### 修正grafana dashboard

导出

![image-20210315192318137](http://myapp.img.mykernel.cn/image-20210315192318137.png)

把job=的内容替换正常的，再导入



## 监控etcd

![image-20210315192406801](http://myapp.img.mykernel.cn/image-20210315192406801.png)

修正每个master的地址和暴露 metrics port

![image-20210315192429130](http://myapp.img.mykernel.cn/image-20210315192429130.png)

检验

![image-20210315192440393](http://myapp.img.mykernel.cn/image-20210315192440393.png)

```bash
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
```



grafana 3070



## 监控kube-proxy

```bash
kubectl edit cm -n kube-system kube-proxy
```

![image-20210315192601643](http://myapp.img.mykernel.cn/image-20210315192601643.png)

删除后，加载新配置

![image-20210315192628437](http://myapp.img.mykernel.cn/image-20210315192628437.png)

确保端口正常

![image-20210315192646025](http://myapp.img.mykernel.cn/image-20210315192646025.png)

![image-20210315192655877](http://myapp.img.mykernel.cn/image-20210315192655877.png)

![image-20210315192707390](http://myapp.img.mykernel.cn/image-20210315192707390.png)

## 监控es

https://grafana.com/grafana/dashboards/13562

 

下载模块，修改$database 为 ${DS_PROMETHEUS-SYS-01} 

 "timezone": "Asia/Shanghai",

 

 

 

https://www.datadoghq.com/blog/elasticsearch-unassigned-shards/



## 监控mysql

配置mysql exporter

```bash
MySQL [(none)]> GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost' IDENTIFIED BY 'exporter';

Query OK, 0 rows affected (0.00 sec)

MySQL [(none)]> GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'::1' IDENTIFIED BY 'exporter';

Query OK, 0 rows affected (0.00 sec)

 

 

 

[root@elk ~]# tar xvf mysqld_exporter-0.12.1.linux-amd64.tar.gz -C /usr/local

mysqld_exporter-0.12.1.linux-amd64/

mysqld_exporter-0.12.1.linux-amd64/NOTICE

mysqld_exporter-0.12.1.linux-amd64/mysqld_exporter

mysqld_exporter-0.12.1.linux-amd64/LICENSE

 

[root@elk mysqld_exporter-0.12.1.linux-amd64]# cat ~/.my.cnf

[client]

user=exporter

password=exporter

```



```bash
./mysqld_exporter  --no-collect.auto_increment.columns


```

http://192.168.0.171:9104/metrics 网页查看metric



配置prom

```bash
    - job_name: 'mysql' 
      static_configs:
      - targets: ['192.168.0.171:9104']
      relabel_configs:
      - target_label: address
        replacement: 192.168.0.171:3306 # 方便后期知道哪个位置的问题

```

https://grafana.com/grafana/dashboards/13106

## 监控redis

```bash
[root@elk ~]# tar xvf redis_exporter-v1.18.0.linux-amd64.tar.gz -C /usr/local
redis_exporter-v1.18.0.linux-amd64/
redis_exporter-v1.18.0.linux-amd64/redis_exporter
redis_exporter-v1.18.0.linux-amd64/LICENSE
redis_exporter-v1.18.0.linux-amd64/README.md
[root@elk ~]# cd /usr/local
[root@elk local]# cd redis_exporter-v1.18.0.linux-amd64/

[root@elk redis_exporter-v1.18.0.linux-amd64]# ./redis_exporter  -redis.password 
INFO[0000] Redis Metrics Exporter v1.18.0    build date: 2021-03-11-03:26:58    sha1: d0597c841d2c9fa30ce8b6ded6251d1994822e27    Go: go1.16.1    GOOS: linux    GOARCH: amd64 
INFO[0000] Providing metrics at :9121/metrics           
```

```bash
    - job_name: 'redis' 
      static_configs:
      - targets: ['192.168.0.171:9121']
      relabel_configs:
      - target_label: address
        replacement: 192.168.0.171:6379 # 方便后期知道哪个位置的问题
```

https://grafana.com/grafana/dashboards/11835



# 报警

Peding 表示for的周期未满，但是已经处理异常状态

等待5分钟之后,firing



## prometheus集成alertmanager

### 规则

```bash
  prometheus.yml: |
    global:
      scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
      evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
    # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
    rule_files:
    - "/etc/rules/*.yml"    # 由于deploy挂载configmap在此路径 
    - "/etc/node-exporter-rules/*.yml" # 加载node相关的通用报警  https://www.cnblogs.com/zhaojiedi1992/p/zhaojiedi_liunx_65_prometheus_alertmanager_rule.html
```

更多可以参考

https://awesome-prometheus-alerts.grep.to/rules

### 集成

```bash
apiVersion: v1
data:
  prometheus.yml: |
    global:
      scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
      evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
    # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
    rule_files:
    - "/etc/rules/*.yml"    # 由于deploy挂载configmap在此路径 
    - "/etc/node-exporter-rules/*.yml" # 加载node相关的通用报警  https://www.cnblogs.com/zhaojiedi1992/p/zhaojiedi_liunx_65_prometheus_alertmanager_rule.html
      # - "second_rules.yml"

    .....
    
    
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

## 配置alertmanager

```bash
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
      group_by: ['alertname', 'cluster']
      #在一个新的告警分组被创建后，需要等待至少 group_wait 时间来初始化通知，这种方式可以确保有足够的时间为同一分组收获多条告警，然后一起触发这条告警信息
      group_wait: 30s
      #在第 1 条告警发送后，等待group_interval时间来发送新的一组告警信息
      group_interval: 1m
      #如果某条告警信息已经发送成功，则等待repeat_interval时间重新发送他们。这里不启用这个功能~
      #repeat_interval: 5h
      #默认的receiver：如果某条告警没有被一个route匹配，则发送给默认的接收器
      receiver: default 
      #上面的所有属性都由所有子路由继承，并且可以在每个子路由上覆盖
      routes:
      - receiver: email
        group_wait: 10s      # 测试
        repeat_interval: 30s  # 测试
        match:
          severity: page   # prometheus定义的labels为这个  
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

