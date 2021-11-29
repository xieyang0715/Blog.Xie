---
title: Service、ingress资源
date: 2021-01-27 02:59:40
tags:
---





- Pod Network

  - flannel: 10.244.0.0/16

  - service: (4层调度)

    - proxy-mode: userspace/iptables/ipvs
      - kube-proxy
    - service type: ClusterIP,NodePort, LoadBalancer, ExternalName
    - Label selector: Pod资源，自动为每个Pod对象创建Endpoint
    - 
    - service -> Endpoint -> Pod
      - Endpoint: apiversion/kind/metadata/subsets
    - service实现会话粘性, clientip
    - 不指定clientip: headless service
      - clusterip: servicename -> clusterip
      - 无clusterip: servicename -> podip

  - ingress: (7层调度)

    - http/https
    - location /bbs {
      - proxy_pass http://10.244.3.55/forum/;
    - }
    - ingress controller完成代理
    - ingress来配置controller.
- 接入流量还是ingress controller 的service
      - service loadblanacer
      - service nodePort 
      - daemonset + nodeSelector + hostNetwork
      - daemonset + nodeSelector + hostPort
    



# service



## 为什么使用service资源？

- [x] pod重建ip会改变 
- [x] pod规模伸缩



不删除或人为修改service,就是固定的。修改时会重新向dns注册。

endpoint的后端

- [x] service -> endpoints -> **pods**
- [x] endpoints后端是**集群外的服务**，集群访问集群外的mysql, tomcat直接访问service -> endpoints -> 集群外的mysql.



## service

k8s实际上把每个service转换为每个节点上的ipvs/iptables规则，DNAT或简单的转发规则。

service创建和删除，需要及时反映到节点上的规则，所以节点上运行进程。kubeadm运行kube-proxy daemonset控制器并且容忍主节点的污点。 二进制部署运行kube-proxy。



kube-proxy监视到api server中service变动, 并实时转换为当前节点的ipvs/iptables规则（kube-proxy定义的模型）。k8s v1.11之后默认ipvs, 但是如果节点没有装载ipvs模块，自动降级iptables模型





## 代理类型

- [x] usrspace k8s v1.2 来使用userspace调度流量
- [x] iptables  us性能低
- [x] ipvs         使用netlink接口配置规则，k8s v1.11开始默认支持，仅限ipvs模块已经加载好了使用。



## 手工创建service

- [x] 标签选择器
- [x] ports

```bash
[root@master chapter5]# kubectl explain service
KIND:     Service
VERSION:  v1      # 核心群组的v1版本


[root@master chapter5]# kubectl explain service.spec | grep '>'
RESOURCE: spec <Object>
   allocateLoadBalancerNodePorts	<boolean>
   clusterIP	<string>        # 不指定，自动分配 
   clusterIPs	<[]string>
   externalIPs	<[]string>
   externalName	<string>
   externalTrafficPolicy	<string>
   healthCheckNodePort	<integer>     # 使用节点Ip,要不要检查ip/port
   ipFamilies	<[]string>
   ipFamilyPolicy	<string>
   loadBalancerIP	<string>
   loadBalancerSourceRanges	<[]string>
   ports	<[]Object>                # 调度器自身的port -> 调度至后端哪个或哪些端点
   publishNotReadyAddresses	<boolean>
   selector	<map[string]string>       # 哪些pod的端口
   sessionAffinity	<string>
   sessionAffinityConfig	<Object>
   topologyKeys	<[]string>
   type	<string>
   
   

[root@master chapter5]# kubectl explain service.spec.ports | grep '>'
RESOURCE: ports <[]Object>
   appProtocol	<string>
   name	<string>
   nodePort	<integer>
   port	<integer> -required-
   protocol	<string>
   targetPort	<string>

```

启动控制器

yaml文件参考：http://blog.mykernel.cn/2021/01/18/Pod%E6%8E%A7%E5%88%B6%E5%99%A8/#deployment

```diff
[root@master chapter5]# kubectl apply -f deploy-nginx.yaml 
deployment.apps/myapp-deploy created

[root@master chapter5]# kubectl get deploy -n prod -o wide
NAME           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                 SELECTOR
myapp-deploy   1/3     3            0           7s    myapp        ikubernetes/myapp:v2   app=myapp,rel=stable
```



添加service

```bash
kubectl create service clusterip myapp --tcp=80:80 --dry-run -o yaml > myapp-svc.yaml


[root@master chapter5]# kubectl get pod -n prod --show-labels -o wide
NAME                           READY   STATUS      RESTARTS   AGE    IP            NODE                NOMINATED NODE   READINESS GATES   LABELS
filebeat-ds-rtbn9              1/1     Running     0          169m   10.244.1.13   node01.magedu.com   <none>           <none>            app=filebeat,controller-revision-hash=77b97bbb96,pod-template-generation=1
job-example-wcskd              0/1     Completed   0          154m   10.244.3.15   node03.magedu.com   <none>           <none>            app=myjob,controller-uid=f1dae7bd-52f4-4b0c-bedc-d0fe53639601,job-name=job-example
myapp-deploy-b9f79945c-8zbgj   1/1     Running     0          70s    10.244.3.32   node03.magedu.com   <none>           <none>            app=myapp,pod-template-hash=b9f79945c,rel=stable
myapp-deploy-b9f79945c-wlpsh   1/1     Running     0          71s    10.244.3.31   node03.magedu.com   <none>           <none>            app=myapp,pod-template-hash=b9f79945c,rel=stable
myapp-deploy-b9f79945c-x7fcz   1/1     Running     0          70s    10.244.2.15   node02.magedu.com   <none>           <none>            app=myapp,pod-template-hash=b9f79945c,rel=stable
myapp-rs-npmms                 1/1     Running     0          8d     10.244.1.8    node01.magedu.com   <none>           <none>            app=myapp-pod
myapp-rs-p2fq2                 1/1     Running     0          8d     10.244.1.9    node01.magedu.com   <none>           <none>            app=myapp-pod
myapp-rs-qlgv9                 1/1     Running     0          8d     10.244.3.8    node03.magedu.com   <none>           <none>            app=myapp-pod
myapp-rs-t6cnj                 1/1     Running     0          8d     10.244.2.7    node02.magedu.com   <none>           <none>            app=myapp-pod

pod-demo                       2/2     Running     9          9d     10.244.3.6    node03.magedu.com   <none>           <none>            app=myapp-pod,rel=stable,tier=frontend

# 注意
# myapp-deploy-* pod的标签 app=myapp,pod-template-hash=*,rel=stable
# pod-demo pod的标签 app=myapp-pod
```

