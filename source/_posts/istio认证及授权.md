---
title: istio认证及授权
date: 2021-09-27 17:08:38
tags:
- istio
---

# 概览

服务间通信流量，是否能被恶意用户嗅探？

东西向流量，服务端接收客户端的访问时，确信来源方是否是受信任用户？

终端用户，从网格外接入网格，是否需要认证？



**零信任网络**：client访问server, 不能认可client是合理合法的client. 常用安全手段: `MTLS`,` RBAC`, `ABAC`, 证书轮替，`IAAA`

- 服务惟一受信标识
- 用户来源认证 authentication, `mTLS`, `JWT`
- 授权 authorization, 不同用户对资源不同权限
- accountability: 审计，避免用户操作滥用权限，对用户行为记录并审计。



# 常见实现

- 网络级别控制
  - 通信限制在本机
  - 网络分区，分区内的客户端受信
  - MTLS：双向认证TLS， 很好的spiffe/spire框架
- 应用级别控制
  - 传统的web信令牌
  - api 导向的token
    - api key
    - oatuh 2.0
    - openid
    - jwt
  - token类型
    - by-reference

服务网络中4种可用认证

- jwt
- ingress tls passthrough + jwt
- mtls + jwt



认证机制

- 传输认证：mtls单向或双向，各服务的证书签发，证书管理非常重要。
- 用户认证：envoy通过jwt实现

# envoy中实现tls传输安全

侦听器面向下游实现tls终止通信(listener)，向面上游作为TLS始发端(cluster定义)。

面向client, 支持多个证书。面向上游只支持1个证书。

