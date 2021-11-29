---
title: "Helm管理器"
date: 2021-02-05 05:03:35
---



# 前言

- Chart + Config -> Release

<!--more-->
# 为什么使用helm?

之前部署k8s的一个应用prometheus, 需要周边组件均要部署

- pod -> Pod Controller(Deployment, StatefulSet, DaemonSet)
  - replicas, template, label selector, label
  - env, volumeMounts
  - serviceName: role/clusterrole/rolebinding/clusterrolebinding
- Service, Ingress（nodeport,clusterip,loadbalancer,externalname）
- volumes(直接指/pvc->pv/configmap/secret)

启动应用、配置清单：方便二次启动

- helm 类似于yum, 一键安装部署k8s应用。部署jenkins: `helm install jenkins`.
- helm只包含`配置清单` chart，清单中的镜像不包含。

应用程序升级、回滚

- helm 调用内部控制完成一键升级、回滚（helm是外壳，真正的还是Pod Controller）

```bash
chart      打包的chart
 
chart repo  http/https服务器

chart release tiler部署chart之后叫chart release
```

<!--more-->



# helm命令

k8s集群之上运行一个pod`tiler`, 完成部署任务执行。

helm是`tiler`的客户端，指挥`tiler`完成工作。helm可以在集群外工作，就需要给`tiler`作service