```bash
# 添加svc
[root@master ~]# cat manifests/basic/myapp-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: prod
spec:
  selector:           # 标签选择器
    app: myapp
  ports:
  - name: http
    port: 80              # service port
    protocol: TCP
    targetPort: 80        # pod port
  type: ClusterIP


[root@master basic]# kubectl apply -f myapp-svc.yaml 
[root@master basic]# kubectl describe svc/myapp -n prod
Name:              myapp
Namespace:         prod
Labels:            <none>
Annotations:       <none>
Selector:          app=myapp
Type:              ClusterIP
IP Families:       <none>
IP:                10.103.230.174
IPs:               10.103.230.174
Port:              http  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.2.15:80,10.244.3.31:80,10.244.3.32:80 # 3个service
Session Affinity:  None
Events:            <none>
[root@master basic]# 

```

重新打标pod

```bash
[root@master basic]# kubectl label pod -n prod pod-demo app=myapp --overwrite
pod/pod-demo labeled
[root@master basic]# kubectl describe svc/myapp -n prod
Name:              myapp
Namespace:         prod
Labels:            <none>
Annotations:       <none>
Selector:          app=myapp
Type:              ClusterIP
IP Families:       <none>
IP:                10.103.230.174
IPs:               10.103.230.174
Port:              http  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.2.15:80,10.244.3.31:80,10.244.3.32:80 + 1 more... # 添加1个service
Session Affinity:  None
Events:            <none>


```

> 在真正使用svc匹配时，应该多个标签来引用Pod, 避免引用到其他pod.

重新生成svc文件

```diff
kubectl delete -f myapp-svc.yaml 
[root@master basic]# cat myapp-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: prod
spec:
  selector:           # 标签选择器
    app: myapp
+    rel: stable
  ports:
  - name: http
    port: 80              # service port
    protocol: TCP
    targetPort: 80        # pod port
  type: ClusterIP

```

```bash
[root@master basic]# cat myapp-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: prod
spec:
  selector:           # 标签选择器
    app: myapp
    rel: stable
  ports:
  - name: http
    port: 80              # service port
    protocol: TCP
    targetPort: 80        # pod port
  type: ClusterIP
[root@master basic]# kubectl apply -f myapp-svc.yaml
service/myapp created
[root@master basic]# kubectl get svc -n prod
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
myapp       ClusterIP   10.109.105.24   <none>        80/TCP         8s
myapp-svc   NodePort    10.104.73.196   <none>        80:32001/TCP   8d


# 直接访问svc的地址
[root@master chapter5]# curl 10.109.105.24/hostname.html
myapp-deploy-b9f79945c-8zbgj
[root@master chapter5]# curl 10.109.105.24/hostname.html
myapp-deploy-b9f79945c-wlpsh
[root@master chapter5]# curl 10.109.105.24/hostname.html
myapp-deploy-b9f79945c-8zbgj

```

## endpoint

创建service自动创建

```bash
[root@master chapter5]# kubectl get endpoints -n prod
NAME        ENDPOINTS                                                AGE
myapp       10.244.2.15:80,10.244.3.31:80,10.244.3.32:80             2m42s
myapp-svc   10.244.1.8:80,10.244.1.9:80,10.244.2.16:80 + 2 more...   8d
[root@master chapter5]# kubectl describe  endpoints/myapp -n prod
Name:         myapp
Namespace:    prod
Labels:       <none>
Annotations:  <none>
Subsets:
  Addresses:          10.244.2.15,10.244.3.31,10.244.3.32
  NotReadyAddresses:  <none>
  Ports:
    Name  Port  Protocol
    ----  ----  --------
    http  80    TCP

Events:  <none>


# endpoint没有spec
[root@master chapter5]# kubectl get  endpoints/myapp -n prod -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  creationTimestamp: "2021-01-27T04:48:46Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:subsets: {}
    manager: kube-controller-manager
    operation: Update
    time: "2021-01-27T04:49:32Z"
  name: myapp
  namespace: prod
  resourceVersion: "1170572"
  uid: 9b61f9b2-9d69-482b-8c23-65f57958fcb3
subsets:
- addresses:
  - ip: 10.244.2.15
    nodeName: node02.magedu.com
    targetRef:
      kind: Pod
      name: myapp-deploy-b9f79945c-x7fcz
      namespace: prod
      resourceVersion: "1169285"
      uid: 60d97900-668d-488d-a38f-211eb27e64e0
  - ip: 10.244.3.31
    nodeName: node03.magedu.com
    targetRef:
      kind: Pod
      name: myapp-deploy-b9f79945c-wlpsh
      namespace: prod
      resourceVersion: "1169345"
      uid: 18672f66-017b-487b-905c-57c501185ce3
  - ip: 10.244.3.32
    nodeName: node03.magedu.com
    targetRef:
      kind: Pod
      name: myapp-deploy-b9f79945c-8zbgj
      namespace: prod
      resourceVersion: "1169363"
      uid: be280baf-a4f7-42a0-8393-5f66bdfe4998
  ports:
  - name: http
    port: 80
    protocol: TCP

```



