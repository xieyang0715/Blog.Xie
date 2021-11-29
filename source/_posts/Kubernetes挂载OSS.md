---
title: Kubernetes挂载OSS
date: 2021-11-02 11:49:02
tags:
- kubernetes
- OSS
photos:
- https://ucc.alicdn.com/pic/developer-ecology/827b01acd90341d096b462f5d2dc07b0.png
---

# 前言

容器中非日志文件需要拷贝出来，如果使用`kubectl cp` 会中断，因为文件太大了，故将OSS挂载到pod


> 引用自：https://www.alibabacloud.com/help/zh/doc-detail/130911.htm

不同的驱动，需要集群事先安装插件，控制台会提示。手工yaml, 就需要注意。

<!--more-->
# 挂载OSS到本地文件系统 

> https://partners-intl.aliyun.com/help/doc-detail/153892.htm?spm=a2c63.p38356.b99.184.212865d08TydXr

如果挂载到一个普通用户运行的容器上时，需要编辑yaml

```yaml
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: myapp
    spec:
      containers:
      - command:
        - cat
        image: nginx
        imagePullPolicy: Always
        name: nginx
        tty: true
        volumeMounts:
        - mountPath: /data/db
          name: db
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        fsGroupChangePolicy: "OnRootMismatch"
```

> OnRootMismatch 避免挂载慢

编辑oss

```diff
  csi:
    driver: ossplugin.csi.alibabacloud.com
    nodePublishSecretRef:
      name: ishop-pos-deploy-dump
      namespace: default
    volumeAttributes:
      bucket: ishop-prod-dump-files
+      otherOpts: '-o uid=1000 -o gid=1000 -o allow_other -o umask=022'
```

> 挂载目录对应的`UID`, `GID`, 允许普通用户挂载，挂载后的掩码

PV是不可变的资源，当修改选项时，需要重建PV

# OSS存储不重要的文件

需要定期自动清除，不然占用空间

![image-20211109162217337](http://myapp.img.mykernel.cn/image-20211109162217337.png)

