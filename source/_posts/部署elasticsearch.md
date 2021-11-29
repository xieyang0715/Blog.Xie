---
title: 部署elasticsearch
date: 2020-12-23 08:18:29
tags:
- elk
---

 elasticsearch-7.6.1-x86_64.rpm  

通常3节点部署集群，实验环境可以单机部署。

```bash
[root@k8s-master1 elk]# cat /etc/elasticsearch/elasticsearch.yml
# 3个节点一致
cluster.name: weizhixiu
# 节点名，每个节点一致。单机部署时此节点名必须hosts解析
node.name: node-1
path.data: /mnt/elasticsearch/data
path.logs: /mnt/elasticsearch/log
bootstrap.memory_lock: true
network.host: 0.0.0.0
http.port: 9200
# 单机部署时，节点名必须解析，此处填解析的名
discovery.seed_hosts: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
cluster.initial_master_nodes: ["172.16.0.222", "172.16.0.223","172.16.0.224"]

gateway.recover_after_nodes: 2
action.destructive_requires_name: true
# 跨域
http.cors.enabled: true
http.cors.allow-origin: "*"
```



unit

```bash
# /usr/lib/systemd/system/elasticsearch.service
LimitMEMLOCK=infinity
```



资源限制

```bash
# /etc/security/limits.conf 
elasticsearch soft nofile 65535
elasticsearch hard nofile 65535
```