## service 类型

service的`spec.type`指定，默认ClusterIP

如何接收和调度

### ClusterIP

10.96.0.0/12

- [ ] 同主机：源地址pod地址
- [ ] 跨主机：源地址flannel接口地址

### NodePort

节点上生成ipvs/iptables规则， client -> 节点端点 -> 路由到 ClusterIP

```diff
"myapp-svc.yaml" 15L, 286C                                                                                                       
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: prod
spec:
  selector:           # 标签选择器
    app: myapp
    rel: stable
  ports:
  - name: http
    port: 80              # service port
    protocol: TCP
    targetPort: 80        # pod port
+    nodePort: 30080        # 不指定端口是随机。30000-60000
+  type: NodePort

```

```bash
[root@master basic]# kubectl apply -f myapp-svc.yaml

[root@master basic]# kubectl get svc -n prod
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
myapp       NodePort   10.109.105.24   <none>        80:30080/TCP   13m

```



nodeport非标准端口，集群外部使用负载均衡器, LoadBlancer，K8S必须在公有云上，并且购买了LBaaS。就可以使用`loadbalancer`

如果不在公有云，就使用nodeport -> 外部的负载均衡器(nginx/haproxy/lvs)



## 引用集群外部服务(externalname/clusterip)

假设service后端不是当前集群的pod, 而是集群外部的服务甚至是互联网上的服务



###  ExternalName dns域名

- [x] 不指定标签选择器，不需要pod
- [x] 直接指定名字



- [x] type externalname
- [x] externalname
- [x] externalips

这样客户端访问服务名，就是外部的服务。



### endpoint指定ip

引用外部服务的另一种方法

- [x] 不定义标签选择器
- [x] 手工定义endpoint资源

```yaml
# service定义
[root@master basic]# kubectl get endpoints/kubernetes -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    endpointslice.kubernetes.io/skip-mirror: "true"
  name: kubernetes
  namespace: default

subsets:
- addresses:
  - ip: 172.16.1.100 # 外部的ip
  ports:
  - name: https
    port: 6443
    protocol: TCP


# service定义
[root@master basic]# kubectl get svc/kubernetes -o yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    component: apiserver
    provider: kubernetes
  name: kubernetes # 此名字必须同名于上面的endpoints，这样就会自动关联至endpoints.
  namespace: default
spec:
  # 注意不定义selecotr
  clusterIP: 10.96.0.1
  clusterIPs:
  - 10.96.0.1
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: 6443
  sessionAffinity: None
  type: ClusterIP

```



service 需要有clusterip, 存在于dns的A记录中



## headless

```bash
# service解析结果
[root@master chapter5]# kubectl get svc -n prod
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
					   # SERVICE集群IP
myapp       NodePort   10.109.105.24   <none>        80:30080/TCP   27m
myapp-svc   NodePort   10.104.73.196   <none>        80:32001/TCP   8d

[root@master basic]# kubectl get pod
NAME                          READY   STATUS             RESTARTS   AGE
liveness-exec                 0/1     CrashLoopBackOff   1618       8d
liveness-http                 1/1     Running            1          8d
myapp-7d4b7b84b-986qx         1/1     Running            0          11d
ngx-deploy-6b44c98c55-22j8m   1/1     Running            0          11d
pod-demo                      1/1     Running            0          9d
readiness-exec                1/1     Running            0          8d

[root@master basic]# kubectl exec -it pod-demo -- sh
/ # nslookup myapp.prod
nslookup: can't resolve '(null)': Name does not resolve

Name:      myapp.prod
Address 1: 10.109.105.24 myapp.prod.svc.cluster.local # 解析结果是service的Ip
```



```diff
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: prod
spec:
+  clusterIP: None
  selector:           # 标签选择器
    app: myapp
    rel: stable
  ports:
  - name: http
    port: 80              # service port
    protocol: TCP
    targetPort: 80        # pod port
    # 没有nodePort
+  type: ClusterIP
```

```bash
[root@master basic]# kubectl apply -f myapp-svc.yaml
service/myapp created
[root@master basic]# kubectl get svc -n prod
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
myapp       ClusterIP   None            <none>        80/TCP         5s
myapp-svc   NodePort    10.104.73.196   <none>        80:32001/TCP   8d


# 再次curl
[root@master basic]# kubectl exec -it pod-demo -- sh
/ # nslookup myapp.prod
nslookup: can't resolve '(null)': Name does not resolve

Name:      myapp.prod
Address 1: 10.244.2.15 10-244-2-15.myapp.prod.svc.cluster.local
Address 2: 10.244.3.32 10-244-3-32.myapp.prod.svc.cluster.local
Address 3: 10.244.3.31 10-244-3-31.myapp.prod.svc.cluster.local


# 注意一样可以访问，没有调度是因为dns有缓存
/ # wget -O - -q myapp.prod/hostname.html
myapp-deploy-b9f79945c-x7fcz
/ # wget -O - -q myapp.prod/hostname.html
myapp-deploy-b9f79945c-x7fcz
/ # wget -O - -q myapp.prod/hostname.html
myapp-deploy-b9f79945c-x7fcz
/ # wget -O - -q myapp.prod/hostname.html
myapp-deploy-b9f79945c-x7fcz

```



