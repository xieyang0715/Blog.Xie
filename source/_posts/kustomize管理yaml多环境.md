---
title: kustomize管理yaml多环境
date: 2021-04-26 11:06:27
tags:
- kubernetes
- 个人日记
---

# 如果一个yaml发版多个环境
<!--more-->

公司发版一个app有3种环境，`test`, `staging`, `prod`

常用解决方法：https://stackoverflow.com/questions/47655363/adjusting-kubernetes-configurations-depending-on-environment

## 3套yaml

 每套一个独立的yaml, 难管理。

## 环境变量修改

### kubectl set env

```bash
kubectl set env 
```

### envsubst

 Ubuntu / Debian `gettext`软件包

一个更容易/更清洁的解决方案： `envsubst`

在deploy.yml中：

```
LoadbalancerIP: $LBIP
```

然后只需创建您的env var并像这样运行kubectl：

```
export LBIP="1.2.3.4"
envsubst < deploy.yml | kubectl apply -f -
```

## kustomize 打补丁

安装：下载[release](https://github.com/kubernetes-sigs/kustomize/releases), 解压给执行权限即可

github地址：https://github.com/kubernetes-sigs/kustomize

https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/

https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#kustomize-feature-list

# kustomize

## 单一基础应用

![image-20210426144536105](http://myapp.img.mykernel.cn/image-20210426144536105.png)

![image-20210426144629833](http://myapp.img.mykernel.cn/image-20210426144629833.png)

> 注意：bases由resources替换，v1.2版本bases废弃。

```bash
root@ccea447f1caa:/kustomize# tree .
.
├── abc.yaml # 配置文件
├── base # 基础环境
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   ├── service.yaml
│   └── ns.yaml
├── kustomization.yaml # 打补丁的方法
├── replicas-patch.yaml # 补丁
└── update-patch.yaml   # 补丁
```



### 生成基础环境

#### 生成目录结构

```bash
root@ccea447f1caa:~# install -dv /kustomize
install: 正在创建目录'/kustomize'
root@ccea447f1caa:~# cd /kustomize
root@ccea447f1caa:/kustomize# ls
root@ccea447f1caa:/kustomize# install -dv base

# 生成通用的deployment
root@ccea447f1caa:/kustomize# kubectl create deploy myapp --image=ikubernetes/myapp:v1 --dry-run -oyaml > base/deployment.yaml

# 生成通用的svc
root@ccea447f1caa:/kustomize# kubectl create svc nodeport myapp --tcp=30080:80  --dry-run -oyaml > base/service.yaml

# 生成通用的ns
root@ccea447f1caa:/kustomize# kubectl create ns mytest --dry-run -oyaml > base/ns.yaml
```

修正yaml

```diff
root@ccea447f1caa:/kustomize# cat base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myapp
  name: myapp
spec:
  replicas: 1
+  strategy:
+    type: Recreate
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - image: ikubernetes/myapp:v1
        name: myapp
```

```bash
root@ccea447f1caa:/kustomize# cat base/service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: myapp
  name: myapp
spec:
  ports:
  - name: 30080-80
    port: 30080
    protocol: TCP
    targetPort: 80
  selector:
    app: myapp
  type: NodePort
```

```bash
root@ccea447f1caa:/kustomize# cat  base/ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mytest
```

#### 生成基础环境的kustomize配置

```bash
root@ccea447f1caa:/kustomize# cat base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
- ns.yaml
```

或者

```bash
root@ccea447f1caa:/kustomize# cd base/
root@ccea447f1caa:/kustomize/base# kustomize create --autodetect
# 生成配置
root@ccea447f1caa:/kustomize/base# ls
deployment.yaml  kustomization.yaml  ns.yaml  service.yaml

# 查看配置
root@ccea447f1caa:/kustomize/base# cat kustomization.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- ns.yaml
- service.yaml
```



#### 测试

```bash
root@ccea447f1caa:/kustomize/base# cd ..
root@ccea447f1caa:/kustomize# kustomize build base/
apiVersion: v1
kind: Service
metadata:
  labels:
    app: myapp
  name: myapp
spec:
  ports:
  - name: 30080-80
    port: 30080
    protocol: TCP
    targetPort: 80
  selector:
    app: myapp
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myapp
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - image: ikubernetes/myapp:v1
        name: myapp
```

### 添加名字和名称空间补丁

```bash
root@ccea447f1caa:/kustomize# kustomize create --resources ./base --namespace myapp --nameprefix myapp-

root@ccea447f1caa:/kustomize# cat kustomization.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./base
namePrefix: myapp-
namespace: myapp
```

```bash
root@ccea447f1caa:/kustomize#  kustomize build  ./
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: myapp
  name: myapp-myapp
  namespace: myapp
spec:
  ports:
  - name: 30080-80
    port: 30080
    protocol: TCP
    targetPort: 80
  selector:
    app: myapp
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myapp
  name: myapp-myapp
  namespace: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - image: ikubernetes/myapp:v1
        name: myapp
```

对比补丁

```diff
root@ccea447f1caa:/kustomize# kustomize build > 2.yaml
root@ccea447f1caa:/kustomize# kustomize build base/ > 1.yaml
root@ccea447f1caa:/kustomize# diff -u 1.yaml 2.yaml 
--- 1.yaml	2021-04-26 07:52:50.404796000 +0000
+++ 2.yaml	2021-04-26 07:52:46.664796000 +0000
@@ -1,14 +1,15 @@
 apiVersion: v1
 kind: Namespace
 metadata:
-  name: mytest
+  name: myapp
 ---
 apiVersion: v1
 kind: Service
 metadata:
   labels:
     app: myapp
-  name: myapp
+  name: myapp-myapp
+  namespace: myapp
 spec:
   ports:
   - name: 30080-80
@@ -24,7 +25,8 @@
 metadata:
   labels:
     app: myapp
-  name: myapp
+  name: myapp-myapp
+  namespace: myapp
 spec:
   replicas: 1
   selector:
```

名称空间会修改

名字会修改，加前缀

### 添加滚动更新策略补丁

https://github.com/kubernetes-sigs/kustomize/blob/master/examples/inlinePatch.md

```diff
root@ccea447f1caa:/kustomize# cat kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
\- ./base
namePrefix: myapp-
namespace: myapp

# 打补丁
patchesStrategicMerge:
+- update-patch.yaml
```

![image-20210426155349175](http://myapp.img.mykernel.cn/image-20210426155349175.png)

编辑补丁

```bash
root@ccea447f1caa:/kustomize# cp base/deployment.yaml update-patch.yaml 
```

```diff
root@ccea447f1caa:/kustomize# cat update-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
+  strategy:
+    rollingUpdate:
+      maxSurge: 25%
+      maxUnavailable: 25%
+    type: RollingUpdate
```

测试生成最终文件

```bash
root@ccea447f1caa:/kustomize# kustomize build . > 3.yaml

```

```diff
root@ccea447f1caa:/kustomize# diff -u 2.yaml 3.yaml 
--- 2.yaml	2021-04-26 05:25:56.822563000 +0000
+++ 3.yaml	2021-04-26 05:36:22.840188000 +0000
@@ -28,7 +28,10 @@
     matchLabels:
       app: myapp
   strategy:
-    type: Recreate
+    rollingUpdate:
+      maxSurge: 25%
+      maxUnavailable: 25%
+    type: RollingUpdate
   template:
     metadata:
       labels:
```

相比ns,namespace补丁，把更新策略换了



### 添加副本补丁

```bash
root@ccea447f1caa:/kustomize# cat replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
```

```diff
root@ccea447f1caa:/kustomize# cat kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
\- ./base
namePrefix: myapp-
namespace: myapp


patchesStrategicMerge:
\- update-patch.yaml
+- replicas-patch.yaml
```

> `\`这个只是转义作用

生成结果 

```bash
kustomize build . > 4.yaml
```

```diff
diffroot@ccea447f1caa:/kustomize# diff -u 3.yaml 4.yaml 
--- 3.yaml	2021-04-26 05:36:22.840188000 +0000
+++ 4.yaml	2021-04-26 05:40:58.895556000 +0000
@@ -23,7 +23,7 @@
   name: myapp-myapp
   namespace: myapp
 spec:
-  replicas: 1
+  replicas: 3
   selector:
     matchLabels:
       app: myapp
```

### 添加secret或configmap

```diff
root@ccea447f1caa:/kustomize# cat kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./base
namePrefix: myapp-
namespace: myapp


patchesStrategicMerge:
- update-patch.yaml
- replicas-patch.yaml

# 自动生成configmap
+configMapGenerator:
+- name: test     # 此名称为前缀的cm, 后面会生成哈希值，始终稳定
+  files:
+  - abc.yaml

+- name: myvar
+  literals:
+  - JAVA_HOME=/opt/java/
+  - JAVA_TOOL_OPT=-agent

# 自动生成secret
+secretGenerator:
+- name: postgres
+  files:
+  - abc.yaml
+- name: myvar-secret
+  literals:
+  - JAVA_HOME=/opt/secret/
+  - JAVA_TOOL_OPT=-agent
```

```bash
touch abc.yaml
```

```bash
kustomize build  .

# 会生成以下2个cm, 2个secret. 每次修改kustomize时， 其生成的名称不会变。
root@ccea447f1caa:/kustomize# kustomize build  .
apiVersion: v1
data:
  JAVA_HOME: /opt/java/
  JAVA_TOOL_OPT: -agent
kind: ConfigMap
metadata:
  name: myapp-myvar-9kthb54hmm
  namespace: myapp
---
apiVersion: v1
data:
  abc.yaml: ""
kind: ConfigMap
metadata:
  name: myapp-test-7bm7668d4g
  namespace: myapp
---
apiVersion: v1
data:
  JAVA_HOME: L29wdC9zZWNyZXQv
  JAVA_TOOL_OPT: LWFnZW50
kind: Secret
metadata:
  name: myapp-myvar-secret-dt96ktb56c
  namespace: myapp
type: Opaque
---
apiVersion: v1
data:
  abc.yaml: ""
kind: Secret
metadata:
  name: myapp-postgres-mf8htfgt2b
  namespace: myapp
type: Opaque
```

### 所有资源添加标签

```diff
root@ccea447f1caa:/kustomize# cat kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
\- ./base
namePrefix: myapp-
namespace: myapp
+commonLabels:
+  environment: test

patchesStrategicMerge:
\- update-patch.yaml
\- replicas-patch.yaml

# 自动生成configmap
configMapGenerator:
\- name: test     # 此名称为前缀的cm, 后面会生成哈希值，始终稳定
  files:
  - abc.yaml

\- name: myvar
  literals:
  - JAVA_HOME=/opt/java/
  - JAVA_TOOL_OPT=-agent

# 自动生成secret
secretGenerator:
\- name: postgres
  files:
  - abc.yaml
\- name: myvar-secret
  literals:
  - JAVA_HOME=/opt/secret/
  - JAVA_TOOL_OPT=-agent
```

```bash
root@ccea447f1caa:/kustomize# kustomize build  .
kind: Service
metadata:
  labels:
    app: myapp
    environment: test
  selector:
    app: myapp
    environment: test

kind: Deployment
metadata:
  labels:
    app: myapp
    environment: test
  selector:
    matchLabels:
      app: myapp
      environment: test
  template:
    metadata:
      labels:
        app: myapp
        environment: test


kind: ConfigMap
metadata:
  labels:
    environment: test

...
```

### 添加变量

```bash
root@ccea447f1caa:/kustomize# cp base/deployment.yaml var-patch.yaml
```

```diff
root@ccea447f1caa:/kustomize# cat var-patch.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
+        env:
+        - name: host-property
+          value: $(MYSVC) # 会引用kustomize的MYSVC变量， 必须大写
+        - name: host
+          value: $(HOST) # 会引用kustomize的HOST变量， 必须大写
+        - name: java_home
+          valueFrom:
+            secretKeyRef:
+              name: myvar-secret  # 会自动引用kustomize定义的myvar-secret自动生成的secret
+              key: JAVA_HOME
```

```diff
root@ccea447f1caa:/kustomize# cat kustomization.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./base
namePrefix: myapp-
namespace: myapp
commonLabels:
  environment: test

patchesStrategicMerge:
- update-patch.yaml
- replicas-patch.yaml
- var-patch.yaml

# 自动生成configmap
configMapGenerator:
- name: test     # 此名称为前缀的cm, 后面会生成哈希值，始终稳定
  files:
  - abc.yaml

- name: myvar
  literals:
  - JAVA_HOME=/opt/java/
  - JAVA_TOOL_OPT=-agent

# 自动生成secret
secretGenerator:
- name: postgres
  files:
  - abc.yaml
- name: myvar-secret
  literals:
  - JAVA_HOME=/opt/secret/
  - JAVA_TOOL_OPT=-agent


# 添加变量会在patch中 $() 引用
+vars:
+  - name: MYSVC
+    objref:
+      kind: Service
+      name: myapp
+      apiVersion: v1
+    fieldref:
+      fieldpath: metadata.name # 不加字段引用，如下面HOST变量， 也默认引用metadata.name.
+  - name: HOST
+    objref:
+      kind: Service
+      name: myapp
+      apiVersion: v1
```

生成结果 

```bash
root@ccea447f1caa:/kustomize# kustomize build . > 5.yaml
```

```diff
root@ccea447f1caa:/kustomize# diff -u 4.yaml 5.yaml 
--- 4.yaml	2021-04-26 05:40:58.895556000 +0000
+++ 5.yaml	2021-04-26 06:00:31.709398000 +0000
@@ -1,8 +1,53 @@
 apiVersion: v1
+data:
+  JAVA_HOME: /opt/java/
+  JAVA_TOOL_OPT: -agent
+kind: ConfigMap
+metadata:
+  labels:
+    environment: test
+  name: myapp-myvar-9kthb54hmm
+  namespace: myapp
+---
+apiVersion: v1
+data:
+  abc.yaml: ""
+kind: ConfigMap
+metadata:
+  labels:
+    environment: test
+  name: myapp-test-7bm7668d4g
+  namespace: myapp
+---
+apiVersion: v1
+data:
+  JAVA_HOME: L29wdC9zZWNyZXQv
+  JAVA_TOOL_OPT: LWFnZW50
+kind: Secret
+metadata:
+  labels:
+    environment: test
+  name: myapp-myvar-secret-dt96ktb56c
+  namespace: myapp
+type: Opaque
+---
+apiVersion: v1
+data:
+  abc.yaml: ""
+kind: Secret
+metadata:
+  labels:
+    environment: test
+  name: myapp-postgres-mf8htfgt2b
+  namespace: myapp
+type: Opaque
+---
+apiVersion: v1
 kind: Service
 metadata:
   labels:
     app: myapp
+    environment: test
   name: myapp-myapp
   namespace: myapp
 spec:
@@ -13,6 +58,7 @@
     targetPort: 80
   selector:
     app: myapp
+    environment: test
   type: NodePort
 ---
 apiVersion: apps/v1
@@ -20,6 +66,7 @@
 metadata:
   labels:
     app: myapp
+    environment: test
   name: myapp-myapp
   namespace: myapp
 spec:
@@ -27,6 +74,7 @@
   selector:
     matchLabels:
       app: myapp
+      environment: test
   strategy:
     rollingUpdate:
       maxSurge: 25%
@@ -36,7 +84,18 @@
     metadata:
       labels:
         app: myapp
+        environment: test
     spec:
       containers:
-      - image: ikubernetes/myapp:v1
+      - env:
+        - name: host-property
+          value: myapp-myapp
+        - name: host
+          value: myapp-myapp
+        - name: java_home
+          valueFrom:
+            secretKeyRef:
+              key: JAVA_HOME
+              name: myapp-myvar-secret-dt96ktb56c
+        image: ikubernetes/myapp:v1
         name: myapp
```

注意：由于以上添加secret, cm, 加标签，我没有将生成结果保存，所以这一步的结果相对于加了副本补丁那一次的结果。多了：标签、cm、secret、变量。。

### 更新镜像

```diff
root@ccea447f1caa:/kustomize# cat kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./base
namePrefix: myapp-
namespace: myapp
commonLabels:
  environment: test

patchesStrategicMerge:
- update-patch.yaml
- replicas-patch.yaml
- var-patch.yaml

# 自动生成configmap
configMapGenerator:
- name: test     # 此名称为前缀的cm, 后面会生成哈希值，始终稳定
  files:
  - abc.yaml

- name: myvar
  literals:
  - JAVA_HOME=/opt/java/
  - JAVA_TOOL_OPT=-agent

# 自动生成secret
secretGenerator:
- name: postgres
  files:
  - abc.yaml
- name: myvar-secret
  literals:
  - JAVA_HOME=/opt/secret/
  - JAVA_TOOL_OPT=-agent


# 添加变量会在patch中 $() 引用
vars:
  - name: MYSVC
    objref:
      kind: Service
      name: myapp
      apiVersion: v1
    fieldref:
      fieldpath: metadata.name
  - name: HOST
    objref:
      kind: Service
      name: myapp
      apiVersion: v1

# 镜像
+images:
+- name: ikubernetes/myapp
+  newName: ikubernetes/myapp # 新镜像
+  newTag: v2                 # 新tag

```

```bash
root@ccea447f1caa:/kustomize# kustomize build .
root@ccea447f1caa:/kustomize# kustomize build . | kubectl apply -f  -
```

### 非标准更新, patches操作

由于kustomize一般是默认的base + merge来渲染的。例如ingress.yaml中的域名，你需要修改，但是文本太多了，不容易突出。或者prometheus的matchlabel在kustomize并不能识别。这个时候patches操作显得非常重要

#### 更新ingress.yaml的域名

目录结构

```bash
pos-prod@1227f652bef0:~/pos/k8s/service/ishop-pos$ ll
总用量 48
drwxrwxr-x 2 pos-prod pos-prod 4096 8月  16 07:33 ./
drwxrwxr-x 4 pos-prod pos-prod 4096 8月  16 07:25 ../
-rw-rw-r-- 1 pos-prod pos-prod  880 8月  16 07:32 env-patch.yaml
-rw-rw-r-- 1 pos-prod pos-prod  155 8月  16 07:20 ingress-patch.yaml
-rw-rw-r-- 1 pos-prod pos-prod  403 8月  16 06:34 ingress.yaml
-rw-rw-r-- 1 pos-prod pos-prod  893 8月  16 07:23 kustomization.yaml
-rw-rw-r-- 1 pos-prod pos-prod  231 8月  16 07:33 README.md
-rw-rw-r-- 1 pos-prod pos-prod   82 8月  12 01:54 replicas-patch.yaml
-rw-rw-r-- 1 pos-prod pos-prod  302 8月  12 01:54 resouce-patch.yaml
-rw-rw-r-- 1 pos-prod pos-prod   87 8月  16 07:20 servicemonitor-patch.yaml
-rw-rw-r-- 1 pos-prod pos-prod  452 8月  16 07:20 servicemonitor.yaml
-rw-rw-r-- 1 pos-prod pos-prod  238 8月  12 01:54 updatestrategy-patch.yaml
```

kustomize

```diff
pos-prod@1227f652bef0:~/pos/k8s/service/ishop-pos$ cat kustomization.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
    - ../../tpl/nodejs-base # 目前所有应用共享，相同端口，相同健康检测接口
+    - ingress.yaml

# 应用名
namePrefix: v1ishop-pos-
# 名称空间
namespace: default
# 标签
commonLabels:
  app: v1ishop-pos
  tier: backend
# 补丁，配置变量
patchesStrategicMerge:
    - env-patch.yaml
    - updatestrategy-patch.yaml
    - replicas-patch.yaml
    - resouce-patch.yaml



# 镜像更新
images:
    - name: ikubernetes/myapp
      newName: registry-vpc.cn-hangzhou.aliyuncs.com/ishop-pos/ishop_pos-svc
      newTag: 1.0.27
```

`ingress.yaml`

```yaml
pos-prod@1227f652bef0:~/pos/k8s/service/ishop-pos$ cat ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: 'true'
  name: ingress # kustomization未渲染前的名称。
spec:
  rules:
    - host: test.aliyun.com
      http:
        paths:
          - backend:
              serviceName: service
              servicePort: 7001
            path: /
  tls:
    - hosts:
        - test.aliyun.com
      secretName: graspishop.com
```

> 常规情况会把name，修改。但是不能修改host. 可以编辑 这个文件实现更新，但是长期来看，不利于更新，可能就忘记了。

使用patches更新 `kustomization.yaml` 文件中添加

```yaml
patches:
    - path: ingress-modify-patch.yaml
      target: # 对哪个资源更新
        group: extensions
        version: v1beta1
        kind: Ingress
        name: ingress # kustomization未渲染前的名称。
```

`ingress-patch.yaml`

```yaml
pos-prod@1227f652bef0:~/pos/k8s/service/ishop-pos$ cat ingress-modify-patch.yaml 
- op: replace
  path: "/spec/rules/0/host"
  value: "v100pos.graspishop.com"
- op: replace
  path: "/spec/tls/0/hosts/0"
  value: "v100pos.graspishop.com"
```

> 也可以使用Json格式
>
> [{"op": "", ...}, {""}]

#### 更新servicemonitor的标签

先看看这个资源定义方式`servicemonitor.yaml `

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: servicemonitor
spec:
  endpoints:
  - interval: 30s
    # Mysqld Grafana模版ID为7362
    # 填写service.yaml中Prometheus Exporter对应的Port的Name字段的值
    port: http
    # 填写Prometheus Exporter代码中暴露的地址
    path: /metrics
  namespaceSelector:
    any: true
    # Demo的命名空间
    matchLabels:
      app: ishop-pos
      tier: backend

```

prometheus exporter定义监控: https://help.aliyun.com/document_detail/196092.html?spm=a2c4g.11186623.2.25.3f444a597Hm0LF

如果把这个作为base资源，并不会自动给matchLabels添加标签，所以添加这个文件后，还需要手工定义和kustomization相同的标签。避免忘记，使用专用文件来操作。
`servicemonitor-modify-patch.yaml`

```yaml
- op: replace
  path: "/spec/namespaceSelector/matchLabels/app"
  value: "v1ishop-pos"
```

然后在kustomize中定义

```yaml
resources:
- ../../tpl/nodejs-base # 目前所有应用共享，相同端口，相同健康检测接口
- servicemonitor.yaml   # 监控

patches:
- path: ingress-modify-patch.yaml
  target:
    group: extensions
    version: v1beta1
    kind: Ingress
    name: ingress
- path: servicemonitor-modify-patch.yaml
  target:
    group: monitoring.coreos.com
    version: v1
    kind: ServiceMonitor
    name: servicemonitor
```





## 多个app结合起来部署

https://github.com/kubernetes-sigs/kustomize/tree/master/examples

![image-20210426144723650](http://myapp.img.mykernel.cn/image-20210426144723650.png)

![image-20210426144514136](http://myapp.img.mykernel.cn/image-20210426144514136.png)



wordpress + mysql 组合起来才可以使用博客，却又可以单一使用mysql，这样在开发、测试、预发、生产的配置就不一样了

```bash
root@ccea447f1caa:/kustomize# tree ~/kustomize/
/root/kustomize/
├── base-modules
│   ├── mysql
│   │   ├── deployment.yaml
│   │   ├── kustomization.yaml
│   │   ├── secret.yaml
│   │   └── service.yaml
│   └── wordpress
│       ├── deployment.yaml
│       ├── kustomization.yaml
│       └── service.yaml
└── staging
    ├── abc.yaml
    ├── deploy-patch.yaml
    ├── kustomization.yaml
    ├── patch-1.yaml
    └── patch.yaml
```

```bash
root@ccea447f1caa:/kustomize# cat ~/kustomize/staging/kustomization.yaml 
# https://www.youtube.com/watch?v=ahMIBxufNR0

# 所有东西放在这个名称空间中
namespace: mytest

# 引用资源
resources:
- ../base-modules/wordpress
- ../base-modules/mysql
```



