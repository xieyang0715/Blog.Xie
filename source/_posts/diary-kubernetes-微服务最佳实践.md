---
title: "kubernetes 微服务最佳实践"
date: 2021-05-24 16:10:40
tags:
- 个人日记
---



# 参考
<!--more-->

https://support.huaweicloud.com/bestpractice-cce/cce-bestpractice.pdf

http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/download/pdf/DNCSK1894097_zh-CN_intl_181113204317_public_41ff921ec2d58b349cb6ee4ddc4114ec.pdf

https://main.qcloudimg.com/raw/document/product/pdf/457_6786_cn.pdf

https://www.bookstack.cn/read/kubernetes-practice-guide/best-practice-service-ha.md

百度网盘离线：

```bash
链接：https://pan.baidu.com/s/1RtfWCiX3hDUR4Uu2_FXNlA 
提取码：j6n1 
复制这段内容后打开百度网盘手机App，操作更方便哦--来自百度网盘超级会员V1的分享
```



# yaml文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: myapp
    env: test
  name: myapp
  namespace: default
spec:
    affinity: 
      nodeAffinity: 
        requiredDuringSchedulingIgnoredDuringExecution: 
          nodeSelectorTerms: 
          - matchExpressions: 
          - key: "placement-set-uniq" 
    podAntiAffinity: 
      preferredDuringSchedulingIgnoredDuringExecution: 
      - weight: 100 
      podAffinityTerm: 
        labelSelector: 
          matchExpressions: 
          - key: app 
            operator: In 
            values: 
            - myapp 
        topologyKey: kubernetes.io/hostname
  containers:
  - env:
    - name: ENVIRONMENT
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.labels['env']
    - name: CPU_LIMIT
      valueFrom:
        resourceFieldRef:
          divisor: "0"
          resource: limits.cpu
    image: ikubernetes/myapp:v1
    imagePullPolicy: IfNotPresent


    name: deploy
    ports:
    - containerPort: 80
      name: http
      protocol: TCP
    readinessProbe:
      failureThreshold: 2
      httpGet:
        path: /
        port: 80
        scheme: HTTP
      initialDelaySeconds: 10
      periodSeconds: 3
      successThreshold: 1
      timeoutSeconds: 1
    livenessProbe:
      failureThreshold: 2
      httpGet:
        path: /
        port: 80
        scheme: HTTP
      initialDelaySeconds: 10
      periodSeconds: 3
      successThreshold: 1
      timeoutSeconds: 1
    lifecycle:
      preStop:
        exec:
          # Introduce a delay to the shutdown sequence to wait for the
          # pod eviction event to propagate. Then, gracefully shutdown
          # nginx.
          command:
          - "sh"
          - "-c"
    	  - "sleep 15"
    resources:
      limits:
        cpu: 100m
        memory: 50Mi
      requests:
        cpu: 100m
        memory: 50Mi
  restartPolicy: Always
  terminationGracePeriodSeconds: 90
  
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  creationTimestamp: null
  name: myapp
  namespace: default
spec:
  maxReplicas: 5
  minReplicas: 2
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: myapp
  targetCPUUtilizationPercentage: 80
```



# 区别不同的环境

应用程序可以读取pod中的环境变量，环境变量来自yaml定义的env。其中可以引用pod的标签

```yaml
metadata:
  labels:
    app: deploy-selector
    env: test
  
...
  - env:
    - name: ENVIRONMENT
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.labels['env']
```

容器中环境变量将存在`ENVIRONMENT=test`

# nginx的work数量

由于默认pod启动后，pod中使用top命令看到的cpu核心 数是和系统一样的数量，而nginx配置文件中使用worker进程数配置为auto时，就会导致nginx启动进程数是系统的cpu核心数量。

而我们在limit中限制的可能才10m。会启动过多的进程。

将yaml中定义的limit.cpu限制读取，divisor 0表示不足1核心，自动进位为1.

```yaml
    - name: CPU_LIMIT
      valueFrom:
        resourceFieldRef:
          divisor: "0"
          resource: limits.cpu
