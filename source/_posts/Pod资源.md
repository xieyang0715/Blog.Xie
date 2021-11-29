---
title: Pod资源
date: 2021-01-18 03:09:29
tags:
---



- [x] pod uts, ipc, network
- [x] side car, 代理
- [x] 服务注册、服务发现
- [x] pod暴露3种方式
  - [x] 入口80和443端口监控
  - [x] 自动创建入口
- [x] ingress https -> http pod
- [x] pod Label, Label Selector
- [x] 注释
- [x] pod生命周期
- [x] pod相位
- [x] pod创建pod过程
- [x] 容器重启策略
- [x] pod终止过程
- [x] pod安全上下文
- [x] pod优先级
- [x] pod容器资源限制、pod服务质量



<!--more-->

# pod架构

k8s主要目标是：通过pod运行容器

pod内部可以运行1到多个容器，共享网络名称空间，ipc, uts

pod有一个`pause`的基础架构容器，一直处于暂停状态，`pause`容器为其他加入的容器提供`IPC, Network, UTS`, 每个容器有自已的PID, MOUNT, USER. 多个pod中可以共享其他名称空间。

- [x] 共享Network: lo接口通信
- [x] 共享UTS: 同主机名，ip地址
- [x] 共享mount: 多个pod挂载底层pause的存储卷

pod 更像一个虚拟机，让多个容器结合更紧密。

通常让pod只运行一个`主容器`，让pod运行多个容器的目的，是单一容器无法完成多个任务，比如：pod运行`nginx`，这时要收集nginx的日志就需要额外的日志收集代理`fluentd, filebeat`。这里的辅助容器也叫**边车容器**(side car)。



**代理容器**：主容器提供的服务，不直接访问。而流量进入时，通过辅助容器接入流量，代理给主容器。`istio`工具，实现网格化，给主容器添加前端代理。可以按需链路侦测，流量调度。



# pod服务注册和发现

K8S中部署Nginx, MariaDB, Tomcat. 除了nginx都不需要暴露. t只访问m. n只访问t.

- [x] host network： 与外部网络接口通信。
- [x] service network: iptables/ipvs规则。pod是动态的，直接使用地址通信并不理想，应该使用service通信。
- [x] pod network: flannel插件提供的网络，仅内部网络。pod间可以通信



NMT静态配置没有问题。

