---
title: 文件系统-cephfs
date: 2021-04-06 05:37:53
---



# cephfs

与rbd不同，cephfs需要额外的mds守护进程。

用户与ceph通信：数据直接写入datapool,    元数据与mds通信，mds再将元数据写入metadatapool，所以mds需要高可用。

高可用mds: 2个rank, 每个rank一个reply备用.



配置cephfs请参考：[提供cephfs接口](http://blog.mykernel.cn/2021/02/24/ceph%E7%B3%BB%E7%BB%9F%E9%83%A8%E7%BD%B2/#%E6%8F%90%E4%BE%9Bcephfs%E6%8E%A5%E5%8F%A3)

- [x] 启动cephfs的守护进程mds
- [x] 创建存储池
- [x] 高可用mds



# 客户端挂载CephFS

- [x] 挂载文件系统cephfs
- [x] 用户空间文件系统fuse



## 内核文件系统

### ceph模块

```bash
root@ceph-client:~# apt install -y ceph-common

root@ceph-client:~# modinfo ceph
filename:       /lib/modules/4.15.0-55-generic/kernel/fs/ceph/ceph.ko

```

### 授权用户

 由于用户写mds和cephfs-data池，所以仅授权如下

```bash
[root@ceph-admin01 ~]# ceph osd pool ls
device_health_metrics
mypool
rbdpool
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
cephfs-metadata
cephfs-data # 数据池
kube
mykernel
mykernelpool

[cephadm@ceph-admin01 ceph-cluster]$ ceph auth get-or-create client.fsclient mon 'allow r' mds 'allow rw' osd 'allow rwx pool=cephfs-data' -o ceph.client.fsclient.keyring

[cephadm@ceph-admin01 ceph-cluster]$ cat ceph.client.fsclient.keyring
[client.fsclient]
	key = AQChAGxge/YfExAAu/n955VgExywCyGRK1BNfQ==


```

> mds只需要allow权限，allow rw一样的。
>
> allow rwx 相当于allow *
>
> 保存结果为key, 一般客户端仅需要这个

### 保存授权用户的key

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph auth print-key client.fsclient
AQChAGxge/YfExAAu/n955VgExywCyGRK1BNfQ==[cephadm@ceph-admin01 ceph-cluster]$ 
[cephadm@ceph-admin01 ceph-cluster]$ 
[cephadm@ceph-admin01 ceph-cluster]$ ceph-authtool -p -n client.fsclient ceph.client.fsclient.keyring 
AQChAGxge/YfExAAu/n955VgExywCyGRK1BNfQ==
[cephadm@ceph-admin01 ceph-cluster]$ 

[cephadm@ceph-admin01 ceph-cluster]$ ceph auth print-key client.fsclient > fsclient.key

```

### 分发授权用户至客户端

```bash
[cephadm@ceph-admin01 ceph-cluster]$ scp fsclient.key ceph.conf root@172.18.199.100:/etc/ceph
root@ceph-client:~# ls -l /etc/ceph/
total 12
-rw-r--r-- 1 root root 716 Apr  6 14:37 ceph.conf #    配置
-rw-r--r-- 1 root root  40 Apr  6 14:37 fsclient.key # 认证
-rw-r--r-- 1 root root  92 Dec  8 02:15 rbdmap

```

### 挂载

```bash
root@ceph-client:~# 
root@ceph-client:~# install -dv /data

root@ceph-client:~# mount -t ceph 172.29.0.11:6789,172.29.0.12:6789,172.29.0.13:6789:/ /data -o name=fsclient,secretfile=/etc/ceph/fsclient.key 

root@ceph-client:~# df -TH /data
Filesystem                                           Type  Size  Used Avail Use% Mounted on
172.29.0.11:6789,172.29.0.12:6789,172.29.0.13:6789:/ ceph  331G     0  331G   0% /data

root@ceph-client:~# stat -f /data
  File: "/data"
    ID: 1695f30626a78b51 Namelen: 255     Type: ceph # Ceph
Block size: 4194304    Fundamental block size: 4194304
Blocks: Total: 78767      Free: 78767      Available: 78767
Inodes: Total: 0          Free: -1

```

### 开机自动挂载

```bash
# /etc/fstab
 172.29.0.11:6789,172.29.0.12:6789,172.29.0.13:6789:/ /data ceph name=fsclient,secretfile=/etc/ceph/fsclient.key,_netdev,noatime 0 0                
```

```bash
root@ceph-client:~# umount /data
root@ceph-client:~# mount -a
root@ceph-client:~# df -TH /data
Filesystem                                           Type  Size  Used Avail Use% Mounted on
172.29.0.11:6789,172.29.0.12:6789,172.29.0.13:6789:/ ceph  331G     0  331G   0% /data
```

```bash
root@ceph-client:~# find /usr/share/ -name '*.png' -exec cp {} /data \;
```



### [性能测试](http://blog.mykernel.cn/2021/03/03/kvm-io%E6%B5%8B%E8%AF%95/)

![image-20210406151248515](http://myapp.img.mykernel.cn/image-20210406151248515.png)

## 用户空间文件系统(fuse)

比内核空间挂载性能更差

### fuse包

```bash
root@ceph-client:~# apt install -y ceph-fuse
```

### 准备keyring

```bash
[cephadm@ceph-admin01 ceph-cluster]$ scp ceph.client.fsclient.keyring  ceph.conf root@172.18.199.100:/etc/ceph
```



### 挂载

```bash
root@ceph-client:~# ceph-fuse -n client.fsclient -m 172.29.0.11:6789,172.29.0.12:6789,172.29.0.13:6789 /data 
ceph-fuse[11070]: starting ceph client2021-04-06 14:46:25.876278 7fd4c1f7e680 -1 init, newargv = 0x55cedfe463a0 newargc=9

ceph-fuse[11070]: starting fuse

```

```bash
root@ceph-client:~# df -TH /data
Filesystem     Type            Size  Used Avail Use% Mounted on
ceph-fuse      fuse.ceph-fuse  331G     0  331G   0% /data
root@ceph-client:~# stat -f /data
  File: "/data"
    ID: 0        Namelen: 255     Type: fuseblk
Block size: 4194304    Fundamental block size: 4194304
Blocks: Total: 78766      Free: 78766      Available: 78766
Inodes: Total: 1          Free: 0
```



### 开机自动挂载

```bash
# /etc/fstab
none /data fuse.ceph  ceph.id=fsclient,ceph.conf=/etc/ceph/ceph.conf,_netdev,defaults 0  0
```

```bash
root@ceph-client:~# mount -a
ceph-fuse[11110]: starting ceph client
2021-04-06 14:48:22.398510 7f564d33a680 -1 init, newargv = 0x55f0a9903b80 newargc=11
ceph-fuse[11110]: starting fuse
```



```bash
root@ceph-client:~# find /usr/share/ -name '*.xml'  -exec cp {} \data \;
```



### [性能测试](http://blog.mykernel.cn/2021/03/03/kvm-io%E6%B5%8B%E8%AF%95/)

![image-20210406150403757](http://myapp.img.mykernel.cn/image-20210406150403757.png)