![image-20210929133056854](http://myapp.img.mykernel.cn/image-20210929133056854.png)

为envoy提供证书

- 静态路径配置，更新证书的话，需要热重启envoy。频繁的证书更新，频繁的envoy重启，对服务网格性能影响大。 对应K8S中的secret volume
- `SDS API`统一推送.                                                                                                                                                                  对应k8s的node-agent + citedal



istio的citadel就是spire一种强大实现



# istio的安全功能

身份标识、策略(policy)、TLS加密、AAA(认证、授权、审计)工具保护网格中的服务和数据

- citadel 管理私钥和证书
- sidecar 和perimeter proxies 服务间的安全通信
- pilot向代理分发认证策略和名称信息
- Mixer管理授权和审计

## 身份标识

标识一个pod身份，spiffe协议为通信参与者分发一个身份标识。 

istio中使用serviceaccount结合spiffe机制完成标识。

标识一个pod:

`spiffe://<domain>/ns/<namespace>/sa/<serviceaccount>`



## pki证书供给机制

### 卷挂载机制

citadel生成的证书，会生成secret资源，会关联到特定的serviceaccount上，此时只需要拥有此SA的pod，关联了此secret, 就有了证书文件。

因此envoy就在listener或cluster就是静态文件提供了证书。

应用于kubernetes场景。

kubernetes场景

- citadel watch api server. 为新增的SA创建spiffe证书，将存为Secret.
- 创建pod时，k8s基于其SA通过kubernetes secret volume将证书和私钥注入到pod中。
- citadel 持续监视每一个pod的证书有效期，一旦到期，更新k8s资源对象secret, 实现证书轮替。
- pilot生成安全命名规范信息，该信息定义了哪些SA可以运行哪个service. 而后由Pilot将安全名称传递给sidecar使用。

缺点：secret在本地，需要重启envoy.

<div class="admonition-title warning">
    <p>
        istio现在使用的Citadel是alpha版本，使用时需要小心 
    </p>
</div>

`default` 配置部署 istioctl

### 基于`SDS`机制

kubernetes节点之上，运行node agent代理 守护进程，理解为一个CA代理，当前节点任何一个服务sidecar需要证书和私钥时，node agent负责为其生成私钥和证书签署请求，nodeagent把证书签署请求`csr`(certificat sign request), 发给 citadel, citadel将签署后的证书发给agent. agent就把证书和私钥和CA证书提供给sidecar.

应用于非kubernetes场景，kubernetes场景一样可以使用node agent。

SDS **nodeagent**

1. citadel创建grpc服务接收CSR请求
2. envoy通过SDS api向本地的node agent发证书和密钥请求
3. node agent生成密钥和CSR, 并把CSR发给citadel
4. citadel验证CSR中携带的凭据并签署CSR生成证书
5. 从citadel发送证书，agent将其通过sds api发给envoy
6. 定期以上述步骤进行证书和密钥轮替。

优势：密钥和证书是内存中，不需要k8s secret, 支持密钥和证书动态更新无需重启envoy.

`sds`  配置部署 istioctl



# istio认证

服务到服务的认证：mutual tls实现

始发认证：终端用户认证：JWT

mtls认证支持：permissive和strict：前一种和客户端协商使用tls/mtls. 后一种严格意义上的mtls. 还有一个none, 就是不使用tls.

pilot持续监控api server, 为每个sa生成VID。



## istio认证架构

- 配置内部服务启用认证机制，身份认证策略, api群组的 "authentication.istio.io"
- 配置好的策略在api server中
- pilot持续监控apiserver, 策略变动，就立即转换为envoy的配置并应用于服务网格中。
  - 为每个listener和cluster，添加证书和私钥。



## 配置istio认证策略

Policy资源

- peers 通信双方
- origin 始发认证/终端用户配置
  - principal
- 默认MTLS认证模式为strict.
- 哪些pod设定，targets. 不定义时，所有名称空间所有pod生效。



## 配置认证机制

kind:

- meshpolicy 生效范围是整个网格级别的 **server**端认证策略，默认策略时，名称必须是default
- policy  名称空间级别**server**的策略， name为default, 就是默认策略。不是default，指定targets 当前名称空间中指定pod的指定端口
- destinationRule   CRD来定义**客户端**，配置tls.



要使用MTLS

- policy/meshpolicy定义服务端, 的ingress listener
- destinationrul配置客户端，的egress的cluster

![image-20210929142010559](http://myapp.img.mykernel.cn/image-20210929142010559.png)

istio的alpha阶段的自动双向认证功能，需要istio中此功能处于启用状态，此时仅需要通过policy配置server端

## 尝试mtls功能

### 启动MTLS

https://istio.io/v1.4/docs/tasks/security/authentication/auto-mtls/

citadel服务正常

```bash
root@ip-10-0-0-245:~/servicemesh_in_practise-master/security/envoy-spire-ext-authz# kubectl get pod -n istio-system -l istio=citadel
NAME                            READY   STATUS    RESTARTS   AGE
istio-citadel-f74b87856-zml9p   1/1     Running   0          47h

```

`demo`配置自动双向认证

```bash
istioctl manifest apply --set profile=demo \
  --set values.global.mtls.auto=true \
  --set values.global.mtls.enabled=false                 
```

`default` 基础之上添加此参数

```bash
istioctl manifest apply --set values.global.proxy.accessLogFile=""  --set profile=default --set values.kiali.enabled=true --set values.grafana.enabled=true --set values.tracing.enabled=true --set "values.kiali.dashboard.jaegerURL=http://jaeger-query:16686" --set "values.kiali.dashboard.grafanaURL=http://grafana:3000"  --set values.global.mtls.auto=true   --set values.global.mtls.enabled=false       
```



验证证书和私钥注入

```bash
root@ip-10-0-0-245:~/servicemesh_in_practise-master/security/envoy-spire-ext-authz# kubectl get secrets | grep key-and-cert
istio.bookinfo-details             istio.io/key-and-cert                 3      47h
istio.bookinfo-productpage         istio.io/key-and-cert                 3      47h
istio.bookinfo-ratings             istio.io/key-and-cert                 3      47h
istio.bookinfo-ratings-v2          istio.io/key-and-cert                 3      47h
istio.bookinfo-reviews             istio.io/key-and-cert                 3      47h
istio.default                      istio.io/key-and-cert                 3      2d

```

查看注入的证书

```bash
root@ip-10-0-0-245:~/servicemesh_in_practise-master/security/envoy-spire-ext-authz# POD=$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')
root@ip-10-0-0-245:~/servicemesh_in_practise-master/security/envoy-spire-ext-authz# kubectl exec -it $POD -c istio-proxy sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
$ ls /etc/certs
cert-chain.pem  key.pem  root-cert.pem

# curl localhost:15000/certs

$ openssl x509 -in /etc/certs/cert-chain.pem -text -noout

# 过滤Validity
        Validity
            Not Before: Sep 27 06:45:55 2021 GMT
            Not After : Dec 26 06:45:55 2021 GMT

# MTLS功能准备
            X509v3 Subject Alternative Name: critical
                URI:spiffe://cluster.local/ns/default/sa/bookinfo-productpage
```

### 检验prodctpage到reviews服务，是mtls

- productpage 客户端访问  reviews 服务

```bash
root@ip-10-0-0-245:~/servicemesh_in_practise-master/security/envoy-spire-ext-authz# istioctl authn tls-check $POD reviews.default.svc.cluster.local
HOST:PORT                                  STATUS     SERVER         CLIENT           AUTHN POLICY     DESTINATION RULE
reviews.default.svc.cluster.local:9080     OK         PERMISSIVE     ISTIO_MUTUAL     /default         default/reviews
```

> 双方 status: ok自动协商 
>
> server: permissive
>
> client: istio_mutual 双向认证
>
> auth policy: /default, 左侧名称空间没有写，就是网格的默认测试。
>
> DESTINATION RULE： reviews服务的策略，应用default名称空间的reviews策略

```bash
root@ip-10-0-0-245:~/servicemesh_in_practise-master/security/envoy-spire-ext-authz# kubectl get dr reviews -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
spec:
  host: reviews
  subsets:
  - labels:
      version: v1
    name: v1
  - labels:
      version: v2
    name: v2
  - labels:
      version: v3
    name: v3
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL # 服务是双向TLS。 即要求到达reviews服务的客户端需要MTLS。
```

> 此是设置客户端
>
> `samples/bookinfo/networking/destination-rule-all-mtls.yaml`

由于上面的默认网格是permissive.  服务端的PERMISSIVE来自默认设定，DestinationRule MUTUAL.



现在让服务端强制使用TLS, 需要定义policy

```bash
kubectl apply -f - << EOF
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  name: reviews-mtls
  namespace: default
spec:
  targets: # 应用到哪个服务
  - name: reviews
    port: # 不指定port表示所有port
    - number: 9080 
  peers:
    # 空/ {}, '' 都表示RESTRICT， 强制TLS
  - mtls: {}
EOF
```

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# kubectl get policies.authentication.istio.io 
NAME           AGE
reviews-mtls   29s
```

测试

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# istioctl authn tls-check $POD reviews.default.svc.cluster.local
HOST:PORT                                  STATUS     SERVER     CLIENT           AUTHN POLICY             DESTINATION RULE
reviews.default.svc.cluster.local:9080     OK         STRICT     ISTIO_MUTUAL     default/reviews-mtls     default/reviews
```

> 当前reviews是强制TLS时，则后端,服务检验TLS，就是严格了

### 验证 单个子集限制不通过MTLS通信 其他子集基于MTLS通信，失败

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# kubectl exec -it $POD -c istio-proxy -- bash
istio-proxy@productpage-v1-596598f447-l77sh:/$ curl  http://reviews.default.svc.cluster.local:9080/reviews/1
curl: (56) Recv failure: Connection reset by peer
istio-proxy@productpage-v1-596598f447-l77sh:/$ curl -k  https://reviews.default.svc.cluster.local:9080/reviews/1
curl: (16) SSL_write() returned SYSCALL, errno = 104

istio-proxy@productpage-v1-596598f447-l77sh:/$ curl --cacert /etc/certs/root-cert.pem --cert /etc/certs/cert-chain.pem --key /etc/certs/key.pem -k  https://reviews.default.svc.cluster.local:9080/reviews/1
{"id": "1","reviews": [{  "reviewer": "Reviewer1",  "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!", "rating": {"stars": 5, "color": "red"}},{  "reviewer": "Reviewer2",  "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare.", "rating": {"stars": 4, "color": "red"}}]}istio-proxy@productpage-v1-596598f447-l77sh:/$ 
```

> 客户端不提供证书时，就通信不了。
>
> 提供证书后，可以访问

浏览器访问 http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage 可以发现productpage均正常访问

> 恢复3个子集
>
> ```bash
> kubectl delete vs reviews
> kubectl apply -f /usr/local/istio-1.4.0/samples/bookinfo/networking/destination-rule-all.yaml
> ```





reviews的9080端口，必须严格MTLS，dr中设定v1客户端是严格

```bash
kubectl delete dr reviews

kubectl << EOF apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      tls:
        mode: DISABLE
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      tls:
        mode: ISTIO_MUTUAL
  - name: v3
    labels:
      version: v3
    trafficPolicy:
      tls:
        mode: ISTIO_MUTUAL
EOF
```

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# istioctl x des pod $POD
Pod: productpage-v1-596598f447-l77sh
   Pod Ports: 9080 (productpage), 15090 (istio-proxy)
--------------------
Service: productpage
   Port: http 9080/HTTP targets pod port 9080
DestinationRule: productpage for "productpage"
   Matching subsets: v1
   No Traffic Policy
Pod is PERMISSIVE, clients configured automatically
VirtualService: productpage
   1 HTTP route(s)


Exposed on Ingress Gateway http://10.0.0.245
VirtualService: bookinfo
   /productpage, /static*, /login, /logout, /api/v1/products*



root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking#  istioctl authn tls-check $POD reviews.default.svc.cluster.local
HOST:PORT                                  STATUS     SERVER     CLIENT     AUTHN POLICY             DESTINATION RULE
reviews.default.svc.cluster.local:9080     AUTO       STRICT     -          default/reviews-mtls     default/reviews

```

> client配置就是 reviews的DR配置，会配置到所有pod的cluster中，，没有统一的限定。任何客户端访问reviews，可有v1的不限制MTLS, V2/V3限制了，所以上面显示-，不统一。
>
> 而访问server, 严格要求。即policy的配置会变成reviews的SIDECAR的ingress的tls配置。
>
> 所以定义了mtls的v2/v3是正常的。否则通过不正常。



```bash
kubectl delete dr reviews
kubectl apply -f destination-rule-all.yaml 
```

查看

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# istioctl authn tls-check $POD reviews.default.svc.cluster.local
HOST:PORT                                  STATUS     SERVER     CLIENT     AUTHN POLICY             DESTINATION RULE
reviews.default.svc.cluster.local:9080     AUTO       STRICT     -          default/reviews-mtls     default/reviews

```

> AUTO， 有证书，就正常通信



客户端完全禁用

```bash
kubectl << EOF apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
    tls:
      mode: DISABLE
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
EOF
```

现在网页访问就不正常了

恢复

```bash
kubectl apply -f destination-rule-all.yaml 
```



### 让所有服务强制ISTIO_MUTUAL

https://istio.io/v1.4/docs/tasks/security/authentication/authn-policy/

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# kubectl delete policy reviews-mtls
policy.authentication.istio.io "reviews-mtls" deleted
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# istioctl authn tls-check $POD reviews.default.svc.cluster.local
HOST:PORT                                  STATUS     SERVER         CLIENT     AUTHN POLICY     DESTINATION RULE
reviews.default.svc.cluster.local:9080     AUTO       PERMISSIVE     -          /default         default/reviews

```

> PERMISSIVE 都正常了

让名称空间级别的默认策略

```bash
kubectl apply -f - <<EOF
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "default"
  namespace: "default"
spec:
  peers:
  - mtls: {}
EOF
```

验证

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# istioctl authn tls-check $POD reviews.default.svc.cluster.local
HOST:PORT                                  STATUS     SERVER     CLIENT     AUTHN POLICY        DESTINATION RULE
reviews.default.svc.cluster.local:9080     AUTO       STRICT     -          default/default     default/reviews
```

> 当前名称空间的应用，被访问时，强制`TLS`
>
> 客户端访问此应用时，是否应用MUTUAL，AUTO。

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# istioctl authn tls-check $POD  | grep .default.svc
details.default.svc.cluster.local:9080                        AUTO       STRICT         -                default/default                              default/details
details.default.svc.cluster.local|v1:9080                     AUTO       STRICT         -                default/default                              default/details
details.default.svc.cluster.local|v2:9080                     AUTO       STRICT         -                default/default                              default/details
fortio.default.svc.cluster.local:8080                         AUTO       STRICT         -                default/default                              -
kubernetes.default.svc.cluster.local:443                      AUTO       STRICT         -                default/default                              -
mongodb.default.svc.cluster.local:27017                       AUTO       STRICT         -                default/default                              -
productpage.default.svc.cluster.local:9080                    AUTO       STRICT         -                default/default                              default/productpage
productpage.default.svc.cluster.local|v1:9080                 AUTO       STRICT         -                default/default                              default/productpage
ratings.default.svc.cluster.local:9080                        AUTO       STRICT         -                default/default                              default/ratings
ratings.default.svc.cluster.local|v1:9080                     AUTO       STRICT         -                default/default                              default/ratings
ratings.default.svc.cluster.local|v2:9080                     AUTO       STRICT         -                default/default                              default/ratings
ratings.default.svc.cluster.local|v2-mysql:9080               AUTO       STRICT         -                default/default                              default/ratings
ratings.default.svc.cluster.local|v2-mysql-vm:9080            AUTO       STRICT         -                default/default                              default/ratings
reviews.default.svc.cluster.local:9080                        AUTO       STRICT         -                default/default                              default/reviews
reviews.default.svc.cluster.local|v1:9080                     AUTO       STRICT         -                default/default                              default/reviews
reviews.default.svc.cluster.local|v2:9080                     AUTO       STRICT         -                default/default                              default/reviews
reviews.default.svc.cluster.local|v3:9080                     AUTO       STRICT         -                default/default                              default/reviews
```

> `productpage`到达 所有此名称空间的服务集群，均走的STRICT, 说明服务端一定是`TLS`. 而客户端是否使用`TLS`， AUTO表示看客户端是否有证书。
>
> 其他名称空间，不受default名称空间限制，所以不存在限制

恢复

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# kubectl delete policies.authentication.istio.io default
```

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# istioctl authn tls-check $POD reviews.default.svc.cluster.local
HOST:PORT                                  STATUS     SERVER         CLIENT     AUTHN POLICY     DESTINATION RULE
reviews.default.svc.cluster.local:9080     AUTO       PERMISSIVE     -          /default         default/reviews
```



### 让整个bookinfo都走mtls

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# kubectl apply -f destination-rule-all-mtls.yaml 
destinationrule.networking.istio.io/productpage configured
destinationrule.networking.istio.io/reviews configured
destinationrule.networking.istio.io/ratings configured
destinationrule.networking.istio.io/details configured
```

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# istioctl authn tls-check $POD reviews.default.svc.cluster.local
HOST:PORT                                  STATUS     SERVER         CLIENT           AUTHN POLICY     DESTINATION RULE
reviews.default.svc.cluster.local:9080     OK         PERMISSIVE     ISTIO_MUTUAL     /default         default/reviews
```

> 显示的结果，表示对于`reviews.default.svc.cluster.local:9080 ` 这个api的mtls机制
>
> status ok 配置ok. auto表示客户端是否使用tls由客户端有没有证书决定，有就使用，没有就不使用。
>
> server PERMISSIVE 表示服务端可以使用tls, 可以不使用tls.
>
> CLIENT: 表示reviews的DR配置中是`ISTIO_MUTUAL`(双向TLS)，这个DR配置会被Pilot转换成其他sidecar的cluster, 所以sidecar访问reviews时，就是客户端，将会使用此DR的配置。
>
> AUTHN POLICY： server配置策略继承自`/default`  这个格式表示`名称空间/策略名`, 不给名称空间表示网格提供的默认策略。
>
> DESTINATION RULE：表示DR的策略配置来自于哪，这个配置会导致`CLIENT`字段显示不同的值。

# istio授权机制

## 优势

为istio 工作负载提供mesh-level, ns-level和workload-level访问控制。

- 单一api, authorizationpolicy CRD就定义了授权
- 灵活主义，运维人员可以istio属性的基础上自定义检查条件 
- 授权检查仅在envoy本地执行，性能好

## 授权架构

授权功能不需要显式启用，不像认证功能还需要显式启用。

只需要把authorizationPolicy应用到k8s即可，存在policy应用于workload时，会立即生效, 默认策略是拒绝。因为目前仅支持ALLOW操作。当多个authorizationPolicy应用相同workload时，取并集。

如果workload没有应用authorizationPolicy, 则该workload不受任何访问控制限制。

![image-20210929160441731](http://myapp.img.mykernel.cn/image-20210929160441731.png)

## 定义格式

- 不写名称空间时，表示应用所有名称空间
- `WorkloadSelector` 限制策略应用的范围，pod标签选择目标workload
- `Rule` 授权规则
  - from 来源
  - to 动作 到当前服务的...允许
    - 哪些服务、哪些端口、哪些URL上、哪些方法的操作
  - when 何时, 大多数HTTP, 一部分HTTP and TCP
    - 条件
    - source.ip
    - source.namespace
    - source.principal, JWT或证书的subject

![image-20210929161349422](http://myapp.img.mykernel.cn/image-20210929161349422.png)

## 示例

了解当前系统之上的授权策略定义

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# istioctl experimental authz check $POD
Checked 18/37 listeners with node IP 10.244.0.19.
LISTENER[FilterChain]     CERTIFICATE                   mTLS (MODE)          JWT (ISSUERS)     AuthZ (RULES)
0.0.0.0_80[0]             none                          no (none)            no (none)         no (none)
0.0.0.0_80[1]             none                          no (none)            no (none)         no (none)
0.0.0.0_3000[0]           none                          no (none)            no (none)         no (none)
0.0.0.0_3000[1]           none                          no (none)            no (none)         no (none)
0.0.0.0_8060[0]           none                          no (none)            no (none)         no (none)
0.0.0.0_8060[1]           none                          no (none)            no (none)         no (none)
0.0.0.0_8080[0]           none                          no (none)            no (none)         no (none)
0.0.0.0_8080[1]           none                          no (none)            no (none)         no (none)
0.0.0.0_9080[0]           none                          no (none)            no (none)         no (none)
0.0.0.0_9080[1]           none                          no (none)            no (none)         no (none)
0.0.0.0_9090[0]           none                          no (none)            no (none)         no (none)
0.0.0.0_9090[1]           none                          no (none)            no (none)         no (none)
0.0.0.0_9091[0]           none                          no (none)            no (none)         no (none)
0.0.0.0_9091[1]           none                          no (none)            no (none)         no (none)
0.0.0.0_9411[0]           none                          no (none)            no (none)         no (none)
0.0.0.0_9411[1]           none                          no (none)            no (none)         no (none)
0.0.0.0_9901[0]           none                          no (none)            no (none)         no (none)
0.0.0.0_9901[1]           none                          no (none)            no (none)         no (none)
virtualOutbound[0]        none                          no (none)            no (none)         no (none)
virtualOutbound[1]        none                          no (none)            no (none)         no (none)
0.0.0.0_15004[0]          none                          no (none)            no (none)         no (none)
0.0.0.0_15004[1]          none                          no (none)            no (none)         no (none)
virtualInbound[0]         none                          no (none)            no (none)         no (none)
virtualInbound[1]         none                          no (none)            no (none)         no (none)
virtualInbound[2]         /etc/certs/cert-chain.pem     yes (PERMISSIVE)     no (none)         no (none)
virtualInbound[3]         none                          no (PERMISSIVE)      no (none)         no (none)
0.0.0.0_15010[0]          none                          no (none)            no (none)         no (none)
0.0.0.0_15010[1]          none                          no (none)            no (none)         no (none)
0.0.0.0_15014[0]          none                          no (none)            no (none)         no (none)
0.0.0.0_15014[1]          none                          no (none)            no (none)         no (none)
0.0.0.0_15019[0]          none                          no (none)            no (none)         no (none)
0.0.0.0_15019[1]          none                          no (none)            no (none)         no (none)
0.0.0.0_20001[0]          none                          no (none)            no (none)         no (none)
0.0.0.0_20001[1]          none                          no (none)            no (none)         no (none)
10.244.0.19_9080[0]       /etc/certs/cert-chain.pem     yes (PERMISSIVE)     no (none)         no (none)
10.244.0.19_9080[1]       none                          no (PERMISSIVE)      no (none)         no (none)
10.244.0.19_15020         none                          no (none)            no (none)         no (none)
```

> - LISTENER[FilterChain]
> - CERTIFICATE       
> - mTLS (MODE)
> - JWT (ISSUERS)
> - AuthZ (RULES) 没有任何授权策略的定义

### HTTP流量授权

https://istio.io/v1.4/docs/tasks/security/authorization/authz-http/

当前名称空间，所有策略拒绝

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  {}
EOF
```

现在访问    http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage 返回

`RBAC: access denied`

允许读取productpage pod

```bash
kubectl apply -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "productpage-viewer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  rules:
  - to:
    - operation:
        methods: ["GET"]
EOF
```

> 访问productpage正常，但是productpage访问review不正常
>
> ![image-20210929163017749](http://myapp.img.mykernel.cn/image-20210929163017749.png)

允许productpage 访问 details

```bash
kubectl apply -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "details-viewer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: details
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
EOF
```

> 访问时，detail可以，但是reviews时，无法显示
>
> ![image-20210929163059219](http://myapp.img.mykernel.cn/image-20210929163059219.png)

开放reviews给prodectpage

```bash
kubectl apply -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "reviews-viewer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
EOF
```

reviews不能访问ratings

```bash
kubectl apply -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "ratings-viewer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: ratings
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-reviews"]
    to:
    - operation:
        methods: ["GET"]
EOF
```

> 稍后片刻，所有服务正常了



现在测试在 productpage 上访问ratings, 就不可以

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# kubectl exec -it $POD -c istio-proxy -- bash
istio-proxy@productpage-v1-596598f447-l77sh:/$ curl -s http://ratings.default.svc.cluster.local:9080/ratings
RBAC: access denied
```

清理

```bash
kubectl delete authorizationpolicy.security.istio.io/deny-all
kubectl delete authorizationpolicy.security.istio.io/productpage-viewer
kubectl delete authorizationpolicy.security.istio.io/details-viewer
kubectl delete authorizationpolicy.security.istio.io/reviews-viewer
kubectl delete authorizationpolicy.security.istio.io/ratings-viewer
```

现在两次请求

```bash
root@ip-10-0-0-245:/usr/local/istio/samples/bookinfo/networking# kubectl exec -it $POD -c istio-proxy -- bash
istio-proxy@productpage-v1-596598f447-l77sh:/$ curl -s http://ratings.default.svc.cluster.local:9080/ratings/0
{"id":0,"ratings":{"Reviewer1":5,"Reviewer2":4}}istio-proxy@productpage-v1-596598f447-l77sh:/$ exit
```

现在就正常了

### TCP 流量授权

如果部署了mongodb的pod, 不是HTTP协议，所以就使用TCP授权
https://istio.io/v1.4/docs/tasks/security/authorization/authz-tcp/

部署v2版本的ratings

```bash
cd /usr/local/istio

kubectl apply -f samples/bookinfo/platform/kube/bookinfo-ratings-v2.yaml # v2 ratings
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml # ratings调用可以到达 v2
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-db.yaml # ratings的ingress的仅路由到mongodb的v2
kubectl apply -f samples/bookinfo/platform/kube/bookinfo-db.yaml # 安装mongo

```

流量器访问，仅到v2 http://ec2-52-83-161-189.cn-northwest-1.compute.amazonaws.com.cn:30080/productpage



让mongodb的pod拒绝所有

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
spec:
  selector:
    matchLabels:
      app: mongodb
EOF
```

现在访问时`Reviewer1 *Ratings service is currently unavailable*`

让mongodb仅允许v2访问

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: bookinfo-ratings-v2
spec:
  selector:
    matchLabels:
      app: mongodb
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-ratings-v2"]
    to:
    - operation:
        ports: ["27017"] # 访问当前的27017端口
EOF
```

现在访问正常了

> 类似于`iptables`的限制

清理

```bash
kubectl delete authorizationpolicy.security.istio.io/deny-all
kubectl delete authorizationpolicy.security.istio.io/bookinfo-ratings-v2
```

```bash
kubectl delete -f samples/bookinfo/platform/kube/bookinfo-ratings-v2.yaml
kubectl delete -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
kubectl delete -f samples/bookinfo/networking/virtual-service-ratings-db.yaml
kubectl delete -f samples/bookinfo/platform/kube/bookinfo-db.yaml
```

查看

```bash
root@ip-10-0-0-245:/usr/local/istio# istioctl x des pod $POD
Pod: productpage-v1-596598f447-l77sh
   Pod Ports: 9080 (productpage), 15090 (istio-proxy)
--------------------
Service: productpage
   Port: http 9080/HTTP targets pod port 9080
Pod is PERMISSIVE, clients configured automatically
VirtualService: productpage
   WARNING: No destinations match pod subsets (checked 1 HTTP routes)
      Warning: Route to subset v1 but NO DESTINATION RULE defining subsets!


Exposed on Ingress Gateway http://10.0.0.245
VirtualService: bookinfo
   /productpage, /static*, /login, /logout, /api/v1/products*

```

```bash
root@ip-10-0-0-245:/usr/local/istio# istioctl x des svc reviews
Service: reviews
   Port: http 9080/HTTP targets pod port 9080
Pod is PERMISSIVE, clients configured automatically
```



# 总结

<div class="admonition-title note">
    <p>
        网络内能使用的：双向TLS, 授权。认证这个功能不会使用
    </p>
    <p>
        <ul>
            <li>TLS就是在客户端配置证书，服务端配置证书。</li>
		<li>授权：就是类似于iptables, 哪个源可以访问目标(自己)什么端口什么主机什么url.</li>
    </ul>
    </p>
</div>
