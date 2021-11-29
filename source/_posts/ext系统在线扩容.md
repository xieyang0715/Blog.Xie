---
title: ext系统在线扩容
date: 2021-04-06 03:30:23
tags:
- kvm
---



先快照镜像后扩容

```bash
apt install -y cloud-guest-utils
#vda后面有空格
growpart /dev/vda 1
#vda后面无空格
resize2fs /dev/vda1
```

