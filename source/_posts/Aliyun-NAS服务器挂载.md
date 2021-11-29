---
title: Aliyun NAS服务器挂载
date: 2020-12-04 02:24:58
tags: 日常挖掘
---


今天在aliyun看到这个位置可以学习一些ecs使用操作，以下记录了nas怎么挂载。
https://developer.aliyun.com/plan/promotion/1?spm=a2c6h.13651102.1364563.40.3e221b11ldnwk4



```bash
# nfs
yum install nfs-utils
# 内核模块
sudo echo "options sunrpc tcp_slot_table_entries=128" >>  /etc/modprobe.d/sunrpc.conf 
sudo echo "options sunrpc tcp_max_slot_table_entries=128" >>  /etc/modprobe.d/sunrpc.conf
# nfs挂载
sudo mount -t nfs -o vers=3,nolock,proto=tcp,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport 3de2c4a777-wue93.cn-shanghai.nas.aliyuncs.com:/ /mnt
# 检验
df -h | grep aliyun
```