```

然后启动nginx, 不直接使用 `nginx -g 'daemon off;'`, 而是使用一个脚本， 先读环境变量，修正nginx配置的核心数，然后再进行启动nginx.

# 使用 preStopHook 和 readinessProbe 保证服务平滑更新不中断

升级服务的操作

```bash
# 手工更新yaml的镜像名
sed -i 's@image: myapp:v1@image: myapp:v2@g' deploy.yaml
kubectl apply -f  deploy.yaml

# set image
kubectl set image deploy myapp=myapp:v2
```

升级之后，出现异常的原因：1、pod一更新就会创建新的pod, 新的pod创建完了，pod中的应用还没有完全启动，就加入service。kube-proxy watch到更新，就更新节点的iptables/ipvs规则，有请求到达这个还未完全就绪的pod, pod还不能处理请求，导致连接拒绝。2、pod被销毁，从endpoint从service中移除，但是kube-proxy更新节点的iptables/ipvs规则还没有更新完，pod就销毁，导致仍有流量 到达这个pod, 导致连接拒绝。

平滑更新：针对情况1， pod的container添加readiniess probe, 只有探测到http/port正常，才接入service. 针对情况2，pod 的container添加prestop hook, 销毁前sleep等待 kube-proxy更新完iptables规则，这段时间pod牌terminating状态，即便在转发规则更新完全之前有请求被转发到 Terminating 的 Pod，依然可以被正常处理，因为 Pod 还在 sleep 没有被真正销毁。

```yaml
    readinessProbe:
      failureThreshold: 2
      httpGet:
        path: /
        port: 80
        scheme: HTTP
      initialDelaySeconds: 10
      periodSeconds: 3
      successThreshold: 1
      timeoutSeconds: 1
    lifecycle:
      preStop:
        exec:
          # Introduce a delay to the shutdown sequence to wait for the
          # pod eviction event to propagate. Then, gracefully shutdown
          # nginx.
          command:
          - "sh"
          - "-c"
    	  - "sleep 20"
```

# 拉镜像的账户

避免每个yaml都配置一次镜像拉取的账号，可以直接在名称空间上的默认serviceaccount上配置账户

```bash
    image: ikubernetes/myapp:v1
    imagePullPolicy: IfNotPresent
```

```bash
root@ccea447f1caa:~# kubectl get sa default -o yaml
apiVersion: v1
imagePullSecrets:
- name: <你的拉镜像的账户1 docker registry>
- name: <你的拉镜像的账户2 docker registry>
- name: <你的拉镜像的账户3 docker registry>
kind: ServiceAccount
metadata:
  name: default
  namespace: default
```

配置私有registry账号的方法有3种. 1. 直接指定imagepullsecret. 2. 指定serviceaccount, sa里面加载secret, 3.直接在default的sa上添加secret.

你镜像所在的名称空间的default的账户,配置一个imagepullsecret, 然后你镜像启动多个这个名称空间中的deploy都不需要配置imagepullsecret

# 容器自动恢复

容器运行过程中，可能应用内部故障，但是进程没有挂，就是不能给用户提供服务了。所以需要探测应用是否存活，如果不存活，就是失败了。使用restartPolicy来决定pod怎么操作。默认Always，重启。

```yaml
    livenessProbe:
      failureThreshold: 2
      httpGet:
        path: /
        port: 80
        scheme: HTTP
      initialDelaySeconds: 10
      periodSeconds: 3
      successThreshold: 1
      timeoutSeconds: 1
      
  restartPolicy: Always
```

# 定义容器端口

此端口定义，一般没有啥用 ，prometheus可以依据此端口来输出/metrics指标。istio可以依据此端口来转发流量 。

```yaml
ports:
- containerPort: 80
  name: http
  protocol: TCP