## service会话粘性 

大多数情况不需要指定session affinity, 

```bash
kubectl explain svc.spec
	sessionAffinity # lvs: sh;  nginx: ip_hash; haproxy: sourceip

```



## 启用ipvs

生产环境中，应该在部署时就启动ipvs



修改kube-proxy配置（k8s在容器外部修改容器配置，可以自动加载为容器的配置）

```bash
[root@master basic]# kubectl get configmap -n kube-system
NAME                                 DATA   AGE
coredns                              1      12d
extension-apiserver-authentication   6      12d
kube-flannel-cfg                     2      12d
kube-proxy                           2      12d
kube-root-ca.crt                     1      12d
kubeadm-config                       2      12d
kubelet-config-1.20                  1      12d

[root@master basic]# kubectl get cm/kube-proxy -n kube-system -o yaml
apiVersion: v1
data:
  config.conf: |-
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    bindAddress: 0.0.0.0
    bindAddressHardFail: false
    clientConnection:
      acceptContentTypes: ""
      burst: 0
      contentType: ""
      kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
      qps: 0
    clusterCIDR: 10.244.0.0/16
    configSyncPeriod: 0s
    conntrack:
      maxPerCore: null
      min: null
      tcpCloseWaitTimeout: null
      tcpEstablishedTimeout: null
    detectLocalMode: ""
    enableProfiling: false
    healthzBindAddress: ""
    hostnameOverride: ""
    iptables:
      masqueradeAll: false
      masqueradeBit: null
      minSyncPeriod: 0s
      syncPeriod: 0s
    ipvs:
      excludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      strictARP: false
      syncPeriod: 0s
      tcpFinTimeout: 0s
      tcpTimeout: 0s
      udpTimeout: 0s
    kind: KubeProxyConfiguration
    metricsBindAddress: ""
    mode: ""
    nodePortAddresses: null
    oomScoreAdj: null
    portRange: ""
    showHiddenMetricsForVersion: ""
    udpIdleTimeout: 0s
    winkernel:
      enableDSR: false
      networkName: ""
      sourceVip: ""
  kubeconfig.conf: |-
    apiVersion: v1
    kind: Config
    clusters:
    - cluster:
        certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        server: https://172.16.1.100:6443
      name: default
    contexts:
    - context:
        cluster: default
        namespace: default
        user: default
      name: default
    current-context: default
    users:
    - name: default
      user:
        tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
kind: ConfigMap
metadata:
  annotations:
    kubeadm.kubernetes.io/component-config.hash: sha256:33c54bbf237f8a3dad6fc8ec7977a3b96b13dfa7f3acb9d11c762f0a3387c4b8
  creationTimestamp: "2021-01-14T10:34:48Z"
  labels:
    app: kube-proxy
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:config.conf: {}
        f:kubeconfig.conf: {}
      f:metadata:
        f:annotations:
          .: {}
          f:kubeadm.kubernetes.io/component-config.hash: {}
        f:labels:
          .: {}
          f:app: {}
    manager: kubeadm
    operation: Update
    time: "2021-01-14T10:34:48Z"
  name: kube-proxy
  namespace: kube-system
  resourceVersion: "256"
  uid: c46f5435-a60f-40c1-8bc0-e051af4493b3

```

修改配置

```bash
[root@master basic]# kubectl edit cm/kube-proxy -n kube-system
mode: ""        # 空，可能是iptables. 
```





所有节点添加ipvs模块

```bash
[root@k8s-master1 ~]# cat /etc/modules-load.d/10-k8s-modules.conf 
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4


for i in `cat /etc/modules-load.d/10-k8s-modules.conf`; do modprobe $i; done

```

方式一：部署时，使用yaml文件http://blog.mykernel.cn/2021/01/13/Kubernetes%E9%9B%86%E7%BE%A4%E9%83%A8%E7%BD%B2%E5%8F%8A%E9%99%88%E8%BF%B0%E5%BC%8F%E5%91%BD%E4%BB%A4%E7%AE%A1%E7%90%86/#master

方式二：部署后

```bash
kubectl edit cm/kube-proxy -n kube-system
mode: "ipvs"
ipvs:
  scheduler: "" # 默认rr, wrr, wlc
```

由于现在是实验环境，可以重建kube-proxy的pod

```bash
kubectl delete pod -n kube-system -l k8s-app=kube-proxy
```



验证service规则有没有生成ipvs规则

```bash
yum -y install ipvsadm

ipvsadm -Ln
TCP  10.104.73.196:80 rr
  -> 10.244.1.8:80                Masq    1      0          0         
  -> 10.244.1.9:80                Masq    1      0          0         
  -> 10.244.2.7:80                Masq    1      0          0         
  -> 10.244.2.16:80               Masq    1      0          0         
  -> 10.244.3.8:80                Masq    1      0          0        
```



验证curl

```bash
[root@master ~]# curl myapp.default.svc.cluster.local/hostname.html
pod-demo
```



# ingress

**ingress controller**: 就是pod, controller表示管理ingress

**ingress**是配置文件

**ingress controller的pod受deploy controller控制**





ingress controller 对应多个deploy

```bash
#一个nginx代理多个集群


upstream shop{}

upstream user{}

```



ingress来配置不同的deploy

```bash
## 多个location
server { # 一个server调度不同应用
	servername www.ilinux.io;
	# 不同的location映射不同的应用
	location /eshop {
		proxy_pass http://shop;
	} 
	location /user {
		proxy_pass http://user;
	}
}


## 多个server
server { 
	servername www.ilinux.io;
	location /eshop {
		proxy_pass http://shop;
	} 
server { 
	servername www.magedu.com;
	location /user {
		proxy_pass http://user;
	}
}
```