但是在k8s之上，pod是动态的(删除、重启、...），所以得加一个前端(service)。 所以NMT有3个service。每一层得转发一次，多一次，性能有影响。**动态逻辑中，必须有一个机制帮助实现服务注册、服务发现。service就是帮助把服务注册到k8s之上的服务总线(dns)，动态注册和更新的dns服务**。

>  dns有3个版本
>
> - [x] skydns
> - [x] kubedns
> - [x] coredns: 比kubedns更有优势，k8s v1.12版本默认coredns.



不应该手工创建自主式pod, 宕机不会恢复，所以应该使用控制器管理pod. 控制器才是k8s核心，通过和解循环让pod的当前状态和目标状态一致。控制器完成了发布、变更、故障处理，可以当成一个运维工程师。

> 不需要运维，有危机。但是这个世界机遇和挑战并存的，有危机就需要换技术环境，换工具的时代。跟的上,,, 跟不上就淘汰。



# 用户访问pod

pod创建默认地址是，flannel地址，不能被集群外访问。需要暴露给互联网用户访问，怎么将外部流量在必要时接入集群内部？

- [x] service nodePort                      每个节点打开DNAT -> service-> 容器
- [x] containers.port.hostPort        pod所在节点打开DNAT
- [x] hostNetwork                              pod所在节点容器运行的端口



service nodePort需要监听在30000-60000的端口，用户访问的是80/443, 所以需要在k8s前端添加负载均衡器。所以必要时需要keepalived高可用前端的nginx， 并且监控前端的nginx.

这样映射层转换4层: `request -> external lb -> service -> pod`

需要人为创建外部负载均衡器，能否按需创建外部负载均衡器，可以把k8s部署云计算环境中`openstack`, 有一个LBaaS服务, 也是软件实现，需要时，自动启2个haproxy, 生成负载均衡服务。当service删除时，可以动态删除外部负载器。k8s的一个组件`cloud-controller-manager`与外部云计算环境交互的api.



围绕pod的service: iptables/ipvs规则，4层代理。不能卸载https会话。所以pod不能运行为https, 前端卸载，就需要使用ingress: haproxy/nginx



# pod标签

管理Pod动态的，service动态发现Pod, Pod故障使用Pod Controller重建。service/pod controller如何识别pod？



pod属于名称空间(目录)，同一名称空间中pod(文件)不可以同名，跨名称空间不能同名。通过label, label selector(元数据)来引用pod。service/deployment可以通过无数据来过滤pod。

>  k8s一切都是资源。



一个对象可以多个标签，一个标签可以添加到多个资源之上。

标签：版本(stable/develop)、环境(生产、研发、测试)、分层架构(前端N、后端T、数据层M)等...



键名称 = [键前缀/]键名 

- [x] 键名最多63个字符，命名与变量命名规则很像。
- [x] 前缀子域名格式。kubernetes.io/ 前缀是预留给k8s核心组件使用。
- [x] 键值不多于63个字符，要么空，要么变量命名规则一致。

![image-20210118135400399](http://myapp.img.mykernel.cn/image-20210118135400399.png)

> 示例是2个标签，可以n个。



标签选择器，2个选择器

- [x] 等值 =, ==, !=
- [x] 集合: 
  - [x] key in (v1, v2, ..)
  - [x] key not in  (v1, v2, ..)
  - [x] key 所有存在此键名标签的资源
  - [x] !key  所有不存在此键名标签的资源

同时指定多个选择器，逻辑关系 “与”

空值标签选择器，意味着key对应的每个资源对象。

空的标签选择器，不选择资源。



定义标签选择器

- [x] matchlabels: 给Key, value
- [x] matchexpresssions:  通过表达式指定 {key: , value:, operator:}

```bash
# 显示pod之上的标签
[root@master ~]# kubectl get pod --show-labels
NAME                          READY   STATUS    RESTARTS   AGE     LABELS
myapp-7d4b7b84b-986qx         1/1     Running   0          2d21h   app=myapp,pod-template-hash=7d4b7b84b
ngx-deploy-6b44c98c55-22j8m   1/1     Running   0          2d22h   app=ngx-deploy,pod-template-hash=6b44c98c55
pod-demo                      1/1     Running   0          3h14m   app=myapp

	# crate/run自动附加标签：app=控制器名  pod模板hash值
	


```



```diff
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  namespace: prod
+  labels:
+    app: pod-demo
+    rel: stable
spec:            # 属性
  containers:           # pod运行多个容器. 
  - name: bbox
    image: busybox
    imagePullPolicy: IfNotPresent
    #command: # kubectl explain pod.spec.containers.command  command <[]string>
    #- "/bin/sh"
    #- "-c"
    #- "sleep 86400"
    command: ["/bin/sh","-c","sleep 86400"] # kubectl explain pod.spec.containers.command  command <[]string>
  - image: ikubernetes/myapp:v1
    imagePullPolicy: IfNotPresent
    name: myapp

```

```bash
[root@master ~]# kubectl apply -f manifests/basic/pod-demo-2.yaml
pod/pod-demo configured

[root@master ~]# kubectl get pod -n prod --show-labels
NAME       READY   STATUS    RESTARTS   AGE     LABELS
pod-demo   2/2     Running   0          3h40m   app=pod-demo,rel=stable

```



kubectl label给资源打标签

```bash
 ~]# kubectl label -h
Examples:
  # Update pod 'foo' with the label 'unhealthy' and the value 'true'.
  kubectl label pods foo unhealthy=true
  # 更新key的原有值
  kubectl label --overwrite pods foo status=unhealthy
kubectl label [--overwrite] (-f FILENAME | TYPE NAME) KEY_1=VAL_1 ... KEY_N=VAL_N [--resource-version=version]

```

# 管理标签

```bash
# 添加标签
[root@master ~]# kubectl label pod -n prod pod-demo tier=frontend
pod/pod-demo labeled
[root@master ~]# kubectl get pod -n prod --show-labels
NAME       READY   STATUS    RESTARTS   AGE     LABELS
																		# 已经添加
pod-demo   2/2     Running   0          3h43m   app=pod-demo,rel=stable,tier=frontend

# 修改标签
[root@master ~]# kubectl label pod -n prod pod-demo app=myapp
error: 'app' already has a value (pod-demo), and --overwrite is false
[root@master ~]# kubectl label pod -n prod pod-demo app=myapp --overwrite
pod/pod-demo labeled
[root@master ~]# kubectl get pod -n prod --show-labels
NAME       READY   STATUS    RESTARTS   AGE     LABELS
pod-demo   2/2     Running   0          3h43m   app=myapp,rel=stable,tier=frontend


# 删除标签
[root@master ~]# kubectl label pod -n prod pod-demo rel-
pod/pod-demo labeled
[root@master ~]# kubectl get pod -n prod --show-labels
NAME       READY   STATUS    RESTARTS   AGE     LABELS
pod-demo   2/2     Running   0          3h44m   app=myapp,tier=frontend

```

# 依据标签显示

## -l

- [x] -l app=myapp
- [x] -l "app in (myapp,ngx-deploy)"
- [x] -l app
- [x] -l '!app'

```bash
 kubectl get
[(-o|--output=)json|yaml|wide|custom-columns=...|custom-columns-file=...|go-template=...|go-template-file=...|jsonpath=...|jsonpath-file=...]
(TYPE[.VERSION][.GROUP] [NAME | -l label] | TYPE[.VERSION][.GROUP]/NAME ...) [flags] [options]


[root@master ~]# kubectl get pod  --show-labels
NAME                          READY   STATUS    RESTARTS   AGE     LABELS
myapp-7d4b7b84b-986qx         1/1     Running   0          2d21h   app=myapp,pod-template-hash=7d4b7b84b
ngx-deploy-6b44c98c55-22j8m   1/1     Running   0          2d22h   app=ngx-deploy,pod-template-hash=6b44c98c55
pod-demo                      1/1     Running   0          3h22m   app=myapp


# 过滤
# key=value
[root@master ~]# kubectl get pod --show-labels -l app=myapp
NAME                    READY   STATUS    RESTARTS   AGE     LABELS
myapp-7d4b7b84b-986qx   1/1     Running   0          2d21h   app=myapp,pod-template-hash=7d4b7b84b
pod-demo                1/1     Running   0          3h23m   app=myapp
# key!=value
[root@master ~]# kubectl get pod --show-labels -l app!=myapp
NAME                          READY   STATUS    RESTARTS   AGE     LABELS
ngx-deploy-6b44c98c55-22j8m   1/1     Running   0          2d22h   app=ngx-deploy,pod-template-hash=6b44c98c55
# 仅过滤key对应的pod
[root@master ~]# kubectl get pod --show-labels -l app
NAME                          READY   STATUS    RESTARTS   AGE     LABELS
myapp-7d4b7b84b-986qx         1/1     Running   0          2d21h   app=myapp,pod-template-hash=7d4b7b84b
ngx-deploy-6b44c98c55-22j8m   1/1     Running   0          2d22h   app=ngx-deploy,pod-template-hash=6b44c98c55
pod-demo                      1/1     Running   0          3h23m   app=myapp
# 仅过滤不是key对应的pod
kubectl get pod --show-labels -l '!app'


# 有app键，且值是myap或pod-deploy
[root@master ~]# kubectl get pod --show-labels -l "app  in (myapp, ngx-deploy)"
NAME                          READY   STATUS    RESTARTS   AGE     LABELS
myapp-7d4b7b84b-986qx         1/1     Running   0          2d22h   app=myapp,pod-template-hash=7d4b7b84b
ngx-deploy-6b44c98c55-22j8m   1/1     Running   0          2d22h   app=ngx-deploy,pod-template-hash=6b44c98c55
pod-demo                      1/1     Running   0          3h24m   app=myapp

# notin
[root@master ~]# kubectl get pod --show-labels -l "app  notin (myapp, ngx-deploy)"
No resources found in default namespace.

```

## -L

```bash
# 显示app键对应的值
[root@master ~]# kubectl get pod -L app -l "app  in (myapp, ngx-deploy)"
NAME                          READY   STATUS    RESTARTS   AGE     APP
																   # 新增app字段，显示app键对应的值
myapp-7d4b7b84b-986qx         1/1     Running   0          2d22h   myapp
ngx-deploy-6b44c98c55-22j8m   1/1     Running   0          2d22h   ngx-deploy
pod-demo                      1/1     Running   0          3h26m   myapp
```

# 资源注解

资源提供元数据信息，元数据不受字符数据限制，可结构化，可非结构化。

```bash
kubectl annotate -h
  kubectl annotate [--overwrite] (-f FILENAME | TYPE NAME) KEY_1=VAL_1 ... KEY_N=VAL_N [--resource-version=version]
[options]

[root@master ~]# kubectl explain pod.metadata.annotations
KIND:     Pod
VERSION:  v1

FIELD:    annotations <map[string]string>

DESCRIPTION:
     Annotations is an unstructured key value map stored with a resource that
     may be set by external tools to store and retrieve arbitrary metadata. They
     are not queryable and should be preserved when modifying objects. More
     info: http://kubernetes.io/docs/user-guide/annotations

```

```diff
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  namespace: prod
  labels:
    app: pod-demo
    rel: stable
+  annotations:
+    ik8s.io/project: hello
spec:            # 属性
  containers:           # pod运行多个容器. 
  - name: bbox
    image: busybox
    imagePullPolicy: IfNotPresent
    #command: # kubectl explain pod.spec.containers.command  command <[]string>
    #- "/bin/sh"
    #- "-c"
    #- "sleep 86400"
    command: ["/bin/sh","-c","sleep 86400"] # kubectl explain pod.spec.containers.command  command <[]string>
  - image: ikubernetes/myapp:v1
    imagePullPolicy: IfNotPresent
    name: myapp

```

```bash
[root@master ~]# kubectl apply -f manifests/basic/pod-demo-2.yaml


[root@master ~]# kubectl get -f manifests/basic/pod-demo-2.yaml
NAME       READY   STATUS    RESTARTS   AGE
pod-demo   2/2     Running   0          3h59m
[root@master ~]# kubectl get pod  pod-demo  -n prod -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    ik8s.io/project: hello
    # 上一次apply的完整内容，下一次apply会对比当前和上一次版本，将变化的作为补丁打上。
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{"ik8s.io/project":"hello"},"labels":{"app":"pod-demo","rel":"stable"},"name":"pod-demo","namespace":"prod"},"spec":{"containers":[{"command":["/bin/sh","-c","sleep 86400"],"image":"busybox","imagePullPolicy":"IfNotPresent","name":"bbox"},{"image":"ikubernetes/myapp:v1","imagePullPolicy":"IfNotPresent","name":"myapp"}]}}


```



# pod生命周期

从创建到结束终止会经过哪些阶段

![image-20210118142343200](http://myapp.img.mykernel.cn/image-20210118142343200.png)

- [x] 初始容器(optional): 主容器启动前，环境初始化。 init container

  > 可以不存在。
  >
  > 可以有多个，只有所有初始化工作都完成，主容器才可以启动。
  >
  > 有多个容器，是串行执行。

- [x] 主容器: main container

  > 其他容器和主容器可以同时运行。

  - [x] 创建完成后：poststart hook, 可以做启动后设置。

  - [x] 结束后: prestop hook，结束前设置。

  - [x] 中间阶段，正常运行阶段：liveness/readniess

    > 容器进程运行未必容器可以提供服务, 所以使用健康状态检查。docker 叫healthcheck. k8s叫liveness Probe/readniess Probe。
    >
    > liveness    不健康：逐渐递增的时间长度，反复重启
    >
    > readiness 就绪不健康：容器running, 未必进程running, 只是容器自身的running, 所以就绪状态检测。与service结合。
    >
    > 生产必须配置，用户请求调度就失败了，看上去正常，实际上进程还没有真正启动。



```bash
kubectl explain pod.spec.initContainers
 
 https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers: # 初始化
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```



```bash
[root@master ~]# kubectl explain pod.spec.initContainers | grep '>'
RESOURCE: initContainers <[]Object>
    lifecycle	<Object>
    	
        postStart	<Object> # 启动后勾子
        	 https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#container-hooks
           exec	<Object>        # 执行命令
           httpGet	<Object>    # http，多数情况使用
           tcpSocket	<Object> # 向端口请求，端口成功，未必可用

        preStop	<Object>
			 https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#container-hooks
           exec	<Object>        # 执行命令
           httpGet	<Object>    # http，多数情况使用
           tcpSocket	<Object> # 向端口请求，端口成功，未必可用
   livenessProbe	<Object>   # 不正常，重启
   readinessProbe	<Object>   # 不正常，不接入service


# google: kubernetes lifecycle demo
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```

## livenessProbe和readinessProbe

均支持三种格式

| 正常运行pod的探针 | 检测失败时                               |
| ----------------- | ---------------------------------------- |
| livenessProbe     | 重启容器。                               |
| readinessProbe    | 从service上摘除，不会重启容器。          |
| lifecycle         | 不检测，仅在启动后或停止前进行一些配置。 |



```bash
[root@master chapter4]# kubectl explain pod.spec.containers.livenessProbe | grep '>'
RESOURCE: livenessProbe <Object>
   exec	<Object>
   failureThreshold	<integer>   # 失败阈值，失败几次，从成功转为失败。默认3次，最小1。
   httpGet	<Object>
       host	<string>            # 不指定，默认pod自已的ip
       httpHeaders	<[]Object>  # 额外首部。
       path	<string>
       port	<string> -required- 
       scheme	<string>        # 默认http
   initialDelaySeconds	<integer>    # 初始化多久才检查？一启动就检查，进程还没有启动，就会导致，一启动就重启。
   periodSeconds	<integer>   # 每隔多长时间检查一次。默认10s
   successThreshold	<integer>   # 成功阈值，成功几次，从失败转为成功。默认1。只能是1.
   tcpSocket	<Object>
   timeoutSeconds	<integer>   # 你检查没有响应，连接会挂着，需要超时时长。默认1s, 最小1.
[root@master chapter4]# kubectl explain pod.spec.containers.readinessProbe | grep '>'
RESOURCE: readinessProbe <Object>
   exec	<Object>
   failureThreshold	<integer>
   httpGet	<Object>

   initialDelaySeconds	<integer>
   periodSeconds	<integer>
   successThreshold	<integer>
   tcpSocket	<Object>
   timeoutSeconds	<integer>
[root@master chapter4]# kubectl explain pod.spec.containers.lifecycle.postStart | grep '>'
RESOURCE: postStart <Object>
   exec	<Object>
   httpGet	<Object>
   tcpSocket	<Object>
[root@master chapter4]# kubectl explain pod.spec.containers.lifecycle.preStop | grep '>'
RESOURCE: preStop <Object>
   exec	<Object>
   httpGet	<Object>
   tcpSocket	<Object>

```





## livenessProbe exec

进入ikubernetes github`https://github.com/iKubernetes/Kubernetes_Advanced_Practical`

```bash
yum -y install git
git clone https://github.com/iKubernetes/Kubernetes_Advanced_Practical.git

[root@master ~]# cd Kubernetes_Advanced_Practical/
[root@master Kubernetes_Advanced_Practical]# cd chapter4/
[root@master chapter4]# ls
lifecycle-demo-pod.yaml  memleak-pod.yaml        pod-example-update.yaml   pod-with-nodeselector.yaml  stress-pod.yaml
liveness-exec.yaml       namespace-example.yaml  pod-use-hostnetwork.yaml  pod-with-seccontext.yaml
liveness-http.yaml       pod-alpine.yaml         pod-with-labels.yaml      readiness-exec.yaml


[root@master chapter4]# cat liveness-exec.yaml 
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness-exec
  name: liveness-exec
spec:
  containers:
  - name: liveness-demo
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 60; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - test
        - -e
        - /tmp/healthy

# 成功60s
# 会重建容器，又会生成文件，60s后又会重启
# ...

[root@master chapter4]# kubectl apply -f liveness-exec.yaml
pod/liveness-exec created

[root@master chapter4]# kubectl get pod -w
NAME                          READY   STATUS    RESTARTS   AGE
liveness-exec                 1/1     Running   0          20s
#ctrl+c


```

```bash
[root@master chapter4]# kubectl describe pod liveness-exec
Name:         liveness-exec
Namespace:    default
Priority:     0
Node:         node01.magedu.com/172.16.1.101
Start Time:   Mon, 18 Jan 2021 15:00:44 +0800
Labels:       test=liveness-exec
Annotations:  <none>
Status:       Running
IP:           10.244.1.4
IPs:
  IP:  10.244.1.4
Containers:
  liveness-demo:
    Container ID:  docker://07d3b565399b4363b9cee557878b58b7842e206c730a7621c8b8d3824d5c1a0e
    Image:         busybox
    Image ID:      docker-pullable://busybox@sha256:c5439d7db88ab5423999530349d327b04279ad3161d7596d2126dfb5b02bfd1f
    Port:          <none>
    Host Port:     <none>
    Args:
      /bin/sh
      -c
      touch /tmp/healthy; sleep 60; rm -rf /tmp/healthy; sleep 600
    State:          Running
      Started:      Mon, 18 Jan 2021 15:01:01 +0800
    Ready:          True
    				# 目前还没有重启, 所以重启次数为0
    Restart Count:  0      
    											# 延迟为0，表示一启动，立即进行liveness检测
    													 # 检测的超时
    													 			# 每次检测的周期 10s
    													 						# 成功一次认为是成功
    													 								  # 失败3次认为失败
    Liveness:       exec [test -e /tmp/healthy] delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-8qddj (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-8qddj:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-8qddj
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  43s   default-scheduler  Successfully assigned default/liveness-exec to node01.magedu.com
  Normal  Pulling    41s   kubelet            Pulling image "busybox"
  Normal  Pulled     27s   kubelet            Successfully pulled image "busybox" in 14.784441565s
  Normal  Created    26s   kubelet            Created container liveness-demo
  Normal  Started    26s   kubelet            Started container liveness-demo



# 重复describe, 可以发现
    Restart Count:  11

[root@master chapter4]# kubectl get pod
NAME                          READY   STATUS    RESTARTS   AGE
												# 重启次数
liveness-exec                 1/1     Running   2          4m58s

```

## livenessProbe httpGet

```diff
[root@master chapter4]# cat liveness-http.yaml 
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness-demo
+    image: nginx:1.14-alpine
    ports:
+    - name: http         # 定义了http是80
      containerPort: 80
    lifecycle:
+      postStart: # 启动后事件，专门生成网页
        exec:
          command:
          - /bin/sh
          - -c
          - 'echo Healty > /usr/share/nginx/html/healthz'
    livenessProbe:
      httpGet:
        path: /healthz
+        port: http           # 调用http,就是80. 没有定义就必须使用80.
        scheme: HTTP
+      initialDelaySeconds: 3
+      timeoutSeconds: 1
+      periodSeconds: 2

```

> 实际配置时要去掉+号

```bash
kubectl apply -f liveness-http.yaml

[root@master chapter4]# kubectl get pod -o wide -w
NAME                          READY   STATUS    RESTARTS   AGE     IP           NODE                NOMINATED NODE   READINESS GATES
liveness-exec                 1/1     Running   5          11m     10.244.1.4   node01.magedu.com   <none>           <none>
liveness-http                 1/1     Running   0          21s     10.244.1.5   node01.magedu.com   <none>           <none>
#ctrl +c


[root@master chapter4]# kubectl describe pod/liveness-http 
Name:         liveness-http
Namespace:    default
Priority:     0
Node:         node01.magedu.com/172.16.1.101
Start Time:   Mon, 18 Jan 2021 15:11:59 +0800
Labels:       test=liveness
Annotations:  <none>
Status:       Running
IP:           10.244.1.5
IPs:
  IP:  10.244.1.5
Containers:
  liveness-demo:
    Container ID:   docker://2a6e0711873caad0d135cc57f4815dd13c19e342b9eda0aece7a029534745bc4
    Image:          nginx:1.14-alpine
    Image ID:       docker-pullable://nginx@sha256:485b610fefec7ff6c463ced9623314a04ed67e3945b9c08d7e53a47f6d108dc7
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running              # running
      Started:      Mon, 18 Jan 2021 15:12:01 +0800
    Ready:          True
    Restart Count:  0
    				# http get
    						# 没有主机地址表示本机，http表示port定义的http
    											  # 延迟3s 
    											  		   # 超时1s
    											  		   			  # 间隔2s
    Liveness:       http-get http://:http/healthz delay=3s timeout=1s period=2s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-8qddj (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-8qddj:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-8qddj
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  47s   default-scheduler  Successfully assigned default/liveness-http to node01.magedu.com
  Normal  Pulled     46s   kubelet            Container image "nginx:1.14-alpine" already present on machine
  Normal  Created    46s   kubelet            Created container liveness-demo
  Normal  Started    45s   kubelet            Started container liveness-demo

```

验证删除

```bash
[root@master chapter4]# kubectl exec -it liveness-http  -- /bin/sh
/ # rm -f /usr/share/nginx/html/healthz 
/ # command terminated with exit code 137      # 2s一次检测，检测失败就重启

[root@master chapter4]# kubectl describe pod/liveness-http
    Last State:     Terminated # 上一次
      Reason:       Completed
      Exit Code:    0
      Started:      Mon, 18 Jan 2021 15:12:01 +0800
      Finished:     Mon, 18 Jan 2021 15:14:18 +0800
    Ready:          True
    Restart Count:  1      # 重启次数
```

## livenessProbe tcpsocket

不使用，端口正常未必正常



## readinessProbe exec

```bash
[root@master chapter4]# cat readiness-exec.yaml 
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: readiness-exec
  name: readiness-exec
spec:
  containers:
  - name: readiness-demo
    image: busybox
    args: ["/bin/sh", "-c", "while true; do rm -f /tmp/ready; sleep 30; touch /tmp/ready; sleep 300; done"] 
    readinessProbe:
      exec:
        command: ["test", "-e", "/tmp/ready"]
      initialDelaySeconds: 5
      periodSeconds: 5

```

```bash
[root@master chapter4]# kubectl apply -f readiness-exec.yaml


^C[root@master chapter4]# kubectl get pod 
NAME                          READY   STATUS             RESTARTS   AGE
liveness-exec                 0/1     CrashLoopBackOff   8          23m
liveness-http                 1/1     Running            1          12m
myapp-7d4b7b84b-986qx         1/1     Running            0          2d23h
ngx-deploy-6b44c98c55-22j8m   1/1     Running            0          2d23h
pod-demo                      1/1     Running            0          4h39m
							 # 左侧是：容器是否就绪？ 就绪才可以接入service.
							 # 右侧是：容器已经正常。
readiness-exec                0/1     Running            0          33s

```

测试删除ready文件

```bash
[root@master chapter4]# kubectl exec readiness-exec -- rm -f /tmp/ready

[root@master chapter4]# kubectl get pod 
NAME                          READY   STATUS             RESTARTS   AGE
liveness-exec                 0/1     CrashLoopBackOff   8          26m
liveness-http                 1/1     Running            1          15m
myapp-7d4b7b84b-986qx         1/1     Running            0          2d23h
ngx-deploy-6b44c98c55-22j8m   1/1     Running            0          2d23h
pod-demo                      1/1     Running            0          4h42m
							  # 删除后，经过多次检测，失败后就未就绪
readiness-exec                0/1     Running            0          3m34s



# 重建文件
[root@master chapter4]# kubectl exec readiness-exec -- touch /tmp/ready
[root@master chapter4]# kubectl get pod 
NAME                          READY   STATUS    RESTARTS   AGE
liveness-exec                 1/1     Running   9          27m
liveness-http                 1/1     Running   1          15m
myapp-7d4b7b84b-986qx         1/1     Running   0          2d23h
ngx-deploy-6b44c98c55-22j8m   1/1     Running   0          2d23h
pod-demo                      1/1     Running   0          4h43m
							  # 成功后就绪
readiness-exec                1/1     Running   0          4m29s
```



# pod对象的相位(phase)

- [x] running: pod被调度，容器已经被kubelet创建完成 。 未必启动。
- [x] succeeded: 容器成功终止并不重启。
- [x] failed: 所有容器终止，至少一个失败。返回非0状态或系统终止。
- [x] unknown: api无法获取pod对象状态信息。通常无法与kubelet通信所致。
- [x] pending: pod调度到节点上，节点还在下载pod镜像。或调度不成功。pod依赖的资源，没有哪个节点提供。





# pod创建过程

![image-20210118153243586](http://myapp.img.mykernel.cn/image-20210118153243586.png)

1. 用户提交create pod请求给api server.
2. api 补全默认字段(准入控制器)，再存储提交请求给etcd.
3. schduler监视到新pod创建事件，bind pod: 开始依据算法选择node，调度结果放在etcd中，填充到yaml文件中的nodename。
4. 调度后，会将事件通知给对应节点的kubelet. kubelet读取pod属性，调用docker引擎启动容器。将创建后的状态回馈给kubelet。
5. kubelet将实际状态回填到status字段。
6. api server告诉kubelet启动ok



# 容器重启策略

因为以下原因，pod对象终止，是否重建它，取决于重启策略(restartPolicy字段)定义

- [x] 容器程序崩溃
- [x] 容器申请超时限制的资源
- [ ] ....

| 策略      | 描述                    |
| --------- | ----------------------- |
| Always    | 默认，pod终止就重启。   |
| OnFailure | Pod对象出现错误才重启。 |
| Never     | 从不重启。              |



# pod终止过程

![image-20210118154128194](http://myapp.img.mykernel.cn/image-20210118154128194.png)

1. 用户发起delete pod，**此pod立即标记为不可调度**。

2. api 记录删除操作于etcd.  不会立即终止pod, 需要设定宽限期(假如30s之内必须终止)。将结果返回给api

3. api 标记为终止，通信给pod运行所在节点的kubelet.

4. kubelet把终止信息发给docker引擎，docker终止前执行preStop hook。运行结束，下一步。

5. api 通知endpoint controller，需要把此pod终止，端点控制器会把pod从自已的列表中移除

   > service不是直接关联pod, service -> endpoint -> pod

6. 如果超出宽限期，kubelet就发送kill信号给docker引擎。
7. 立即终止pod, 返回给api server
8. api server 从etcd中删除此pod所有定义。



# Pod安全上下文

- [x] Pod级别：对所有容器生效
- [x] 容器级别：仅对当前容器生效



默认可以使用节点管理员运行，它可以映射 为节点管理员，不安全。

```bash 
kubectl explain pod.spec.securityContext
   fsGroup	<integer>
   fsGroupChangePolicy	<string>
   runAsGroup	<integer> # 容器内进程只能以哪个组id运行
   runAsNonRoot	<boolean> # 不能以root运行. true不能以管理员运行。false可以以root运行。
   runAsUser	<integer> # 以哪个用户运行，指定user id
   seLinuxOptions	<Object> #以selinux限制，节点selinux不能禁用。一般不会使用此项。
   seccompProfile	<Object>
   supplementalGroups	<[]integer>  # 额外补充的附加组，组列表中的组，可以作为进程运行的组。
   sysctls	<[]Object>               # 设定为你的容器内的环境，设定哪个内核参数，由于共享同一个内核，你的设定，可能对当前节点有影响。
   windowsOptions	<Object>
```

容器级别

```bash
[root@master chapter4]# kubectl explain pod.spec.containers.securityContext | grep '>'
RESOURCE: securityContext <Object>
   allowPrivilegeEscalation	<boolean>  # 容器是否允许升级特权运行？
   capabilities	<Object> # 功能，当前容器允许执行或必须放弃内核中哪种功能
   privileged	<boolean>              # 是否直接特权容器。
   procMount	<string>               # 是否可以挂载proc目录
   readOnlyRootFilesystem	<boolean>  # 是否对rootfs只读
   runAsGroup	<integer> #
   runAsNonRoot	<boolean> #
   runAsUser	<integer> #
   seLinuxOptions	<Object>
   seccompProfile	<Object>
   windowsOptions	<Object>

# capabilities 管理操作变成能力
# 将能力给用户，用户就有管理操作
[root@master chapter4]# kubectl explain pod.spec.containers.securityContext.capabilities | grep '>'
RESOURCE: capabilities <Object>
   add	<[]string> # 添加能力
   drop	<[]string> # 删除能力


# 有哪些能力？
kernel capabilities
只有对安全逻辑有精确了解才有必须调整。
```



# Pod优先级

pod资源紧缺，有些pod关键，有些不关键

分配pod属于哪个优先级，可以优先调度哪个pod

```bash
[root@master chapter4]# kubectl explain pod.spec. | grep '>'
   priority	<integer>
   priorityClassName	<string>

```



# 容器资源限制

不限制资源，必要时可能把节点上的计算资源强占完了

```bash
[root@master chapter4]# kubectl explain pod.spec.containers.resources | grep '>'
RESOURCE: resources <Object>
   limits	<map[string]string> # 上限 最多给你多少资源。
   requests	<map[string]string> # 下限  资源需求，启动这个容器，必须有的资源。会影响调度结果。
```



如果节点资源少于下限，就不可以启动容器

cpu可压缩，可以超分。1核=1000m, 0.5 -> 500m

ram不可压缩，不可以超分。耗尽导致OOM。



```bash
[root@master chapter4]# ls memleak-pod.yaml stress-pod.yaml 
memleak-pod.yaml   # 内存压测
stress-pod.yaml    # 


[root@master chapter4]# cat stress-pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod
spec:
  containers:
  - name: stress
    image: ikubernetes/stress-ng
    command: ["/usr/bin/stress-ng", "-c 1", "-m 1", "--metrics-brief"] # 申请1C,1G内存。无论怎么申请都超不过limit限制
    resources:
      requests: 
        memory: "128Mi" # 确保128Mi
        cpu: "200m"
      limits:
        memory: "512Mi"
        cpu: "400m"

[root@master chapter4]# cat memleak-pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: memleak-pod
spec:
  containers:
  - name: simmemleak # 简单的内存泄漏
    image: saadali/simmemleak # 不断的请求内存，只申请不回收，一会超过64M就会导致OOM。
    resources:
      requests:
        memory: "64Mi"
        cpu: "1"
      limits:
        memory: "64Mi"
        cpu: "1"

[root@master chapter4]# kubectl apply -f memleak-pod.yaml
[root@master chapter4]# kubectl describe pod  memleak-pod
    Last State:     Terminated
      Reason:       OOMKilled # OOMKilled
      Exit Code:    137
      Started:      Mon, 18 Jan 2021 16:18:09 +0800
      Finished:     Mon, 18 Jan 2021 16:18:09 +0800

QoS Class:       Guaranteed # 服务质量类别

```



## pod服务质量类型

Guaranteed：具有相同值的requests和limits, k8s必须让其运行。**优先级最高。**

bustable: 至少一个容器设置了cpu/ram的下限，不满足guaranteed,自动归属于类，此优先级为**中优先级**。

besteffort: 未为任何一个容器设置request或limit属性的pod资源自动归属此类，优先级为**低优先级**。



在`kubectl describe pod`中的QoS Class 字段指定

