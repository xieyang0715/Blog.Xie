---
title: windows一键启动kubernetes
date: 2021-09-06 09:44:34
tags:
- kubernetes
- docker
---



# 前言

今天浏览kubernetes官方项目下，有一个[minikube](https://github.com/kubernetes/minikube), 可以在maxos, linux, windows本地运行kubernetes, 主要用来开发，并且支持了所有k8s特性。



<!--more-->
# 安装

https://minikube.sigs.k8s.io/docs/start/

win: https://storage.googleapis.com/minikube/releases/latest/minikube-installer.exe

linux:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

deb:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

rpm:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -Uvh minikube-latest.x86_64.rpm
```

# 启动

```bash
setx  MINIKUBE_HOME "G:\minikube" # 文件大，如果放在C盘，C盘很快就满了。 桌面就卡死了。
minikube start --image-mirror-country='cn' --base-image="anjone/kicbase"  --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'
```

> 管理 员权限

# 使用

```bash
kubectl get po -A
```

dashboard

```bash
minikube dashboard
```

部署应用在8080端点

```bash
kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.4
kubectl expose deployment hello-minikube --type=NodePort --port=8080
```

访问8080

```bash
kubectl get services hello-minikube
minikube service hello-minikube
```

使用kubectl转发端口

```bash
kubectl port-forward service/hello-minikube 7080:8080
```

> Your application is now available at http://localhost:7080/

# 集群管理

Pause Kubernetes without impacting deployed applications:

```shell
minikube pause
```

Unpause a paused instance:

```shell
minikube unpause
```

Halt the cluster:

```shell
minikube stop
```

Increase the default memory limit (requires a restart):

```shell
minikube config set memory 16384
```

Browse the catalog of easily installed Kubernetes services:

```shell
minikube addons list
```

Create a second cluster running an older Kubernetes release:

```shell
minikube start -p aged --kubernetes-version=v1.16.1
```

Delete all of the minikube clusters:

```shell
minikube delete --all
```

# 排错

当启动失败时

```bash
minikube delete
```

> 并且删除用户目录下的minikube/cache, cert