简单的ingress示例

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules: # 对象列表
  - http: # 一个对象指定虚拟主机
    host:
      paths:
      - path: /testpath # 相当于指定location
        pathType: Prefix
        backend:
          service:
            name: test   # 相当于指定upstream；k8s指定servicename可以根据标签选择器自动映射为一组pod.
            port:
              number: 80
```

> service提供给ingress selector
>
> service识别pod变动，ingress controller是pod可以直接与selector选择的pod通信。



## 接入流量

ingress controller pod也是私网地址，也需要service接入外部流量(3种方法)。

- nodePort 每个节点是入口, 外部负载均衡。（service配置） - elb - nodeport 
- hostNetwork。                    pod直接共享节点网络名称空间。(pod controller配置)  用户 -  运行pod的节点 - pod
- container.port.hostport。专门留出80/443端口。     (pod controller配置)                用户 -   运行pod的节点 - pod





ingress controller运行在有限个主机上，让ingress controller pod, 通过daemonset启动，要么 

​	方式一：节点打污点，让 ingress controller pod 容忍。 其他pod不能调度至这些节点。

​    方式二：节点打标签让ingress controller pod选择这些节点。



## ingress controller

k8s官方将nginx改造为ingress

https://github.com/kubernetes/

![image-20210127151559501](http://myapp.img.mykernel.cn/image-20210127151559501.png)

红帽将haproxy改造为ingress



即使nginx/haproxy可以改造为ingress使用，但是并非云原生应用，所以不太适合动态的规则不断变动的场景。为此有专门的应用程序，专门在k8s之上完成ingress。traefik、envoy(c++)

service/ingress controller只实现了如何调度，如何代理；没有更为强大的功能：流控、AB发布、...；将来应该部署istio服务网格，**istio除了具有nginx ingress 7层代理功能，还支持流控、灰度发布、AB测试 ...**

**流控**：金丝雀发布有1组新版本，一组老版本，istio支持：哪些**流量引入**到哪些pod中。**流量的速率**怎么样，也可以精心定义。



中小企业使用nginx即可，用不了高级功能



## 部署ingress

![image-20210127152750575](http://myapp.img.mykernel.cn/image-20210127152750575.png)

进入看看deploy doc

https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md#bare-metal

```bash
# Source: ingress-nginx/templates/controller-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    helm.sh/chart: ingress-nginx-3.19.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.43.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller # 名称叫nginx-controller, 实际对应一个pod controller。
  namespace: ingress-nginx
spec:
  # 未指定副本，默认1个
  selector:  # 选择pod
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/component: controller
  revisionHistoryLimit: 10
  minReadySeconds: 0
  template:
    metadata:
      labels: # 定义pod标签
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/component: controller
    spec:
      dnsPolicy: ClusterFirst
      containers:
        - name: controller
          image: k8s.gcr.io/ingress-nginx/controller:v0.43.0@sha256:9bba603b99bf25f6d117cf1235b6598c16033ad027b143c90fa5b3cc583c5713
          imagePullPolicy: IfNotPresent
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown
	  ...
      nodeSelector: # 选择节点
        kubernetes.io/os: linux
```

引入集群外部流量

```bash 
# Source: ingress-nginx/templates/controller-service.yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
  labels:
    helm.sh/chart: ingress-nginx-3.19.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.43.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: http
    - name: https
      port: 443
      protocol: TCP
      targetPort: https
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
```

部署ingress

```bash
[root@master ~]# kubectl apply -f deploy.yaml 
namespace/ingress-nginx created # 名称空间
serviceaccount/ingress-nginx created
configmap/ingress-nginx-controller created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
service/ingress-nginx-controller-admission created
service/ingress-nginx-controller created # 默认ClusterIP
deployment.apps/ingress-nginx-controller created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
serviceaccount/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created



```

给ingress添加service

## service nodePort

```bash
[root@master ingress]# cat ingress-nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
      nodePort: 30080
    - name: https
      port: 443
      protocol: TCP
      targetPort: 80
      nodePort: 30443
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
[root@master ingress]# kubectl apply -f ingress-nginx-svc.yaml
service/ingress-nginx-controller configured
[root@master ingress]# kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.102.234.189   <none>        80:30080/TCP,443:30443/TCP   23m
ingress-nginx-controller-admission   ClusterIP   10.104.162.197   <none>        443/TCP                      23m

```

尝试访问节点ip的30080



## daemonset + nodeselector （略,仅介绍）

不依赖service访问

```diff
# Source: ingress-nginx/templates/controller-deployment.yaml
apiVersion: apps/v1
+ kind: DaemonSet
metadata:
  labels:
    helm.sh/chart: ingress-nginx-3.19.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.43.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller # 名称叫nginx-controller, 实际对应一个pod controller。
  namespace: ingress-nginx
spec:
  # 未指定副本，默认1个
  selector:  # 选择pod
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/component: controller
  revisionHistoryLimit: 10
  minReadySeconds: 0
  template:
    metadata:
      labels: # 定义pod标签
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/component: controller
    spec:
      dnsPolicy: ClusterFirst
      containers:
        - name: controller
          image: k8s.gcr.io/ingress-nginx/controller:v0.43.0@sha256:9bba603b99bf25f6d117cf1235b6598c16033ad027b143c90fa5b3cc583c5713
          imagePullPolicy: IfNotPresent
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown
	  ...
