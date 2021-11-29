---
title: Pod控制器
date: 2021-01-18 09:09:33
tags:
---



controller作用：创建pod活动对象，和解循环：status -> spec(desired state), 让pod当前状态无限接近或吻合目标状态。

kube-controller-manager k8s组件，打包了k8s 40-50个控制器的守护进程为一个。所有控制器都被此组件管理。控制器不正常，由此组件恢复。master节点需要高可用，来高可用`controller-manager`.  所有的控制器都可以保证不宕机，也保证了所有Pod不宕机。



控制器操作: 创建、删除、修改

- [x] 创建Pod 
- [x] 和解循环，status -> spec
  - [x] 健康状态
- [x] 扩容或缩容Pod



<!--more-->



# controller类型

控制Pod

| 控制器类型  | 实例化                              |
| ----------- | ----------------------------------- |
| Replicaset  | 对象: nginx-rs, myapp-rs, tomcat-rs |
| deploy      |                                     |
| daemonset   |                                     |
| statefulset |                                     |
| job         |                                     |
| cronjob     |                                     |

其他控制器

| 控制器类型      | 描述                                     |
| --------------- | ---------------------------------------- |
| namespace       | 名称空间                                 |
| bootstrapsigner |                                          |
| endpoint        | 端点                                     |
| service         | 服务                                     |
| serviceaccount  | 账户                                     |
| node            | 管理节点，节点变动时当前状态吻合目标状态 |



kube-controller-manager 守护进程启动时，--controllers的值

```bash
--controllers
	*    启用所有控制器，有2-3控制器不包含。 bootstrapsigner（引导创建集群）, tokencleaner(令牌清理器). kubeadm部署也未必包含
	foo  启用指定控制器
	-foo 禁用指定控制器
```



kubeadm部署的k8s, 配置pod中的进程

```bash
[root@master chapter4]# kubectl get pods -n kube-system
NAME                                        READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-k6zwb                     1/1     Running   0          3d23h
coredns-74ff55c5b-q4qrv                     1/1     Running   0          3d23h
etcd-master.magedu.com                      1/1     Running   0          3d23h
kube-apiserver-master.magedu.com            1/1     Running   4          3d23h
kube-controller-manager-master.magedu.com   1/1     Running   4          3d23h
kube-flannel-ds-2s6p8                       1/1     Running   0          3d22h
kube-flannel-ds-4l4hp                       1/1     Running   0          3d22h
kube-flannel-ds-qqrw2                       1/1     Running   0          3d22h
kube-flannel-ds-vbw8m                       1/1     Running   0          3d22h
kube-proxy-fmj6w                            1/1     Running   0          3d23h
kube-proxy-j8vkr                            1/1     Running   0          3d22h
kube-proxy-ldvj9                            1/1     Running   0          3d22h
kube-proxy-nxcsj                            1/1     Running   0          3d22h
kube-scheduler-master.magedu.com            1/1     Running   4          3d23h

[root@master chapter4]# cd /etc/kubernetes/manifests/
[root@master manifests]# ls
	# 这些就是创建kube-system名称空间的k8s核心pod使用的资源清单
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml


[root@master manifests]# cat kube-controller-manager.yaml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-controller-manager # 守护进程
    - --allocate-node-cidrs=true
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --bind-address=127.0.0.1
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --cluster-cidr=10.244.0.0/16
    - --cluster-name=kubernetes
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --controllers=*,bootstrapsigner,tokencleaner # 所有控制器和...，禁用前面加-
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --leader-elect=true
    - --port=0
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --use-service-account-credentials=true

# 修改完之后，k8s会自动重建新pod
```



# Pod Controller

用户不应该手工创建pod, 而是pod controller创建pod

这是一个统称，实际上这些, 应用程序划分多种类型

- 守护进程：后面请求与前面请求有关联。
  - 有状态：个体可以被单独对待。**statefulset**, 例如mysql、redis集群一个控制器不可以通用。
  - 无状态：个体可以被任意取代。
    - 非系统级：**replicaset, deployment**
    - 系统级：集群中每个节点运行一份：**daemonset**,  例如：收集每个节点上的docker日志。
- 非守护进程
  - job
  - cronjob

| 控制器类型  | 实例化                           |
| ----------- | -------------------------------- |
| Replicaset  | 无状态、守护进程：nginx/tomcat   |
| deploy      | 无状态、守护进程：nginx/tomcat   |
| daemonset   |                                  |
| statefulset | 有状态：kv/messagequeue/服务总线 |
| job         |                                  |
| cronjob     |                                  |

