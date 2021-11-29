---
title: df查看文件系统容量
date: 2020-11-16 09:41:39
tags: 日常挖掘
---

```bash
df --out=fstype,ipcent,pcent,target -x tmpfs -x devtmpfs -x overlay

# target 挂载点
# pcent  使用百分比
# ipcent inode使用百分比
# fstype 文件系统类型
# -x 是排除文件系统
```



```bash
df -h --out=fstype,size,used,pcent,ipcent,target
类型      容量  已用 已用% 已用(I)% 挂载点
devtmpfs  956M     0    0%       1% /dev
tmpfs     198M   12M    6%       2% /run
ext4       40G   20G   53%       5% /
tmpfs     986M     0    0%       1% /dev/shm
tmpfs     5.0M     0    0%       1% /run/lock
tmpfs     986M     0    0%       1% /sys/fs/cgroup
tmpfs     198M     0    0%       1% /run/user/0
overlay    40G   20G   53%       5% /var/lib/docker/overlay2/ad6b2c05d58904ce39ca91d102bcb3f502edce98312fbd10097d99cb73985965/merged

```

