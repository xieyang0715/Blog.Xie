---
title: "Istio Pilot和流量治理"
date: 2021-09-23 13:46:34
tags:
- istio
---



# 本节话题

[istio架构与实践](http://blog.mykernel.cn/2021/09/22/istio架构与实践/) 以前bookinfo的部署，并开放到k8s之外

- istio流量治理基础及相关的CRD
- ingress gateway和egress gateway功能概述
- 网格东西向流量管理及配置机制
  - virtual service 配置envoy路由
  - `destinationrule` 配置envoy集群及相关管理机制
- 南北流量管理及配置机制
  - 网关: gateway, listner
  - 服务端点 serviceentry
- 配置envoy
  - 特定sidecar定义 
  - envoyfilter定制宽泛的envoy配置。





# Pilot分发机制

相当于XDS的server, 通过用户配置或服务注册表, 获取数据平面的配置，并转换为`XDS API`标准格式，经由GRPC分发给相关的envoy（gateway, sidecar）.

pilot将这些配置保存在底层平台的api server:

- `service registry`   某种意义上讲还是kuberentes api server, 通过k8s注册表获取的配置。
- `config storage` kubernetes api server， 用户通过kubectl apply的CRD配置。

pilot通过id和cluster, 来标识配置信息属于具体哪个envoy.

![image-20210923142712420](http://myapp.img.mykernel.cn/image-20210923142712420.png)

- 数据平面组件

  - `istio proxyv2`就提供了pilot-agent和envoy, 
    - pilot-agent: 监控和管理当前容器的envoy状态，envoy有问题时，重启。envoy配置变动时，有必要情况下会重载envoy.
    - envoy：基于`k8s api`生成的bootstrap配置启动，根据配置中的pilot地址，通过XDS API获取动态配置信息。工作为app的sidecar，流量拦截机制为应用程序实现入站和出站代理功能。

- 控制平面组件

  - `pilot-discovery` 即图中 `XDS SERVER`, 主要有以下几个功能
    - 从`k8s api`获取配置，从service registry(`k8s api`中)获取配置
    - 将配置信息和**服务信息**转换成envoy的配置格式，通过`XDS API`分发

  - `k8s api`: 存储用户`CRD`格式(`VirtualService`和`DestinationRule`)提供配置信息。

- 生成CRD

  - kubectl命令
  - istioctl命令

  

# `CRD`

| name              | effect                                  | 归属  |
| ----------------- | --------------------------------------- | ----- |
| `VirtualService`  | 路由                                    | pilot |
| `DestinationRule` | cluster                                 | pilot |
| `Gateway`         | ingress gateway的listener               | pilot |
| `ServiceEntry`    | egress 中使用                           | pilot |
| `EnvoyFilter`     | envoy添加自定义的过滤器                 | pilot |
| `sidecar`         | 为sidecar模式运行的envoy，定制listener. |       |

# ingress gateway和egress gateway功能概述

ingress gateway 引入南北流量，从网格外到网格内

egress gateway 引入南北流量 ，从网格内到网格外

# 网格东西向流量管理及配置机制

## virtual services

路由规则组成

按顺序自上而下评估

路由到何处: route/redirect/direct_response

- service: 服务，就是kubernetes的service. 
  - hostname
  - port
- source:  发起请求服务
- host: 对方请求的目标服务名

关键字段

- hosts: 必给，可以DNS/IP地址，相当于envoy的单个virtualhost，
- gateways：可选 
  - 不给时，, 则表示此virtual service不会应用于gateway(ingress/egress), 此应用于工作于sidecar的envoy.
  - 给了gateway, 此vs应用于gateway或sidecar.
    - 仅用于gateway, 给gateway字段合适的值。
    - 同时应用时，gateways字段应该有一个值为mesh
- http:  7层
- tls：https
- tcp: 4层

## destination rules

为cluster提供具体而特定的定义

负载均衡配置

sidecar连接池

异常值检测

字段

- host 短域名，服务注册中心注册的服务名或serviceentry注册的网格外的服务
- trafficpolicy: 负载均衡、连接池、异常值检测
- subsets 子集
- 控制destinationrule可见性，"."当前名称空间可见， "*"表示所有名称空间

## 字段定义

- DR: 当前pod定义在其他sidecar的集群

  - 熔断，只需要DR就生效
  - 子集，必须VS配置服务路由指定指定子集才生效。

- VS: 当前pod的路由

  - fault

  - weighted

  - timeout

  - mirror

    > vs 
    >
    > - host+subset, 需要dr。
    > - host + port, 不需要dr, 直接路由到service.

  vs配置gateways时，host是service，配置前端到service. `destination: host/port`

   一般 vs不配置gateways时，host是service的名称。配置哪个service的路由。 `destination: host/subset`

  dr配置哪个service相关的熔断，或者将service的主机分组

  

  单个版本：

  1. 定义service, 标签只能引用1个版本
  2. vs直接，引用服务名+对应的端口。

  多版本并存

  1. 定义service, 通用标签引用所有版本。
  2. destinationrule, 将service里面的不同版本分组。
  3. 使用vs，基于不同路由匹配不同版本。

![VirtualService-DestinationRule](http://myapp.img.mykernel.cn/VirtualService-DestinationRule.png)

# 南北流量管理及配置机制

## gateway

- ingress gateway 网格外部访问内部
- egress gateway   网格内部访问外部

## ingress gateway

引入外部流量 

- gateway
- virtual service
- destination rule

ingress gateway 可以有多个副本，或者甚至多个ingress gateway deploy

字段

- selector:  应用于哪个ingress gateway deploy, 此为选择Envoy pod的标签。

- servers 开放的服务列表

  - port 发布的端口
  - hosts 监听的ip, * 表示所有地址, 也表示流量请求的目标地址。
  - defaultEndpoint:  默认后端
  - tls: port工作在tls

  

## serviceEntry

定义网格外部的服务或网格内但未注册到网格的服务 为serviceEntry, 从而让网格统一管理网格外的服务

https://my.oschina.net/u/4197945/blog/5265187

## 字段定义

![gateway-serviceentry](http://myapp.img.mykernel.cn/gateway-serviceentry.png)

# 配置`default` istio

## 部署

生产环境中，不一定每一套环境都使用一套grafana, kiali, tracing. 

```bash
root@ip-10-0-0-245:~# kubectl get node
NAME                STATUS   ROLES    AGE   VERSION
master-10.0.0.245   Ready    master   23h   v1.16.3
```

```bash
root@ip-10-0-0-245:~# istioctl profile list
Istio configuration profiles:
    demo
    minimal
    remote
    sds
    default
```

安装检验

```bash
root@ip-10-0-0-245:~# istioctl verify-install

Checking the cluster to make sure it is ready for Istio installation...

#1. Kubernetes-api
-----------------------
Can initialize the Kubernetes client.
Can query the Kubernetes API Server.

#2. Kubernetes-version
-----------------------
Istio is compatible with Kubernetes: v1.16.0.

#3. Istio-existence
-----------------------
Istio will be installed in the istio-system namespace.

#4. Kubernetes-setup
-----------------------
Can create necessary Kubernetes configurations: Namespace,ClusterRole,ClusterRoleBinding,CustomResourceDefinition,Role,ServiceAccount,Service,Deployments,ConfigMap. 

#5. SideCar-Injector
-----------------------
This Kubernetes cluster supports automatic sidecar injection. To enable automatic sidecar injection see https://istio.io/docs/setup/kubernetes/additional-setup/sidecar-injection/#deploying-an-app

-----------------------
Install Pre-Check passed! The cluster is ready for Istio installation. # 通过检查
```

### 正常部署

安装`default`

```bash
root@ip-10-0-0-245:~# istioctl manifest apply --set profile=default
```

> 相对于demo, 缺了egress, nodeagent, grafana, istio-tracing, kiali组件

> 启动后
>
> ```bash
> root@ip-10-0-0-245:~# kubectl get ns istio-system 
> NAME           STATUS   AGE
> istio-system   Active   109s
> 
> root@ip-10-0-0-245:~# kubectl get pods -n istio-system 
> NAME                                      READY   STATUS    RESTARTS   AGE
> istio-citadel-f74b87856-9rlhb             1/1     Running   0          76s
> istio-galley-6b9d654b8c-zrzk8             2/2     Running   0          76s
> istio-ingressgateway-76c6d6485c-xxj5f     1/1     Running   0          76s
> istio-pilot-d546f6d75-p2s2b               2/2     Running   0          76s
> istio-policy-bbcc666cc-5sjqf              2/2     Running   0          76s
> istio-sidecar-injector-85657c7bcb-rfjmr   1/1     Running   0          76s
> istio-telemetry-7f8bc9476-fbw6h           2/2     Running   1          76s
> prometheus-66c5887c86-n2b9m               1/1     Running   0          76s
> ```

### values部署

查看`default`配置文件,  像是helm的values文件

```bash
root@ip-10-0-0-245:~# istioctl profile dump default
```

应用时设定属性, 这样即便default, 也可以开grafana

```bash 
root@ip-10-0-0-245:~# istioctl manifest apply --set  profile=default --set values.grafana.enabled=true
```

> 由于我们仅需要values开头的段，所以为了方便查看
>
> ```bash
> istioctl profile dump default --config-path values.grafana
> ```

### 纯手工定制安装

如果完全想手工安装

```bash
root@ip-10-0-0-245:~# istioctl manifest generate --set profile=default > /tmp/istio-default.yaml
root@ip-10-0-0-245:~# kubectl apply -f /tmp/istio-default.yaml 
```

验证安装部署

```bash
root@ip-10-0-0-245:~# istioctl verify-install -f /tmp/istio-default.yaml 
.......
Checked 23 crds
Checked 7 Istio Deployments
Istio is installed successfully
```

如果基于原有方式添加额外的组件，参考values部署


## 部署kiali, grafana, tracing

https://istio.io/latest/zh/docs/tasks/observability/kiali/#install-Via-%60istioctl%60

创建kiali使用的账号和密码

   https://istio.io/latest/zh/docs/tasks/observability/kiali/

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

启动3个组件

```bash
istioctl manifest apply --set profile=default \
	--set values.kiali.enabled=true --set values.grafana.enabled=true --set values.tracing.enabled=true \
	--set "values.kiali.dashboard.jaegerURL=http://jaeger-query:16686" \
	--set "values.kiali.dashboard.grafanaURL=http://grafana:3000" 
```

开放kiali, grafana, jaeger-query 为nodePort

```bash
root@ip-10-0-0-245:~# kubectl edit svc -n istio-system   jaeger-query
root@ip-10-0-0-245:~# kubectl edit svc -n istio-system   grafana
root@ip-10-0-0-245:~# kubectl edit svc -n istio-system   kiali
```

## 部署bookinfo

https://istio.io/latest/zh/docs/tasks/traffic-management/request-routing/

![image-20210922164004891](http://myapp.img.mykernel.cn/image-20210922164004891.png)

### 部署

https://istio.io/latest/zh/docs/examples/bookinfo/

```bash
kubectl label namespace default istio-injection-
kubectl label namespace default istio-injection=enabled --overwrite
```

应用

```bash
# cd /usr/local/istio
# kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

### 验证 东西流量

集群内部流量OK

```bash
kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```

### 引入南北

ingress流量

```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml           # gateway(listener), virtualservice(route)
kubectl apply -f  samples/bookinfo/networking/destination-rule-all.yaml      # Destination(cluster)
kubectl edit svc -n istio-system   istio-ingressgateway                      # NodePort 80->30080, 443->30443;
```

> ```yaml
> apiVersion: networking.istio.io/v1alpha3
> kind: Gateway
> metadata:
>   name: bookinfo-gateway
> spec:
>   selector:
>     istio: ingressgateway # use istio default controller
>   servers:
>   - port:
>       number: 80
>       name: http
>       protocol: HTTP
>     hosts:
>     - "*"
> ---
> apiVersion: networking.istio.io/v1alpha3
> kind: VirtualService
> metadata:
>   name: bookinfo
> spec:
>   hosts:
>   - "*"
>   gateways:
>   - bookinfo-gateway
>   http:
>   - match:
>     - uri:
>         exact: /productpage
>     - uri:
>         prefix: /static
>     - uri:
>         exact: /login
>     - uri:
>         exact: /logout
>     - uri:
>         prefix: /api/v1/products
>     route:
>     - destination:
>         host: productpage
>         port:
>           number: 9080
> 
> ```
>
> > `Gateway` 定义的listener
> >
> > `  selector:
> >     istio: ingressgateway` 这里就是选择的ingress pod.
> >
> > ```bash
> > root@ip-10-0-0-245:/usr/local/istio# kubectl get pod -n istio-system -l istio=ingressgateway
> > NAME                                    READY   STATUS    RESTARTS   AGE
> > istio-ingressgateway-76c6d6485c-xxj5f   1/1     Running   0          41m
> > ```
> >
> > ```yaml
> >   servers:
> >   - port:
> >       number: 80
> >       name: http
> >       protocol: HTTP
> >     hosts:
> >     - "*"
> > ```
> >
> > 表示这个pod会监听在80端口,  暴露的地址是*, 所以域名或ip
>
> > `VirtualService` 定义的流量分发策略， 路由
> >
> > ```yaml
> >   gateways:
> >   - bookinfo-gateway
> > ```
> >
> > 表示访问bookinfo-gateway这个listener
> >
> > ```yaml
> >   hosts:
> >   - "*"
> > ```
> >
> > 表示访问任何域名或Ip
>
> > ```yaml
> >     route:
> >     - destination:
> >         host: productpage # 路由到哪个集群
> >         port:
> >           number: 9080
> > ```
> >
> > 表示路由到哪个集群
>
> ```yaml
> apiVersion: networking.istio.io/v1alpha3
> kind: DestinationRule
> metadata:
>   name: productpage
> spec:
>   host: productpage
>   subsets:
>   - name: v1
>     labels:
>       version: v1
> ---
> apiVersion: networking.istio.io/v1alpha3
> kind: DestinationRule
> metadata:
>   name: reviews
> spec:
>   host: reviews
>   subsets:
>   - name: v1
>     labels:
>       version: v1
>   - name: v2
>     labels:
>       version: v2
>   - name: v3
>     labels:
>       version: v3
> ---
> apiVersion: networking.istio.io/v1alpha3
> kind: DestinationRule
> metadata:
>   name: ratings
> spec:
>   host: ratings
>   subsets:
>   - name: v1
>     labels:
>       version: v1
>   - name: v2
>     labels:
>       version: v2
>   - name: v2-mysql
>     labels:
>       version: v2-mysql
>   - name: v2-mysql-vm
>     labels:
>       version: v2-mysql-vm
> ---
> apiVersion: networking.istio.io/v1alpha3
> kind: DestinationRule
> metadata:
>   name: details
> spec:
>   host: details
>   subsets:
>   - name: v1
>     labels:
>       version: v1
>   - name: v2
>     labels:
>       version: v2
> ---
> ```
>
> > ```yaml
> > apiVersion: networking.istio.io/v1alpha3
> > kind: DestinationRule
> > metadata:
> >   name: productpage
> > spec:
> >   host: productpage
> >   subsets:
> >   - name: v1
> >     labels:
> >       version: v1
> > ```
> >
> > 表示路由到此集群为`name: productpage` 
>
> > `  host: productpage` 表示请求productpage service
>
> ```yaml
> apiVersion: networking.istio.io/v1alpha3
> kind: DestinationRule
> metadata:
>   name: reviews
> spec:
>   host: reviews
>   subsets:
>   - name: v1
>     labels:
>       version: v1
>   - name: v2
>     labels:
>       version: v2
>   - name: v3
>     labels:
>       version: v3
> ```
>
> > 请求` host: reviews`服务会到达3个版本
> >
> > ```bash
> > root@ip-10-0-0-245:/usr/local/istio# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl  reviews:9080/reviews/0  | awk '{if ($0 ~ "rating") {  if ($0 ~ "black"){print "v2"} else {print "v3"}} else {print "v1"}}'; sleep 0.2; done
> > v2
> > v1
> > v3
> > v2
> > v1
> > ```
>
> 如果现在定义reviews service只到1个版本
>
> ```yaml
> kubectl <<EOF apply -f -
> apiVersion: networking.istio.io/v1alpha3
> kind: DestinationRule
> metadata:
>   name: reviews
> spec:
>   host: reviews
>   subsets:
>   - name: v1
>     labels:
>       version: v1
> EOF
> ```
>
> 没有影响，上面请求的结果还是v1,v2,v3轮循
>
> 但是我们修改reviews集群的定义
>
> ```bash
> kubectl delete dr reviews
> ```
>
> ```diff
> kubectl <<EOF apply -f -
> apiVersion: networking.istio.io/v1alpha3
> kind: DestinationRule
> metadata:
> +  name: reviews1
> spec:
>   host: reviews
>   subsets:
>   - name: v1
>     labels:
> +      version: v2
> EOF
> ```
>
> ```yaml
> kubectl <<EOF apply -f -
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
> EOF
> ```
>
> > 没有gateway, 所以定义的是sidecar
>
> 现在请求, 发现还是通的，只不过全部到v2版本
>
> 现在脚本通了
>
> ```bash
> root@ip-10-0-0-245:/usr/local/istio# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl  reviews:9080/reviews/0  | awk '{if ($0 ~ "rating") {  if ($0 ~ "black"){print "v2"} else {print "v3"}} else {print "v1"}}'; sleep 0.2; done
> v2
> v2
> v2
> v2
> v2
> v2
> v2
> v2
> v2
> ```
>
> > 可以发现版本是交替的， 说明**vs用来控制service的访问逻辑**, **路由的目标是destination的host**
>
> 而服务还是3个主机
>
> ```bash
> root@ip-10-0-0-245:/usr/local/istio# kubectl describe svc reviews
> Name:              reviews
> Namespace:         default
> Labels:            app=reviews
>                    service=reviews
> Annotations:       <none>
> Selector:          app=reviews
> Type:              ClusterIP
> IP Families:       <none>
> IP:                10.107.212.14
> IPs:               <none>
> Port:              http  9080/TCP
> TargetPort:        9080/TCP
> Endpoints:         10.244.0.35:9080,10.244.0.36:9080,10.244.0.38:9080
> Session Affinity:  None
> Events:            <none>
> ```
>
> 恢复
>
> ```bash
> kubectl delete vs reviews    # 不控制service
> kubectl delete dr reviews1
> kubectl apply -f  samples/bookinfo/networking/destination-rule-all.yaml      # Destination(cluster) reviews集群恢复
> ```

访问网页: http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage

![image-20210924093619727](http://myapp.img.mykernel.cn/image-20210924093619727.png)

现在查看kiali



### 请求路由

https://istio.io/latest/docs/tasks/traffic-management/request-routing/#route-based-on-user-identity

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

> ```yaml
> apiVersion: networking.istio.io/v1alpha3
> kind: VirtualService
> metadata:
> name: reviews
> spec:
> hosts:
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



### 故障注入 （vs)

#### 依赖

https://istio.io/latest/docs/tasks/traffic-management/fault-injection/

```bash
$ kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml # 所有服务到v1
$ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml # review的指定用户到v2
```

```bash
root@ip-10-0-0-245:/usr/local/istio# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl  reviews:9080/reviews/0  | awk '{if ($0 ~ "rating") {  if ($0 ~ "black"){print "v2"} else {print "v3"}} else {print "v1"}}'; sleep 0.2; done
v1
v1
```

> 可以看出，只要是没有用户标识就是到v1

添加用户标头

```bash
root@ip-10-0-0-245:/usr/local/istio# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl --header end-user:jason  reviews:9080/reviews/0  | awk '{if ($0 ~ "rating") {  if ($0 ~ "black"){print "v2"} else {print "v3"}} else {print "v1"}}'; sleep 0.2; done
v2
v2
v2
```

> 可以看出到v1, 还是请求相同的服务

- `productpage` → `reviews:v2` → `ratings` (only for user `jason`)
- `productpage` → `reviews:v1` (for everyone else)



#### 注入delay故障

```bash
 kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

> ```yaml
> apiVersion: networking.istio.io/v1alpha3
> kind: VirtualService
> metadata:
>   name: ratings
> spec:
>   hosts:
>   - ratings
>   http:
>   - match:
>     - headers:
>         end-user:
>           exact: jason
>     fault:
>       delay:
>         percentage:
>           value: 100.0
>         fixedDelay: 7s
>     route:
>     - destination:
>         host: ratings
>         subset: v1
>   - route:
>     - destination:
>         host: ratings
>         subset: v1
> ```
>
> > 配置service
> >
> > 当访问`host: ratings`指定的ratings service服务时，带了用户标头，100%的请求，注入延迟故障，路由到ratings集群的v1子集。
> >
> > 当没有带用户标头时，就不会有延迟，到ratings集群的v1子集
> >
> > ```yaml
> > apiVersion: networking.istio.io/v1alpha3
> > kind: DestinationRule
> > metadata:
> >   name: ratings # VS引用的名
> > spec:
> >   host: ratings # 服务
> >   subsets:
> >   - name: v1
> >     labels: # 标签 
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
> > ```
> >
> > > ratings集群的v1子集就是匹配ratings服务的v1版本。

现在测试

- 没有延迟故障

  ```bash
  root@ip-10-0-0-245:/usr/local/istio# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl -w "\ntime_namelookup:  %{time_namelookup}s\ntime_connect:  %{time_connect}s\ntime_appconnect:  %{time_appconnect}s\ntime_pretransfer:  %{time_pretransfer}s\ntime_redirect:  %{time_redirect}s\ntime_starttransfer:  %{time_starttransfer}s\n----------\ntime_total:  %{time_total}s\n\n\n"  ratings:9080/ratings/0; done
  
  ```

- 有延迟故障

  ```bash
  while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl --header end-user:jason -w "\ntime_namelookup:  %{time_namelookup}s\ntime_connect:  %{time_connect}s\ntime_appconnect:  %{time_appconnect}s\ntime_pretransfer:  %{time_pretransfer}s\ntime_redirect:  %{time_redirect}s\ntime_starttransfer:  %{time_starttransfer}s\n----------\ntime_total:  %{time_total}s\n\n\n"  ratings:9080/ratings/0; done
  {"id":0,"ratings":{"Reviewer1":5,"Reviewer2":4}}
  time_namelookup:  0.004216s
  time_connect:  0.004324s
  time_appconnect:  0.000000s
  time_pretransfer:  0.004365s
  time_redirect:  0.000000s
  time_starttransfer:  7.006975s
  ----------
  time_total:  7.007015s
  
  {"id":0,"ratings":{"Reviewer1":5,"Reviewer2":4}}
  time_namelookup:  0.004231s
  time_connect:  0.004333s
  time_appconnect:  0.000000s
  time_pretransfer:  0.004374s
  time_redirect:  0.000000s
  time_starttransfer:  7.006731s
  ----------
  time_total:  7.006769s
  
  ```

  > 可以发现每一次请求一次耗时需要7s

如果在网页上加载，请求超时是6s, 这个接口是7s, 所以网页显示`Unavailable`

#### 注入abort故障

https://istio.io/latest/docs/tasks/traffic-management/fault-injection/#injecting-an-http-abort-fault

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml
```

> ```yaml
> apiVersion: networking.istio.io/v1alpha3
> kind: VirtualService
> metadata:
>   name: ratings
> spec:
>   hosts:
>   - ratings
>   http:
>   - match:
>     - headers:
>         end-user:
>           exact: jason
>     fault:
>       abort:
>         percentage:
>           value: 100.0
>         httpStatus: 500
>     route:
>     - destination:
>         host: ratings
>         subset: v1
>   - route:
>     - destination:
>         host: ratings
>         subset: v1
> ```
>
> > 注入ratings服务，jason用户请求时，100%的请求终止故障，直接响应500. 非这个用户，到v1版本

非用户请求，正常

```bash
root@ip-10-0-0-245:~# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl  ratings:9080/ratings/0; echo; done
{"id":0,"ratings":{"Reviewer1":5,"Reviewer2":4}}
{"id":0,"ratings":{"Reviewer1":5,"Reviewer2":4}}
{"id":0,"ratings":{"Reviewer1":5,"Reviewer2":4}}
{"id":0,"ratings":{"Reviewer1":5,"Reviewer2":4}}
{"id":0,"ratings":{"Reviewer1":5,"Reviewer2":4}}
{"id":0,"ratings":{"Reviewer1":5,"Reviewer2":4}}

```

用户请求,全部是500

```bash
root@ip-10-0-0-245:/usr/local/istio# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl --header end-user:jason -fsL -w "%{http_code}"  ratings:9080/ratings/0; done
500command terminated with exit code 22
500command terminated with exit code 22
500command terminated with exit code 22
```



### 流量迁移 （vs)

#### http流量迁移

https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/

```bash
 kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

> ```yaml
> apiVersion: networking.istio.io/v1alpha3
> kind: VirtualService
> metadata:
>   name: productpage
> spec:
>   hosts:
>   - productpage
>   http:
>   - route:
>     - destination:
>         host: productpage
>         subset: v1
> ---
> apiVersion: networking.istio.io/v1alpha3
> kind: VirtualService
> metadata:
>   name: reviews
> spec:
>   hosts:
>   - reviews
>   http:
>   - route:
>     - destination:
>         host: reviews
>         subset: v1
> ---
> apiVersion: networking.istio.io/v1alpha3
> kind: VirtualService
> metadata:
>   name: ratings
> spec:
>   hosts:
>   - ratings
>   http:
>   - route:
>     - destination:
>         host: ratings
>         subset: v1
> ---
> apiVersion: networking.istio.io/v1alpha3
> kind: VirtualService
> metadata:
>   name: details
> spec:
>   hosts:
>   - details
>   http:
>   - route:
>     - destination:
>         host: details
>         subset: v1
> ---
> ```
>
> > 负载均衡子集，匹配指定元数据，到不同的集群。
> >
> > 以上均是所有service到v1子集

请求reviews

```bash
root@ip-10-0-0-245:/usr/local/istio# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl  reviews:9080/reviews/0  | awk '{if ($0 ~ "rating") {  if ($0 ~ "black"){print "v2"} else {print "v3"}} else {print "v1"}}'; sleep 0.2; done
v1
v1

```

定义流量分割

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
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
> 用户请求reviews的service时，流量在v1,v3版本调度。
>
> 确保权重之和为100. 此处是route到不同端点有不同的权重。 envoy中的route目标，cluster/weighted_cluster.

现在请求reviews

```bash
root@ip-10-0-0-245:/usr/local/istio# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl  reviews:9080/reviews/0  | awk '{if ($0 ~ "rating") {  if ($0 ~ "black"){print "v2"} else {print "v3"}} else {print "v1"}}'; sleep 0.2; done
v3
v1
v1
v3
```

> 1半流量到达v3. 1半在v1

补丁版本发布：

- v1->替换成v3. 就将流量慢慢迁移到v3. 最后，仅保留一个子集，就是v3

新版本发布：

- 应该新加route, 匹配新版本，到新版本。

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
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
>   - route:
>     - destination:
>         host: reviews
>         subset: v3
> ```
>
> 迁移完成， 现在请求将只剩v3
>
> ```bash
> root@ip-10-0-0-245:/usr/local/istio# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl  reviews:9080/reviews/0  | awk '{if ($0 ~ "rating") {  if ($0 ~ "black"){print "v2"} else {print "v3"}} else {print "v1"}}'; sleep 0.2; done
> v3
> ```

#### tcp流量迁移

https://istio.io/latest/zh/docs/tasks/traffic-management/tcp-traffic-shifting/

逻辑与http一致，不示例了

### 超时 (vs)

https://istio.io/latest/zh/docs/tasks/traffic-management/request-timeouts/

在客户端，快速处理错误的机制

`reviews` 会请求 `ratings`

1. 给ratings注入2s超时延迟

   ```yaml
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: ratings
   spec:
     hosts:
     - ratings
     http:
     - fault:
         delay:
           percent: 100
           fixedDelay: 2s
       route:
       - destination:
           host: ratings
           subset: v1
   EOF
   ```

   命令测试

   ```bash
   root@ip-10-0-0-245:/usr/local/istio# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl  -w "\ntime_namelookup:  %{time_namelookup}s\ntime_connect:  %{time_connect}s\ntime_appconnect:  %{time_appconnect}s\ntime_pretransfer:  %{time_pretransfer}s\ntime_redirect:  %{time_redirect}s\ntime_starttransfer:  %{time_starttransfer}s\n----------\ntime_total:  %{time_total}s\n\n\n"   ratings:9080/ratings/0; echo; done
   {"id":0,"ratings":{"Reviewer1":5,"Reviewer2":4}}
   time_namelookup:  0.004221s
   time_connect:  0.004327s
   time_appconnect:  0.000000s
   time_pretransfer:  0.004368s
   time_redirect:  0.000000s
   time_starttransfer:  2.005848s
   ----------
   time_total:  2.005883s
   
   
   
   {"id":0,"ratings":{"Reviewer1":5,"Reviewer2":4}}
   time_namelookup:  0.004228s
   time_connect:  0.004337s
   time_appconnect:  0.000000s
   time_pretransfer:  0.004375s
   time_redirect:  0.000000s
   time_starttransfer:  2.009074s
   ----------
   time_total:  2.009110s
   
   ```

   > 可以发现每次请求reviews时，会有2s延迟

2. 给客户端默认配置

   ```yaml
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: reviews
   spec:
     hosts:
       - reviews
     http:
     - route:
       - destination:
           host: reviews
           subset: v2
   EOF
   ```

   > 表示请求reviews服务时，就会将所有流量搞到reviews v2, 这个服务会请求ratings.
   >
   > ```bash
   > root@ip-10-0-0-245:/usr/local/istio# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl  -w "\ntime_namelookup:  %{time_namelookup}s\ntime_connect:  %{time_connect}s\ntime_appconnect:  %{time_appconnect}s\ntime_pretransfer:  %{time_pretransfer}s\ntime_redirect:  %{time_redirect}s\ntime_starttransfer:  %{time_starttransfer}s\n----------\ntime_total:  %{time_total}s\n\n\n"   reviews:9080/reviews/0 ; sleep 0.2; done
   > {"id": "0","reviews": [{  "reviewer": "Reviewer1",  "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!", "rating": {"stars": 5, "color": "black"}},{  "reviewer": "Reviewer2",  "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare.", "rating": {"stars": 4, "color": "black"}}]}
   > time_namelookup:  0.004228s
   > time_connect:  0.004332s
   > time_appconnect:  0.000000s
   > time_pretransfer:  0.004371s
   > time_redirect:  0.000000s
   > time_starttransfer:  2.015552s
   > ----------
   > time_total:  2.015594s
   > 
   > 
   > {"id": "0","reviews": [{  "reviewer": "Reviewer1",  "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!", "rating": {"stars": 5, "color": "black"}},{  "reviewer": "Reviewer2",  "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare.", "rating": {"stars": 4, "color": "black"}}]}
   > time_namelookup:  0.004227s
   > time_connect:  0.004350s
   > time_appconnect:  0.000000s
   > time_pretransfer:  0.004387s
   > time_redirect:  0.000000s
   > time_starttransfer:  2.013520s
   > ----------
   > time_total:  2.013560s
   > 
   > ```
   >
   > > 可以发现每次请求reviews时，均会到v2版本，这个会返回black,  并且会请求评分

3. 给客户端`reviews`添加 半秒的超时

   ```yaml
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: reviews
   spec:
     hosts:
     - reviews
     http:
     - route:
       - destination:
           host: reviews
           subset: v2
       timeout: 0.5s
   EOF
   ```

   > ```bash
   > root@ip-10-0-0-245:/usr/local/istio# while true; do kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl  -w "\ntime_namelookup:  %{time_namelookup}s\ntime_connect:  %{time_connect}s\ntime_appconnect:  %{time_appconnect}s\ntime_pretransfer:  %{time_pretransfer}s\ntime_redirect:  %{time_redirect}s\ntime_starttransfer:  %{time_starttransfer}s\n----------\ntime_total:  %{time_total}s\nhttp_code: %{http_code}\n\n"   reviews:9080/reviews/0 ; sleep 0.2; done
   > upstream request timeout
   > time_namelookup:  0.004265s
   > time_connect:  0.004366s
   > time_appconnect:  0.000000s
   > time_pretransfer:  0.004404s
   > time_redirect:  0.000000s
   > time_starttransfer:  0.506039s
   > ----------
   > time_total:  0.506071s
   > http_code: 504
   > 
   > upstream request timeout
   > time_namelookup:  0.004250s
   > time_connect:  0.004477s
   > time_appconnect:  0.000000s
   > time_pretransfer:  0.004549s
   > time_redirect:  0.000000s
   > time_starttransfer:  0.507918s
   > ----------
   > time_total:  0.507954s
   > http_code: 504
   > ```
   >
   > 现在可以发现，reviews服务将不会等待ratings 超过0.5s
   >
   > 并且，返回504的网关超时



### 熔断 (dr)

https://istio.io/latest/zh/docs/tasks/traffic-management/circuit-breaking/

#### 依赖

其他部署，不能在bookinfo上实现

```bash
kubectl apply -f samples/httpbin/httpbin.yaml
```

#### 配置断路器

**集群配置**中为httpbin服务添加断路器

```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool: # istio: 连接池,  envoy: 熔断器 circuit_breakers; 潜在峰值, 超出 maxConnections+pending会快速503.
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection: # istio: 熔断,   envoy: 异常值检测； 对应故障的能力，如果后端响应5xx, 就会驱逐后端端点
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

> 异常值检测 `outlierDetection`
>
> - `consecutiveErrors` 连续错误1次开始驱逐
> - `interval` 1s检测一次
> - `baseEjectionTime` 每次驱逐3m
> - `maxEjectionPercent` 最大驱逐比率：100%

添加客户端

```bash
kubectl apply -f samples/httpbin/sample-client/fortio-deploy.yaml
```

请求测试

```bash
$ export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/get
```

验证熔断

```bash
root@ip-10-0-0-245:/usr/local/istio#  kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get
07:56:43 I logger.go:127> Log level is now 3 Warning (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 4->4 procs, for 20 calls: http://httpbin:8000/get
Starting at max qps with 2 thread(s) [gomax 4] for exactly 20 calls (10 per thread + 0)
07:56:43 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 41.00504ms : 20 calls. qps=487.74
Aggregated Function Time : count 20 avg 0.0040035718 +/- 0.0006972 min 0.002260217 max 0.005564946 sum 0.080071436
# range, mid point, percentile, count
>= 0.00226022 <= 0.003 , 0.00263011 , 5.00, 1
> 0.003 <= 0.004 , 0.0035 , 60.00, 11
> 0.004 <= 0.005 , 0.0045 , 90.00, 6
> 0.005 <= 0.00556495 , 0.00528247 , 100.00, 2
# target 50% 0.00381818
# target 75% 0.0045
# target 90% 0.005
# target 99% 0.00550845
# target 99.9% 0.0055593
Sockets used: 3 (for perfect keepalive, would be 2)
Jitter: false
Code 200 : 19 (95.0 %)
Code 503 : 1 (5.0 %)   # 多余的连接会503
```

```bash
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning http://httpbin:8000/get
```

```bash
$ kubectl exec "$FORTIO_POD" -c istio-proxy -- pilot-agent request GET stats | grep httpbin | grep pending
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.remaining_pending: 1
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.high.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_overflow: 21    # 超出的连接，会快速响应503
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_total: 29
cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_active: 1 # 驱逐
```



### 流量镜像 （vs)

https://istio.io/latest/zh/docs/tasks/traffic-management/mirroring/

#### 依赖

先清理之前的httpbin的dr

```bash
kubectl delete deploy httpbin # 清理之前的httpbin
kubectl delete dr httpbin # 熔断的配置
```

`httpbin-v1`

```bash
cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        ports:
        - containerPort: 80
EOF
```

`httpbin-v2`

```bash
cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v2
  template:
    metadata:
      labels:
        app: httpbin
        version: v2
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        ports:
        - containerPort: 80
EOF
```

`httpbin Kubernetes service`

```yaml
kubectl create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
EOF
```

验证服务正常启动

```bash
root@ip-10-0-0-245:~# kubectl get pod -o wide -l app=httpbin
NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE                NOMINATED NODE   READINESS GATES
httpbin-v1-5d4dcb7f64-xtp8d   2/2     Running   0          13m   10.244.0.41   master-10.0.0.245   <none>           <none>
httpbin-v2-74df788486-b9wk5   2/2     Running   0          13m   10.244.0.42   master-10.0.0.245   <none>           <none>
```

请求服务, 是正常的。

```bash
export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl http://httpbin:8000/get
```

查看日志

```bash
root@ip-10-0-0-245:~# kubectl logs --tail 20 -f httpbin-v1-5d4dcb7f64-xtp8d   -c httpbin
root@ip-10-0-0-245:~# kubectl logs --tail 20 -f httpbin-v2-74df788486-b9wk5 -c httpbin
# 再请求
export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl http://httpbin:8000/get
```

可以发现是在版本`v1, v2`之间轮循请求的

#### 配置流量镜像

- 让所有的流量到达httpbin v1

  ```yaml
  kubectl apply -f - <<EOF
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: httpbin
  spec:
    hosts:
      - httpbin
    http:
    - route:
      - destination:
          host: httpbin
          subset: v1
        weight: 100
  ---
  apiVersion: networking.istio.io/v1alpha3
  kind: DestinationRule
  metadata:
    name: httpbin
  spec:
    host: httpbin
    subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
  EOF
  ```

- 向服务发送一部分流量 

  ```yaml
  export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
  kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl http://httpbin:8000/get
  ```

  > 现在可以发现所有流量均在v1.

- 镜像部分流量到v2版

  ```yaml
  kubectl apply -f - <<EOF
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: httpbin
  spec:
    hosts:
      - httpbin
    http:
    - route:
      - destination:
          host: httpbin
          subset: v1
        weight: 100
      mirror:
        host: httpbin
        subset: v2
      mirrorPercent: 100
  EOF
  ```

- 查看日志

  ```yaml
  export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
  kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl http://httpbin:8000/get
  ```

  现在可以观察到，到达v1一份流量，v2必然有相同一份流量 

### ingress ( Gateway)

#### 依赖

先清理上一节的2个版本及vs, dr

```bash
kubectl delete vs httpbin
kubectl delete dr httpbin
kubectl delete deploy httpbin-v1
kubectl delete deploy httpbin-v2
```

应用httpbin服务

```bash
root@ip-10-0-0-245:~# cd /usr/local/istio
root@ip-10-0-0-245:/usr/local/istio# kubectl apply -f samples/httpbin/httpbin.yaml
```

> 验证
>
> ```bash
> root@ip-10-0-0-245:/usr/local/istio# kubectl get deploy httpbin
> NAME      READY   UP-TO-DATE   AVAILABLE   AGE
> httpbin   1/1     1            1           26s
> ```

#### istio Gateway来配置ingress

监控80端口的的http路由

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
EOF

```

配置ingress gateway的路由，路由到httpbin集群

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /headers
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

现在请求ingress gateway的nodeport， 访问headers就可以到达httpbin服务了

http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/headers

![image-20210926093041555](http://myapp.img.mykernel.cn/image-20210926093041555.png)

### Egress 

https://istio.io/latest/zh/docs/tasks/traffic-management/egress/egress-control/

1. 允许 Envoy 代理将请求传递到未在网格内配置过的服务。
2. 配置 [service entries](https://istio.io/latest/zh/docs/reference/config/networking/service-entry/) 以提供对外部服务的受控访问。
3. 对于特定范围的 IP，完全绕过 Envoy 代理。