# replicaset

- [x] pod副本数量足够: replicas
- [x] 标签找pod:   pod selector
- [x] pod模板创建pod: pod Template

```bash
# 查看 
kubectl get rs

# 
[root@master manifests]# kubectl explain rs.spec | grep '>'
RESOURCE: spec <Object>
   minReadySeconds	<integer>
   replicas	<integer> # 副本
   selector	<Object> -required- # 标签
       matchExpressions	<[]Object>       # 表达式 [{key: ,operator:, values:}]
           key	<string> -required-
           operator	<string> -required-
           values	<[]string>
       matchLabels	<map[string]string>  # k/v。多个与

   template	<Object>  # 模板
       metadata	<Object> 
       spec	<Object>  # pod的spec
```

```bash
[root@master ~]# cd Kubernetes_Advanced_Practical/chapter5
[root@master chapter5]# cat rs-example.yaml 
apiVersion: apps/v1   # 群组，不同的组，定义格式不一样
kind: ReplicaSet      # 大小写
metadata:
  name: myapp-rs      # 名字
  namespace: prod
spec:
  replicas: 2         # 期望副本
  selector:
     matchLabels:     # 匹配app=myapp-pod的pod
       app: myapp-pod
  template:
    metadata:
      labels:         # 模板的pod定义标签，要和上面选择器一样，才能匹配到pod.
        app: myapp-pod
    spec:             # 自主式pod的定义格式
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        ports:
        - name: http
          containerPort: 80


[root@master chapter5]# kubectl apply -f rs-example.yaml
replicaset.apps/myapp-rs created
[root@master chapter5]# kubectl get rs -n prod
NAME       DESIRED   CURRENT   READY   AGE
名称        期望        当前     就绪     运行时长
myapp-rs   2         2         2       5s

[root@master chapter5]# kubectl describe rs -n prod myapp-rs
Name:         myapp-rs
Namespace:    prod
Selector:     app=myapp-pod 此选择器选择的pod
Labels:       <none>
Annotations:  <none>
Replicas:     2 current / 2 desired
Pods Status:  2 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=myapp-pod
  Containers:
   myapp:
    Image:        ikubernetes/myapp:v1
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  # 创建2个        													 rs名 - 随机字符
  Normal  SuccessfulCreate  37s   replicaset-controller  Created pod: myapp-rs-t6cnj
  Normal  SuccessfulCreate  37s   replicaset-controller  Created pod: myapp-rs-d94h2

```

如果已经存在5个pod有这个标签，就会导致多出的pod会被kill. 

所以在同一个名称空间有多个控制器，定义的pod的标签，使用单一的标签选择器很容易互相有关联关系。最好多定义标签，选择时也使用多个条件。

举例，人为创建一个pod, 刚好一个标签和以上的标签一致，就会影响rs运行

```bash
kubectl label pod -n prod pod-demo app=myapp-pod --overwrite
# 一个新建的pod会影响原来的pod
[root@master basic]# kubectl get pod -n prod 
NAME             READY   STATUS        RESTARTS   AGE
myapp-rs-d94h2   0/1     Terminating   0          12m
myapp-rs-t6cnj   1/1     Running       0          12m
pod-demo         2/2     Running       0          7h48m
```



## 扩容/缩容(变更)

### 直接修改yaml的replicas字段

```diff
[root@master chapter5]# cat rs-example.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
  namespace: prod
spec:
+  replicas: 5
  selector:
     matchLabels:
       app: myapp-pod
  template:
    metadata:
      labels:
        app: myapp-pod
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        ports:
        - name: http
          containerPort: 80

```

```bash
[root@master chapter5]# kubectl apply -f rs-example.yaml 
kubectl get poreplicaset.apps/myapp-rs configured
service/myapp-svc unchanged

# 补充3个
[root@master chapter5]# kubectl get pod -n prod
NAME             READY   STATUS              RESTARTS   AGE
myapp-rs-4l89n   0/1     ContainerCreating   0          1s
myapp-rs-nhkvs   0/1     ContainerCreating   0          1s
myapp-rs-qlgv9   0/1     ContainerCreating   0          2s
myapp-rs-t6cnj   1/1     Running             0          15m
pod-demo         2/2     Running             0          7h50m
```



### scale

