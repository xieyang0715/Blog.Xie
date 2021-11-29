---
title: StatefulSet控制器
date: 2021-01-29 04:45:09
tags:
---



k8s基础运行单元Pod, 其他是丰富Pod。

- service: 为Pod提供访问接口，Pod自动发现；服务注册和发现使用CoreDNS总线。
- Pod Controller: 自动伸缩、发布
  - deployment(replicaset)：无状态应用的运维管理，无法界定一个节点运行几个Pod
  - daemonset:    运行系统级守护进程
  - Job, CronJob(job): 非守护进程
  - 有状态应用：
    - statefulSet  大多数为：有状态有存储
- ConfigMap， Secret：变更配置



# 为什么使用statefulset?

deploy一个管理逻辑可以管理nginx/tomcat/haproxy, ...

但是有状态应用，每一种集群维护操作不一样。

Pod崩了，不使用随意使用其他Pod的数据，每个Pod独立的存储

mysql集群，添加节点，通常为从节点，从为之前2个从之后。节点崩了，崩从的替换和崩主的替换逻辑不一样。缩容，只能从从节点减小(redis集群是数据分散的，需要先迁移数据, ...)。



# statefulset

statefulset

- [x] 固定名称
- [x] 固定存储

k8s解决不了不同应用的不同操作，你要使用statefulset, 得自已写一个清单，把成熟的运维操作逻辑和过程封装成pod模板，确保加、减Pod不会出故障。

 听上去美好，并不是最终解决方案。

早期使用有状态服务在k8s之外 ，使用有状态服务使用 externalName或endpoint定义IP。

CoreOS让用户使用任何编程语言开发代码，代码封装成熟的运维步骤，经由充份测试，封装成应用程序，应用程序叫controller（全能的运维工程师），并把这个程序以Pod形式运行在K8S之上，CoreOS 起一个通用的名称**operator**

**operator本身是pod, 只需要deployment来控制这个 operator即可**，云原生是2018关键词，很多开源项目官方有自已的operator, redis-operator, mysql-operator, ...。就算官方没有，第3方人员也开发了operator



operator内部也是statefulset，用户更快的开发operator, **coreos**让用户更快开发operator, **Operator SDK**

> coreos被redhat收购，ibm收购了redhat。



查看 operator 列表

