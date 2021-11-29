---
title: k8s pods 137退出状态码
date: 2020-12-11 10:42:56
tags: 生产小经验
---

今天启动pod，一上线就崩溃，pod的日志提示

command terminated with exit code 137  # 内存分配不足

```bash
# kubectl describe pods -n namespace pod
state: Error
```

pod的yaml修改请求内存资源如下: 

        resources:
          limits:
            cpu: 2
            memory: "2048Mi" # 修改内存
          requests:
            cpu: 1
            memory: "1024Mi" # 修改内存
<!--more-->

137就是内存分配不足，原来是最初pod启动，为了节省内存, 写的deployment里面的请求的内存50m, 最大1024m. 

```bash
kind: Deployment
#apiVersion: extensions/v1beta1
apiVersion: apps/v1
metadata:
spec:
  replicas: 1
  selector:
  revisionHistoryLimit: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate

  template:
    metadata:
      labels:
    spec:
      imagePullSecrets: 
      volumes:
      containers:
      - name: weizhixiu-jar-MICROSERVICE-container
        livenessProbe:
        readinessProbe:
        lifecycle: 
        image: 
        imagePullPolicy: Always
        volumeMounts:
        ports:
        resources:
          limits:
            cpu: 2
            memory: "2048Mi" # 修改内存
          requests:
            cpu: 1
            memory: "1024Mi" # 修改内存
```






参考：https://imroc.io/posts/kubernetes/analysis-exitcode/
