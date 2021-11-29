---
title: ceph dashboard及prometheus监控
date: 2021-04-07 01:29:35
---

# ceph启用dashboard v2

```bash
[root@ceph-admin01 ~]# ceph mgr module ls | grep name
# 新版没有dashboard插件
```



# ceph启用promethes

```bash
[root@ceph-admin01 ~]# ceph mgr module enable prometheus

#mgr节点
lsof -ni:9283
lsof -ni:9283
```