+     hostNetwork: true                       # 共享节点的网络名称空间
      nodeSelector: # 选择节点
        kubernetes.io/os: linux
+        ingress-nginx-controller: "on"       # 运行于特定节点上，还工作为daemonSet, 并且hostNetwork
```

## ingress 规则

host

path

https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#ingress-rules

## 默认后端

没有服务

https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#default-backend

## 基于url

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-fanout-example
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 4200
      - path: /bar
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 8080
```

## 基于主机名



```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host-ingress-no-third-host
spec:
  rules:
  - host: first.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: second.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service2
            port:
              number: 80
  - http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service3
            port:
              number: 80
```

## tls

定义secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 编码的 cert
  tls.key: base64 编码的 key
type: kubernetes.io/tls
```

定义tls

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
  namespace: default
spec:
  tls: # 表示规则对应https
  - hosts:
      - https-example.foo.com
    secretName: testsecret-tls
  rules:
  - host: https-example.foo.com
    http: # https会话卸载器，面向后端是http
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
```

## annotation

配置nginx特性

https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md

### ingress控制器

多个ingress 控制器时，通过ingress指定 annotation，指定哪个控制器解析这个ingress

```diff
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
+    kubernetes.io/ingress.class: "nginx" # nginx controller解析
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```

### 流控

annotation完成



## 示例ingress(http)

```bash
kubectl create ns myns

# kubectl create deployment -n myns --dry-run=client myapp --image=a -o yaml
# kubectl --namespace='myns'   create  service  clusterip myapp  --tcp=80:80 --dry-run=client -o yaml
[root@master ingress]# pwd
/root/manifests/ingress
[root@master ingress]# cat myns-ingress-example.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      rel: beta
  strategy: {}
  template:
    metadata:
      namespace: myns
      labels:
        app: myapp
        rel: beta
    spec:
      containers:
      - image: ikubernetes/myapp:v1
        name: myapp
        resources: {}


--- 
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myns
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: myapp
    rel: beta
  type: ClusterIP

```

```bash
[root@master ingress]# kubectl apply -f myns-ingress-example.yaml 
deployment.apps/myapp created
service/myapp created
[root@master ingress]# kubectl get pod -n myns -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE                NOMINATED NODE   READINESS GATES
myapp-85476d78cc-kwhsx   1/1     Running   0          33s   10.244.3.37   node03.magedu.com   <none>           <none>
myapp-85476d78cc-wlr9x   1/1     Running   0          33s   10.244.3.36   node03.magedu.com   <none>           <none>

[root@master ingress]# kubectl get svc -n myns -o wide
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
myapp   ClusterIP   10.96.180.205   <none>        80/TCP    45s   app=myapp,rel=beta


[root@master ingress]# kubectl get all -n myns -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP            NODE                NOMINATED NODE   READINESS GATES
pod/myapp-85476d78cc-kwhsx   1/1     Running   0          52s   10.244.3.37   node03.magedu.com   <none>           <none>
pod/myapp-85476d78cc-wlr9x   1/1     Running   0          52s   10.244.3.36   node03.magedu.com   <none>           <none>

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/myapp   ClusterIP   10.96.180.205   <none>        80/TCP    53s   app=myapp,rel=beta

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                 SELECTOR
deployment.apps/myapp   2/2     2            2           53s   myapp        ikubernetes/myapp:v1   app=myapp,rel=beta

NAME                               DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                 SELECTOR
replicaset.apps/myapp-85476d78cc   2         2         2       53s   myapp        ikubernetes/myapp:v1   app=myapp,pod-template-hash=85476d78cc,rel=beta


```

访问测试

```bash
[root@master ingress]# curl 10.96.180.205/hostname.html
myapp-85476d78cc-kwhsx
[root@master ingress]# curl 10.96.180.205/hostname.html
myapp-85476d78cc-wlr9x

```

现在把这组pod通过ingress controller完成7层代理

```diff
"myapp-ingress.yaml" 21L, 469C                                                                                                                                                                                                                                                                           5,3           All
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
+  name: myapp # 不同类型下的名称一样
  namespace: myns
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
+    kubernetes.io/ingress.class: "nginx" # 选哪个控制器

spec:
  rules:
+  - host: www.magedu.com # 省略默认, 域名需要映射到ingress
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
+            name: myapp      # 上面的后端的svc
            port:
              number: 80

```

提交 

```bash
[root@master ingress]# kubectl apply -f myapp-ingress.yaml
ingress.networking.k8s.io/myapp created

# 查看ingress资源
[root@master ingress]# kubectl get ingress -n myns
NAME    CLASS    HOSTS            ADDRESS        PORTS   AGE
myapp   <none>   www.magedu.com   172.16.1.103   80      18s


[root@master ingress]# kubectl describe ingress -n myns 
Name:             myapp
Namespace:        myns
Address:          172.16.1.103
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host            Path  Backends
  ----            ----  --------
  www.magedu.com  
                  /   myapp:80 (10.244.3.36:80,10.244.3.37:80) # 对www.magedu.com/ 访问映射到myapp:80
Annotations:      kubernetes.io/ingress.class: nginx
                  nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type    Reason  Age               From                      Message
  ----    ------  ----              ----                      -------
  Normal  Sync    4s (x3 over 90s)  nginx-ingress-controller  Scheduled for sync

```

验证是否生效

