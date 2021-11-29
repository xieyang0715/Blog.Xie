---
title: kubernetes集群升级迁移
date: 2021-09-16 15:24:23
tags:
- kubernetes
- 个人日记
---

# 前言
<!--more-->

今天我们部分集群快到期了，通过阿里云获取到ACK的升级文档。

过期公告：https://help.aliyun.com/document_detail/213516.html

https://help.aliyun.com/document_detail/86497.htm?spm=a2c4g.11186623.0.0.336b5adaf0npIr#task-1664343

由于升级过程难免会出现问题，导致网络不通，为了避免这个问题，我们采用**蓝绿部署**的方式升级。

优势：

- 有问题，可以立即回滚。而不是升级集群，如果有些k8s资源在升级完成不可用，肯定会导致用户不可用，现在再回滚肯定来不及了。
- 手工蓝绿部署，可以控制很小一部分的流量到达新版本，测试新版本是否正常。让用户影响降到最低。

<!--more-->
# 大体流程

![集群迁移](http://myapp.img.mykernel.cn/集群迁移.png)
