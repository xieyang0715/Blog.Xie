---
title: rancher安装及部署
tags:
  - kubernetes
photos:
  - http://myapp.img.mykernel.cn/rancher-logo-horiz-color.png
date: 2021-11-24 22:22:53
---

# 前言

公司有一个老项目是rancher管理的，突然rancher的UI不能访问了，通过日志获取到rancher证书已经过期。网上说rancher新版本可以自动更新证书，而当前应该是旧版本。不清楚rancher的原理，不能随便停止rancher。

故通过以下方式来学习

[官方文档](https://github.com/rancher/rancher)

[介绍视频](https://www.bilibili.com/video/BV194411p7mL?from=search&seid=8773559222818409749&spm_id_from=333.337.0.0)

<!--more-->

# 介绍

rancher管理集群通过agent来拉起集群或管理集群

1. 添加现有集群。
2. 自定义集群。
3. 通过云提供商的驱动来启动集群。

rancher只是一个管理界面，rancher停止并不影响k8s集群。



选择rancher版本:   https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-4-9/

启动rancher:

```bash
docker run -d --restart=unless-stopped -p 80:80 -p 443:443 --privileged rancher/rancher
```

rancher停止后，如何恢复？

- 自己停止了，启动。
- 过期了，会自己更新证书。
- import的状态数据丢了，就删除目标集群的cattle-system下的东西, 然后重新导入即可。

