---
title: "Pod资源调度"
date: 2021-02-02 09:50:00
---



# 前言

master节点内部的scheduler内部隐藏调度器程序，监听api server中pod资源属性字段`nodeName`是否为空？为空时，选择最佳节点，并运行pod。选择结果更新nodeName。

api server 知道pod调度到节点上, api server连接kubelet读取pod定义，调用docker引擎创建pod。

不同的调度算法评估pod运行节点位置不一样



评估过程有3个阶段

1. **预选(predicate)**：满足pod运行的节点，污点
2. **优选(priority)**：满足运行需求的优选级，资源需求
3. **选定(select)**:  随机选择一个节点



高级调度

- nodeselector/nodename            mysql定位在mysql主机
- nodeaffinity                                   mysql定位在mysql主机
- podaffinity/podantiaffinity          tomcat+mysql在一起

- taints/tolerations  节点配置非常高，运行特定类型的opd
  - noscheduler
  - prefernoscheduler
  - noexecute



<!--more-->
# 预选(predicate)

一组程序对每个节点评估，任何一个程序不符合，这个节点就排队在外。（一票否决）

剩下：一组节点

| 预选程序                | 功能                                                         | 一票否决                                                     |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| CheckNodeCondition      | 文件系统已满，网络不可用或者 kubelet 尚未准备好运行 Pod      | 文件系统满的节点，不被选择                                   |
| hostname                | `nodeName` 的值                                              | **只能过滤nodeName一致的节点**                               |
| PodFitsHostPorts        | `ports.hostPort` 在节点上没有冲突端口                        | 有冲突的节点不被选择                                         |
| MatchNodeSelector       | `nodeSelector`的值 是节点标签的子集                          | **节点没有符合标签不选择**                                   |
| PodFitsResources        | CPU和内存是否满足 Pod 的要求                                 |                                                              |
| PodToleratesNodeTaints  | Pod 的[容忍](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/taint-and-toleration/) 是否能容忍节点的[污点](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/taint-and-toleration/)。 | **节点存在污点时**pod容忍的污点和节点污点匹配，就选择。如果节点存在的污点pod不容忍，就不选择节点。 |
| CheckVolumeBinding      | 已经绑定卷与pod是否冲突                                      |                                                              |
| NoVolumeZoneConflict    | 存储卷区域没有冲突？                                         |                                                              |
| CheckNodeMemoryPressure | 节点内存不足                                                 |                                                              |
| CheckNodePIDPressure    | PID不足                                                      |                                                              |
| CheckNodeDiskPressure   | 文件系统不足                                                 |                                                              |
| MatchInterPodAffinity   | pod倾向的节点                                                |                                                              |

> nodeName, nodeSelector可以硬性调度
>
> 硬亲和：必须满足

打开k8s源码文件：go源码 pkg

https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/algorithmprovider/registry.go



# 优选(priority)

一组优先函数对每个节点评估，任何一个函数都对节点打分，一个节点的整体得分是所有得分之和。

剩下：逆序排序，选择得分最高的