```bash
[root@master chapter5]# kubectl scale --replicas=3 rs myapp-rs -n prod
replicaset.apps/myapp-rs scaled


[root@master chapter5]# kubectl get pods -n prod -o wide
NAME             READY   STATUS        RESTARTS   AGE     IP           NODE                NOMINATED NODE   READINESS GATES
myapp-rs-nhkvs   0/1     Terminating   0          3m28s   10.244.1.7   node01.magedu.com   <none>           <none>
myapp-rs-qlgv9   1/1     Running       0          3m29s   10.244.3.8   node03.magedu.com   <none>           <none>
myapp-rs-t6cnj   1/1     Running       0          18m     10.244.2.7   node02.magedu.com   <none>           <none>
pod-demo         2/2     Running       0          7h54m   10.244.3.6   node03.magedu.com   <none>           <none>
```

## 发布

- [x] 灰度
- [x] 蓝绿
- [x] 金丝雀

myapp-rs是v1版本，更新至v2

### 修改yaml

```diff
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
  namespace: prod
spec:
  replicas: 5
  selector:
     matchLabels:
       app: myapp-pod
  template:
    metadata:
      labels:
        app: myapp-pod
    spec:
      containers:
      - name: myapp
+        image: ikubernetes/myapp:v2
        ports:
        - name: http
          containerPort: 80

```

```bash
kubectl apply -f rs-example.yaml
```

```bash
[root@master chapter5]# kubectl describe pod -n prod myapp-rs-npmms
发现有些pod版本是v1，有些是v2
```

一会删除一个, 固定一段时间删除一个，滚动发布。

删除后，新建需要时间。你删除一个，剩下5个可能承担不了这么大的负载。

### set image



# deployment

- [x] replicaset存在是管理pod
- [x] deployment来管理replicaset, 帮我们完成程序**滚动更新**策略。
- [x] **金丝雀发布**，让新更新出现pod接入部分用户流量。
  - [ ] 路由哪部分流量，使用第三方组件完成，`istio`，服务网桥。
- [x] deployment 支持10个版本的回滚，回滚也是灰度的。

名称：myapp-rs的哈希码-pod名随机



```diff
[root@master chapter5]# cat deploy-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
+  name: myapp-deploy
+  namespace: prod
spec:
  replicas: 3
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      # 同时为0，不更新
      # 可以多1个，可以不少
      # 可以同时多1个，可以少1个
      # 可以多0个，少1个。
      maxSurge: 1        # 可以超出用户期望pod
+      maxUnavailable: 0 # 最大不可用
    type: RollingUpdate # 滚动更新
  selector:
    matchLabels:
+      app: myapp
+      rel: stable 
  template:
    metadata:
      labels:
+        app: myapp
+        rel: stable 
    spec:
      containers:
      - name: nginx
+        image: ikubernetes/myapp:v1
        ports:
        - containerPort: 80
          name: http
        readinessProbe:
          periodSeconds: 1
          httpGet:
            path: /
            port: http

```

提交 

```bash
[root@master chapter5]# cat deploy-nginx.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy
  namespace: prod
spec:
  replicas: 3
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  selector:
    matchLabels:
      app: myapp
      rel: stable
  template:
    metadata:
      labels:
        app: myapp
        rel: stable
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        ports:
        - containerPort: 80
          name: http
        readinessProbe:
          periodSeconds: 1
          httpGet:
            path: /
            port: http

kubectl apply -f deploy-nginx.yaml 

[root@master chapter5]# kubectl get pod -n prod
NAME                            READY   STATUS    RESTARTS   AGE
myapp-deploy-85d8db5944-8248j   1/1     Running   0          9s
myapp-deploy-85d8db5944-lgpr6   1/1     Running   0          9s
myapp-deploy-85d8db5944-pkj7f   1/1     Running   0          9s

[root@master chapter5]# kubectl get deploy -n prod
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deploy   3/3     3            3           2m2s

[root@master chapter5]# kubectl get rs -n prod
NAME                      DESIRED   CURRENT   READY   AGE
			  # rs模板 hash值
myapp-deploy-85d8db5944   3         3         3       2m11s

```

> pod是rs管理，不是deploy管理

## 发布

### 修改yaml

yaml修改镜像，apply.

一下全部更新

```bash
# 维护2个版本的rs
[root@master chapter5]# kubectl get rs -n prod -o wide
NAME                      DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES                 SELECTOR
myapp-deploy-85d8db5944   2         2         2       5m46s   myapp        ikubernetes/myapp:v1   app=myapp,pod-template-hash=85d8db5944,rel=stable
myapp-deploy-b9f79945c    2         2         2       34s     myapp        ikubernetes/myapp:v2   app=myapp,pod-template-hash=b9f79945c,rel=stable
myapp-rs                  5         5         5       41m     myapp        ikubernetes/myapp:v2   app=myapp-pod
```