```

# 资源请求限制

## request值含义

Request 的值，与pod无关。此值用于提供给kube-scheduler组件，此组件会拿着此值与每个节点的可用资源对比，只有节点可用资源大于此request资源，此节点才可能被scheduler认为此pod可以运行在其上。

## 提高资源利用率

如果**reqeust值过高**，而实际占用的资源远小于设定值，会导致节点整体的资源利用率较低。**除**对时延非常敏感的业务外，敏感的业务本身并不期望节点利用率过高，影响网络包收发速度。

>  建议对非核心，并且资源非长期占用的应用，适当减少 request 以提高资源利用率。若您的服务支持水平扩容，则除 CPU 密集型应用外，单副本的 request 值通常可设置为不大于1核。例如，coredns 设置为0.1核，即100m即可。

## 避免request与limit值过大

若您的服务使用单副本或少量副本，且 request 及 limit 的值设置过大，这样使服务可分配到足够多的资源去支撑业务。但是一旦某个副本发生故障时，可能会给业务带来较大影响。

当 Pod 所在节点发生故障时，由于 request 值过大，且集群内资源分配的较为碎片化，其余节点无足够可分配资源满足该 Pod 的 request，则该 Pod 无法实现漂移，无法自愈，会加重对业务的影响。

> 建议尽量减小 request 及 limit，通过增加副本的方式对您的服务支撑能力进行水平扩容，使系统更加灵活可靠。

## 测试和生产同集群

测试 namespace 消耗过多资源，如不加以限制，则可能导致集群负载过高，影响生产业务。可以使用ResourceQuota 限制测试 namespace 的 request 与 limit 的总大小。示例如下：

```yaml
apiVersion: v1 
kind: ResourceQuota 
metadata: 
  name: quota-test 
  namespace: test 
spec: 
  hard: 
    requests.cpu: "1" 
    requests.memory: 1Gi 
    limits.cpu: "2" 
    limits.memory: 2Gi
```

## 节点驱逐优先级

节点资源不足时，会触发自动驱逐，删除低优先级的 Pod 以释放资源使节点自愈。Pod 优先级由低到高排序如下：

1. 未设置 request 及 limit 的 Pod。
2. 设置 request 值不等于 limit 值的 Pod。
3.  设置 request 值等于 limit 值的 Pod。

建议重要线上应用设置 request 值等于 limit 值，此类 Pod 优先级较高，在节点故障时不易被驱逐导致线上业务受到影响。

```yaml
    resources:
      limits:
        cpu: 100m
        memory: 50Mi
      requests:
        cpu: 100m
        memory: 50Mi
```

## 实践resources

通过观察nginx-controller的资源分配, 并不会限制nginx的资源，说明只需要节点可以运行即可。

limit 2核足够承载400万并发。

```yaml
    resources:
      limits:
        cpu: 2
        memory: 2Gi
      requests:
        cpu: 100m
        memory: 70Mi
```

而后端的应用(java, python)必须限制，避免程序不合理过度使用内存。 调试方法

**重要的应用。**

前期，固定replicas=2, 保证高可用，并且request, limit 至少2核心，2Gi内存。

经过一段时间的观察。

deployment占用的资源（每个pod最高使用cpu/ram之和，一般看deploy更准确），例如最高看cpu才300m, ram才750Mi, 所以后期，固定请求和限制1核心，1Gi内存，HPA配置min=2,max=5,CPU达到80.



### 测试环境

nginx

```yaml
    resources:
      limits:
        cpu: 2
        memory: 2Gi
      requests:
        cpu: 100m
        memory: 70Mi
```

后端, 因为在观察过程中发现最高才使用300m, 750Mi，所以请求低：可以被调度，限制高，让测试人员使用时不卡。

```yaml
    resources:
      limits:
        cpu: 1
        memory: 1Gi
      requests:
        cpu: 200m
        memory: 256Mi
```

### 生产环境

nginx

```yaml
    resources:
      limits:
        cpu: 2
        memory: 2Gi
      requests:
        cpu: 100m
        memory: 70Mi
```

后端, 请求和限制相同，避免被OOM. 请求1核心也低，因为在最初上线时，给的是5核心，10Gi内存。

```yaml
    resources:
      limits:
        cpu: 1
        memory: 1Gi
      requests:
        cpu: 1
        memory: 1Gi