```bash
[root@master ingress]# kubectl exec -it -n ingress-nginx ingress-nginx-controller-85df779996-jnm4q -- sh
/etc/nginx $ cat nginx.conf

    ...
	## start server www.magedu.com
	server {
		server_name www.magedu.com ;
		
		listen 80  ;
		listen 443  ssl http2 ;
		
		set $proxy_upstream_name "-";
		
		ssl_certificate_by_lua_block {
			certificate.call()
		}
		
		location / {
			
			set $namespace      "myns";
			set $ingress_name   "myapp";
			set $service_name   "myapp";
			set $service_port   "80";
			set $location_path  "/";
			
			rewrite_by_lua_block {
				lua_ingress.rewrite({
					force_ssl_redirect = false,
					ssl_redirect = true,
					force_no_ssl_redirect = false,
					use_port_in_redirects = false,
				})
				balancer.rewrite()
				plugins.run()
			}
			
			# be careful with `access_by_lua_block` and `satisfy any` directives as satisfy any
			# will always succeed when there's `access_by_lua_block` that does not have any lua code doing `ngx.exit(ngx.DECLINED)`
			# other authentication method such as basic auth or external auth useless - all requests will be allowed.
			#access_by_lua_block {
			#}
			
			header_filter_by_lua_block {
				lua_ingress.header()
				plugins.run()
			}
			
			body_filter_by_lua_block {
			}
			
			log_by_lua_block {
				balancer.log()
				
				monitor.call()
				
				plugins.run()
			}
			
			port_in_redirect off;
			
			set $balancer_ewma_score -1;
			set $proxy_upstream_name "myns-myapp-80";
			set $proxy_host          $proxy_upstream_name;
			set $pass_access_scheme  $scheme;
			
			set $pass_server_port    $server_port;
			
			set $best_http_host      $http_host;
			set $pass_port           $pass_server_port;
			
			set $proxy_alternative_upstream_name "";
			
			client_max_body_size                    1m;
			
			proxy_set_header Host                   $best_http_host;
			
			# Pass the extracted client certificate to the backend
			
			# Allow websocket connections
			proxy_set_header                        Upgrade           $http_upgrade;
			
			proxy_set_header                        Connection        $connection_upgrade;
			
			proxy_set_header X-Request-ID           $req_id;
			proxy_set_header X-Real-IP              $remote_addr;
			
			proxy_set_header X-Forwarded-For        $remote_addr;
			
			proxy_set_header X-Forwarded-Host       $best_http_host;
			proxy_set_header X-Forwarded-Port       $pass_port;
			proxy_set_header X-Forwarded-Proto      $pass_access_scheme;
			
			proxy_set_header X-Scheme               $pass_access_scheme;
			
			# Pass the original X-Forwarded-For
			proxy_set_header X-Original-Forwarded-For $http_x_forwarded_for;
			
			# mitigate HTTPoxy Vulnerability
			# https://www.nginx.com/blog/mitigating-the-httpoxy-vulnerability-with-nginx/
			proxy_set_header Proxy                  "";
			
			# Custom headers to proxied server
			
			proxy_connect_timeout                   5s;
			proxy_send_timeout                      60s;
			proxy_read_timeout                      60s;
			
			proxy_buffering                         off;
			proxy_buffer_size                       4k;
			proxy_buffers                           4 4k;
			
			proxy_max_temp_file_size                1024m;
			
			proxy_request_buffering                 on;
			proxy_http_version                      1.1;
			
			proxy_cookie_domain                     off;
			proxy_cookie_path                       off;
			
			# In case of errors try the next upstream server before returning an error
			proxy_next_upstream                     error timeout;
			proxy_next_upstream_timeout             0;
			proxy_next_upstream_tries               3;
			
			proxy_pass http://upstream_balancer;       # 通过lua脚本配置
			
			proxy_redirect                          off;
			
		}
		
	}
	## end server www.magedu.com

```

将主机名解析至ingress的service Nodeport对应的ip上

```bash
172.16.1.100 www.magedu.com
```

访问http://www.magedu.com:30080/

## 示例ingress(https)

需要openssl自签证书

```bash
cd manifests/ingress/
ls
openssl genrsa -out myapp.key 2048

# 自签证书
openssl req -new -x509 -key myapp.key -out myapp.crt -subj /C=CN/ST=Beijing/L=Beijing/O=Ops/CN=www.ilinux.io -days 3650
	# C 
	# ST 
	# L 
	# 部门
	# 域名
kubectl create secret -h
kubectl create secret tls -h
kubectl create secret tls ilinux-cert -n myns --cert=myapp.crt --key=myapp.key --dry-run -o yaml
kubectl create secret tls ilinux-cert -n myns --cert=myapp.crt --key=myapp.key --dry-run 、
# 创建
kubectl create secret tls ilinux-cert -n myns --cert=myapp.crt --key=myapp.key 
# 描述
kubectl describe secret  ilinux-cert -n myns

```

引用

```diff
"myapp-tls-ingress-example.yaml" 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
+  name: myapp-tls
+  namespace: myns
+  annotations:
+    kubernetes.io/ingress.class: "nginx" # nginx controller解析
spec:
  tls: # 表示规则对应https
  - hosts:
+      - www.ilinux.io
+    secretName: ilinux-cert
  rules:
+  - host: www.ilinux.io
    http: # https会话卸载器，面向后端是http
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
+            name: myapp
            port:
              number: 80
```