### set-image

逐个更新



系统级任务不需要在节点上运行多个，deploy不能规定在哪个节点上运行多份。期望只在一个节点上仅运行一个Pod。



# DaemonSets

- [x] pod模板，不需要replicas, 因为仅一个pod
- [x] 标签选择器

添加守护进程仅在一个节点上运行一个pod。而且在有限个节点上(带ssd)运行一个pod(监控ssd)。`daemonset + 节点选择器`

## 功能

滚动更新的惟一策略：先减一个pod. 后添加一个pod



```bash
[root@master chapter5]# pwd
/root/Kubernetes_Advanced_Practical/chapter5
[root@master chapter5]# cat filebeat-ds.yaml 

```

```yaml
apiVersion: apps/v1          # group/version
kind: DaemonSet             
metadata:
  name: filebeat-ds
+  namespace: prod
  labels:                    # 控制器的标签, 非必须
    app: filebeat
spec:
  selector: # 选择器, 判定集群节点上有我们期望的pod
    matchLabels:
      app: filebeat
  template: # 创建pod的模板
    metadata:
      labels: # pod的标签
        app: filebeat
      name: filebeat
    spec:
      containers:
      - name: filebeat # pod容器名
        image: ikubernetes/filebeat:5.6.5-alpine # 镜像
        env: # 环境变量, 云原生使用变量。 非云原生使用entrypoint脚本处理
        - name: REDIS_HOST  # 变量名
          value: db.ikubernetes.io:6379      # 值
        - name: LOG_LEVEL
          value: info
```

```bash
[root@master chapter5]# kubectl apply -f filebeat-ds.yaml
kubedaemonset.apps/filebeat-ds created
[root@master chapter5]# kubectl get pod -n prod
NAME                           READY   STATUS              RESTARTS   AGE
filebeat-ds-njrtf              0/1     ContainerCreating   0          3s
filebeat-ds-sqnm7              0/1     ContainerCreating   0          3s
filebeat-ds-sx4lt              0/1     ContainerCreating   0          3s
[root@master chapter5]# kubectl get pod -n prod --show-labels
NAME                           READY   STATUS              RESTARTS   AGE   LABELS
filebeat-ds-njrtf              0/1     ContainerCreating   0          22s   app=filebeat,controller-revision-hash=775b7d8456,pod-template-generation=1
filebeat-ds-sqnm7              1/1     Running             0          22s   app=filebeat,controller-revision-hash=775b7d8456,pod-template-generation=1
filebeat-ds-sx4lt              0/1     ContainerCreating   0          22s   app=filebeat,controller-revision-hash=775b7d8456,pod-template-generation=1

[root@master chapter5]# kubectl get pod -n prod -l app=filebeat
NAME                READY   STATUS    RESTARTS   AGE
filebeat-ds-njrtf   1/1     Running   0          42s
filebeat-ds-sqnm7   1/1     Running   0          42s
filebeat-ds-sx4lt   1/1     Running   0          42s
[root@master chapter5]# kubectl get pod -n prod -l app=filebeat -o wide
NAME                READY   STATUS    RESTARTS   AGE   IP            NODE                NOMINATED NODE   READINESS GATES
																     # 没有master, master有污点。pod不容忍污点就不能调度。
filebeat-ds-njrtf   1/1     Running   0          48s   10.244.1.11   node01.magedu.com   <none>           <none>
filebeat-ds-sqnm7   1/1     Running   0          48s   10.244.3.12   node03.magedu.com   <none>           <none>
filebeat-ds-sx4lt   1/1     Running   0          48s   10.244.2.11   node02.magedu.com   <none>           <none>

```

查看污点 

```bash
[root@master chapter5]# kubectl get node
NAME                STATUS   ROLES                  AGE   VERSION
master.magedu.com   Ready    control-plane,master   12d   v1.20.2
node01.magedu.com   Ready    <none>                 12d   v1.20.2
node02.magedu.com   Ready    <none>                 12d   v1.20.2
node03.magedu.com   Ready    <none>                 12d   v1.20.2
[root@master chapter5]# kubectl describe node/master.magedu.com
					# 污点node-role.kubernetes.io/master  效果：不容忍这个污点怎么办？
Taints:             node-role.kubernetes.io/master:NoSchedule       
```

## 滚动更新

