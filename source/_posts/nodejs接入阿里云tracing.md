---
title: nodejs接入阿里云tracing
date: 2021-08-26 17:22:22
tags:
- 生产小经验
- 分布式链路追踪
- 个人日记
---



# 前言
<!--more-->

由于过程非常麻烦，所以记录一下



jaeger 对接SLS： 已经过时，参考：https://help.aliyun.com/document_detail/68035.html

其上引用的github地址：https://github.com/aliyun/aliyun-log-jaeger/blob/master/README_CN.md?spm=a2c4g.11186623.0.0.534130a8tT57ar&file=README_CN.md



回到主页后，会跳转到新版本：https://github.com/aliyun/aliyun-log-jaeger/

此版本对应的sls日志，

project位置: https://sls.console.aliyun.com/lognext/profile

实例位置： https://sls.console.aliyun.com/lognext/trace



<!--more-->
# 搭建

```bash
git clone https://github.com/aliyun/aliyun-log-jaeger.git
```

## go代理

编辑 dockerfile

```diff
+ENV GOPROXY=https://goproxy.cn,direct
+ENV GO111MODULE=on
RUN go mod download && CGO_ENABLED=0 GOARCH=amd64 GOOS=linux go build -o /jaeger-sls-plugin
```

## 配置阿里云的配置

```yaml
[root@iZbp1fbhlx6573ely01az2Z aliyun-log-jaeger]# cat config.yaml
ACCESS_KEY_SECRET: ""
ACCESS_KEY_ID: ""
PROJECT: ""
ENDPOINT: ""
INSTANCE: ""
```

> 注意docker-compose.yaml 中也需要配置
>
> ```bash
> PROJECT: 即 https://sls.console.aliyun.com/lognext/profile 创建的project
> INSTANCE: 即 https://sls.console.aliyun.com/lognext/trace 创建的instance
> 当创建project之后，会提供内网or外网的地址。
> secret/key在个人信息会有提供
> ```

## docker-compose服务

由于nodejs写的是udp

```diff
  jaeger:
    build: .
    networks:
      - backend
    ports:
      - "6831:6831"
+      - "6832:6832/udp"
```



## 启动

```bash
docker-compose up --build -d
```



## kubernetes反代到这个宿主机服务

### service引用宿主机服务

```yaml
pos-prod@1227f652bef0:~/pos/k8s/dep/aliyun-log-jaeger$ cat jaeger-agent-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  labels:
    app: jaeger-agent
    env: test
  name: jaeger-agent-svc
  namespace: default
spec:
  ports:
  - port: 6832
    protocol: UDP
    name: app1
  - port: 5778
    name: app2
  - port: 16686
    name: query
---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    app: jaeger-agent
    env: test
  name: jaeger-agent-svc
  namespace: default
subsets:
- addresses:
   - ip: 192.168.7.160
  ports:
  - port: 6832
    protocol: UDP
    name: app1
  - port: 5778
    name: app2
  - port: 16686
    name: query
```

## ingress basic认证反代到这个service

需要添加认证

```bash
pos-prod@1227f652bef0:~/pos/k8s/dep/aliyun-log-jaeger$ cat ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required - for you company"
    nginx.ingress.kubernetes.io/auth-secret: jaeger-query-basic-auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/force-ssl-redirect: 'true'
  name: jager-query-ingress
spec:
  rules:
    - host: a.google.cn
      http:
        paths:
          - backend:
              serviceName: jaeger-agent-svc
              servicePort: 16686
            path: /
  tls:
    - hosts:
        - a.google.cn
      secretName: google.cn
```

添加basic认证的secret

```bash
# 首次 需要添加-c, 之后不需要
 htpasswd -cb  auth user1 password1

# 生成secret
kubectl create secret  generic jaeger-query-basic-auth --from-file=auth
```



# 查看

## tracing服务

https://sls.console.aliyun.com/lognext/trace

## query服务

http://a.google.cn