![image-20210205132516884](http://myapp.img.mykernel.cn/image-20210205132516884.png)



交互过程

helm 下载清单, 清单不是单个文件，是多个文件的打包结果，叫`chart`

tiler 应用清单

> helm本地有chart(没有就从repo仓库下载), 提交chart给tiler, chart叫 chart release(即chart一次部署)
>
> chart是静态的
>
> repo仓库：互联网很多人将流行的程序打包放在一个位置，需要配置给helm仓库位置。
>
> release是chart运行起来的副本

![image-20210205133706575](http://myapp.img.mykernel.cn/image-20210205133706575.png)

安装jenkins

1. helm: repo 检索chart,下载至本地， 提交给tiler(grpc: google rpc协议)

   > 分布式通用协作格式：2000, http restful风格
   >
   > google发布grpc协议

2. tiler: 与api server通信，应用chart, 成为chart release



## config

以一个chart生成多个release, 能够向chart传递配置参数，完成自定义release. `config` 结合charts生成release，确保release不同名



ansible templates一个配置完成任意节点适用配置。python开发，python模板引擎。

puppet模板引擎， ruby研发，ruby模板引擎。

php, 嵌入html中，php生成数据，生成真正的html。



config 的变量 结合 chart模板配置清单 生成真正的配置清单。 大多时候不需要配置config, 默认值即可以适用chart模板。

chart是模板，被模板引擎渲染时使用config的值 生成配置清单，提交到tiler, 生成release.      k8s是go开发，go模板引擎。



## helm v2 和 helm v3

helm v2 tiller serviceaccount 需要集群权限

由于我们是使用helm v3版本，所以不需要tiler，直接使用kubeconfig文件验证集群

https://helm.sh/zh/docs/faq/

- install名称空间必须事先存在 

- release属于名称空间 `helm ls -n myapp`

- 将 `requirements.yaml` 合并到了 `Chart.yaml`

- Name (或者 --generate-name) 安装时是必需的

- 请查看 `helm help chart` 和 `helm help registry` 了解如何打包chart并推送到Docker注册中心的更多信息。

  更多信息请查看 [注册中心](https://docs.helm.sh/zh/docs/topics/registries/)页面。



# 部署helm

- [x] 部署tiler
- [x] 部署helm



先部署helm, helm与api server交互部署tiler.

之后helm仅与tiller交互



## 部署helm

https://github.com/helm/helm#install

To rapidly get Helm up and running, start with the [Quick Start Guide](https://helm.sh/docs/intro/quickstart/).

See the [installation guide](https://helm.sh/docs/intro/install/) for more options, including installing pre-releases.

https://helm.sh/docs/intro/install/

Every [release](https://github.com/helm/helm/releases) of Helm provides binary releases for a variety of OSes. These binary versions can be manually downloaded and installed.

1. Download your [desired version](https://github.com/helm/helm/releases) # 选择版本
2. Unpack it (`tar -zxvf helm-v3.0.0-linux-amd64.tar.gz`) # 展开
3. Find the `helm` binary in the unpacked directory, and move it to its desired destination (`mv linux-amd64/helm /usr/local/bin/helm`)



## Installation and Upgrading

Download Helm v3.5.2. The common platform binaries are here:

- [Linux amd64](https://get.helm.sh/helm-v3.5.2-linux-amd64.tar.gz) ([checksum](https://get.helm.sh/helm-v3.5.2-linux-amd64.tar.gz.sha256sum) / 01b317c506f8b6ad60b11b1dc3f093276bb703281cb1ae01132752253ec706a2)

```bash
# 部署位置：
# 1. master
# 2. 任意linux系统，他只是client。但是需要调用kubectl命令读取$HOME/.kube/config, 有管理仅限

export https_proxy=http://192.168.0.33:808 # docker使用HTTPS_PROXY大写
wget https://get.helm.sh/helm-v3.5.2-linux-amd64.tar.gz
```



展开

```bash
[root@master ~]# tar xvf helm-v3.5.2-linux-amd64.tar.gz 
linux-amd64/
linux-amd64/helm
linux-amd64/LICENSE
linux-amd64/README.md
[root@master ~]# 
[root@master linux-amd64]# install helm /usr/local/bin/
```

帮助

```bash
[root@master linux-amd64]# helm -h
```

- 



# 使用helm v3

列出helm管理的release

```bash
[root@master ~]# helm list
NAME	NAMESPACE	REVISION	UPDATED	STATUS	CHART	APP VERSION
```

安装jenkins

```bash
[root@master ~]# helm search jenkins

Search provides the ability to search for Helm charts in the various places
they can be stored including the Artifact Hub and repositories you have added.
Use search subcommands to search different locations for charts.

Usage:
  helm search [command]

Available Commands:
  hub         search for charts in the Artifact Hub or your own hub instance # 官方仓库：https://artifacthub.io/
  repo        search repositories for a keyword in charts # 自定义仓库

```

搜索jenkins

```bash
[root@master ~]# helm search hub jenkins
URL                                               	CHART VERSION      	APP VERSION  	DESCRIPTION                                       
https://artifacthub.io/packages/helm/jenkinsci/...	3.1.8              	2.263.3      	Jenkins - Build great things at any scale! The ...
https://artifacthub.io/packages/helm/cloudposse...	0.1.2              	             	A Jenkins Helm chart for Kubernetes               
https://artifacthub.io/packages/helm/bitnami/je...	7.3.3              	2.263.3      	The leading open source automation server         
https://artifacthub.io/packages/helm/choerodon/...	0.1.0              	2.60.3-alpine	A Helm chart for Kubernetes                       
https://artifacthub.io/packages/helm/edu/jenkins  	2.7.1              	lts          	Open source continuous integration server. It s...
https://artifacthub.io/packages/helm/jenkins/je...	0.4.2              	0.5.0        	Kubernetes native operator which fully manages ...
https://artifacthub.io/packages/helm/cloudbees/...	2.263.102          	2.263.1.2    	CloudBees Jenkins Distribution provides develop...
https://artifacthub.io/packages/helm/odavid/my-...	0.1.145            	2.263.3-239  	A Helm chart for my-bloody-jenkins - a self con...
https://artifacthub.io/packages/helm/cloudbees/...	3.25.3+73c21ed5fcda	2.263.2.3    	Enterprise Continuous Integration with Jenkins    
https://artifacthub.io/packages/helm/olli-ai/ex...	1.0.3              	v1.0.3       	Controller for automatically exposing services....
https://artifacthub.io/packages/helm/webhookrel...	0.3.1              	0.5.1        	Webhook Relay Operator provides an easy way to ...
```

管理仓库

```bash
[root@master ~]# helm repo list
Error: no repositories to show
```

## 安装jenkins

https://artifacthub.io/packages/helm/jenkinsci/jenkins

```bash
helm repo add jenkins https://charts.jenkins.io


[root@master ~]# helm repo list
NAME   	URL                      
jenkins	https://charts.jenkins.io

[root@master ~]# export https_proxy=http://www.ik8s.io:10070
[root@master ~]# export https_proxy=http://192.168.0.33:808
[root@master ~]# export no_proxy=172.16.0.0/16,127.0.0.0/8
[root@master ~]# helm repo update # 更新本地缓存同步官方仓库的更新
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jenkins" chart repository
Update Complete. ⎈Happy Helming!⎈

```

查看charts

```bash
[root@master ~]# helm search repo jenkins
NAME           	CHART VERSION	APP VERSION	DESCRIPTION                                       
jenkins/jenkins	3.1.8        	2.263.3    	Jenkins - Build great things at any scale! The ...
```



探测inspect

```bash
helm inspect -h
Available Commands:
  all         show all information of the chart
  chart       show the chart's definition
  readme      show the chart's README
  values      show the chart's values

```

```bash
[root@master ~]# helm inspect  readme jenkins/jenkins

依赖关系：k8s版本
安装方式：
自定义配置参数：configuration

```

一旦inspect之后, 就会下载

```bash
[root@master ~]# ls ~/.cache/helm/repository/
jenkins-3.1.8.tgz  jenkins-charts.txt  jenkins-index.yaml
```

```bash
[root@master ~]# kubectl create ns jenkins
[root@master ~]# helm install jenkins/jenkins --generate-name -n jenkins
```



## 安装redis

以以上方式安装redis

选择Verified publishers

https://artifacthub.io/packages/helm/wyrihaximusnet/redis-db-assignment-operator

![image-20210205145726242](http://myapp.img.mykernel.cn/image-20210205145726242.png)



```bash
# 添加repo
helm repo add wyrihaximusnet https://helm.wyrihaximus.net/

# 查看 
[root@master ~]# helm repo list
NAME          	URL                          
jenkins       	https://charts.jenkins.io    
wyrihaximusnet	https://helm.wyrihaximus.net/

# 查看 charts
[root@master ~]# helm search repo wyrihaximusnet
NAME                                       	CHART VERSION	APP VERSION	DESCRIPTION                                       
wyrihaximusnet/commento                    	0.1.3        	v1.8.0     	Helm chart to install commento on a kubernetes ...
wyrihaximusnet/commons                     	0.1.1        	           	commons library                                   
wyrihaximusnet/cron-jobs                   	0.1.2        	           	CronJobs library                                  
wyrihaximusnet/default-backend             	0.4.4        	random     	A Helm chart for Kubernetes                       
wyrihaximusnet/docker-hub-exporter         	0.5.4        	d05df48    	Docker Hub Exporter                               
wyrihaximusnet/horizontal-pod-autoscalers  	0.2.0        	           	Horizontal Pod Autoscalers library                
wyrihaximusnet/pi-hole-exporter            	0.1.2        	d05df48    	Pi-Hole Exporter                                  
wyrihaximusnet/redirect                    	0.9.4        	random     	Redirect                                          
wyrihaximusnet/redis-db-assignment-operator	1.0.5        	v1.0.1     	Redis Database Assignment Operator            


# inspect readme
[root@master ~]# export https_proxy=http://www.ik8s.io:10070
[root@master ~]# export https_proxy=http://192.168.0.33:808
[root@master ~]# export no_proxy=172.16.0.0/16,127.0.0.0/8
[root@master ~]# helm inspect readme wyrihaximusnet/redis-db-assignment-operator

```

安装redis

```bash
[root@master ~]# helm install  wyrihaximusnet/redis-db-assignment-operator -n redis-release-1 --dry-run --generate-name
# --generate-name helm v3和v2默认一样随机名


# 生成信息
NAME: redis-db-assignment-operator-1612508398
LAST DEPLOYED: Fri Feb  5 15:00:04 2021
NAMESPACE: redis-release-1
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:

NOTES:
Congratulations, you have just installed or upgraded the Redis Databases Assignment Operator!

You can now get a redis database assigned on the Redis server of your picking with the following definition:
apiVersion: wyrihaximus.net/v1
kind: RedisDatabase
metadata:
  name: example
spec:
  secret:
    name: example-redis-database
  service:
    read: redis://redis-follower.redis.svc.cluster.local:6379/
    write: redis://redis-leader.redis.svc.cluster.local:6379/


The operator will then create the following secret:
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: example-redis-database
  namespace: default
data:
  DATABASE: BASE64_ENCODED
  READ: BASE64_ENCODED
  WRITE: BASE64_ENCODED
```

```bash
[root@master ~]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "wyrihaximusnet" chart repository
...Successfully got an update from the "jenkins" chart repository
Update Complete. ⎈Happy Helming!⎈
[root@master ~]# kubectl create ns redis-release-1 # helmv3
[root@master ~]# helm install  wyrihaximusnet/redis-db-assignment-operator -n redis-release-1  --generate-name


# 定义redis集群
apiVersion: wyrihaximus.net/v1
kind: RedisDatabase # CRD
metadata:
  name: example
  namespace: default
spec:
  secret:
    name: example-redis-database
  service:
    read: redis://redis-follower.redis.svc.cluster.local:6379/
    write: redis://redis-leader.redis.svc.cluster.local:6379/


The operator will then create the following secret:
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: example-redis-database
  namespace: default
data:
  DATABASE: BASE64_ENCODED
  READ: BASE64_ENCODED
  WRITE: BASE64_ENCODED

```

# 了解chart格式

不会go语言，但是要知道chart格式，定制chart

https://helm.sh/zh/docs/topics/charts/

| 文件名              | 作用                                        |
| ------------------- | ------------------------------------------- |
| chart.yaml          | chart元数据                                 |
| LICENSE             | 开源许可，公有云公司一定要看                |
| README              |                                             |
| requirements.yaml   | 依赖的charts                                |
| values.yaml         | config默认值                                |
| templates           | go模板                                      |
| templates/NOTES.txt | 配置清单模板使用说明 ：helm install之后显示 |
| crds/               | 自定义资源的定义                            |
| charts              | 包含chart依赖的其他chart                    |



根据以上的文件解开chart

先下载，后才可以inspect/install

```bash
[root@master ~]# cd .cache/helm/repository/
[root@master repository]# ls
jenkins-3.1.8.tgz  jenkins-charts.txt  jenkins-index.yaml  redis-db-assignment-operator-1.0.5.tgz  wyrihaximusnet-charts.txt  wyrihaximusnet-index.yaml
```

```bash
[root@master repository]# tar xvf jenkins-3.1.8.tgz 

[root@master repository]# tree jenkins
jenkins
├── CHANGELOG.md
├── Chart.yaml                # 描述当前chart的元数据文件；版本、chart名
├── README.md                 # 自述文件 helm inspect readme
├── templates                      # go template文件                         
│   ├── config-init-scripts.yaml
│   ├── config.yaml
│   ├── deprecation.yaml
│   ├── _helpers.tpl
│   ├── home-pvc.yaml
│   ├── jcasc-config.yaml
│   ├── jenkins-agent-svc.yaml
│   ├── jenkins-backup-cronjob.yaml
│   ├── jenkins-backup-rbac.yaml
│   ├── jenkins-controller-alerting-rules.yaml
│   ├── jenkins-controller-backendconfig.yaml
│   ├── jenkins-controller-ingress.yaml
│   ├── jenkins-controller-networkpolicy.yaml
│   ├── jenkins-controller-route.yaml
│   ├── jenkins-controller-secondary-ingress.yaml
│   ├── jenkins-controller-servicemonitor.yaml
│   ├── jenkins-controller-statefulset.yaml # 可以观察到所有配置全是模板，由模板引擎渲染，生成字符串替换至此处 
│   ├── jenkins-controller-svc.yaml
│   ├── NOTES.txt
│   ├── rbac.yaml
│   ├── secret-https-jks.yaml
│   ├── secret.yaml
│   ├── service-account-agent.yaml
│   ├── service-account.yaml
│   └── tests
│       ├── jenkins-test.yaml
│       └── test-config.yaml
├── tests
│   ├── config-test.yaml
│   ├── home-pvc-test.yaml
│   ├── jcasc-config-test.yaml
│   ├── jenkins-agent-svc-test.yaml
│   ├── jenkins-backup-cronjob-test.yaml
│   ├── jenkins-controller-alerting-rules-test.yaml
│   ├── jenkins-controller-ingress-1.19-test.yaml
│   ├── jenkins-controller-ingress-test.yaml
│   ├── jenkins-controller-networkpolicy-test.yaml
│   ├── jenkins-controller-secondary-ingress-1.19-test.yaml
│   ├── jenkins-controller-secondary-ingress-test.yaml
│   ├── jenkins-controller-servicemonitor_test.yaml
│   ├── jenkins-controller-statefulset-test.yaml
│   ├── jenkins-controller-svc-test.yaml
│   ├── rbac-test.yaml
│   ├── secret-test.yaml
│   ├── service-account-agent-test.yaml
│   └── service-account-test.yaml
├── Tiltfile
├── VALUES_SUMMARY.md
└── values.yaml        # helm inspect values， 为模板引擎提供值

如果依赖其他文件，requirements.yaml文件定义
```

## chart.yaml

https://helm.sh/zh/docs/topics/charts/

```bash
annotations:
  artifacthub.io/links: |
    - name: Chart Source
      url: https://github.com/jenkinsci/helm-charts/tree/main/charts/jenkins
    - name: Jenkins
      url: https://www.jenkins.io/
apiVersion: v2 #chart版本
appVersion: 2.263.3 #部署程序的版本


description: Jenkins - Build great things at any scale! The leading open source automation server, Jenkins provides hundreds of plugins to support building, deploying and automating any project.
home: https://jenkins.io/
icon: https://wiki.jenkins-ci.org/download/attachments/2916393/logo.png # 仓库图标链接
maintainers:
- email: maor.friedman@redhat.com
  name: maorfr
- email: mail@torstenwalter.de
  name: torstenwalter
- email: garridomota@gmail.com
  name: mogaal
- email: wmcdona89@gmail.com
  name: wmcdona89
- email: timjacomb1@gmail.com
  name: timja
name: jenkins # chart名
sources: # chart自身所在仓库
- https://github.com/jenkinsci/jenkins
- https://github.com/jenkinsci/docker-inbound-agent
- https://github.com/maorfr/kube-tasks
- https://github.com/jenkinsci/configuration-as-code-plugin
version: 3.1.8 

```

Helm 中，chart可能会依赖其他任意个chart。 这些依赖可以使用`Chart.yaml`文件中的`dependencies` 字段动态链接，或者被带入到`charts/` 目录并手动配置。







## values.yaml

`jenkins/values.yaml `为模板引擎提供值

- 命令行提供值 
- 修改values.yaml文件

`values.yaml`文件提供的必要值如下：

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

## templates

所有模板文件存储在chart的 `templates/` 文件夹。 当Helm渲染chart时，它会通过模板引擎遍历目录中的每个文件。

模板的Value通过以下方式提供：

- Chart开发者可以在chart中提供一个命名为 `values.yaml` 的文件。这个文件包含了默认值。
- Chart用户可以提供一个包含了value的YAML文件。可以在命令行使用 `helm install`命令时提供。
- helm install --set 提供

用户提供自定义value时，这些value会覆盖chart的`values.yaml`文件中value。

### 调用内建值

```bash
以下值是预定义的，对每个模板都有效，并且可以被覆盖。和所有值一样，名称 区分大小写。

Release.Name: 版本名称(非chart的)
Release.Namespace: 发布的chart版本的命名空间
Release.Service: 组织版本的服务
Release.IsUpgrade: 如果当前操作是升级或回滚，设置为true
Release.IsInstall: 如果当前操作是安装，设置为true
Chart: Chart.yaml的内容。因此，chart的版本可以从 Chart.Version 获得， 并且维护者在Chart.Maintainers里。
Files: chart中的包含了非特殊文件的类图对象。这将不允许您访问模板， 但是可以访问现有的其他文件（除非被.helmignore排除在外）。 使用{{ index .Files "file.name" }}可以访问文件或者使用{{.Files.Get name }}功能。 您也可以使用{{ .Files.GetBytes }}作为[]byte方位文件内容。
Capabilities: 包含了Kubernetes版本信息的类图对象。({{ .Capabilities.KubeVersion }} 和支持的Kubernetes API 版本({{ .Capabilities.APIVersions.Has "batch/v1" }})
```

### 调用values文件提供的值

考虑到前面部分的模板，`values.yaml`文件提供的必要值如下：

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"

volumePermissions:
  enabled: false
  image:
   reistry: docker.io
   repository: bitnami/minideb
   tag: latest
   pullPolicy: IfNotPresent

```

然后使用模板中的`.Values`对象就可以任意访问这些值了：

```diff
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
+          image: {{ .Values.imageRegistry }}/postgres:{{ .Values.dockerTag }}
+          imagePullPolicy: {{ .Values.pullPolicy }} # 单级
+          imagePullPolicy: {{ .Values.volumePermissions.image.pullPolicy }} # 多级
          ports:
            - containerPort: 5432
          env:
+            - name: DATABASE_STORAGE
+              value: {{ default "minio" .Values.storage }}
```

> 有些值有作用域

# 自定义chart

快速生成框架

```bash
# helm create -h
For example, 'helm create foo' will create a directory structure that looks
something like this:

    foo/
    ├── .helmignore   # Contains patterns to ignore when packaging Helm charts.
    ├── Chart.yaml    # Information about your chart
    ├── values.yaml   # The default values for your templates
    ├── charts/       # Charts that this chart depends on
    └── templates/    # The template files
        └── tests/    # The test files

```



生成myapp

```bash
# 切换成内建的repo目录
[root@master ~]# cd .cache/helm/repository/
[root@master repository]# helm create myapp
Creating myapp

[root@master repository]# tree myapp/
myapp/
├── charts  # 依赖的chart
├── Chart.yaml
├── templates  # 不需要的删除，需要留着
│   ├── deployment.yaml # deploy 
│   ├── _helpers.tpl
│   ├── hpa.yaml # hpa
│   ├── ingress.yaml # ingress 发布
│   ├── NOTES.txt
│   ├── serviceaccount.yaml # 账户
│   ├── service.yaml # service
│   └── tests
│       └── test-connection.yaml
└── values.yaml

```

## 配置chart.yaml元数据

```diff
[root@master repository]# cat myapp/Chart.yaml
apiVersion: v2
+name: myapp # chart名
+description: A Helm chart for Kubernetes # 描述

# A chart can be either an 'application' or a 'library' chart.
#
# Application charts are a collection of templates that can be packaged into versioned archives
# to be deployed.
#
# Library charts provide useful utilities or functions for the chart developer. They're included as
# a dependency of application charts to inject those utilities and functions into the rendering
# pipeline. Library charts do not define any templates and therefore cannot be deployed.
type: application

# This is the chart version. This version number should be incremented each time you make changes
# to the chart and its templates, including the app version.
# Versions are expected to follow Semantic Versioning (https://semver.org/)
+version: 0.1.1 # 修正chart的版本

# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application. Versions are not expected to
# follow Semantic Versioning. They should reflect the version the application is using.
# It is recommended to use it with quotes.
+appVersion: "v1"

```

## 模板template/deployment.yaml

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
+  name: {{ include "myapp.fullname" . }} # 部署时指定的名字
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
+        - name: {{ .Chart.Name }} # Chart文件的name
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
+          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}" # 镜像。 默认是chart内的appversion
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}

```

## service.yaml

```diff
[root@master repository]# cat myapp/templates/service.yaml 
apiVersion: v1
kind: Service
metadata:
+  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
+    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
+    {{- include "myapp.selectorLabels" . | nindent 4 }}
```

## ingress.yaml

定义Ingress发布服务

```diff
[root@master repository]# cat myapp/templates/ingress.yaml 
{{- if .Values.ingress.enabled -}}
{{- $fullName := include "myapp.fullname" . -}}
{{- $svcPort := .Values.service.port -}}
{{- if semverCompare ">=1.14-0" .Capabilities.KubeVersion.GitVersion -}}
apiVersion: networking.k8s.io/v1beta1
{{- else -}}
apiVersion: extensions/v1beta1
{{- end }}
kind: Ingress
metadata:
  name: {{ $fullName }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            backend:
              serviceName: {{ $fullName }}
              servicePort: {{ $svcPort }}
          {{- end }}
    {{- end }}
  {{- end }}
```



## values.yaml

```diff
# Default values for myapp.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 2

image:
+  repository: ikubernetes/myapp  # {{ .Values.image.repository }}
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
+  tag: "v1"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}

podSecurityContext: {}
  # fsGroup: 2000

securityContext: {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

service:
  type: ClusterIP
+  port: 80  # 集群端口

ingress:
  enabled: false # 启用ingress?
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths: []
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

resources: 
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
   limits:
     cpu: 100m
     memory: 128Mi
   requests:
     cpu: 100m
     memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

nodeSelector: {} # pod运行节点

tolerations: []

affinity: {}

```

##  templates/NOTES.txt 

```bash
1. Get the application URL by running these commands:
{{- if .Values.ingress.enabled }}
{{- range $host := .Values.ingress.hosts }}
  {{- range .paths }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ $host.host }}{{ .path }}
  {{- end }}
{{- end }}
{{- else if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "myapp.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if contains "LoadBalancer" .Values.service.type }}
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace {{ .Release.Namespace }} svc -w {{ include "myapp.fullname" . }}'
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "myapp.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else if contains "ClusterIP" .Values.service.type }}
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "myapp.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace {{ .Release.Namespace }} $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:$CONTAINER_PORT
{{- end }}

```



## 打包myapp

检验myapp格式

```bash
[root@master repository]# helm lint myapp
==> Linting myapp
[INFO] Chart.yaml: icon is recommended # INFO级别可以不管

1 chart(s) linted, 0 chart(s) failed

```

## 本地部署

```bash
[root@master repository]# helm install -h


[root@master repository]# kubectl create ns myapp
namespace/myapp created
[root@master repository]# helm install -n myapp ./myapp/ --generate-name


[root@master repository]# helm ls -n myapp
NAME            	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART      	APP VERSION
myapp-1612512015	myapp    	1       	2021-02-05 16:00:18.545381924 +0800 CST	deployed	myapp-0.1.1	1.3.0      
[root@master repository]# kubectl get pods -n myapp
NAME                                READY   STATUS    RESTARTS   AGE
myapp-1612512015-69c774d988-8fkdb   1/1     Running   0          11m
myapp-1612512015-69c774d988-jmtwk   1/1     Running   0          11m
```

## 升级版本

修改chart.yaml

```diff
[root@master repository]# cat myapp/Chart.yaml
apiVersion: v2
name: myapp
description: A Helm chart for Kubernetes

# A chart can be either an 'application' or a 'library' chart.
#
# Application charts are a collection of templates that can be packaged into versioned archives
# to be deployed.
#
# Library charts provide useful utilities or functions for the chart developer. They're included as
# a dependency of application charts to inject those utilities and functions into the rendering
# pipeline. Library charts do not define any templates and therefore cannot be deployed.
type: application

# This is the chart version. This version number should be incremented each time you make changes
# to the chart and its templates, including the app version.
# Versions are expected to follow Semantic Versioning (https://semver.org/)
+version: 0.1.2

# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application. Versions are not expected to
# follow Semantic Versioning. They should reflect the version the application is using.
# It is recommended to use it with quotes.
+appVersion: "1.4"

```

修改values.yaml

```diff
image:
  repository: ikubernetes/myapp  # {{ .Values.image.repository }}
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
+  tag: "v2"
```

升级

```bash
[root@master repository]# helm ls -n myapp
NAME            	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART      	APP VERSION
myapp-1612512015	myapp    	1       	2021-02-05 16:00:18.545381924 +0800 CST	deployed	myapp-0.1.1	1.3.0      
[root@master repository]# helm upgrade myapp-1612512015 ./myapp/ -n myapp
Release "myapp-1612512015" has been upgraded. Happy Helming!
NAME: myapp-1612512015
LAST DEPLOYED: Fri Feb  5 16:16:05 2021
NAMESPACE: myapp
STATUS: deployed
REVISION: 2
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace myapp -l "app.kubernetes.io/name=myapp,app.kubernetes.io/instance=myapp-1612512015" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace myapp $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace myapp port-forward $POD_NAME 8080:$CONTAINER_PORT
  
  
[root@master repository]# kubectl get pod -n myapp
NAME                                READY   STATUS        RESTARTS   AGE
myapp-1612512015-68bf4985b7-mbbvh   1/1     Running       0          28s
myapp-1612512015-68bf4985b7-zkv49   1/1     Running       0          60s


# 版本
[root@master repository]# kubectl get deploy -n myapp -o wide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                 SELECTOR
																	  # v2
myapp-1612512015   2/2     1            2           18m   myapp        ikubernetes/myapp:v2   app.kubernetes.io/instance=myapp-1612512015,app.kubernetes.io/name=myapp

[root@master repository]# helm list -n myapp
NAME            	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART      	APP VERSION	
																								# 0.1.2     # app 1.4
myapp-1612512015	myapp    	4       	2021-02-05 16:18:24.636764179 +0800 CST	deployed	myapp-0.1.2	1.4        


```

如果有问题，可以修改以上`tag`, 再回滚 `helm rollback`

## 回滚

```bash
[root@master repository]# helm rollback myapp-1612512015  -n myapp
Rollback was a success! Happy Helming!


[root@master repository]# kubectl get deploy -n myapp -o wide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                 SELECTOR
myapp-1612512015   2/2     2            2           17m   myapp        ikubernetes/myapp:v1   app.kubernetes.io/instance=myapp-1612512015,app.kubernetes.io/name=myapp


[root@master repository]#  helm list -n myapp
NAME            	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART      	APP VERSION
																							   # 0.1.1       # app 1.3.0
myapp-1612512015	myapp    	5       	2021-02-05 16:19:16.826643818 +0800 CST	deployed	myapp-0.1.1	1.3.0      

```

## 其他命令

查看release状态,  安装后的输出信息

```bash
[root@master repository]# helm status myapp-1612512015 -n myapp 
NAME: myapp-1612512015
LAST DEPLOYED: Fri Feb  5 16:19:16 2021
NAMESPACE: myapp
STATUS: deployed
REVISION: 5
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace myapp -l "app.kubernetes.io/name=myapp,app.kubernetes.io/instance=myapp-1612512015" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace myapp $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace myapp port-forward $POD_NAME 8080:$CONTAINER_PORT
```

查看详细信息

```bash
[root@master repository]# helm inspect values ./myapp/
[root@master repository]# helm inspect readme ./myapp/
...
  all         show all information of the chart
  chart       show the chart's definition
  readme      show the chart's README
  values      show the chart's values


```

```bash
本地模板 `helm template`

检验chart `helm verify`

download chart `helm fetch` 下载解压

helm get 仅下载

helm dependency 依赖

completion 补全
```

## 分发chart

供别人使用

```bash
[root@master repository]# helm package -h

This command packages a chart into a versioned chart archive file. If a path
is given, this will look at that path for a chart (which must contain a
Chart.yaml file) and then package that directory.

Versioned chart archives are used by Helm package repositories.

To sign a chart, use the '--sign' flag. In most cases, you should also
provide '--keyring path/to/secret/keys' and '--key keyname'.

  $ helm package --sign ./mychart --key mykey --keyring ~/.gnupg/secring.gpg

If '--keyring' is not specified, Helm usually defaults to the public keyring
unless your environment is otherwise configured.

Usage:
  helm package [CHART_PATH] [...] [flags]

```

```bash
[root@master repository]# helm package ./myapp/
Successfully packaged chart and saved it to: /root/.cache/helm/repository/myapp-0.1.2.tgz
# 自动生成版本号对应的压缩包
```

直接使用打包文件

```bash
[root@master repository]# helm inspect values ./myapp-0.1.2.tgz 
```



## chart 镜像仓库

镜像仓库 https://docs.helm.sh/zh/docs/topics/registries/

```bash
export HELM_EXPERIMENTAL_OCI=1
docker run -dp 5000:5000 --restart=always --name registry registry
如果你希望持久化存储，可以在上面的命令中添加 -v $(pwd)/registry:/var/lib/registry

如果您想启用注册中心认证，需要使用用户名和密码先创建 auth.htpasswd 文件：

htpasswd -cB -b auth.htpasswd myuser mypass
然后启动服务，启动时挂载文件并设置 REGISTRY_AUTH环境变量：

docker run -dp 5000:5000 --restart=always --name registry \
  -v $(pwd)/auth.htpasswd:/etc/docker/registry/auth.htpasswd \
  -e REGISTRY_AUTH="{htpasswd: {realm: localhost, path: /etc/docker/registry/auth.htpasswd}}" \
  registry
 
 
 $ helm registry login -u myuser localhost:5000
Password:
Login succeeded


$ helm registry logout localhost:5000
Logout succeeded


保存chart目录到本地缓存
$ helm chart save mychart/ localhost:5000/myrepo/mychart:2.7.0
ref:     localhost:5000/myrepo/mychart:2.7.0
digest:  1b251d38cfe948dfc0a5745b7af5ca574ecb61e52aed10b19039db39af6e1617
size:    2.4 KiB
name:    mychart
version: 0.1.0
2.7.0: saved


列举出所有的chart
 helm chart list
REF                                                     NAME                    VERSION DIGEST  SIZE            CREATED
localhost:5000/myrepo/mychart:2.7.0                     mychart                 2.7.0   84059d7 454 B           27 seconds
localhost:5000/stable/acs-engine-autoscaler:2.2.2       acs-engine-autoscaler   2.2.2   d8d6762 4.3 KiB         2 hours
localhost:5000/stable/aerospike:0.2.1                   aerospike               0.2.1   4aff638 3.7 KiB         2 hours
localhost:5000/stable/airflow:0.13.0                    airflow                 0.13.0  c46cc43 28.1 KiB        2 hours
localhost:5000/stable/anchore-engine:0.10.0             anchore-engine          0.10.0  3f3dcd7 34.3 KiB        2 hours

导出chart到目录
helm chart export localhost:5000/myrepo/mychart:2.7.0
ref:     localhost:5000/myrepo/mychart:2.7.0
digest:  1b251d38cfe948dfc0a5745b7af5ca574ecb61e52aed10b19039db39af6e1617
size:    2.4 KiB
name:    mychart
version: 0.1.0
Exported chart to mychart/



推送chart到远程
 helm chart push localhost:5000/myrepo/mychart:2.7.0
The push refers to repository [localhost:5000/myrepo/mychart]
ref:     localhost:5000/myrepo/mychart:2.7.0
digest:  1b251d38cfe948dfc0a5745b7af5ca574ecb61e52aed10b19039db39af6e1617
size:    2.4 KiB
name:    mychart
version: 0.1.0
2.7.0: pushed to remote (1 layer, 2.4 KiB total)


从缓存中移除chart
helm chart remove localhost:5000/myrepo/mychart:2.7.0
2.7.0: removed


从远程拉取chart
helm chart pull localhost:5000/myrepo/mychart:2.7.0
2.7.0: Pulling from localhost:5000/myrepo/mychart
ref:     localhost:5000/myrepo/mychart:2.7.0
digest:  1b251d38cfe948dfc0a5745b7af5ca574ecb61e52aed10b19039db39af6e1617
size:    2.4 KiB
name:    mychart
version: 0.1.0
Status: Downloaded newer chart for localhost:5000/myrepo/mychart:2.7.0


使用上述命令存储的chart会被缓存到文件系统中。

OCI 镜像设计规范 严格遵守文件系统布局的。例如：

$ tree ~/Library/Caches/helm/
/Users/myuser/Library/Caches/helm/
└── registry
    ├── cache
    │   ├── blobs
    │   │   └── sha256
    │   │       ├── 1b251d38cfe948dfc0a5745b7af5ca574ecb61e52aed10b19039db39af6e1617
    │   │       ├── 31fb454efb3c69fafe53672598006790122269a1b3b458607dbe106aba7059ef
    │   │       └── 8ec7c0f2f6860037c19b54c3cfbab48d9b4b21b485a93d87b64690fdb68c2111
    │   ├── index.json
    │   ├── ingest
    │   └── oci-layout
    └── config.json
    
    
index.json示例， 包含了所有的Helm chart manifests的参考：

$ cat ~/Library/Caches/helm/registry/cache/index.json  | jq
{
  "schemaVersion": 2,
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:31fb454efb3c69fafe53672598006790122269a1b3b458607dbe106aba7059ef",
      "size": 354,
      "annotations": {
        "org.opencontainers.image.ref.name": "localhost:5000/myrepo/mychart:2.7.0"
      }
    }
  ]
}
```

从经典 [chart 仓库](https://helm.sh/zh/docs/topics/chart_repository) （基于index.yaml的仓库） 尽量简单地 `helm fetch` (Helm 2 CLI), `helm chart save`, `helm chart push`。



# 部署ELK









## 获取es

elasticsearch 7.10.2版本

![image-20210205165231883](http://myapp.img.mykernel.cn/image-20210205165231883.png)

```bash
export https_proxy=http://192.168.0.33:808
export https_proxy=http://192.168.0.33:808
export no_proxy=172.16.0.0/16,127.0.0.0/8

# helm repo add elastic https://helm.elastic.co

# 查看repo
[root@master repository]# helm repo list
NAME          	URL                                       
jenkins       	https://charts.jenkins.io                 
wyrihaximusnet	https://helm.wyrihaximus.net/             
groundhog2k   	https://groundhog2k.github.io/helm-charts/
elastic       	https://helm.elastic.co       


# #查看chart
[root@master repository]# helm search repo elastic
NAME                     	CHART VERSION	APP VERSION	DESCRIPTION                                       
elastic/elasticsearch    	7.10.2       	7.10.2     	Official Elastic helm chart for Elasticsearch     
groundhog2k/elasticsearch	0.1.1        	7.10.1     	A Helm chart for Elasticsearch on Kubernetes      
elastic/apm-server       	7.10.2       	7.10.2     	Official Elastic helm chart for Elastic APM Server
elastic/eck-operator     	1.3.1        	1.3.1      	A Helm chart for deploying the Elastic Cloud on...
elastic/eck-operator-crds	1.3.1        	1.3.1      	A Helm chart for installing the ECK operator Cu...
elastic/filebeat         	7.10.2       	7.10.2     	Official Elastic helm chart for Filebeat          
elastic/kibana           	7.10.2       	7.10.2     	Official Elastic helm chart for Kibana            
elastic/logstash         	7.10.2       	7.10.2     	Official Elastic helm chart for Logstash          
elastic/metricbeat       	7.10.2       	7.10.2     	Official Elastic helm chart for Metricbeat        



#fetch chart
[root@master repository]# helm fetch elastic/elasticsearch

[root@master repository]# ls ~/.cache/helm/repository/
elasticsearch-7.10.2.tgz


# 展开
[root@master repository]# tar xvf elasticsearch-7.10.2.tgz 
elasticsearch/values.yaml # 配置存储
persistence:
  enabled: false
esJavaOpts: "-Xmx200m -Xms200m"
resources:
  requests:
    cpu: "200m"
    memory: "200Mi"
  limits:
    cpu: "500m"
    memory: "500Mi"


```

```bash
# kubectl create ns elastic

# helm install elastic/elasticsearch -n elastic -f ./elasticsearch/values.yaml --generate-name

[root@master repository]# helm list -n elastic
NAME                    	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART               	APP VERSION
elasticsearch-1612517181	elastic  	1       	2021-02-05 17:26:25.838713732 +0800 CST	deployed	elasticsearch-7.10.2	7.10.2     
[root@master repository]# kubectl get pods -n elastic
NAME                     READY   STATUS     RESTARTS   AGE
elasticsearch-master-0   0/1     Init:0/1   0          12s
elasticsearch-master-1   0/1     Init:0/1   0          12s
elasticsearch-master-2   0/1     Init:0/1   0          12s


[root@node01 ~]# kubectl get pods -n elastic -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE                NOMINATED NODE   READINESS GATES
elasticsearch-master-0   1/1     Running   0          16m   10.244.0.67   master.magedu.com   <none>           <none>
elasticsearch-master-1   0/1     Running   0          16m   10.244.3.59   node03.magedu.com   <none>           <none>
elasticsearch-master-2   1/1     Running   0          16m   10.244.1.50   node01.magedu.com   <none>           <none>


# 获取service
[root@node01 ~]# kubectl get svc -n elastic
NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
elasticsearch-master            ClusterIP   10.106.195.188   <none>        9200/TCP,9300/TCP   16m
elasticsearch-master-headless   ClusterIP   None             <none>        9200/TCP,9300/TCP   16m

 
# 访问
[root@master ~]# curl elasticsearch-master.elastic.svc.cluster.local:9200
{
  "name" : "elasticsearch-master-2",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "Ehpxb1iBQNah5ysd5-b96Q",
  "version" : {
    "number" : "7.10.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "747e1cc71def077253878a59143c1f785afa92b9",
    "build_date" : "2021-01-13T00:42:12.435326Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

```



## 获取fluentd

fluentd 版本和es版本无所谓

https://artifacthub.io/packages/helm/kokuwa/fluentd-elasticsearch

> 以上elasticsearch关联的位置

![image-20210205175712125](http://myapp.img.mykernel.cn/image-20210205175712125.png)

默认配置可以收集每个节点及容器日志

fluentd 收集节点自已日志、容器日志 -> es-client ->  master索引后保存data

部署时告诉es的集群入口



daemonset

```bash
helm repo add kokuwa https://kokuwaio.github.io/helm-charts
#helm repo update
#helm repo list
helm search repo kokuwa

# 获取安装包
helm fetch kokuwa/fluentd-elasticsearch
helm inspect readme kokuwa/fluentd-elasticsearch

[root@master ~]# ls .cache/helm/repository
fluentd-3.5.1.tgz


# 展开
[root@master ~]# cd .cache/helm/repository
[root@master repository]# tar xvf fluentd-elasticsearch-11.7.2.tgz
fluentd-elasticsearch/values.yaml
elasticsearch:
  auth:
    enabled: false
    user: null
    password: null
  includeTagKey: true
  setOutputHostEnvVar: true
  # If setOutputHostEnvVar is false this value is ignored
  hosts: ["elasticsearch-master.elastic.svc.cluster.local:9200"]


helm install fluentd bitnami/fluentd -n elastic --version 3.5.1 -f fluentd/values.yaml
```

系统资源实在不足，后续在做



## 获取kibana

kibana版本一定要和elasticsearch版本一致, 7.10.xx

![image-20210205165313345](http://myapp.img.mykernel.cn/image-20210205165313345.png)

kibana联系master, data 展示数据

惟一修改kibana服务类型







# 部署gitlab-runner

https://docs.gitlab.com/runner/install/kubernetes.html

https://docs.gitlab.com/runner/install/kubernetes.html#configuring-gitlab-runner-using-the-helm-chart

```bash
helm repo add gitlab https://charts.gitlab.io

root@ccea447f1caa:~# helm repo update

root@ccea447f1caa:~# helm search repo gitlab-runner
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /root/.kube/config
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /root/.kube/config
NAME                	CHART VERSION	APP VERSION	DESCRIPTION  
gitlab/gitlab-runner	0.28.0       	13.11.0    	GitLab Runner



helm fetch gitlab/gitlab-runner



cd ~/.cache/helm/repository
[root@izbp1canomrhu2o9bo8mjsz repository]# tar xvf gitlab-runner-0.28.0.tgz 

```

编辑values.yaml

```	diff
[root@izbp1canomrhu2o9bo8mjsz repository]# diff -u /usr/local/src/gitlab-runner/values.yaml gitlab-runner/values.yaml 
--- /usr/local/src/gitlab-runner/values.yaml	2021-04-21 02:34:20.000000000 +0800
+++ gitlab-runner/values.yaml	2021-04-29 13:39:26.808266823 +0800
@@ -105,7 +105,7 @@
 
 ## For RBAC support:
 rbac:
-  create: false
+  create: true
 
   ## Define specific rbac permissions.
   ## DEPRECATED: see .Values.rbac.rules
@@ -117,12 +117,12 @@
   ## - apiGroups: default "" (indicates the core API group) if missing or empty.
   ## - resources: default "*" if missing or empty.
   ## - verbs: default "*" if missing or empty.
-  rules: []
-  # - resources: ["pods", "secrets"]
-  #   verbs: ["get", "list", "watch", "create", "patch", "delete"]
-  # - apiGroups: [""]
-  #   resources: ["pods/exec"]
-  #   verbs: ["create", "patch", "delete"]
+  rules: 
+   - resources: ["pods", "secrets"]
+     verbs: ["get", "list", "watch", "create", "patch", "delete"]
+   - apiGroups: [""]
+     resources: ["pods/exec"]
+     verbs: ["create", "patch", "delete"]
 
   ## Run the gitlab-bastion container with the ability to deploy/manage containers of jobs
   ## cluster-wide or only within namespace
@@ -130,7 +130,7 @@
 
   ## Use the following Kubernetes Service Account name if RBAC is disabled in this Helm chart (see rbac.create)
   ##
-  # serviceAccountName: default
+  serviceAccountName: default
 
   ## Specify annotations for Service Accounts, useful for annotations such as eks.amazonaws.com/role-arn
   ##
@@ -369,7 +369,7 @@
 
   ## Service Account to be used for runners
   ##
-  # serviceAccountName:
+  serviceAccountName: default
 
   ## If Gitlab is not reachable through $CI_SERVER_URL
   ##

```

```bash
helm install --namespace jenkins gitlab-runner ./gitlab-runner  -f ./gitlab-runner/values.yaml 
```

## 配置runner

https://docs.gitlab.com/ee/ci/runners/

要为项目启用或禁用特定运行器：

1. 转到项目的**“设置”>“ CI / CD”，**然后展开“**跑步者”**部分。
2. 单击**“为此项目启用”**或“**对此项目****禁用”**

```bash
helm upgrade --namespace jenkins gitlab-runner         --set gitlabUrl=https://graspgit.graspyun.com/,runnerRegistrationToken=n-NxHzWP6  gitlab/gitlab-runner -f ./gitlab-runner/values.yaml
```