| 优先函数                    | 功能                                                         | 分值总和 |
| --------------------------- | ------------------------------------------------------------ | -------- |
| BalancedResourceAllocation  | 平衡资源分配：cpu/ram占用率比例均为50%，越接近，得分越高。偏差越多，得分越高 |          |
| LeastRequestedPriority      | 节点已用资源/节点容量的比值。比率越小，用的资源少，得分高。 (cpu/总 + ram/总) /2 |          |
| MostRequestedPriority       | 与LeastRequestedPriority相反，谁分的多，就使用谁。保证一个节点先填满资源。 |          |
| NodePreferAvoidPodsPriority | 节点倾向不运行pod, node排斥pod, 越排斥越不被调度             |          |
| NodeAffinityPriority        | PreferredDuringSchedulingIgnoredDuringExecution定义node与pod倾向性，有的话，得分高 |          |
| TaintTolerationPriority     | pod能否容忍node污点的容忍程度。定义污点时有2种机制，1. 不容忍不能调度。2. 不容忍不强行不能调度 |          |
| SelectorSpreadPriority      | 属于同一 [Service](https://kubernetes.io/zh/docs/concepts/services-networking/service/)、 [StatefulSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/) 或 [ReplicaSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicaset/) 的 Pod，尽可能跨主机部署。确保pod 跨节点高可用 |          |

> 软亲和：满足更好，不满足先找个node运行。

最高分胜出



# 选定

随机选

<!--more-->

# pod自定义调度器

目前k8s只有一个调度器，将来k8s允许插件式调度器，允许插件调度器调度对接pod上，完成特定功能的调度。

`pod.spec.schedulername`

```bash
[root@master ~]# kubectl explain pod.spec.schedulerName
KIND:     Pod
VERSION:  v1

FIELD:    schedulerName <string>

DESCRIPTION:
     If specified, the pod will be dispatched by specified scheduler. If not
     specified, the pod will be dispatched by default scheduler.
```

如果没有指定这个属性，就使用默认的scheduler名`schedulerName: default-scheduler`

```bash
[root@master ~]# kubectl get pod -n prod client-pod -oyaml | grep ' schedulerName'
  schedulerName: default-scheduler
```

> pod创建时，没有定义这个字段，准入控制器`变异型`会修改这个值。



# 高级调度机制

施加外部调度条件，影响调度结果

## 调度类型

| 选择机制                    | 干预阶段   | 功能                                                         |
| --------------------------- | ---------- | ------------------------------------------------------------ |
| nodeSelector/nodeName       | 预选       | **pod选择节点**                                              |
| nodeAffinity                | 预选、优选 | pod对node亲和性。 反亲和使用notin/doesnotexist。**pod选择节点**，基于node标签。 |
| podAffinity/podAntiAffinity | 预选、优选 | pod对pod亲和性/反亲和性。pod选择节点，**基于pod选择节点**。节点有我喜欢的pod才运行。 |
|                             |            |                                                              |
| taints+tolerations          | 预选       | *节点选择pod*，pod容忍污点、节点污点定义                     |


### taints + tolerations

> pod容忍度tolerations：默认pod可以运行在所有没有污点的node. 有污点的node, pod需要容忍。
>
> node污点定义，默认节点没有污点，k8s可以根据condition自动添加污点。kubeadm部署的k8s集群，master部署时自动打污点。

![image-20210203151908653](http://myapp.img.mykernel.cn/image-20210203151908653.png)

### node affinity两种限制

- 硬限制：required 调度过程中 **必须满足条件**，**调度完成后对既成事实没有影响。**
- 软限制：preferred 调度过程中 **尽可能满足条件**，调度完成后对既成事实没有影响。可以指定权重，多个prefer权重可不一样。

operator: in ,notin ,exists ,doesnotexist, gt , lt

> In 满足其一

notin doesnotexist 某种意义上 相当anti-affinity功能相似

多个affinity，与关系



### pod亲和性

- 亲和：pod间在同一位置。      位置为节点，topo=hostname, 一起同节点;        位置为机房，topo=zone，一起同机房，未必在同节点；位置为数据中心, topo=dc，一起同数据中心，未必在同节点；

  > 示例：pod 硬亲和，有security=S1标签的pod。结果必须在pod所在区域中任意节点

  <img src="http://myapp.img.mykernel.cn/podAffinity.png" alt="podAffinity" style="zoom:50%;" />

- 不亲和：pod间不在同一位置。SelectorSpreadPriority默认的优选函数就可以将同控制器分散。

  > 示例：pod 软反亲和，有security=S2标签的pod。结果尽量不在pod所在区域中任意节点。

  <img src="http://myapp.img.mykernel.cn/podAntiaffinity.png" alt="podAntiaffinity" />



nginx -> ingress, ingress controller是deploy的pod, pod更高可用性，需要高配置节点来运行2个nginx.

- 节点污点 + pod 容忍
- daemonset + nodeSelector + 节点污点 + pod 容忍



## nodeSelector/nodeName

略

## nodeAffinity 

```bash
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution: 
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-2-name
            operator: In  # 满足其中之一
            values:
            - e2e-az1
            - e2e-az2 
      preferredDuringSchedulingIgnoredDuringExecution: 
      - weight: 1  # 满足硬亲和的分 + 满足软亲和的权重
        preference:
          matchExpressions:
          - key: e2e-az-EastWest 
            operator: In 
            values:
            - e2e-az-East 
            - e2e-az-West 
  containers:
  - name: with-node-affinity
    image: docker.io/ocpqe/hello-pod
    
```

> 硬亲和，满足其中之一的节点，多个。再在其中选择软亲和

### 生效方式

nodeselector + nodeaffinity 需要同时满足

nodeSelectorTerms中多个matchExpressions时，或关系

同一个nodeSelectorTerms.matchExpression的多个key存在与关系。

weight: 1  # 满足硬亲和的分 + 满足软亲和的权重

### 示例nodeaffinity

#### 硬亲和

没有这样的节点不调度

```diff
[root@master ~]# cat Kubernetes_Advanced_Practical/chapter12/required-nodeAffinity-pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: with-required-nodeaffinity
spec:
  affinity:
    nodeAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
+          - {key: zone, operator: In, values: ["foo"]} # 匹配节点
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
```

```bash
[root@master ~]# kubectl apply -f Kubernetes_Advanced_Practical/chapter12/required-nodeAffinity-pod.yaml
pod/with-required-nodeaffinity created
[root@master ~]# kubectl get po
NAME                            READY   STATUS    RESTARTS   AGE
with-required-nodeaffinity      0/1     Pending   0          3s

```

人为修改节点打上zone=foo，即可以运行。

运行后，删除标签不会影响已调度。

#### 硬亲和,多key

与关系，节点必须有zone=foo, ssd

```diff
[root@master ~]# cat Kubernetes_Advanced_Practical/chapter12/required-nodeAffinity-pod2.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: with-required-nodeaffinity-2
spec:
  affinity:
    nodeAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
+          - {key: zone, operator: In, values: ["foo", "bar"]} # zone=foo或zoo=bar
+          - {key: ssd, operator: Exists, values: []} # 表示存在ssd的key即可
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
```

#### 硬亲和，多个matchExpressions

或关系

```diff
[root@master ~]# cat Kubernetes_Advanced_Practical/chapter12/required-nodeAffinity-pod2.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: with-required-nodeaffinity-2
spec:
  affinity:
    nodeAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
+        - matchExpressions:
          - {key: zone, operator: In, values: ["foo", "bar"]}
+        - matchExpressions:
          - {key: ssd, operator: Exists, values: []}
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
```

> 值不能有空格，有空格，apply也不会提示，但是结果就不是或了

#### 软亲和

即便没有任何一个节点满足，也会调度

```bash
[root@master ~]# cat Kubernetes_Advanced_Practical/chapter12/deploy-with-preferred-nodeAffinity.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy-with-node-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 60
            preference:
              matchExpressions:
              - {key: zone, operator: In, values: ["foo"]}
          - weight: 30
            preference:
              matchExpressions:
              - {key: ssd, operator: Exists, values: []}
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
```



## podAffinity

注意：调度前，需要已有pod, 已有pod的调度方式，肯定是nodeSelector/nodeName, nodeAffinity

### 为什么使用pod亲和？

基于业务关联关系，不受控同一个控制器的两pod，根据业务需求，规划需求，需要**2个pod**运行同一个位置(topo)，或者**2个pod**不能运行在同一个位置(topo)之上。高级调度的控制机制，叫inter-pod affinity and anti-affinity

冗余能力：podantiaffinity，硬件IO能力使用提升。

协作：pod affinity,

#### 同一位置

界定两个节点是否同一位置？**取决于键的值（是否相同，相同则同一个位置）**

```diff
[root@master ~]# kubectl get node --show-labels
NAME                STATUS   ROLES                  AGE   VERSION   LABELS
master.magedu.com   Ready    control-plane,master   19d   v1.20.2   
	
+				  # os当前节点架构平台类型，同一个k8s中，可能有arm/arch/ibm/惠普	
+                                                # os linux/unix
+                                                                                               #hostname 当前节点的节点名
+ node-role.kubernetes.io/master= 主节点
beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master.magedu.com,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=
node01.magedu.com   Ready    <none>                 19d   v1.20.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node01.magedu.com,kubernetes.io/os=linux,logcollecting=on
node02.magedu.com   Ready    <none>                 19d   v1.20.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node02.magedu.com,kubernetes.io/os=linux
node03.magedu.com   Ready    <none>                 19d   v1.20.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node03.magedu.com,kubernetes.io/os=linux



```

topologkey=beta.kubernetes.io/arch 节点1-3是同一个位置，因为这3个节点的beta.kubernetes.io/arch=amd64，有相同的值

topologkey=beta.kubernetes.io/os 同一个位置。

topologkey=kubernetes.io/hostname 3个节点不同位置。



将来根据需求做实际物理拓扑结构划分 ，组织k8s集群：

- 冗余：反亲和
- 协作：亲和

| 冗余方式   | 冗余级别                                                 |              |
| ---------- | -------------------------------------------------------- | ------------ |
| 跨DC冗余   | 冗余级别非常高                                           | 地理位置不同 |
| 跨机房     | 较高，                                                   |              |
| 跨配电单元 | 配电单元宕机，下面所有机柜不可用，高                     |              |
| 每排机柜   | 一个机柜有一个栈顶交换机，交换机崩了，另一个机柜不影响。 |              |
| 每个机架   | 一个机柜有一个栈顶交换机，交换机崩了，无冗余。           |              |
| 节点       | 级别非常低                                               | 颗粒度低     |



##### 跨数据中心(data center)

一个集群可以跨多个数据中心，仅需要节点间可以互相通信，就可以构建集群。

联系非常紧密的应用，就算不运行在同一个节点，也应该运行同一个数据中心，通信才更快。

pod 基于数据中心作为topologkey(=dc),  要么运行在A数据中心的某些节点之上，要么运行在B数据中心的某些节点之上。

![image-20210203123928946](http://myapp.img.mykernel.cn/image-20210203123928946.png)

##### 跨机房

一个数据中心有多个room, room间使用中心交换机或路由连接。

期望pod在同一个room中

1. 给每个节点定义一个键：room=服务器所在机房
2. 将来pod 基于机房作为topologkey(=room), 确保**关联的pod**, 要么在同一个room中，要么不能在同一个room中
   - 冗余：不同room，反亲和
   - 协作、通信更紧密：相同room，亲和

![image-20210203124652927](http://myapp.img.mykernel.cn/image-20210203124652927.png)

##### 跨机架、机柜、配电单元

同一个room有多个pdu(配电单元)，不同的pdu之下有多排机柜(rack)，每个机柜一个交换机， 每个机柜上有多个机架，每个机架上有多个服务器。

将来可以跨每个机架、跨每排机柜

### pod亲和调度的缺陷

评估时间：**非常大的集群环境（100节点以上），不建议使用，因为使用pod亲和时每调度一个pod，需要先评估每一个节点是否相同位置。**

> 可以使用node affinity

**pod anti-affinity要求每个节点针对topologkey设置正确的标签, 否则没有topokey, 就不能评估。**

> 添加节点需要配置节点标签，否则反亲和无法工作。

### pod affinity 和 pod antiaffinity

https://docs.openshift.com/container-platform/3.7/admin_guide/scheduling/pod_affinity.html

```diff
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity 

spec:
  affinity: # affinity亲和性
    podAffinity:  # pod亲和. 与pod在同一个位置
      requiredDuringSchedulingIgnoredDuringExecution:  # required硬亲和, 也有软亲和 prefer
      - labelSelector: # 选择pod
          matchExpressions:
          - key: security 
            operator: In  # 可以In，NotIn，Exists，或DoesNotExist
            values:
            - S1 
        topologyKey: failure-domain.beta.kubernetes.io/zone # 定位pod可以在哪些区域，是逻辑概念。每个机房一个zone或自已有5个服务，划分每2台一个区段。
+    # 与关系
    podAntiAffinity:  # pod反亲和
      preferredDuringSchedulingIgnoredDuringExecution:  # required硬亲和, 也有软亲和 preferred
      - weight: 100  
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security 
              operator: In 
              values:
              - S2
          topologyKey: kubernetes.io/hostname
  containers:
  - name: with-pod-affinity
    image: docker.io/ocpqe/hello-pod
```

> affinity同时定义pod affinity/antiaffinity同时生效

#### pod affinity

![podAffinity](http://myapp.img.mykernel.cn/podAffinity.png)

#### pod Antiaffinity

<img src="http://myapp.img.mykernel.cn/podAntiaffinity.png" alt="podAntiaffinity" style="zoom:50%;" />

由于是与关系，所以结果，还是在上面3个节点运行：尽量在1，2节点。实在不行才在第3个节点上运行。

### 亲和

#### 硬亲和

pod创建3个pod必须运行在，zone某个区域中，存在app=db标签的运行的pod的区域中，任何一个节点。

基于业务的逻辑，与app=db运行的pod越近越好，tomcat+mysql一块，访问效率高。

```diff
[root@master ~]#  cat Kubernetes_Advanced_Practical/chapter12/deploy-with-required-podAffinity.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-with-pod-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      name: myapp
      labels:
        app: myapp
    spec:
      affinity:
+        podAffinity: # pod亲和
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
+              - {key: app, operator: In, values: ["db"]} 
+            topologyKey: zone
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
```





运行redis, 打标签

```bash
kubectl create deployment redis --image=redis:4-alpine
[root@master ~]# kubectl get pod -l app=redis -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP           NODE                NOMINATED NODE   READINESS GATES
redis-5bcb8468c7-hfgg6   1/1     Running   0          2m38s   10.244.2.5   node02.magedu.com   <none>           <none>
```

所有节点添加topologyKey, redis 在node02, 我们把node01,02在同一个区域。node03在另一个区域

创建myapp pod,  默认同控制器在不同节点。但是我们定义了优先函数后，myapp应该只会在node01,node02上。

```bash
[root@master ~]# kubectl label nodes node01.magedu.com zone=foo
node/node01.magedu.com labeled
[root@master ~]# kubectl label nodes node02.magedu.com zone=foo
node/node02.magedu.com labeled
[root@master ~]# kubectl label nodes node03.magedu.com zone=bar
node/node03.magedu.com labeled
```



创建一个nginx与数据库离的近

```diff
"Kubernetes_Advanced_Practical/chapter12/deploy-with-required-podAffinity.yaml" 25L, 547C                                       
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-with-pod-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      name: myapp
      labels:
        app: myapp
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
+              - {key: app, operator: In, values: ["redis"]}
+            topologyKey: zone
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1

```

提交

```bash
[root@master ~]#  kubectl apply -f Kubernetes_Advanced_Practical/chapter12/deploy-with-required-podAffinity.yaml 
deployment.apps/myapp-with-pod-affinity created


[root@master ~]# kubectl get pods -l app=myapp -o wide
NAME                                       READY   STATUS    RESTARTS   AGE   IP           NODE                NOMINATED NODE   READINESS GATES
myapp-with-pod-affinity-54df49c854-94n58   1/1     Running   0          30s   10.244.1.4   node01.magedu.com   <none>           <none>
myapp-with-pod-affinity-54df49c854-j5sg7   1/1     Running   0          30s   10.244.2.6   node02.magedu.com   <none>           <none>
myapp-with-pod-affinity-54df49c854-r2vkq   1/1     Running   0          30s   10.244.2.7   node02.magedu.com   <none>           <none>

```

注意：只在node01,node02.

扩容pod观察, 

```bash
[root@master ~]# kubectl scale deploy/myapp-with-pod-affinity --replicas=6
```

```bash
[root@master ~]# kubectl get pods -l app=myapp -o wide
NAME                                       READY   STATUS    RESTARTS   AGE    IP           NODE                NOMINATED NODE   READINESS GATES
myapp-with-pod-affinity-54df49c854-7m99h   1/1     Running   0          6s     10.244.2.8   node02.magedu.com   <none>           <none>
myapp-with-pod-affinity-54df49c854-94n58   1/1     Running   0          5m8s   10.244.1.4   node01.magedu.com   <none>           <none>
myapp-with-pod-affinity-54df49c854-j5sg7   1/1     Running   0          5m8s   10.244.2.6   node02.magedu.com   <none>           <none>
myapp-with-pod-affinity-54df49c854-r2vkq   1/1     Running   0          5m8s   10.244.2.7   node02.magedu.com   <none>           <none>
myapp-with-pod-affinity-54df49c854-sctk9   1/1     Running   0          6s     10.244.1.5   node01.magedu.com   <none>           <none>
myapp-with-pod-affinity-54df49c854-wnhv9   1/1     Running   0          6s     10.244.1.6   node01.magedu.com   <none>           <none>
```

扩容在多的副本，只会在node01, node02区域运行。

#### 软亲和

软限制，不符合条件也会调度

```diff
[root@master ~]#  cat Kubernetes_Advanced_Practical/chapter12/deploy-with-preferred-podAffinity.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-with-preferred-pod-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      name: myapp
      labels:
        app: myapp
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - {key: app, operator: In, values: ["db"]}
+              topologyKey: rack #  尽可能运行在拥有app=db的pod的机架上任何一个节点上。
+          # 或关系
          - weight: 20
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - {key: app, operator: In, values: ["db"]}
+              topologyKey: zone # zone中拥有app=db的zone上任何一个节点上。
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1

```

> 或关系：有的节点+80, 有的节点+20，有的节点既+80又+20
>
> rack倾向更高



### 反亲和

#### 硬限制

pod可以运行在跨节点区域，但是不能运行在有 app=myapp标签的pod所在区域上。

```diff
[root@master ~]#  cat Kubernetes_Advanced_Practical/chapter12/deploy-with-required-podAntiAffinity.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-with-pod-anti-affinity
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      name: myapp
      labels:
        app: myapp
    spec:
      affinity:
+        podAntiAffinity:
+          requiredDuringSchedulingIgnoredDuringExecution: # 硬
          - labelSelector:
              matchExpressions:
+              - {key: app, operator: In, values: ["myapp"]}
+            topologyKey: kubernetes.io/hostname
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1

```



获取app=myapp标签的pod已经在哪些节点运行了？

```bash
[root@master ~]# kubectl get pod -A -l app=myapp -o wide
NAMESPACE   NAME                                       READY   STATUS    RESTARTS   AGE   IP           NODE                NOMINATED NODE   READINESS GATES
default     myapp-with-pod-affinity-54df49c854-7m99h   1/1     Running   0          6m    10.244.2.8   node02.magedu.com   <none>           <none>
default     myapp-with-pod-affinity-54df49c854-94n58   1/1     Running   0          11m   10.244.1.4   node01.magedu.com   <none>           <none>
default     myapp-with-pod-affinity-54df49c854-j5sg7   1/1     Running   0          11m   10.244.2.6   node02.magedu.com   <none>           <none>
default     myapp-with-pod-affinity-54df49c854-r2vkq   1/1     Running   0          11m   10.244.2.7   node02.magedu.com   <none>           <none>
default     myapp-with-pod-affinity-54df49c854-sctk9   1/1     Running   0          6m    10.244.1.5   node01.magedu.com   <none>           <none>
default     myapp-with-pod-affinity-54df49c854-wnhv9   1/1     Running   0          6m    10.244.1.6   node01.magedu.com   <none>           <none>
dev         myapp-7d4b7b84b-4fknw                      1/1     Running   0          21h   10.244.2.3   node02.magedu.com   <none>           <none>
dev         myapp-7d4b7b84b-jhbzx                      1/1     Running   0          21h   10.244.1.2   node01.magedu.com   <none>           <none>
# node01,node02
```

所以启动antiaffinity时，一定不同节点

```bash
[root@master ~]# kubectl apply -f Kubernetes_Advanced_Practical/chapter12/deploy-with-required-podAntiAffinity.yaml 
deployment.apps/myapp-with-pod-anti-affinity created

```

```diff
[root@master ~]# kubectl get pods -l app=myapp -o wide -A
NAMESPACE   NAME                                           READY   STATUS    RESTARTS   AGE     IP           NODE                NOMINATED NODE   READINESS GATES
default     myapp-with-pod-affinity-54df49c854-7m99h       1/1     Running   0          8m37s   10.244.2.8   node02.magedu.com   <none>           <none>
default     myapp-with-pod-affinity-54df49c854-94n58       1/1     Running   0          13m     10.244.1.4   node01.magedu.com   <none>           <none>
default     myapp-with-pod-affinity-54df49c854-j5sg7       1/1     Running   0          13m     10.244.2.6   node02.magedu.com   <none>           <none>
default     myapp-with-pod-affinity-54df49c854-r2vkq       1/1     Running   0          13m     10.244.2.7   node02.magedu.com   <none>           <none>
default     myapp-with-pod-affinity-54df49c854-sctk9       1/1     Running   0          8m37s   10.244.1.5   node01.magedu.com   <none>           <none>
default     myapp-with-pod-affinity-54df49c854-wnhv9       1/1     Running   0          8m37s   10.244.1.6   node01.magedu.com   <none>           <none>
default     myapp-with-pod-anti-affinity-68bbd6f98-rrj7f   0/1     Pending   0          99s     <none>       <none>              <none>           <none>
default     myapp-with-pod-anti-affinity-68bbd6f98-vx5ww   0/1     Pending   0          99s     <none>       <none>              <none>           <none>
default     myapp-with-pod-anti-affinity-68bbd6f98-z84xf   0/1     Pending   0          99s     <none>       <none>              <none>           <none>
+default     myapp-with-pod-anti-affinity-68bbd6f98-zfc2z   1/1     Running   0          99s     10.244.3.7   node03.magedu.com   <none>           <none>
dev         myapp-7d4b7b84b-4fknw                          1/1     Running   0          21h     10.244.2.3   node02.magedu.com   <none>           <none>
dev         myapp-7d4b7b84b-jhbzx                          1/1     Running   0          21h     10.244.1.2   node01.magedu.com   <none>           <none>

```

由于自已的标签也是app=myapp, 启动一个pod在node03之后，之后的pod将不能调度



## taints 和Tolerations

污点和容忍度

用于让节点排队所有pod, 节点的资源运行特定类型的pod, 或者不运行pod就像master一样。

类似标注节点的特性的键值数据，但是taints是由3部分组成：键 值 效用

- key 不超253字符，可以使用字母、数字开头。可以包含letter、number、hyphens、dot、underscores字符。
- value 最多63个字符
- effect pod容忍污点，才可以调度。不容忍时，什么操作？过段时间污点变了，是否驱离？

| 效用             | 含义                     | 作用                                                         |
| ---------------- | ------------------------ | ------------------------------------------------------------ |
| NoSchedule       | 不调度                   | pod容忍污点，才可以调度。不容忍，一定不能调度。过段时间node污点 变了，调度即成事实保持原状。 |
| PreferNoSchedule | 尽量不调度，既定的可能性 | **pod容忍污点，才可以调度。不容忍，也可以调度。**过段时间node污点 变了，调度即成事实保持原状。 |
| NoExecute        | 不执行                   | pod容忍污点，才可以调度。不容忍，一定不能调度。**过段时间node污点 变了，容忍不了污点的pod将被驱离。可以定义驱离宽限期。** |

> 应该明白，无论如何，毕竟相处这么长时间了，即使有不能容忍的污点时，也不能让对方立刻滚，好歹让对方从容离开。

### 匹配2个taint

假设一个node有两个污点：

- NoSchedule/PreferNoSchedule. 
  - pod容忍NoSchedule，不容忍PreferNoSchedule。k8s尽量避免调度
- NoSchedule/NoSchedule
  - pod对NoSchedule都容忍。才可以调度
- NoExecute/NoExecute
  - pod对NoExecute都容忍。才可以调度。
  - 调度完成，运行过程中污点变动。
    - 立即驱离
    - 不指定忍耐时间，可以一起运行。
    - 指定忍耐时间，过段时间才驱离。 

### 匹配单个taint

Equal

- key相同
- value相同
- effect相同

Exists

- key相同
- effect相同

### 污点管理

添加

```bash
kubectl taint nodes NAME KEY_1=VAL_1:TAINT_EFFECT_1 ...
						 KEY_N=VAL_N:TAINT_EFFECT_N ...
```

移除指定效用的key

```bash
kubectl taint nodes NAME KEY:TAINT_EFFECT-
```

移除名为key的tain，不管效用

```bash
kubectl taint nodes NAME KEY-
```



### 添加容忍度

Equal operator

```yaml
tolerations:
- key: "key1" # 容忍污点
  operator: "Equal" # 等值容忍：key且value且effect相同
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600 # 容忍时间，不定义。污点变动时不驱离
```

Use Exists Operator

```yaml
tolerations:
- key: "key1" # 容忍污点
  operator: "Exists" # 存在性容忍：key且effect相同
  effect: "NoExecute"
  tolerationSeconds: 3600 # 容忍时间，不定义。污点变动时不驱离
```



### 示例

查看污点 

```bash
[root@master ~]# kubectl describe nodes/master.magedu.com | grep Taints
Taints:             node-role.kubernetes.io/master:NoSchedule # 没有值，表示只能存在性判断
```

在master上的pod肯定可以容忍此污点

```bash
[root@master ~]# kubectl describe pods -n kube-system   kube-apiserver-master.magedu.com  | grep Tolerations
Tolerations:       :NoExecute op=Exists      # 存在性，效用NoExecute 
```



配置

```bash
[root@master ~]# kubectl delete all --all
# all --all 所有类型所有资源不包含configmap/secret
```



配置ingress controller仅运行在node03

- nodeSelector/nodeName: pod主动选节点
- taints ：节点选pod， pod选择节点：tolerations

节点3打污点

```bash
[root@master ~]# kubectl taint nodes node03.magedu.com ingress=nginx:NoSchedule
[root@master ~]# kubectl describe nodes node03.magedu.com  | grep -i taints
Taints:             ingress=nginx:NoSchedule
```

创建myapp-deploy有6个pod, 默认规则同deploy下pod分散调度，但是并不会运行在node03.

```bash
[root@master ~]# kubectl  create deployment myapp --image=ikubernetes/myapp:v1
deployment.apps/myapp created
[root@master ~]# kubectl scale deployment/myapp --replicas=6
deployment.apps/myapp scaled


[root@master ~]# kubectl get pod -o wide
NAME                                           READY   STATUS    RESTARTS   AGE    IP            NODE                NOMINATED NODE   READINESS GATES
myapp-7d4b7b84b-5jt48                          1/1     Running   0          11s    10.244.2.12   node02.magedu.com   <none>           <none>
myapp-7d4b7b84b-ddz25                          1/1     Running   0          11s    10.244.2.11   node02.magedu.com   <none>           <none>
myapp-7d4b7b84b-jgtjk                          1/1     Running   0          11s    10.244.1.8    node01.magedu.com   <none>           <none>
myapp-7d4b7b84b-m6vv5                          1/1     Running   0          11s    10.244.2.10   node02.magedu.com   <none>           <none>
myapp-7d4b7b84b-n62bd                          1/1     Running   0          21s    10.244.2.9    node02.magedu.com   <none>           <none>
myapp-7d4b7b84b-zl54b                          1/1     Running   0          11s    10.244.1.7    node01.magedu.com   <none>           <none>
# 并不在节点3上

[root@master ~]# kubectl  delete deployment myapp
deployment.apps "myapp" deleted

```

创建ds，也不会调度至3

```diff
[root@master ~]# cat Kubernetes_Advanced_Practical/chapter5/filebeat-ds.yaml 
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
-      nodeSelector:
-        logcollecting: "on"
      containers:
      - name: filebeat # pod容器名
        image: ikubernetes/filebeat:5.6.5-alpine # 镜像
        env: # 环境变量, 云原生使用变量。 非云原生使用entrypoint脚本处理
        - name: REDIS_HOST  # 变量名
          value: db.ikubernetes.io:6379      # 值
        - name: LOG_LEVEL
          value: info
[root@master ~]# kubectl apply -f Kubernetes_Advanced_Practical/chapter5/filebeat-ds.yaml
daemonset.apps/filebeat-ds created
[root@master ~]# kubectl get pods -n prod -o wide -l app=filebeat
NAME                READY   STATUS    RESTARTS   AGE   IP            NODE                NOMINATED NODE   READINESS GATES
filebeat-ds-4zfmg   1/1     Running   0          16s   10.244.1.11   node01.magedu.com   <none>           <none>
filebeat-ds-8gljp   1/1     Running   0          16s   10.244.2.14   node02.magedu.com   <none>           <none>
+# 可以看到不到调度到节点3
```

现在容忍污点

```diff
[root@master ~]# cat Kubernetes_Advanced_Practical/chapter5/filebeat-ds.yaml
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
+      tolerations:
+      - key: ingress
+        operator: Equal
+        value: nginx
+        effect: NoSchedule
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
[root@master ~]# kubectl apply -f Kubernetes_Advanced_Practical/chapter5/filebeat-ds.yaml
```

```diff
[root@master ~]# kubectl get pods -n prod -o wide -l app=filebeat
NAME                READY   STATUS              RESTARTS   AGE     IP            NODE                NOMINATED NODE   READINESS GATES
filebeat-ds-8gljp   1/1     Running             0          2m42s   10.244.2.14   node02.magedu.com   <none>           <none>
filebeat-ds-hvrqm   0/1     ContainerCreating   0          1s      <none>        node01.magedu.com   <none>           <none>
+# 节点3已经调度
filebeat-ds-k6pc6   1/1     Running             0          17s     10.244.3.8    node03.magedu.com   <none>           <none>
```

#### Noschedule 污点变动 

修改污点，Noschedule无影响。

```diff
[root@master ~]# kubectl taint node node03.magedu.com ingress=controller:NoSchedule --overwrite
node/node03.magedu.com modified

+# 现在这个污点和filebeat容忍不一样，由于效用是noschedule不会驱离
[root@master ~]# kubectl get pods -n prod -o wide -l app=filebeat
NAME                READY   STATUS    RESTARTS   AGE     IP            NODE                NOMINATED NODE   READINESS GATES
filebeat-ds-hvrqm   1/1     Running   0          2m50s   10.244.1.12   node01.magedu.com   <none>           <none>
+filebeat-ds-k6pc6   1/1     Running   0          3m6s    10.244.3.8    node03.magedu.com   <none>           <none>
filebeat-ds-v78dl   1/1     Running   0          2m41s   10.244.2.15   node02.magedu.com   <none>           <none>
```

#### NoExecute 污点变动，驱离

修改污点 ，noexecute才有影响

生产Pod不应该立即驱离，应该给tolerationseconds.

```diff
[root@master ~]# kubectl taint node node03.magedu.com ingress=controller:NoExecute --overwrite
node/node03.magedu.com modified
[root@master ~]# kubectl get pods -n prod -o wide -l app=filebeat
NAME                READY   STATUS        RESTARTS   AGE     IP            NODE                NOMINATED NODE   READINESS GATES
filebeat-ds-hvrqm   1/1     Running       0          3m17s   10.244.1.12   node01.magedu.com   <none>           <none>
+# 已经在驱离pod
+filebeat-ds-k6pc6   1/1     Terminating   0          3m33s   10.244.3.8    node03.magedu.com   <none>           <none>
filebeat-ds-v78dl   1/1     Running       0          3m8s    10.244.2.15   node02.magedu.com   <none>           <none>

```

### 清理key

```bash
[root@master ~]# kubectl taint node node03.magedu.com ingress-
node/node03.magedu.com untainted

# 现在又恢复了
[root@master ~]# kubectl get pods -n prod -o wide -l app=filebeat
NAME                READY   STATUS              RESTARTS   AGE     IP            NODE                NOMINATED NODE   READINESS GATES
filebeat-ds-9qrtq   0/1     ContainerCreating   0          4s      <none>        node03.magedu.com   <none>           <none>
filebeat-ds-hvrqm   1/1     Running             0          4m59s   10.244.1.12   node01.magedu.com   <none>           <none>
filebeat-ds-v78dl   1/1     Running             0          4m50s   10.244.2.15   node02.magedu.com   <none>           <none>

```

# 自动添加taints

k8s v1.6 node controller引入表达节点问题的taints, 按需要添加taints。实现pod调度后避免pod出问题。

not-ready 节点未就绪

unreachable 不可达

out-of-disk 磁盘耗尽

memory-pressure 内存压力

disk-pressure 磁盘压力

network-unavailable 网络不可用

unschedulable 不可被调度

uninitialized 未被初始化

```bash
[root@master ~]# kubectl describe node | grep Taint
Taints:             node-role.kubernetes.io/master:NoSchedule
Taints:             <none>
Taints:             <none>
Taints:             <none>
```

daemonset系统级应用，必须容忍unreachable, not-ready实现让node就绪。