![image-20210129140239095](http://myapp.img.mykernel.cn/image-20210129140239095.png)

使用operator比自已写statefulset和一段代码，可靠的多。

operator是statefulset和一段代码封装

<!--more-->

# 使用statefulset？

- 惟一网络标识
- 持久存储
- 有序部署和扩展
- 有序，删除和终止
- 有序，自动滚动更新



应用满足前提

- Pod专有的存储卷，必须有storageclass动态供给或管理事先创建的pv
- 删除statefulset或缩减规模导致pod被删除时，不应该自动删除其存储卷以确保数据安全。(删除Pvc时，pv的回收策略)
- statefulset控制器依赖于事先存储的headless service对象实现每个pod惟一标识；headless由用户手动配置



statefulset框架

![image-20210129144813511](http://myapp.img.mykernel.cn/image-20210129144813511.png)

1. pod模板
2. pvc模板
3. replicas
4. 标签选择器，关联pod
5. 建立headless service
6. pv供给
   - 动态 storageclass
   - 静态：创建的pv，不管标签，都可以使用

确保所有pod使用专用PVC, 引用专用的PV



## pod标识符

固定且惟一

由statefulset创建, 动态生成

- 有序索引, web-0, web-1, web-2. web-0宕机，重建之后，还叫web-0
- 每个Pod的FQDN：$(podname).$(service-name).$(namespace).scv.cluster.local

## pod策略

- 有序ready
- 并行pod

## 更新策略

k8s v1.7+ 支持 spec.updateStrategy

- 删除重建

- 滚动更新

  - 顺序创建、逆序更新

    > Deploy随机更新

  - 金丝雀，更新分区，deploy需要暂时。statefulset partition=3即只逆序更新4 3. 2 1 0不更新

## 创建statefulset

Pv

- 动态供给
- 事先准备pv

1. pod模块
2. pvc模板
3. replicas
4. 标签选择器，关联pod
5. 建立headless service

```diff
[root@master chapter9]# cat statefulset-demo.yaml
apiVersion: v1
kind: Namespace
metadata:
+  name: sts
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-sts-svc # service名, 和statefulset调用的名一致
  # 不指定在default名称空间
  # 方便测试使用sts
+  namespace: sts
  labels:
    app: myapp
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None # None headless service
+  selector: # 选择pod, 一定是statefulset使用pod模板创建pod的标签
    app: myapp-pod
    controller: sts # 避免冲突
---
apiVersion: apps/v1
kind: StatefulSet 
metadata:
  name: statefulset-demo
+  namespace: sts
spec:
+  selector: # 标签选择pod
    matchLabels:
      app: myapp-pod 
      controller: sts # 避免冲突
+  serviceName: "myapp-sts-svc"     # 定义无头服务的服务名
  replicas: 2     # 副本数
+  template: # Pod模板
    metadata:
+      labels: # 标签吻合label selector
        app: myapp-pod 
+        controller: sts # 避免冲突
    spec:
+      terminationGracePeriodSeconds: 10    # 终止宽限期，默认30s
      containers: # 容器
      - name: myapp
+        image: ikubernetes/myapp:v1       # 无状态模拟有状态
        ports:
        - containerPort: 80
          name: web
+        volumeMounts: # 存储卷
        - name: myapp-pvc
          mountPath: /usr/share/nginx/html
+  volumeClaimTemplates: # PVC模板
  - metadata:
+      name: myapp-pvc # 名字
    spec:
+      accessModes: [ "ReadWriteOnce" ] # 创建每个pvc访问模式
+      #storageClassName: "gluster-dynamic" # 存储类，动态供给pvc；没有存储类，注释
      resources: # 期望多大
        requests:
          storage: 2Gi


[root@master chapter9]# kubectl apply -f statefulset-demo.yaml
namespace/sts created
service/myapp-sts-svc created
statefulset.apps/statefulset-demo created


# 由于没有pv， 所以一直启动不了，就需要创建几个集群级别的pv，只要满足pvc模板定义的大小，就可以实现“静态供给”
[root@master chapter9]# kubectl get pod -n sts
NAME                 READY   STATUS    RESTARTS   AGE
statefulset-demo-0   0/1     Pending   0          110s

```

### pv静态供给

```bash
# nfs输出5个目录
[root@node04 ~]# exportfs  -arv
exporting 172.16.0.0/16:/vols
[root@node04 ~]# install -dv /vols/
1/ 2/ 3/ 4/ 5/ 
```

```diff
[root@master ~]# cat ./Kubernetes_Advanced_Practical/chapter7/pv-nfs-0001.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
+  name: pv-nfs-1
  labels:
    storsys: nfs
    release: stable
spec:
  capacity:      # 存储容量
    storage: 5Gi # 大小，不定义, 取决于底层存储空间大小
  volumeMode: Filesystem # 访问存储设备的接口: 块接口或文件系统接口
  accessModes: # 访问模型
    - ReadWriteOnce # 
    - ReadWriteMany
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain     # 回收策略，删除pvc, pv怎么办
  #storageClassName: slow # 存储类
  mountOptions:  # pvc的选项。pv挂载到容器时，关联的选项; 需要才定义。
    - hard
    - nfsvers=4.1
  nfs: # 存储系统的特性
+    path:  "/vols/1"  # 
    server: 172.16.1.104
---
apiVersion: v1
kind: PersistentVolume
metadata:
+  name: pv-nfs-2
  labels:
    storsys: nfs
    release: stable
spec:
  capacity:      # 存储容量
    storage: 5Gi # 大小，不定义, 取决于底层存储空间大小
  volumeMode: Filesystem # 访问存储设备的接口: 块接口或文件系统接口
  accessModes: # 访问模型
    - ReadWriteOnce # 
    - ReadWriteMany
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain     # 回收策略，删除pvc, pv怎么办
  #storageClassName: slow # 存储类
  mountOptions:  # pvc的选项。pv挂载到容器时，关联的选项; 需要才定义。
    - hard
    - nfsvers=4.1
  nfs: # 存储系统的特性
+    path:  "/vols/2"  # 
    server: 172.16.1.104
---
apiVersion: v1
kind: PersistentVolume
metadata:
+  name: pv-nfs-3
  labels:
    storsys: nfs
    release: stable
spec:
  capacity:      # 存储容量
    storage: 5Gi # 大小，不定义, 取决于底层存储空间大小
  volumeMode: Filesystem # 访问存储设备的接口: 块接口或文件系统接口
  accessModes: # 访问模型
    - ReadWriteOnce # 
    - ReadWriteMany
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain     # 回收策略，删除pvc, pv怎么办
  #storageClassName: slow # 存储类
  mountOptions:  # pvc的选项。pv挂载到容器时，关联的选项; 需要才定义。
    - hard
    - nfsvers=4.1
  nfs: # 存储系统的特性
+    path:  "/vols/3"  # 
    server: 172.16.1.104
---
apiVersion: v1
kind: PersistentVolume
metadata:
+  name: pv-nfs-4
  labels:
    storsys: nfs
    release: stable
spec:
  capacity:      # 存储容量
    storage: 5Gi # 大小，不定义, 取决于底层存储空间大小
  volumeMode: Filesystem # 访问存储设备的接口: 块接口或文件系统接口
  accessModes: # 访问模型
    - ReadWriteOnce # 
    - ReadWriteMany
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain     # 回收策略，删除pvc, pv怎么办
  #storageClassName: slow # 存储类
  mountOptions:  # pvc的选项。pv挂载到容器时，关联的选项; 需要才定义。
    - hard
    - nfsvers=4.1
  nfs: # 存储系统的特性
+    path:  "/vols/4"  # 
    server: 172.16.1.104
---
apiVersion: v1
kind: PersistentVolume
metadata:
+  name: pv-nfs-5
  labels:
    storsys: nfs
    release: stable
spec:
  capacity:      # 存储容量
+    storage: 1Gi # 为了演示效果将5Gi修改为1Gi就不符合我们的statefulset pvc模板的需要，一定不会绑定这个
  volumeMode: Filesystem # 访问存储设备的接口: 块接口或文件系统接口
  accessModes: # 访问模型
    - ReadWriteOnce # 
    - ReadWriteMany
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain     # 回收策略，删除pvc, pv怎么办
  #storageClassName: slow # 存储类
  mountOptions:  # pvc的选项。pv挂载到容器时，关联的选项; 需要才定义。
    - hard
    - nfsvers=4.1
  nfs: # 存储系统的特性
+    path:  "/vols/5"  # 
    server: 172.16.1.104

```

```bash
[root@master ~]# kubectl apply -f ./Kubernetes_Advanced_Practical/chapter7/pv-nfs-0001.yaml
persistentvolume/pv-nfs-1 created
persistentvolume/pv-nfs-2 created
persistentvolume/pv-nfs-3 created
persistentvolume/pv-nfs-4 created
persistentvolume/pv-nfs-5 created

[root@master ~]# kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                              STORAGECLASS   REASON   AGE
																  # 绑定后，自动绑定 生成pvc为 pvc模板名-statefulset名-index
pv-nfs-1   5Gi        RWO,ROX,RWX    Retain           Bound       sts/myapp-pvc-statefulset-demo-1                           21s
pv-nfs-2   5Gi        RWO,ROX,RWX    Retain           Available                                                              21s
																  # 绑定后，自动绑定 生成pvc为 pvc模板名-statefulset名-index
pv-nfs-3   5Gi        RWO,ROX,RWX    Retain           Bound       sts/myapp-pvc-statefulset-demo-0                           21s
pv-nfs-4   5Gi        RWO,ROX,RWX    Retain           Available                                                              21s
pv-nfs-5   1Gi        RWO,ROX,RWX    Retain           Available                                                              21s

# sts正常
[root@master ~]# kubectl get sts -n sts
NAME               READY   AGE
statefulset-demo   2/2     9m49s

# sts名称空间中的所有资源
[root@master ~]# kubectl get all -n sts
# pod
NAME                     READY   STATUS    RESTARTS   AGE
pod/statefulset-demo-0   1/1     Running   0          11m
pod/statefulset-demo-1   1/1     Running   0          109s

# service, headless
NAME                    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
									# headless service
service/myapp-sts-svc   ClusterIP   None         <none>        80/TCP    11m

# statefulset控制器
NAME                                READY   AGE
statefulset.apps/statefulset-demo   2/2     11m

```



## 扩容statefulset

```bash
# 终端1
[root@master ~]# kubectl get pod -n sts -w
NAME                 READY   STATUS    RESTARTS   AGE
statefulset-demo-0   1/1     Running   0          14m
statefulset-demo-1   1/1     Running   0          5m9s

# 终端2
# kubectl scale -h
 kubectl scale --replicas=4 sts statefulset-demo -n sts

# 终端1观察
statefulset-demo-2   0/1     Pending   0          0s
statefulset-demo-2   0/1     Pending   0          0s
statefulset-demo-2   0/1     Pending   0          2s
statefulset-demo-2   0/1     ContainerCreating   0          2s
statefulset-demo-2   1/1     Running             0          5s
statefulset-demo-3   0/1     Pending             0          0s
statefulset-demo-3   0/1     Pending             0          0s
statefulset-demo-3   0/1     Pending             0          3s
statefulset-demo-3   0/1     ContainerCreating   0          3s
statefulset-demo-3   1/1     Running             0          6s
#是有序添加 -2, -3, ...


# 查看pv绑定状态
[root@master chapter9]# kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                              STORAGECLASS   REASON   AGE
pv-nfs-1   5Gi        RWO,ROX,RWX    Retain           Bound       sts/myapp-pvc-statefulset-demo-1                           7m21s
pv-nfs-2   5Gi        RWO,ROX,RWX    Retain           Bound       sts/myapp-pvc-statefulset-demo-3                           7m21s
pv-nfs-3   5Gi        RWO,ROX,RWX    Retain           Bound       sts/myapp-pvc-statefulset-demo-0                           7m21s
pv-nfs-4   5Gi        RWO,ROX,RWX    Retain           Bound       sts/myapp-pvc-statefulset-demo-2                           7m21s
													  # 不会绑定这个，因为大小不符合
pv-nfs-5   1Gi        RWO,ROX,RWX    Retain           Available                                                              7m21s
```

## 升级版本

滚动更新

逆序更新

```bash
# 终端1监控pod
[root@master chapter9]# kubectl get pod -n sts -w



# kubectl set image -h
# kubectl set image sts/statefulset-demo myapp=ikubernetes/myapp:v2 -n sts
```

查看更新过程

```bash
# 由kubectl get pod -n sts -w 命令的结果
statefulset-demo-3   1/1     Terminating         0          2m10s
statefulset-demo-3   0/1     Terminating         0          2m11s
statefulset-demo-3   0/1     Terminating         0          2m18s
statefulset-demo-3   0/1     Terminating         0          2m18s
statefulset-demo-3   0/1     Pending             0          0s
statefulset-demo-3   0/1     Pending             0          0s
statefulset-demo-3   0/1     ContainerCreating   0          0s
statefulset-demo-3   1/1     Running             0          3s
statefulset-demo-2   1/1     Terminating         0          2m26s
statefulset-demo-2   0/1     Terminating         0          2m27s
statefulset-demo-2   0/1     Terminating         0          2m33s
statefulset-demo-2   0/1     Terminating         0          2m33s
statefulset-demo-2   0/1     Pending             0          0s
statefulset-demo-2   0/1     Pending             0          0s
statefulset-demo-2   0/1     ContainerCreating   0          0s
statefulset-demo-2   1/1     Running             0          3s
statefulset-demo-1   1/1     Terminating         0          8m47s
statefulset-demo-1   0/1     Terminating         0          8m48s
statefulset-demo-1   0/1     Terminating         0          8m54s
statefulset-demo-1   0/1     Terminating         0          8m54s
statefulset-demo-1   0/1     Pending             0          0s
statefulset-demo-1   0/1     Pending             0          0s
statefulset-demo-1   0/1     ContainerCreating   0          0s
# 可以看出是逆向更新
```

详情

```bash
# kubectl describe sts statefulset-demo -n sts
Update Strategy:    RollingUpdate
  Partition:        0 # 逆向更新至0个pod
```

### 分区更新

逆向更新至pod号partition的Pod



#### 金丝雀发布

仅放出一个pod为新版本

由于一共4个pod, partition最大为3，所以partition=3, 则逆向更新至statefulset-demo-3

```bash
[root@master chapter9]# kubectl explain sts.spec.updateStrategy.rollingUpdate.partition
KIND:     StatefulSet
VERSION:  apps/v1

FIELD:    partition <integer>

DESCRIPTION:
     Partition indicates the ordinal at which the StatefulSet should be
     partitioned. Default value is 0.
```

```diff
"statefulset-demo.yaml" 60L, 1633C                                                                                                                                                                                                                                                                       60,11         Bot
kind: Service
metadata:
  name: myapp-sts-svc # service名, 和statefulset调用的名一致
  # 不指定在default名称空间
  # 方便测试使用sts
  namespace: sts
  labels:
    app: myapp
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None # None headless service
  selector: # 选择pod, 一定是statefulset使用pod模板创建pod的标签
    app: myapp-pod
    controller: sts # 避免冲突
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: statefulset-demo
  namespace: sts
spec:
  selector: # 标签选择pod
    matchLabels:
      app: myapp-pod
      controller: sts # 避免冲突
  serviceName: "myapp-sts-svc"     # 定义无头服务的服务名
+  replicas: 4     # 刚刚扩容后，这个参数没有修改
+  updateStrategy:
+    type: RollingUpdate
+    rollingUpdate:
+      partition: 3 # 逆序更新至第3个pod
  template: # Pod模板
    metadata:
      labels: # 标签吻合label selector
        app: myapp-pod
        controller: sts # 避免冲突
    spec:
      terminationGracePeriodSeconds: 10    # 终止宽限期，默认30s
      containers: # 容器
      - name: myapp
        image: ikubernetes/myapp:v1       # 无状态模拟有状态
        ports:
        - containerPort: 80
          name: web
        volumeMounts: # 存储卷
        - name: myapp-pvc
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates: # PVC模板
  - metadata:
      name: myapp-pvc # 名字
    spec:
      accessModes: [ "ReadWriteOnce" ] # 创建每个pvc访问模式

```

> 每次更新yaml文件，apply麻烦
>
> 1. kubectl edit sts statefulset -n sts
> 2. kubectl patch

```bash
[root@master chapter9]# kubectl apply -f statefulset-demo.yaml
namespace/sts unchanged
service/myapp-sts-svc unchanged
statefulset.apps/statefulset-demo configured


# 分区更新
[root@master chapter9]# kubectl get pod -n sts -w
NAME                 READY   STATUS    RESTARTS   AGE
statefulset-demo-0   1/1     Running   0          5m24s
statefulset-demo-1   1/1     Running   0          5m34s

# 终端1监控pod
[root@master chapter9]# kubectl get pod -n sts -w

# 终端2更新pod
# kubectl set image sts/statefulset-demo myapp=ikubernetes/myapp:v3 -n sts

# 仅逆向更新至pod号partition的Pod
[root@master ~]# kubectl get pod -n sts -w
NAME                 READY   STATUS    RESTARTS   AGE
statefulset-demo-0   1/1     Running   0          7m40s
statefulset-demo-1   1/1     Running   0          7m50s
statefulset-demo-2   1/1     Running   0          95s
statefulset-demo-3   1/1     Running   0          92s
statefulset-demo-3   1/1     Terminating   0          98s
statefulset-demo-3   0/1     Terminating   0          99s

```



#### 全部更新

过一会儿金丝雀正常后, 就可以全部更新

```diff
# kubectl edit sts/statefulset-demo -n sts
  updateStrategy:
    rollingUpdate:
+      partition: 0 # 更新至第0个pod
```

> 也可以修改partition为2，过一会在更新一个，过一会在修改在更新

另一个终端的更新结果

```bash
statefulset-demo-2   1/1     Terminating         0          5m55s
statefulset-demo-2   0/1     Terminating         0          5m57s
statefulset-demo-2   0/1     Terminating         0          6m5s
statefulset-demo-2   0/1     Terminating         0          6m5s
statefulset-demo-2   0/1     Pending             0          0s
statefulset-demo-2   0/1     Pending             0          0s
statefulset-demo-2   0/1     ContainerCreating   0          0s
statefulset-demo-2   1/1     Running             0          3s
statefulset-demo-1   1/1     Terminating         0          12m
statefulset-demo-1   0/1     Terminating         0          12m
statefulset-demo-1   0/1     Terminating         0          12m
statefulset-demo-1   0/1     Terminating         0          12m
statefulset-demo-1   0/1     Pending             0          0s
statefulset-demo-1   0/1     Pending             0          0s
statefulset-demo-1   0/1     ContainerCreating   0          0s
statefulset-demo-1   1/1     Running             0          15s
statefulset-demo-0   1/1     Terminating         0          12m
statefulset-demo-0   0/1     Terminating         0          12m
statefulset-demo-0   0/1     Terminating         0          12m
statefulset-demo-0   0/1     Terminating         0          12m
statefulset-demo-0   0/1     Pending             0          0s
statefulset-demo-0   0/1     Pending             0          0s
statefulset-demo-0   0/1     ContainerCreating   0          0s
statefulset-demo-0   1/1     Running             0          2s
```

## 缩容

pod删除，pvc不会被删除

即使手工删除pvc, pv回收策略为retain, pv也存在

将来在扩容还是挂载相同的pvc, 也存在

```bash
[root@master chapter9]# kubectl exec -it -n sts statefulset-demo-3 -- sh
/ # touch /usr/share/nginx/html/magedu.txt # 需要在挂载点弄个文件
/ # sync
/ # exit

[root@master chapter9]# kubectl scale sts statefulset-demo -n sts --replicas=3
statefulset.apps/statefulset-demo scaled
[root@master chapter9]# kubectl get pvc -n sts
NAME                           STATUS    VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myapp-pvc-statefulset-demo-0   Bound     pv-nfs-3   5Gi        RWO,ROX,RWX                   33m
myapp-pvc-statefulset-demo-1   Bound     pv-nfs-1   5Gi        RWO,ROX,RWX                   23m
myapp-pvc-statefulset-demo-2   Bound     pv-nfs-4   5Gi        RWO,ROX,RWX                   17m
myapp-pvc-statefulset-demo-3   Bound     pv-nfs-2   5Gi        RWO,ROX,RWX                   17m
myapp-pvc-statefulset-demo-4   Pending                                                       8m29s

```

扩容

```bash
# kubectl scale sts statefulset-demo -n sts --replicas=4

# 观察到pod正常
statefulset-demo-3   1/1     Running             0          2s

# 可以看到即使pod删除，pvc数据还在，以后pod恢复数据还可以使用
[root@master chapter9]# kubectl exec -it -n sts statefulset-demo-3 -- sh
/ # ls /usr/share/nginx/html/
dump.rdb    magedu.txt

```

## 删除statefulset

```bash
kubectl delete sts statefulset-demo -n sts
```



# 管理有状态应用

## etcd

1. pod模板
2. pvc模板
3. replicas
4. 标签选择器，关联pod
5. 建立headless service
6. pv供给
   - 动态 storageclass
   - 静态：创建的pv，不管标签，都可以使用

etcd将来可以扩展，但是etcd必须是奇数个节点

```diff
[root@master chapter9]# vim etcd-statefulset.yaml
# headless service
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd
  labels:
    app: etcd
spec:
  serviceName: etcd
  # changing replicas value will require a manual etcdctl member remove/add
  #   # command (remove before decreasing and add after increasing)
  replicas: 3
  selector:
    matchLabels:
      app: etcd-member
+  template: # Pod模板
    metadata:
      name: etcd
      labels:
        app: etcd-member
    spec:
      containers:
      - name: etcd
        image: "quay.io/coreos/etcd:v3.2.16"
        ports:
        - containerPort: 2379
          name: client
        - containerPort: 2380
          name: peer
        env:
        - name: CLUSTER_SIZE
          value: "3"
        - name: SET_NAME
          value: "etcd"
        volumeMounts:
        - name: data
          mountPath: /var/run/etcd
        command:
          - "/bin/sh"
          - "-ecx"
          - |
            IP=$(hostname -i)
            PEERS=""
            for i in $(seq 0 $((${CLUSTER_SIZE} - 1))); do     # 创建集群时，先判断集群是否存储, 非常复杂. statefulset可能没有备份，需要operator
                PEERS="${PEERS}${PEERS:+,}${SET_NAME}-${i}=http://${SET_NAME}-${i}.${SET_NAME}:2380"
            done
            # start etcd. If cluster is already initialized the `--initial-*` options will be ignored.
            exec etcd --name ${HOSTNAME} \
              --listen-peer-urls http://${IP}:2380 \
              --listen-client-urls http://${IP}:2379,http://127.0.0.1:2379 \
              --advertise-client-urls http://${HOSTNAME}.${SET_NAME}:2379 \
              --initial-advertise-peer-urls http://${HOSTNAME}.${SET_NAME}:2380 \
              --initial-cluster-token etcd-cluster-1 \
              --initial-cluster ${PEERS} \
"etcd-statefulset.yaml" 100L, 2670C                                                                                                                                                                                                                                                                      44,13         Top
# headless service
            exec etcd --name ${HOSTNAME} \
              --listen-peer-urls http://${IP}:2380 \
              --listen-client-urls http://${IP}:2379,http://127.0.0.1:2379 \
              --advertise-client-urls http://${HOSTNAME}.${SET_NAME}:2379 \
              --initial-advertise-peer-urls http://${HOSTNAME}.${SET_NAME}:2380 \
              --initial-cluster-token etcd-cluster-1 \
              --initial-cluster ${PEERS} \
              --initial-cluster-state new \
              --data-dir /var/run/etcd/default.etcd
+  volumeClaimTemplates: # PVC模板
  - metadata:
      name: data
    spec:
      storageClassName: gluster-dynamic
      accessModes:
        - "ReadWriteOnce"
      resources:
        requests:
          storage: 1Gi
---
+# headless service
apiVersion: v1
kind: Service
metadata:
  name: etcd
  labels:
    app: etcd
  annotations:
    # Create endpoints also if the related pod isn't ready
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
spec:
  ports:
  - port: 2379
    name: client
  - port: 2380
    name: peer
  clusterIP: None
  selector:
    app: etcd-member
   
---
+ # clusterip
apiVersion: v1
kind: Service
metadata:
  name: etcd-client
  labels:
    app: etcd
spec:
  ports:
  - name: etcd-client
    port: 2379
    protocol: TCP
    targetPort: 2379
  selector:
    app: etcd-member

```

### 静态供给pv

静态创建几个pv

```diff
[root@node04 ~]# mkdir -pv /vols/{6..10}

[root@master ~]# cat ./Kubernetes_Advanced_Practical/chapter7/pv-nfs-0001.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
+  name: pv-nfs-6
  labels:
    storsys: nfs
    release: stable
spec:
  capacity:      # 存储容量
    storage: 5Gi # 大小，不定义, 取决于底层存储空间大小
  volumeMode: Filesystem # 访问存储设备的接口: 块接口或文件系统接口
  accessModes: # 访问模型
    - ReadWriteOnce # 
    - ReadWriteMany
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain     # 回收策略，删除pvc, pv怎么办
  #storageClassName: slow # 存储类
  mountOptions:  # pvc的选项。pv挂载到容器时，关联的选项; 需要才定义。
    - hard
    - nfsvers=4.1
  nfs: # 存储系统的特性
+    path:  "/vols/6"  # 
    server: 172.16.1.104
---
apiVersion: v1
kind: PersistentVolume
metadata:
+  name: pv-nfs-7
  labels:
    storsys: nfs
    release: stable
spec:
  capacity:      # 存储容量
    storage: 5Gi # 大小，不定义, 取决于底层存储空间大小
  volumeMode: Filesystem # 访问存储设备的接口: 块接口或文件系统接口
  accessModes: # 访问模型
    - ReadWriteOnce # 
    - ReadWriteMany
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain     # 回收策略，删除pvc, pv怎么办
  #storageClassName: slow # 存储类
  mountOptions:  # pvc的选项。pv挂载到容器时，关联的选项; 需要才定义。
    - hard
    - nfsvers=4.1
  nfs: # 存储系统的特性
+    path:  "/vols/7"  # 
    server: 172.16.1.104
---
apiVersion: v1
kind: PersistentVolume
metadata:
+  name: pv-nfs-8
  labels:
    storsys: nfs
    release: stable
spec:
  capacity:      # 存储容量
    storage: 5Gi # 大小，不定义, 取决于底层存储空间大小
  volumeMode: Filesystem # 访问存储设备的接口: 块接口或文件系统接口
  accessModes: # 访问模型
    - ReadWriteOnce # 
    - ReadWriteMany
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain     # 回收策略，删除pvc, pv怎么办
  #storageClassName: slow # 存储类
  mountOptions:  # pvc的选项。pv挂载到容器时，关联的选项; 需要才定义。
    - hard
    - nfsvers=4.1
  nfs: # 存储系统的特性
+    path:  "/vols/8"  # 
    server: 172.16.1.104
---
apiVersion: v1
kind: PersistentVolume
metadata:
+  name: pv-nfs-9
  labels:
    storsys: nfs
    release: stable
spec:
  capacity:      # 存储容量
    storage: 5Gi # 大小，不定义, 取决于底层存储空间大小
  volumeMode: Filesystem # 访问存储设备的接口: 块接口或文件系统接口
  accessModes: # 访问模型
    - ReadWriteOnce # 
    - ReadWriteMany
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain     # 回收策略，删除pvc, pv怎么办
  #storageClassName: slow # 存储类
  mountOptions:  # pvc的选项。pv挂载到容器时，关联的选项; 需要才定义。
    - hard
    - nfsvers=4.1
  nfs: # 存储系统的特性
+    path:  "/vols/9"  # 
    server: 172.16.1.104
---
apiVersion: v1
kind: PersistentVolume
metadata:
+  name: pv-nfs-10
  labels:
    storsys: nfs
    release: stable
spec:
  capacity:      # 存储容量
    storage: 1Gi # 大小，不定义, 取决于底层存储空间大小
  volumeMode: Filesystem # 访问存储设备的接口: 块接口或文件系统接口
  accessModes: # 访问模型
    - ReadWriteOnce # 
    - ReadWriteMany
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain     # 回收策略，删除pvc, pv怎么办
  #storageClassName: slow # 存储类
  mountOptions:  # pvc的选项。pv挂载到容器时，关联的选项; 需要才定义。
    - hard
    - nfsvers=4.1
  nfs: # 存储系统的特性
+    path:  "/vols/10"  # 
    server: 172.16.1.104

```

```bash
[root@master ~]# kubectl apply -f ./Kubernetes_Advanced_Practical/chapter7/pv-nfs-0001.yaml
persistentvolume/pv-nfs-6 created
persistentvolume/pv-nfs-7 created
persistentvolume/pv-nfs-8 created
persistentvolume/pv-nfs-9 created
persistentvolume/pv-nfs-10 created
[root@master ~]# kubectl get pv
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                              STORAGECLASS   REASON   AGE
pv-nfs-1    5Gi        RWO,ROX,RWX    Retain           Bound       sts/myapp-pvc-statefulset-demo-1                           37m
pv-nfs-10   1Gi        RWO,ROX,RWX    Retain           Available                                                              1s
pv-nfs-2    5Gi        RWO,ROX,RWX    Retain           Bound       sts/myapp-pvc-statefulset-demo-3                           37m
pv-nfs-3    5Gi        RWO,ROX,RWX    Retain           Bound       sts/myapp-pvc-statefulset-demo-0                           37m
pv-nfs-4    5Gi        RWO,ROX,RWX    Retain           Bound       sts/myapp-pvc-statefulset-demo-2                           37m
pv-nfs-5    1Gi        RWO,ROX,RWX    Retain           Available                                                              37m
pv-nfs-6    5Gi        RWO,ROX,RWX    Retain           Available                                                              1s
pv-nfs-7    5Gi        RWO,ROX,RWX    Retain           Available                                                              1s
pv-nfs-8    5Gi        RWO,ROX,RWX    Retain           Available                                                              1s
pv-nfs-9    5Gi        RWO,ROX,RWX    Retain           Available                                                              1s

```

启动etcd

```bash
[root@master chapter9]# kubectl apply -f etcd-statefulset.yaml 
statefulset.apps/etcd created
service/etcd created
service/etcd-client created
```

## operator运行etcd

任何有状态应用的管理，第1点想到operator, 找稳定版本的operator。就算开发版的operator也比我们自已开发的靠谱的多。beta版本建议使用。alpha不建议使用。

https://github.com/operator-framework/awesome-operators

进入etcd-operator https://github.com/coreos/etcd-operator 

coreos提供生产环境可用

example目录 https://github.com/coreos/etcd-operator/tree/master/example

https://github.com/coreos/etcd-operator/blob/master/example/deployment.yaml

```diff
apiVersion: extensions/v1beta1
+kind: Deployment # 注意operator使用deploy启用
metadata:
  name: etcd-operator
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: etcd-operator
    spec:
      containers:
      - name: etcd-operator
+        image: quay.io/coreos/etcd-operator:v0.9.4 # 自定义的controller专用来维护etcd集群，每个集群就是一个operator实例
        command:
        - etcd-operator
        # Uncomment to act for resources in all namespaces. More information in doc/user/clusterwide.md
        #- -cluster-wide
        env:
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

部署之后，使用创建etcd集群

https://github.com/coreos/etcd-operator/blob/master/example/example-etcd-cluster.yaml

```diff
apiVersion: "etcd.database.coreos.com/v1beta2"
+kind: "EtcdCluster" # 控制器部署时引入CRD自定义资源类型，每个集群为这个类的实例
metadata:
  name: "example-etcd-cluster"
  ## Adding this annotation make this cluster managed by clusterwide operators
  ## namespaced operators ignore it
  # annotations:
  #   etcd.database.coreos.com/scope: clusterwide
spec:
+  size: 3 # 集群规模
+  version: "3.2.13" # 集群版本
```



## 部署redis operator

https://github.com/ucloud/redis-cluster-operator

至少2G内存3个节点





