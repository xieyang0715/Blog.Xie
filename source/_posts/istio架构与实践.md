---
title: istio架构与实践
date: 2021-09-22 09:26:26
tags:
- istio
---



# 前言

数据平面：实例之间网络流量部分。

控制平面：控制运行在实例旁sidecar, 用来控制实例工作行为。提供了api接口，可以使用cli/api操作，网格所有sidecar统一管理界面。



<!--more-->
# 为什么使用服务网格

ingress controller 南北流量，进入服务后的东西流量。

- 服务间安全无从保证.          `TLS/MTLS`
- 跟踪通信延迟非常困难.      分页式跟踪
- 负载均衡功能有限.              ipvs调度，负载均衡能力有限。流量分割、迁移、镜像, ...。 kubernetes ingress对整个集群适用，但是管理特定应用构建高级的负载均衡，也有限。



# 服务网格的功能

- 流量
  - http 1, http2(grpc)
  - 动态路由 - 条件，权重，镜像
  - 服务韧性 - 超时、重试、熔断
  - 策略 - 访问控制 ，速率控制，配额
  - 测试  - 错误注入
- 安全
  - tls， mtls
  - 严格认证 - spiffe 
  - auth
- 可观测
  - metrics - 黄金指标，`prometheus`,` grafana`
  - tracing -  `jaeger`, `zipkin`, `netstep`
  - traffic     - cluster, ingress/egress.
- mesh
  - 部署环境 `kubernetes`, virtual machines
  - 多集群 - gateways, federation



# istio?

## 是什么

是envoy数据平面的控制平面实现之一。

istio本身就包含了envoy，Istio独立服务网格，有控制平面，数据平面是envoy实现的，不是原生 envoy, 而是对envoy有扩展。