```bash
[root@master ingress]# kubectl apply -f myapp-tls-ingress-example.yaml
ingress.networking.k8s.io/myapp-tls created
[root@master ingress]# kubectl get ingress -n myns
NAME        CLASS    HOSTS            ADDRESS        PORTS     AGE
myapp       <none>   www.magedu.com   172.16.1.103   80        17m
													 # 虽然定义443, 但是80是保留的
myapp-tls   <none>   www.ilinux.io    172.16.1.103   80, 443   24s


[root@master ingress]# kubectl describe ingress -n myns myapp-tls
Name:             myapp-tls
Namespace:        myns
Address:          172.16.1.103
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
TLS:
  ilinux-cert terminates www.ilinux.io
Rules:
  Host           Path  Backends
  ----           ----  --------
  www.ilinux.io  
                 /   myapp:80 (10.244.3.36:80,10.244.3.37:80) # 到后端的80
Annotations:     kubernetes.io/ingress.class: nginx
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    38s (x2 over 53s)  nginx-ingress-controller  Scheduled for sync

```

查看是否生成nginx配置

```bash
 kubectl exec -it -n ingress-nginx ingress-nginx-controller-85df779996-jnm4q -- sh

	## start server www.ilinux.io
	server {
		server_name www.ilinux.io ;
		
		listen 80  ;
		listen 443  ssl http2 ;
		
		set $proxy_upstream_name "-";
		
		ssl_certificate_by_lua_block { # 证书位置
			certificate.call()
		}
...
```



配置hosts

> 172.16.1.100 www.magedu.com www.ilinux.io

访问https://www.ilinux.io:30443 失败，会跳转www.ilinux.io:443 (可能新版的ingress nginx不支持非标准端口的443请求)

但是把ingress nginx controller使用hostPort 443

```bash
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
              hostPort: 443
```

 重新apply

```bash
kubectl apply -f
```

查看pod所在的节点

```bash
[root@master ingress]# kubectl get pod -n ingress-nginx -o wide
NAME                                      READY   STATUS      RESTARTS   AGE     IP            NODE                NOMINATED NODE   READINESS GATES
ingress-nginx-admission-create-2m724      0/1     Completed   0          102m    10.244.3.33   node03.magedu.com   <none>           <none>
ingress-nginx-admission-patch-k99g7       0/1     Completed   0          102m    10.244.3.34   node03.magedu.com   <none>           <none>
ingress-nginx-controller-5ff6b455-jmvcz   1/1     Running     0          2m39s   10.244.3.38   node03.magedu.com   <none>           <none>

```

也就是node03节点的443端口可以直接到达nginx controller pod

然后再将www.ilinux.io解析至node03, 再次请求

```bash
[root@master ingress]# curl -k https://www.ilinux.io 
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
[root@master ingress]# curl -k https://www.ilinux.io/hostname.html
myapp-5b596fb756-895fk
```



## 部署Nginx ingress + Tomcat

由于以上尝试ingress(nodeport) -> tomcat 没有外置负载均衡器的原因失败。

所以以下直接使用以上的deploy(hostPort 443) -> tomcat完成https. 80还是走nodePort





准备tomcat的deploy和svc放在eshop目录

```bash
[root@master ingress]# cat tomcat.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: eshop
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat
  namespace: eshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tomcat
      rel: beta
  strategy: {}
  template:
    metadata:
      namespace: eshop
      labels:
        app: tomcat
        rel: beta
    spec:
      containers:
      - image: tomcat:alpine
        name: tomcat
        resources: {}


--- 
apiVersion: v1
kind: Service
metadata:
  name: tomcat
  namespace: eshop
spec:
  ports:
  - name: http
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: tomcat
    rel: beta
  type: ClusterIP

```

### http

准备tomcat 的ingress配置

```bash
[root@master ingress]# cat tomcat-ingress.yaml 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tomcat # 不同类型下的名称一样
  namespace: eshop
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: "nginx"
    
spec:
  rules:
  - host: www.ik8s.io # 省略默认, 域名需要映射到ingress
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tomcat
            port:
              number: 8080

```

将www.ik8s.io解析至各节点任意一个，直接访问http://www.ik8s.io:30080/



### https

准备证书

```bash
cd manifests/ingress/
ls
openssl genrsa -out tomcat.key 2048

# 自签证书
openssl req -new -x509 -key tomcat.key -out tomcat.crt -subj /C=CN/ST=Beijing/L=Beijing/O=Ops/CN=www.ik8s.io -days 3650
	# C 
	# ST 
	# L 
	# 部门
	# 域名

kubectl create secret tls ik8s-cert -n eshop --cert=tomcat.crt --key=tomcat.key 
# 描述
kubectl describe secret  ik8s-cert -n eshop

```

准备ingress

```bash
[root@master ingress]# cat tomcat-tls.yaml 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tomcat-tls
  namespace: eshop
  annotations:
    kubernetes.io/ingress.class: "nginx" # nginx controller解析
spec:
  tls: # 表示规则对应https
  - hosts:
      - www.ik8s.io
    secretName: ik8s-cert
  rules:
  - host: www.ik8s.io
    http: # https会话卸载器，面向后端是http
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tomcat
            port:
              number: 8080

```

解析域名www.ik8s.io至nginx-controller运行的位置

```bash
[root@master ingress]# kubectl get pod -n ingress-nginx -o wide
NAME                                      READY   STATUS      RESTARTS   AGE    IP            NODE                NOMINATED NODE   READINESS GATES
ingress-nginx-admission-create-2m724      0/1     Completed   0          120m   10.244.3.33   node03.magedu.com   <none>           <none>
ingress-nginx-admission-patch-k99g7       0/1     Completed   0          120m   10.244.3.34   node03.magedu.com   <none>           <none>
ingress-nginx-controller-5ff6b455-jmvcz   1/1     Running     0          20m    10.244.3.38   node03.magedu.com   <none>           <none>
```

解析至node03

```bash
172.16.1.103 www.ik8s.io
```

浏览器访问

https://www.ik8s.io/

![image-20210127181615340](http://myapp.img.mykernel.cn/image-20210127181615340.png)