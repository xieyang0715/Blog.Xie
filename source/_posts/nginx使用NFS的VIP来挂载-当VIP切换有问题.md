---
title: 'nginx使用NFS的VIP来挂载, 当VIP切换有问题'
date: 2020-12-03 08:15:00
tags: 
- 马哥企业教练
- nginx
toc: true 
---

我的nfs使用keepalived，当停止nfs vip就迁移, rsync+inotify就停止，另一个keepalived 拿到vip, 启动同步。

但是客户端挂载的nfs路径就出现df: "/apps/nginx/html/wp-content/uploads": 失效文件句柄

- 脚本

<!--more-->

#  搞个脚本自动检测

```bash
如果挂载出现问题了就重新mount一下。我给你写了个 
for i in `seq 30`;do
  df -Th &> /dev/null
  if [ `echo $?` -ne 0 ];then
    umount -lf /mnt && mount -t nfs xxx:/test_nfs /mnt
  fi
  sleep 2
done
```



# autofs

```bash
# /etc/auto.master

/- /etc/nfs.misc # 不与挂载点关联


# /etc/nfs.misc
/apps/nginx/html/wp-content/uploads -fstype=nfs,rw 192.168.101.9:/data/k8s_storage

# systemctl enable --now autofs
# systemctl restart autofs


# vip切换还是有问题
```

