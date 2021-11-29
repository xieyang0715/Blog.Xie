---
title: 获取systemd服务的日志
date: 2021-02-07 07:09:08
tags: 生产小经验
---









```bash
# 终端1
systemctl restart kube-apiserver



# 终端2
journalctl -u kube-apiserver --since="$(date -d"-10 second" +"%F %T")"
```

