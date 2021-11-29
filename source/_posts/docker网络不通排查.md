---
title: docker网络不通排查
date: 2021-06-29 09:02:11
tags:
- 生产小经验
- docker
---

# 前言

今天遇到一个奇怪的问题，主机端口正常，docker端口不正常。

在docker宿主机正常，外部不正常。



<!--more-->
# 查看防火墙

```bash
iptables -vnL

# INPUT 空，没有禁止
# FORWARD DROP，被禁止转发了。
```

解决

```bash
iptables -t filter -P FORWARD ACCEPT
```

# 查看转发内核参数

```bash
[root@dev-ops ~]# sysctl  net.ipv4.ip_forward
net.ipv4.ip_forward = 0
```

解决

```bash
[root@dev-ops ~]# cat /etc/sysctl.d/docker.conf 
net.ipv4.ip_forward = 1
[root@dev-ops ~]# sysctl -p /etc/sysctl.d/docker.conf
net.ipv4.ip_forward = 1
```