```bash
[root@master chapter5]# kubectl explain ds.spec.updateStrategy | grep '>'
RESOURCE: updateStrategy <Object>
   rollingUpdate	<Object>
   type	<string>
[root@master chapter5]# kubectl explain ds.spec.updateStrategy.rollingUpdate | grep '>'
RESOURCE: rollingUpdate <Object>
   maxUnavailable	<string>       # 默认1
   # 没有maxavailable, 只能减，不能加
```

更新镜像至5.6.6

dockerhub.com查找ikubernetes/filebeat镜像

![image-20210127093358632](http://myapp.img.mykernel.cn/image-20210127093358632.png)

```bash
#kubectl set image -h

# 获取ds名 和容器名
[root@master ~]# kubectl get ds -n prod -o wide
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE     CONTAINERS   IMAGES                              SELECTOR
filebeat-ds   3         3         2       2            2           <none>          9m37s   filebeat     ikubernetes/filebeat:5.6.6-alpine   app=filebeat

# 更新命令
[root@master chapter5]# kubectl set image ds/filebeat-ds -n prod filebeat=ikubernetes/filebeat:5.6.6-alpine
daemonset.apps/filebeat-ds image updated

# 更新过程
[root@master chapter5]# kubectl get pod -n prod -w
filebeat-ds-jgfz2              1/1     Running             0          20s # running之后，下一个容器是先终止后启动
filebeat-ds-njrtf              1/1     Terminating         0          9m27s
filebeat-ds-njrtf              0/1     Terminating         0          9m28s
filebeat-ds-njrtf              0/1     Terminating         0          9m37s
filebeat-ds-njrtf              0/1     Terminating         0          9m37s
filebeat-ds-gnwn8              0/1     Pending             0          0s
filebeat-ds-gnwn8              0/1     Pending             0          0s
filebeat-ds-gnwn8              0/1     ContainerCreating   0          0s
filebeat-ds-gnwn8              1/1     Running             0          16s
filebeat-ds-sx4lt              1/1     Terminating         0          9m53s
filebeat-ds-sx4lt              0/1     Terminating         0          9m54s

# 查看新版本
[root@master chapter5]# kubectl get pod -n prod -l app=filebeat
NAME                READY   STATUS    RESTARTS   AGE
filebeat-ds-gnwn8   1/1     Running   0          90s
filebeat-ds-jgfz2   1/1     Running   0          2m
filebeat-ds-kffb8   1/1     Running   0          40s
[root@master chapter5]# kubectl get ds -n prod -o wide
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE   CONTAINERS   IMAGES                              SELECTOR
filebeat-ds   3         3         3       3            3           <none>          11m   filebeat     ikubernetes/filebeat:5.6.6-alpine   app=filebeat

# kubectl delete ds filebeat-ds -n prod

```

## 限制pod仅能运行在哪些节点

```bash
[root@master chapter5]# kubectl get node --show-labels
NAME                STATUS   ROLES                  AGE   VERSION   LABELS
master.magedu.com   Ready    control-plane,master   12d   v1.20.2   	
beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master.magedu.com,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=
node01.magedu.com   Ready    <none>                 12d   v1.20.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node01.magedu.com,kubernetes.io/os=linux
node02.magedu.com   Ready    <none>                 12d   v1.20.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node02.magedu.com,kubernetes.io/os=linux
node03.magedu.com   Ready    <none>                 12d   v1.20.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node03.magedu.com,kubernetes.io/os=linux

```

> beta.kubernetes.io/arch=amd64, 架构
>
> beta.kubernetes.io/os=linux, 系统
>
> kubernetes.io/arch=amd64,
>
> kubernetes.io/hostname=master.magedu.com, 主机名
>
> kubernetes.io/os=linux,
>
> node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=node01.magedu.com 节点角色



定义pod模块中的spec下containers同级别，定义新字段`nodeSelector`

```bash
[root@master chapter5]# kubectl explain pod.spec.nodeSelector
KIND:     Pod
VERSION:  v1

FIELD:    nodeSelector <map[string]string> # 映射，给key:value对

DESCRIPTION:
     NodeSelector is a selector which must be true for the pod to fit on a node.
     Selector which must match a node's labels for the pod to be scheduled on
     that node. More info:
     https://kubernetes.io/docs/concepts/configuration/assign-pod-node/

```

```diff
"filebeat-ds.yaml" 27L, 835C                                                                                                                                                                                                                                                                             24,9          All
apiVersion: apps/v1          # group/version
kind: DaemonSet
metadata:
  name: filebeat-ds
  namespace: prod
  labels:                    # 控制器的标签, 非必须
    app: filebeat
spec:
  selector: # 选择器, 判定集群节点上有我们期望的pod
    matchLabels:
      app: filebeat
  template: # 创建pod的模板
    metadata:
      labels: # pod的标签
        app: filebeat
      name: filebeat
    spec:
+      nodeSelector:
+        logcollecting: "on"
      containers:
      - name: filebeat # pod容器名
        image: ikubernetes/filebeat:5.6.5-alpine # 镜像
        env: # 环境变量, 云原生使用变量。 非云原生使用entrypoint脚本处理
        - name: REDIS_HOST  # 变量名
          value: db.ikubernetes.io:6379      # 值
        - name: LOG_LEVEL
          value: info

```

> 注意：上面已经修改过image版本，现在重新apply，镜像版本又回到5.6.5了

```bash
kubectl apply -f filebeat-ds.yaml
[root@master chapter5]# kubectl get ds -n prod -o wide
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR      AGE   CONTAINERS   IMAGES                              SELECTOR
filebeat-ds   0         0         0       0            0           logcollecting=on   88s   filebeat     ikubernetes/filebeat:5.6.5-alpine   app=filebeat

[root@master chapter5]# kubectl get pod -n prod -l app=filebeat
# 因为没有节点有这个标签logcollecting: "on"
No resources found in prod namespace.


```

人为改变节点标签

```bash
# 添加标签
[root@master chapter5]# kubectl label node/node01.magedu.com logcollecting="on"
node/node01.magedu.com labeled

# 一个
[root@master chapter5]# kubectl get ds -n prod -o wide
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR      AGE     CONTAINERS   IMAGES                              SELECTOR
filebeat-ds   1         1         1       1            1           logcollecting=on   2m51s   filebeat     ikubernetes/filebeat:5.6.5-alpine   app=filebeat
[root@master chapter5]# kubectl get pod -n prod -l app=filebeat -o wide
NAME                READY   STATUS    RESTARTS   AGE   IP            NODE                NOMINATED NODE   READINESS GATES
																	 # 在node01
filebeat-ds-rtbn9   1/1     Running   0          13s   10.244.1.13   node01.magedu.com   <none>           <none>

```



实现pod在指定节点运行

- [x] 节点选择器 pod.spec.nodeSelector
- [x] pod affinite
- [x] nodeName



k8s本身就有daemonset控制器

```bash
[root@master chapter5]# kubectl get ds -n kube-system
NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
# 提供pod网络 10.244.0.0/16. 
kube-flannel-ds   4         4         4       4            4           <none>                   12d
# 生成iptables/ipvs规则
kube-proxy        4         4         4       4            4           kubernetes.io/os=linux   12d

```



# Job

一次性调用，非守护进程

正常pod控制器，pod宕了，会重建。但是有些任务（备份），仅需要执行一次。



创建一个或多个pod, 确保任务“**成功**”完成，不成功就重建

- [x] 模板       `template`
- [x] 选择器    `selector`
- [x] 并行job  `parallel jobs`, 处理任务可以切分时，可以并行。
  - [x] 并行度 .spec.parallelism， 1，                   any。
  - [x] 完成度 .spec.completions，作业总量 1，  w个作业

重启pod的策略

`OnFailure`, 退出码非0， 会重启。

`Never`, 退出码非0， 不重启



## 单作业

```diff
"job-example.yaml" 15L, 353C                                                                                                                                                                                                                                                                             15,7          All
apiVersion: batch/v1
kind: Job
metadata:
  name: job-example
+  namespace: prod
spec:
  template:
    metadata:
      labels:
        app: myjob
    spec:
      containers:
      - name: myjob
        image: alpine
        command: ["/bin/sh",  "-c", "sleep 120"]
+      restartPolicy: Never # 意外终止不重启，但是job退出非0, 会重建，让作业一定完成

```

```bash
kubectl apply -f job-example.yaml
[root@master chapter5]# kubectl get job -n prod -o wide
NAME          COMPLETIONS   DURATION   AGE     CONTAINERS   IMAGES   SELECTOR
			  # 作业完成
job-example   1/1           2m10s      2m20s   myjob        alpine   controller-uid=f1dae7bd-52f4-4b0c-bedc-d0fe53639601
[root@master chapter5]# kubectl get pod -n prod
NAME                           READY   STATUS      RESTARTS   AGE
filebeat-ds-rtbn9              1/1     Running     0          17m	
									   # pod显示完成 
job-example-wcskd              0/1     Completed   0          2m23s
myapp-deploy-b9f79945c-bcnmw   1/1     Running     0          8d
myapp-deploy-b9f79945c-jpsgq   1/1     Running     0          8d
myapp-deploy-b9f79945c-rk8ng   1/1     Running     0          8d
myapp-rs-npmms                 1/1     Running     0          8d
myapp-rs-p2fq2                 1/1     Running     0          8d
myapp-rs-qlgv9                 1/1     Running     0          8d
myapp-rs-t6cnj                 1/1     Running     0          8d
pod-demo                       2/2     Running     8          8d

```

## 多作业

### 单路串行

```diff
"job-multi.yaml" 18L, 406C                                                                                                       

apiVersion: batch/v1
kind: Job
metadata:
  name: job-multi
+  namespace: test
spec:
+  completions: 5     # 任务总量, 5任务
+  # 没有并行度, 则为1
+  # 指了并行为2，2路执行2次，再1路执行第3个. 
  template:
    metadata:
      labels:
        app: myjob
    spec:
      containers:
      - name: myjob
        image: alpine
+        command: ["/bin/sh",  "-c", "sleep 5"] # 快点完成
      restartPolicy: Never

```

串行执行5个任务

```bash
[root@master chapter5]# kubectl create ns test
namespace/test created
[root@master chapter5]# kubectl apply -f job-multi.yaml 
job.batch/job-multi created
[root@master chapter5]# kubectl get pods -n test -w
NAME              READY   STATUS              RESTARTS   AGE
job-multi-brxwk   0/1     ContainerCreating   0          6s
job-multi-brxwk   1/1     Running             0          9s
job-multi-brxwk   0/1     Completed           0          14s
job-multi-ckqbm   0/1     Pending             0          0s
job-multi-ckqbm   0/1     Pending             0          0s
job-multi-ckqbm   0/1     ContainerCreating   0          0s
job-multi-ckqbm   1/1     Running             0          9s
job-multi-ckqbm   0/1     Completed           0          14s
job-multi-2dt5w   0/1     Pending             0          0s
job-multi-2dt5w   0/1     Pending             0          0s
job-multi-2dt5w   0/1     ContainerCreating   0          0s
job-multi-2dt5w   1/1     Running             0          8s
job-multi-2dt5w   0/1     Completed           0          13s
job-multi-6qm2f   0/1     Pending             0          0s
job-multi-6qm2f   0/1     Pending             0          0s
job-multi-6qm2f   0/1     ContainerCreating   0          0s
job-multi-6qm2f   1/1     Running             0          8s
job-multi-6qm2f   0/1     Completed           0          13s
job-multi-kq4r4   0/1     Pending             0          0s
job-multi-kq4r4   0/1     Pending             0          0s
job-multi-kq4r4   0/1     ContainerCreating   0          0s
job-multi-kq4r4   1/1     Running             0          9s
job-multi-kq4r4   0/1     Completed           0          14s

```

### 多路并行

```bash
[root@master chapter5]# kubectl get job -n test
NAME        COMPLETIONS   DURATION   AGE
job-multi   5/5           69s        119s
[root@master chapter5]# kubectl delete job -n test job-multi
job.batch "job-multi" deleted
```

```diff
[root@master chapter5]# cat job-multi.yaml 
apiVersion: batch/v1
kind: Job
metadata:
  name: job-multi
  namespace: test
spec:
  completions: 5     # 任务总量, 5任务
  # 没有并行度, 则为1
  # 指了并行为2，2路执行2次，再1路执行第3个. 
+  parallelism: 2
  template:
    metadata:
      labels:
        app: myjob
    spec:
      containers:
      - name: myjob
        image: alpine
        command: ["/bin/sh",  "-c", "sleep 5"]
      restartPolicy: Never


```

```bash
[root@master chapter5]# kubectl apply -f job-multi.yaml
job.batch/job-multi created
[root@master chapter5]# kubectl get  pod -n test -w
NAME              READY   STATUS    RESTARTS   AGE
# 一次并行2路 tm 4j
job-multi-92ztm   0/1     Pending   0          0s
job-multi-97s4j   0/1     Pending   0          0s
job-multi-92ztm   0/1     Pending   0          0s
job-multi-97s4j   0/1     Pending   0          0s
job-multi-92ztm   0/1     ContainerCreating   0          0s
job-multi-97s4j   0/1     ContainerCreating   0          0s
job-multi-92ztm   1/1     Running             0          10s
job-multi-97s4j   1/1     Running             0          15s
job-multi-92ztm   0/1     Completed           0          15s
# 并行2路 9f ct
job-multi-nvt9f   0/1     Pending             0          1s
job-multi-nvt9f   0/1     Pending             0          1s
job-multi-nvt9f   0/1     ContainerCreating   0          1s
job-multi-97s4j   0/1     Completed           0          20s
job-multi-ssbct   0/1     Pending             0          1s
job-multi-ssbct   0/1     Pending             0          1s
job-multi-ssbct   0/1     ContainerCreating   0          1s
job-multi-nvt9f   1/1     Running             0          11s
job-multi-nvt9f   0/1     Completed           0          16s
# 并行1路 8566
job-multi-v8566   0/1     Pending             0          0s
job-multi-v8566   0/1     Pending             0          0s
job-multi-v8566   0/1     ContainerCreating   0          0s
job-multi-ssbct   1/1     Running             0          14s
job-multi-ssbct   0/1     Completed           0          18s
job-multi-v8566   1/1     Running             0          9s
job-multi-v8566   0/1     Completed           0          14s

```





# cronjob

期望job周期性任务，job按时间点启动

```diff
[root@master chapter5]# cat cronjob-example.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cronjob-example
  labels:
    app: mycronjob
spec:
+  schedule: "*/2 * * * *"  # 每2分钟一次
+  jobTemplate: # job模板，cronjob控制job. job控制pod
    metadata:
      labels:
        app: mycronjob-jobs
    spec:
      parallelism: 2
      template:
        spec:
          containers:
          - name: myjob
            image: alpine
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster; sleep 10
          restartPolicy: OnFailure
```





# Garbage Collection

https://kubernetes.io/zh/docs/concepts/workloads/controllers/garbage-collection/

pod终止时，需要回收

cascading 删除：后台、前台

> 级联：删除控制器，会自动删除pod.   kubectl delete --cascade=true，不指定这个选项,默认true
>
> 不级联：删除deploy, 不自动删除pod.  kubectl delete --cascade=false

- [x] 后台：级联操作，后台默默执行

  ```bash
  [root@master chapter5]# kubectl delete job/job-multi -n test
  job.batch "job-multi" deleted
  # 其实还需要删除pod, 也需要等待一段时间
  ```

- [x] 前台：级联操作，前台删除完，你才可以操作终端。

定义一个资源时可以定义删除方式`propagationPolicy.deleteOptions`

1. Orphan k8s v1.9 之前，不级联删除
2. Foreground 前台
3. Background 后台，kubernetes v1.9之后默认值 



# node和node status

- Nodes

- Node Status ` kubectl describe node/172.16.0.222`

  - addresses地址：hostname, externalip外部地址, internalip

  - condition: 节点处于什么状态

    1. Outofdisk 磁盘耗尽
    2. Ready 就绪
    3. MemoryPressure    内存是否有压力？即是否耗尽
    4. DiskPressure        磁盘是否耗尽?
    5.  PIDPressure          PID是否耗尽？
    6. NetworkUnavailabel  网络不可用
    7. ConfigOK                    配置就绪

    diskpressurce/pidpressure/memorypressure 一般不会调用pod

    但是监控系统这种忽略pressure, 也会调度

  - capacity: 节点的容量。 cpu/ram/最大pod、自身信息

    > Capacity:  总容量
    >
    >  cpu:                8               cpu核心
    >  ephemeral-storage:  41152812Ki 临时 存储
    >  hugepages-1Gi:      0        1Gi 内存页
    >  hugepages-2Mi:      0       2Mi 内存页
    >  memory:             16265568Ki 
    >  pods:               110            最大的pod数量
    >
    > Allocatable:      可分配
    >  cpu:                8
    >  ephemeral-storage:  37926431477
    >  hugepages-1Gi:      0
    >  hugepages-2Mi:      0
    >  memory:             15548768Ki pod可使用的，未分配是系统使用的。
    >  pods:               110

  - 已分配 

    > Allocated resources:
    >   (Total limits may be over 100 percent, i.e., overcommitted.)
    >   Resource           Requests      Limits
    > --------           --------      ------
    >   cpu                3412m (42%)   13670m (170%)                 pod定义了cpu/ram此处才计数。不定义说明cpu/ram可有可无。
    >   memory             3599Mi (23%)  18879Mi (124%)
    >   ephemeral-storage  0 (0%)        0 (0%)

生产环境中，pod启动一定要定义资源