```





# 资源合理分配

## Affinity

- 对节点有特殊要求的服务可使用节点亲和性（Node Affinity）部署，以便调度到符合要求的节点。例如，让 MySQL 调度到高 IO 的机型以提升数据读写效率。
- 需进行关联的服务可使用节点亲和性（Pod Affinity）部署。例如，让 Web 服务与其 Redis 缓存服务都部署在同一可用区，可实现低延时。
- 可使用节点亲和性（Pod antiAffinity）将 Pod 进行打散调度，避免单点故障或流量过于集中导致的一些问题。

## 使用污点（taint)与容忍(toleration)

- 通过给节点打污点，来给某些应用预留资源，避免其他 Pod 调度到此节点。
- 需使用预留资源的 Pod 加上容忍，结合节点亲和性使 Pod 调度到预留节点，即可使用预留资源。

# 流量突发型业务

通常业务会有高峰和低谷，为了更合理的利用资源，可为服务定义 HPA，实现根据 Pod 的资源实际使用情况来对服务进行自动扩缩容。在业务高峰期时自动扩容 Pod 数量来支撑服务，在业务低谷时自动缩容 Pod 释放资源，以供其他服务使用。例如，夜间线上业务低峰，自动缩容释放资源以供大数据类的离线任务运行。

使用 HPA 前，需安装 resource metrics（metrics.k8s.io）或 custom metrics（custom.metrics.k8s.io），使hpa controller 通过查询相关 API 获取到服务资源的占用情况，即 K8s 先获取服务的实际资源占用情况（指标数据）。 早期 HPA 使用 resource metrics 获取指标数据，后推出的 custom metrics 可通过更灵活的指标来控制扩缩容。Kubernetes 官方相关实现为 metrics-server，而社区通常使用基于 prometheus 的 实现 prometheus-adapter，云厂商托管的 Kubernetes 集群通常集成自身的实现。例如容器服务，实现了 CPU、内存、硬盘、网络等维度的指标，可在网页端可视化创建 HPA，最终转化为 Kubernetes 的 yaml。示例如下：

```bash
kubectl autoscale deploy myapp --min=2 --max=5 --cpupercentage=80
# min 最小副本
# max 最大副本
# cpupercentage cpu达到80%自动扩容

```

HPA 能够实现 Pod 水平扩缩容，但如果节点资源不足，则扩容出的 Pod 状态仍会 Pending。如果提前准备好大量节点，使资源冗余，即使不会发生 Pod Pending 问题，但成本可能过高。

通常云厂商托管的 Kubernetes 集群均会实现 cluster-autoscaler，即根据资源使用情况，动态增删节点，使计算资源能够被最大化弹性使用，并通过按量计费的计费模式节约成本。例如，容器服务中的伸缩组，及包含伸缩组功能的拓展特性（节点池）。

对于无法适配水平伸缩的单体应用，或不确定最佳 request 与 limit 超卖比的应用，可尝试使用 VPA 来进行垂直伸缩。即自动更新 request 与 limit，并重启 pod。该特性容易导致服务出现短暂的不可用，不建议在生产环境中大规模使用。

# 应用高可用性

replicas =1 , 必然单点。 replicas>1但所有副本都调度到同一个节点，仍将无法避免单点故障。

为了避免单点故障，我们需要有合理的副本数量，还需要让不同副本调度到不同的节点。可以利用反亲和性来实现

```yaml
# 首先准备应用的yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - args:
        - myapp
        image: ikubernetes/myapp:v1
        name: deploy


# 注意：应用的标签是app=nginx.


# 然后我们把nginx部署在指定一批节点上。 以下表示：node标签的key为placement-set-uniq，的所有节点均可以调度
affinity: 
  nodeAffinity: 
    requiredDuringSchedulingIgnoredDuringExecution: 
      nodeSelectorTerms: 
      - matchExpressions: 
      - key: "placement-set-uniq" 
      

# 然后把nginx与每个部署的nginx 强反亲和 
podAntiAffinity: 
  preferredDuringSchedulingIgnoredDuringExecution: 
  - weight: 100 
  podAffinityTerm: 
    labelSelector: 
      matchExpressions: 
      - key: app 
        operator: In 
        values: 
        - nginx 
    topologyKey: kubernetes.io/hostname
```

> topologKey, 表示 pod可以在 节点有 kubernetes.io/hostname标签的任何节点上运行。
>
> app=nginx, 表示pod 的标签为app=nginx的pod所在的节点，podAntiAffinity表示反亲和，表示pod不能运行在拥有app=nginx的pod标签所在节点上。

详细参考：http://blog.mykernel.cn/2021/02/02/Pod%E8%B5%84%E6%BA%90%E8%B0%83%E5%BA%A6/#pod-Antiaffinity

