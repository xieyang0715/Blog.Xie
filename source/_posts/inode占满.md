---
title: inode占满
date: 2021-02-22 06:25:41
tags: 
- 生产小经验
---



https://help.aliyun.com/knowledge_detail/42531.html

```bash
[root@k8s-master1 ~]# df -THi
Filesystem     Type     Inodes IUsed IFree IUse% Mounted on
devtmpfs       devtmpfs   2.1M   371  2.1M    1% /dev
tmpfs          tmpfs      2.1M     1  2.1M    1% /dev/shm
tmpfs          tmpfs      2.1M   767  2.1M    1% /run
tmpfs          tmpfs      2.1M    16  2.1M    1% /sys/fs/cgroup
/dev/vda1      ext4       2.7M  217k  2.5M    9% /
/dev/vdb1      ext4        33M   27M  6.3M   82% /mnt
```

查看僵尸文件占用

```bash
lsof |grep delete | more
```

```bash
# 停止文件对应的进程
systemctl restart elasticsearch
```

查看挂载目录

```bash
[root@k8s-master1 log]# for i in /mnt/*; do echo $i; find $i | wc -l; done
```

```bash
# https://github.com/jhouze/inodes
[root@k8s-master1 mnt]# python3 inodes3.py 
Total inodes is: 52980499

------------!!No large inode using directories found!!-----------

---Locating directories holding more than 10.00% of total inodes----
/mnt/docker/lib/overlay2/1bcb8ff093ebf047339f61d1cddb4e097e3c090434a97f3cef4ca768e90bb6e8/diff/tmp 26428287 49.9%
/mnt/docker/lib/overlay2/1bcb8ff093ebf047339f61d1cddb4e097e3c090434a97f3cef4ca768e90bb6e8/merged/tmp 26428287 49.9%

-----Largest inode usage directories at the script's target-----
/mnt/docker 52976047
/mnt/elasticsearch 3305
/mnt/weizhixiu 1132
/mnt/harbor_data 9
/mnt/lost+found 0

```

```bash
[root@k8s-master1 mnt]# tree /mnt/docker/lib/overlay2/1bcb8ff093ebf047339f61d1cddb4e097e3c090434a97f3cef4ca768e90bb6e8/ -L 2
/mnt/docker/lib/overlay2/1bcb8ff093ebf047339f61d1cddb4e097e3c090434a97f3cef4ca768e90bb6e8/
├── diff
│   ├── etc
│   └── tmp
├── link
├── lower
├── merged
│   ├── bin -> usr/bin
│   ├── boot
│   ├── clair
│   ├── config
│   ├── dev
│   ├── docker-entrypoint.sh
│   ├── dumb-init
│   ├── etc
│   ├── harbor
│   ├── home
│   ├── lib -> usr/lib
│   ├── lib64 -> usr/lib
│   ├── media -> run/media
│   ├── mnt
│   ├── proc
│   ├── root
│   ├── run
│   ├── sbin -> usr/sbin
│   ├── srv -> var/srv
│   ├── sys
│   ├── tmp
│   ├── usr
│   └── var
└── work
    └── work

27 directories, 4 files

[root@k8s-master1 mnt]# cat /mnt/docker/lib/overlay2/1bcb8ff093ebf047339f61d1cddb4e097e3c090434a97f3cef4ca768e90bb6e8/merged/docker-entrypoint.sh 
#!/bin/bash
set -e

/harbor/install_cert.sh
sudo -E -H -u \#10000 sh -c "/dumb-init -- /clair/clair -config /etc/clair/config.yaml $*"
set +e

```

> 所以清理harbor镜像

```bash
docker-compose down

docker container prune

#删除所有docker-compose容器
#core容器删除半天
```

恢复

```bash
[root@k8s-master1 harbor]# df -THi
Filesystem     Type     Inodes IUsed IFree IUse% Mounted on
devtmpfs       devtmpfs   2.1M   372  2.1M    1% /dev
tmpfs          tmpfs      2.1M     1  2.1M    1% /dev/shm
tmpfs          tmpfs      2.1M   610  2.1M    1% /run
tmpfs          tmpfs      2.1M    16  2.1M    1% /sys/fs/cgroup
/dev/vda1      ext4       2.7M  216k  2.5M    9% /
tmpfs          tmpfs      2.1M     1  2.1M    1% /run/user/0
/dev/vdb1      ext4        33M  9.1M   24M   28% /mnt
```

重新为harbor生成证书，略