istio就实现了 [服务网格的功能](#服务网格的功能) 列出的所有功能。



官方: https://istio.io/



## 架构

![image-20210922102230376](http://myapp.img.mykernel.cn/image-20210922102230376.png)

> `istio`提供了envoy注入：手工注入、自动注入，通用的sidecar, 业务配置，通过`XDS API`可以动态修改其配置。

控制平面组件

- Pilot:       其实就是`XDS Server`, 可以为envoy提供动态配置。 **核心组件**
  - 分发认证策略和名称信息: mtls/tls/jwt?
- Mixer:     与policy相关，可观测性相关。
  - 授权和审计
- Citadel:  安全相关，服务间基于`mtls`认证，此组件用来颁发证书。类似于spire实现，甚至更强大。
  - 证书轮替
- Galley:    服务于控制平面的组件，检验用户提交的用于下发给envoy的配置，是否合规。
  - 验证配置OK，并使用mcp协议分发配置给pilot和mixer.

数据平面组件

- 部署所有服务时，注入到服务pod内部的sidecar组成。



官方架构图，此为最新版本，上课还有mixer，此组件对性能有影响可能已经取消了

![官方架构图](https://istio.io/latest/docs/ops/deployment/architecture/arch.svg)

ingress, egress gateway 与k8s的ingress controller有区别 

![high arch](https://istio.io/latest/docs/concepts/security/arch-sec.svg)

## sidecar injector

手工注入: istioctl客户端工具注入

自动注入：istio sidecar injector 自动完成注入, 需要名称空间打上特别标签来实现。





## istio可视化

- `grafana` 
- `kiali`    redhat研发开源，专用于istio可视化，拓扑、分布式跟踪、....



## 部署istio

部署环境

- 通用`k8s`环境之上
- aliyun ack
- micro k8s
- openshift

1.6平台之上，部署1.4的istio, 尝试服务网格

# istio实践

## 部署依赖

istioctl 快速部署istio-1.4， 基础平台使用kubernetes 1.16.3版本。

系统环境无所谓

## 部署k8s 1.16.3

`aws centos 7` 失败，不支持iptables REDIRECT.

`aws ubuntu 18.04 LTS` 

配置`docker-ce 18.09`

```bash
apt-get -y install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
apt-cache madison docker-ce
apt-get -y install docker-ce=5:18.09.9~3-0~ubuntu-bionic
```

配置源`kubernetes`

```bash
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt update
```

```bash
# k8s部署
root@ip-10-0-0-245:~# apt-cache madison kubeadm | grep 16.3
   kubeadm |  1.16.3-00 | https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages

# apt install kubeadm=1.16.3-00  kubelet=1.16.3-00 ipset ipvsadm
```

`/etc/modules-load.d/10-k8s-modules.conf`

```bash
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
```

>  内核版本大于4.19时，使用`nf_conntrack`
>
>  加载ipvs模块
>
>  ```bash
>  [root@ip-10-0-0-91 kubernetes]# for i in $(cat /etc/modules-load.d/10-k8s-modules.conf); do     modprobe -v $i; done
>  ```
>
>  验证
>
>  ```bash
>  lsmod | grep ip_vs
>  ```



`kubeadm-1.16.3.yaml`

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.0.0.91
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: master-10.0.0.91
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.16.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
```

```bash
[root@ip-10-0-0-91 kubernetes]# kubeadm init --config ./kubeadm-1.16.3.yaml 
```

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
[root@ip-10-0-0-91 kubernetes]# kubectl get node
NAME               STATUS   ROLES    AGE    VERSION
master-10.0.0.91   Ready    master   2m3s   v1.16.3
```

验证ipvs

```bash
[root@ip-10-0-0-91 kubernetes]# kubectl get cm -n kube-system kube-proxy  -o yaml | grep mode
    mode: ipvs
```

部署flannel: https://github.com/flannel-io/flannel/blob/master/Documentation/kubernetes.md#for-kubernetes-v16v115

## 部署istio 1.4

- 安装istio: istioctl
- 安装应用：kubectl apply
- 管理istio, 使用CRD资源配置， pilot将配置应用到envoy.

### 部署istio集群

准备istio 1.4 `https://github.com/istio/istio/releases/tag/1.4.0`

下载`istio-1.4.0-linux.tar.gz`

```bash
[root@ip-10-0-0-91 ~]# tar xvf istio-1.4.0-linux.tar.gz -C /usr/local/
[root@ip-10-0-0-91 ~]# cd /usr/local
[root@ip-10-0-0-91 local]# ln -sv istio-1.4.0/ istio
[root@ip-10-0-0-91 ~]# cat /etc/profile.d/istio.sh
export PATH=/usr/local/istio/bin:$PATH
. /etc/profile.d/istio.sh
[root@ip-10-0-0-91 ~]# cd /usr/local/istio
[root@ip-10-0-0-91 istio]# ls
bin  install  LICENSE  manifest.yaml  README.md  samples  tools

```

> `install` 部署istio的文档
>
> `samples` 示例程序
>
> `tools` 一些工具程序

部署要求

- 使用的很多端口
- pod只能属于1个service, port不能交叉使用，不能乱用
- label添加上应用程序版本
- 1337 uid用户不能被占用
- `netadmin`功能，流量拦截功能需要这个。

> 自己部署的kubernetes环境，这些要求基本都会满足的。

istio内置配置示例

- default:     不指定profile时，默认使用default profile. 内部设定适用于生产环境。  
- demo:       适合bookinfo一类的应用程序
  - 除了`SDS`，均会启用
- minimal:  流量管理的最小化部署，
  - pilot, xds ms
- sds:           类似于default, 额外启用SDS
- remote:    多个kubernetes使用同一个istio.



部署操作

```bash
# 查看可以使用的配置
root@ip-10-0-0-245:/usr/local/istio# istioctl profile list
Istio configuration profiles:
    sds
    default
    demo
    minimal
    remote


# 基于demo启动
istioctl manifest apply --set profile=demo
# 执行一个报错，不关心 no matches for kind "CronJob" in version "batch/v1"
```

> 不同配置的区别，参考: https://istio.io/latest/zh/docs/setup/additional-setup/config-profiles/
>
> 我们学习使用demo, 就不使用`SDS`， 可以完备的了解`istio`了

```bash
[root@ip-10-0-0-91 ~]# kubectl get pods -A
NAMESPACE      NAME                                       READY   STATUS    RESTARTS   AGE
istio-system   grafana-6b65874977-w24ps                   1/1     Running   0          51s
istio-system   istio-citadel-f74b87856-vp8k6              1/1     Running   0          52s
istio-system   istio-egressgateway-c56696f94-nm9ct        1/1     Running   0          52s
istio-system   istio-galley-566bcc7758-hrvk4              1/1     Running   0          52s
istio-system   istio-ingressgateway-6b9987bf94-jjcxn      1/1     Running   0          51s
istio-system   istio-pilot-677c9c688f-btg6p               1/1     Running   0          52s
istio-system   istio-policy-58fb4d586c-bkm67              1/1     Running   1          52s
istio-system   istio-sidecar-injector-85657c7bcb-p8g4g    1/1     Running   0          51s
istio-system   istio-telemetry-666b985499-wzfzr           1/1     Running   1          52s
istio-system   istio-tracing-c66d67cd9-566z8              1/1     Running   0          53s
istio-system   kiali-8559969566-p2w6l                     1/1     Running   0          52s
istio-system   prometheus-66c5887c86-ksbm5                1/1     Running   0          53s

[root@ip-10-0-0-91 ~]# kubectl get svc -A

```

验证集群部署是否正常

```bash
[root@ip-10-0-0-91 ~]# istioctl manifest generate --set profile=demo | istioctl verify-install -f -
Checked 23 crds
Checked 9 Istio Deployments
Istio is installed successfully
```

> `[root@ip-10-0-0-91 ~]# wc -l /tmp/demo.yaml
> 24959 /tmp/demo.yaml`

## 部署bookinfo应用

管理工作,使用 `kubectl` 和 `CRD`完成配置

### 分布式应用架构

![image-20210922164004891](http://myapp.img.mykernel.cn/image-20210922164004891.png)

> - product page: 生产页面
> - reviews-v1/v2/v3 评论
> - ratings 评论
> - details 详情页面
> - 使用服务网格解耦了某种语言和系统关联
> - 后面数据平面都注入了sidecar. 任何到达pod的流量 ，通过Iptables拦截然后转发给应用代码。应用响应代码，再被拦截，再转发出去。
> - ingress envoy 引入服务网格外部流量到网格内部。非sidecar运行。

官方详细介绍: https://istio.io/latest/docs/examples/bookinfo/

Bookinfo应用程序分为四个单独的微服务：

- `productpage`。这个`productpage`微服务调用`details`和`reviews`用于填充页面的微服务。
- `details`。这个`details`微服务包含图书信息。
- `reviews`。这个`reviews`微服务包含书评。它也称`ratings`微型服务。
- `ratings`。这个`ratings`微服务包含与书评相关的图书排名信息。

有3个版本的`reviews`微型服务：

- 版本1不调用`ratings`服务。
- 版本2调用`ratings`服务，并将每个评等显示为1至5颗黑星。
- 版本3调用`ratings`服务，并将每个评等显示为1至5颗红星.



### 部署

部署时没有envoy代码，所以部署应用只有应用本身。要想添加sidecar. 

- 手工添加sidecar
- 名称空间自动添加sidecar， 为了简便，使用此方式
- istioctl手工添加sidecar

#### 启动bookinfo

```bash
# kubectl label namespace default istio-injection=enabled
# kubectl get ns --show-labels
NAME              STATUS   AGE   LABELS
default           Active   37m   istio-injection=enabled


# cd /usr/local/istio/
# kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

> If you disabled automatic sidecar injection during installation and rely on [manual sidecar injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#manual-sidecar-injection), use the [`istioctl kube-inject`](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-kube-inject) command to modify the `bookinfo.yaml` file before deploying your application.
>
> ```
> $ kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
> ```

可以发现名称空间

```bash
root@ip-10-0-0-245:/usr/local/istio# kubectl get pod 
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-78d78fbddf-m2zxr       2/2     Running   0          95s
productpage-v1-596598f447-grr5g   2/2     Running   0          95s
ratings-v1-6c9dbf6b45-qhf8n       2/2     Running   0          95s
reviews-v1-7bb8ffd9b6-rlnpq       2/2     Running   0          95s
reviews-v2-d7d75fff8-cvwjj        2/2     Running   0          96s
reviews-v3-68964bc4c8-bc8rj       2/2     Running   0          96s
```

> 现在deploy的yaml还是原来
>
> ```yaml
> kubectl get deploy productpage-v1 -o yaml
> ```
>
> 但是pod的yaml已经添加initcontainer, sidecar
>
> ```bash
> kubectl get pod productpage-v1-596598f447-grr5g -o yaml
> ```
>
> > 1. 初始化容器会拦截流量 
> > 2. 入站ingress, 出站egress.
> > 3. istio的pod中注入side后，代理容器就istio-proxy, 和业务代码自身

#### 验证bookinfo

```bash
root@ip-10-0-0-245:/usr/local/istio# kubectl get services
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.103.133.168   <none>        9080/TCP   5m18s
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    18m
productpage   ClusterIP   10.109.112.238   <none>        9080/TCP   5m18s
ratings       ClusterIP   10.100.194.95    <none>        9080/TCP   5m18s
reviews       ClusterIP   10.102.85.218    <none>        9080/TCP   5m18s
```

确保productpage, 监听在9080

```bash
root@ip-10-0-0-245:/usr/local/istio# kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```

> 显示出Simple Bookstore App,表示正常了
>
> 只要显示了，说明东西流量 正常了



验证南北向流量正常， product page 通过ingress gateway暴露出去,  

`ingress gateway` -> `productpage`

1. 应用listener和路由

    ```bash
    kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
    ```

    > ```yaml
    > root@ip-10-0-0-245:/usr/local/istio# cat samples/bookinfo/networking/bookinfo-gateway.yaml 
    > apiVersion: networking.istio.io/v1alpha3 # CRD
    > kind: Gateway   # 定义listener
    > metadata:
    >   name: bookinfo-gateway # 名称
    > spec: # 期望状态有2个。
    >   selector:   # selector表示servers定义的规则将生效在哪个envoy之上
    >     istio: ingressgateway # ingressgateway表示 生效在ingress gateway controller， 即 istio-system名称空间的istio-ingressgateway的deploy.
    >   servers: # gateway之上内部是单独的envoy, 而不是sidecar的envoy
    >   - port: # envoy的listener
    >       number: 80      
    >       name: http
    >       protocol: HTTP
    >     hosts: 
    >     - "*"   
    > ---
    > apiVersion: networking.istio.io/v1alpha3 # CRD
    > kind: VirtualService # envoy的路由, 流量转发还需要http-connection-manager, 通过router过滤器定义转发。
    > metadata:
    >   name: bookinfo
    > spec:
    >   hosts:    # 规则会应用在哪个envoy上，*所有envoy都会应用
    >   - "*"
    >   gateways: # 规则应用在 gateway的listener之上
    >   - bookinfo-gateway
    >   http: # 7层代理
    >   - match:
    >     - uri:
    >         exact: /productpage # 精确
    >     - uri:
    >         prefix: /static # 前缀
    >     - uri:
    >         exact: /login
    >     - uri:
    >         exact: /logout
    >     - uri:
    >         prefix: /api/v1/products
    >     route: # 路由目标
    >     - destination:
    >         host: productpage # 调用哪个cluster(由DestinationRule定义)
    >         port:
    >           number: 9080
    > ```
    >
    > > ingress-gateway把流量给了productpage了，没有问题。
    > >
    > > productpage如何把流量发给内部服务？ `samples/bookinfo/networking/destination-rule-all.yaml`
    > >
    > > ```yaml
    > > apiVersion: networking.istio.io/v1alpha3
    > > kind: DestinationRule # CRD envoy的cluster内部细节定义
    > > metadata:
    > >   name: productpage # productpage集群的细节定义
    > > spec:
    > >   host: productpage # 请求到了productpage服务
    > >   subsets: # 定义子集
    > >   - name: v1 # 用户请求发给内部v1版本
    > >     labels:
    > >       version: v1
    > > ---
    > > apiVersion: networking.istio.io/v1alpha3
    > > kind: DestinationRule
    > > metadata:
    > >   name: reviews # 
    > > spec:
    > >   host: reviews ## 请求到了 reviews 服务
    > >   subsets: # 3个版本，负载均衡子集，子集选择器归类到不同子集，流量分配到不同子集。
    > >   - name: v1
    > >     labels:
    > >       version: v1
    > >   - name: v2
    > >     labels:
    > >       version: v2
    > >   - name: v3
    > >     labels:
    > >       version: v3
    > > ---
    > > apiVersion: networking.istio.io/v1alpha3
    > > kind: DestinationRule # CRD 此下的子集，可能不存在mysql, 只管应用即可。
    > > metadata:
    > >   name: ratings # ratings 只有v1版本存在
    > > spec:
    > >   host: ratings
    > >   subsets:
    > >   - name: v1
    > >     labels:
    > >       version: v1
    > >   - name: v2
    > >     labels:
    > >       version: v2
    > >   - name: v2-mysql
    > >     labels:
    > >       version: v2-mysql
    > >   - name: v2-mysql-vm
    > >     labels:
    > >       version: v2-mysql-vm
    > > ---
    > > apiVersion: networking.istio.io/v1alpha3
    > > kind: DestinationRule
    > > metadata:
    > >   name: details # 由架构图 只有v1版本存在
    > > spec:
    > >   host: details
    > >   subsets:
    > >   - name: v1
    > >     labels:
    > >       version: v1
    > >   - name: v2
    > >     labels:
    > >       version: v2
    > > ---
    > > 
    > > ```
    > >
    > > 

    验证

    ```bash
    root@ip-10-0-0-245:/usr/local/istio# kubectl get gateway
    NAME               AGE
    bookinfo-gateway   8s
    root@ip-10-0-0-245:/usr/local/istio# kubectl describe gateway
    ```

    ```bash
    root@ip-10-0-0-245:/usr/local/istio# kubectl get virtualservices.networking.istio.io 
    NAME       GATEWAYS             HOSTS   AGE
    bookinfo   [bookinfo-gateway]   [*]     33s
    root@ip-10-0-0-245:/usr/local/istio# kubectl describe virtualservices.networking.istio.io 
    ```

    > 可以简写 vs

2. 创建集群定义

   ```bash
   kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
   ```

   ```bash
   root@ip-10-0-0-245:/usr/local/istio# kubectl get destinationrules.networking.istio.io
   NAME          HOST          AGE
   details       details       6s
   productpage   productpage   6s
   ratings       ratings       6s
   reviews       reviews       6s
   ```

   > 可以简写为 dr

3. 集群外部访问内部 ingress gateway nodePort/LoadBalancer/ ingress controller实现

    ingress gateway `nodeport`

    ```bash
    kubectl edit svc -n istio-system  istio-ingressgateway
    type: nodeport
    80 -> 30080
    443 -> 30443
    ```

    > 正常应该SLB（80、443）-> ingress gateway的nodeport(30080/30443)

    现在由于 ingress 路由定义的是

    ```yaml
            exact: /productpage # 精确
        - uri:
            prefix: /static # 前缀
        - uri:
            exact: /login
        - uri:
            exact: /logout
        - uri:
            prefix: /api/v1/products
    ```

    所以，我们访问:  http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage

    刷新3次，就有3种显示， 如下，原因是，虽然我们使用了子集选择器，但是我们在路由上调用子集元数据来分割流量 ，所以就是RR调度，更详细参考:

    > http://blog.mykernel.cn/2021/09/03/Envoy%E9%9B%86%E7%BE%A4%E7%AE%A1%E7%90%86/#%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E5%99%A8%E5%AD%90%E9%9B%86%E8%B0%83%E5%BA%A6

    ![image-20210923102123953](http://myapp.img.mykernel.cn/image-20210923102123953.png)

    

![image-20210923102134858](http://myapp.img.mykernel.cn/image-20210923102134858.png)



![image-20210923102143532](http://myapp.img.mykernel.cn/image-20210923102143532.png)

#### 通过kali监控istio

查看svc

```bash
root@ip-10-0-0-245:/usr/local/istio# kubectl get svc -n istio-system   kiali
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
kiali   ClusterIP   10.104.61.176   <none>        20001/TCP   97m
```

暴露到外部

- nodeport
- ingress gateway -> kiali

教程

> https://istio.io/latest/zh/docs/tasks/observability/kiali/

1. 创建账号和密码

   ```bash
   KIALI_USERNAME=$(read -p 'Kiali Username: ' uval && echo -n $uval | base64)
   KIALI_PASSPHRASE=$(read -sp 'Kiali Passphrase: ' pval && echo -n $pval | base64)
   ```

   ```bash
   cat <<EOF | kubectl apply -f -
   apiVersion: v1
   kind: Secret
   metadata:
     name: kiali
     namespace: $NAMESPACE
     labels:
       app: kiali
   type: Opaque
   data:
     username: $KIALI_USERNAME
     passphrase: $KIALI_PASSPHRASE
   EOF
   ```

   > ```bash
   > root@ip-10-0-0-245:/usr/local/istio# kubectl get secret kiali
   > NAME    TYPE     DATA   AGE
   > kiali   Opaque   2      7s
   > ```

   重启kiali的pod, 加载 secret, 或者等一段时间

2. 添加 ingress gateway 暴露kiali服务

   https://istio.io/latest/zh/docs/tasks/observability/gateways/

   其中找到http暴露

   ```bash
   cat <<EOF | kubectl apply -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: kiali-gateway
     namespace: istio-system
   spec:
     selector:
       istio: ingressgateway
     servers:
     - port:
         number: 15029 # ingress gateway的15029的端口，要么nodeport, 要么loadbalancer 15029
         name: http-kiali
         protocol: HTTP
       hosts:
       - "*"
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: kiali-vs
     namespace: istio-system
   spec:
     hosts:
     - "*"
     gateways:
     - kiali-gateway
     http:
     - match:
       - port: 15029
       route:
       - destination:
           host: kiali
           port:
             number: 20001
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: kiali
     namespace: istio-system
   spec:
     host: kiali
     trafficPolicy:
       tls:
         mode: DISABLE
   ---
   EOF
   ```

   > ```bash
   > gateway.networking.istio.io/kiali-gateway created
   > virtualservice.networking.istio.io/kiali-vs created
   > destinationrule.networking.istio.io/kiali created
   > ```
   >
   > kiali端口 `15029:32712`
   >
   > ```bash
   > root@ip-10-0-0-245:/usr/local/istio# kubectl get svc -n istio-system   istio-ingressgateway 
   > NAME                   TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)                                                                                                                      AGE
   > istio-ingressgateway   NodePort   10.109.63.7   <none>        15020:31688/TCP,80:30080/TCP,443:30443/TCP,15029:32712/TCP,15030:31768/TCP,15031:31864/TCP,15032:31307/TCP,15443:30372/TCP   107m
   > ```

3. 访问 `32712`

   ![image-20210923112014401](http://myapp.img.mykernel.cn/image-20210923112014401.png)

   用户: admin, 密码: admin

4. 开放grafana服务

   ```bash
   root@ip-10-0-0-245:/usr/local/istio# kubectl get pod -n istio-system   grafana-6b65874977-tzlmm  
   NAME                       READY   STATUS    RESTARTS   AGE
   grafana-6b65874977-tzlmm   1/1     Running   0          112m
   ```

   ```bash
   root@ip-10-0-0-245:/usr/local/istio# kubectl get svc -n istio-system   grafana
   NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
   grafana   ClusterIP   10.110.177.241   <none>        3000/TCP   113m
   ```

   ```bash
   cat <<EOF | kubectl apply -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: grafana-gateway
     namespace: istio-system
   spec:
     selector:
       istio: ingressgateway
     servers:
     - port:
         number: 15031
         name: http-grafana
         protocol: HTTP
       hosts:
       - "*"
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: grafana-vs
     namespace: istio-system
   spec:
     hosts:
     - "*"
     gateways:
     - grafana-gateway
     http:
     - match:
       - port: 15031
       route:
       - destination:
           host: grafana
           port:
             number: 3000
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: grafana
     namespace: istio-system
   spec:
     host: grafana
     trafficPolicy:
       tls:
         mode: DISABLE
   ---
   EOF
   
   ```

   > ```bash
   > gateway.networking.istio.io/grafana-gateway created
   > virtualservice.networking.istio.io/grafana-vs created
   > destinationrule.networking.istio.io/grafana created
   > ```

   `15031:31864`

   内建Istio相关的面板，有了grafana才会有kiali的数据

5. 访问 kiali 生成图

   graph -> default名称空间

   ![image-20210923112540883](http://myapp.img.mykernel.cn/image-20210923112540883.png)



接下来就可以尝试服务网格的相关特性

- 请求路由
- 故障注入
- 流量迁移
- 查询metric
- collecting log
- 速率限制
- ingress gateway
- 访问外部服务
- 可视化mesh



#### 请求路由

##### 基于用户的路由

https://istio.io/latest/docs/tasks/traffic-management/request-routing/#route-based-on-user-identity

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

> ```yaml
> apiVersion: networking.istio.io/v1alpha3
> kind: VirtualService
> metadata:
>   name: reviews
> spec:
>   hosts:
>     - reviews
>   http:
>   - match:
>     - headers:
>         end-user:
>           exact: jason
>     route:
>     - destination:
>         host: reviews
>         subset: v2
>   - route:
>     - destination:
>         host: reviews
>         subset: v1
> ```

登陆jason时，就路由到v2. 其他的路由到v1.

删除后, 恢复

```bash
kubectl delete -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```



#### 故障注入

https://istio.io/latest/docs/tasks/traffic-management/fault-injection/

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

> ```yaml
> apiVersion: networking.istio.io/v1beta1
> kind: VirtualService
> ...
> spec:
>   hosts:
>   - ratings
>   http:
>   - fault:
>       delay:
>         fixedDelay: 7s
>         percentage:
>           value: 100
>     match:
>     - headers:
>         end-user:
>           exact: jason
>     route:
>     - destination:
>         host: ratings
>         subset: v1
>   - route:
>     - destination:
>         host: ratings
>         subset: v1
> ```



#### 流量迁移

https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/

```bash
 kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

> ```yaml
> apiVersion: networking.istio.io/v1beta1
> kind: VirtualService
> ...
> spec:
>   hosts:
>   - reviews
>   http:
>   - route:
>     - destination:
>         host: reviews
>         subset: v1
>       weight: 50
>     - destination:
>         host: reviews
>         subset: v3
>       weight: 50
> ```
>
> > 负载均衡子集，匹配指定元数据，到不同的集群。

### 清理bookinfo和istio集群

清理bookinfo

```bash
root@ip-10-0-0-245:/usr/local/istio#  samples/bookinfo/platform/kube/cleanup.sh 
```

卸载服务网格

```bash
istioctl manifest generate --set profile=demo | kubectl delete -f -
```

查看

```bash
root@ip-10-0-0-245:/usr/local/istio# kubectl get pods -A 
NAMESPACE     NAME                                        READY   STATUS    RESTARTS   AGE
kube-system   coredns-58cc8c89f4-7944l                    1/1     Running   0          149m
kube-system   coredns-58cc8c89f4-v7hzt                    1/1     Running   0          149m
kube-system   etcd-master-10.0.0.245                      1/1     Running   0          148m
kube-system   kube-apiserver-master-10.0.0.245            1/1     Running   0          148m
kube-system   kube-controller-manager-master-10.0.0.245   1/1     Running   0          148m
kube-system   kube-flannel-ds-r76wj                       2/2     Running   0          140m
kube-system   kube-proxy-6kfsr                            1/1     Running   0          149m
kube-system   kube-scheduler-master-10.0.0.245            1/1     Running   0          148m
root@ip-10-0-0-245:/usr/local/istio# 
```

现在可以定制安装集群，使用`default` 配置，prometheus, kiali, grafana, tracing都不会部署。

