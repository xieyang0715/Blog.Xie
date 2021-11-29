---
title: 块存储(rdb)
date: 2021-04-01 07:43:28
---



# 为什么使用rbd

ceph集群通过librados库输出访问接口，rbd, rgw; cephfs+mds是集群组件的一部分。

存储rbd数据，都切分为对象数据，rbd是最成熟的客户端 。



# 使用rbd的方式

rbd磁盘给linux主机使用，linux主机需要ceph-common包，ceph命令加载ceph.ko内核模块，拥有ceph相应的权限就连接rbd，检索镜像，映射为一块本地的磁盘使用。

kvm虚拟机，通过libvirt访问ceph。



# ceph rbd管理

## 环境准备

### 查看镜像池

镜像属于某个pool

```bash
[root@ceph-admin01 ~]# su - cephadm
上一次登录：四 4月  1 14:03:37 CST 2021pts/0 上
[cephadm@ceph-admin01 ~]$ cd ceph-cluster/
[cephadm@ceph-admin01 ceph-cluster]$ ls
ceph.bootstrap-mds.keyring  ceph.bootstrap-mgr.keyring  ceph.bootstrap-osd.keyring  ceph.bootstrap-rgw.keyring  ceph.client.admin.keyring  ceph.client.kube.keyring  ceph.conf  ceph-deploy-ceph.log  ceph.mon.keyring  cluster.keyring
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool ls
device_health_metrics
mypool
rbdpool
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
cephfs-metadata
cephfs-data
kube

```

### 创建池

准备将来给kubernetes使用的存储池

```diff
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool create mykernelpool 64 64
pool 'mykernelpool' created
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool ls
device_health_metrics
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
cephfs-metadata
cephfs-data
+mykernelpool
```



### 启动池的rbd并初始化

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool application enable mykernelpool rbd
enabled application 'rbd' on pool 'mykernelpool'
```

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rbd pool init mykernelpool
[cephadm@ceph-admin01 ceph-cluster]$ 
```

## 镜像管理

### 查看镜像

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rbd ls -l --pool mykernelpool
```

### 创建镜像

#### 指定选项方式

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rbd create --pool mykernelpool --image vol01 --size 2G
[cephadm@ceph-admin01 ceph-cluster]$ rbd ls -l --pool mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
vol01  2 GiB            2            
```

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rbd create --pool mykernelpool  vol02 --size 2G
[cephadm@ceph-admin01 ceph-cluster]$ rbd ls -l --pool mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
vol01  2 GiB            2            
vol02  2 GiB            2            
```

#### image-spec方式

```bash
[cephadm@ceph-admin01 ceph-cluster]$  rbd create mykernelpool/vol03 --size 2G
[cephadm@ceph-admin01 ceph-cluster]$ rbd ls -l --pool mykernelpool
                # 父  # 格式，目前仅有2
NAME   SIZE   PARENT  FMT  PROT  LOCK
vol01  2 GiB            2            
vol02  2 GiB            2            
vol03  2 GiB            2            
```

### 镜像信息

#### json

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rbd ls -l --pool mykernelpool --format json --pretty-format
[
    {
        "image": "vol01",
        "id": "63f97868939c",
        "size": 2147483648,
        "format": 2
    },
    {
        "image": "vol02",
        "id": "642fee46f26",
        "size": 2147483648,
        "format": 2
    },
    {
        "image": "vol03",
        "id": "64414f8b9ba9",
        "size": 2147483648,
        "format": 2
    }
]
```

#### 详细

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rbd info mykernelpool/vol01
```

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rbd info --pool mykernelpool vol01
```

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rbd info --pool mykernelpool --image vol01
rbd image 'vol01': # 镜像名
	size 2 GiB in 512 objects # 大小，切割 512对象
	order 22 (4 MiB objects) 次序是22. 2^22 = 2^10K*2^10K*4 = 4M 每个对象4M大小
	snapshot_count: 0 # 镜像快照数量
	id: 63f97868939c # ID
	block_name_prefix: rbd_data.63f97868939c # 池中镜像名前缀
	format: 2 # 格式
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten # 镜像特性, 默认支持
		# 1. layering 支持分层快照，可以镜像克隆 ，克隆类似于链接快照
		# 2. exclusive-lock 排他锁，限定一个镜像只能被一个客户端使用。
		# 3. object-map 对象位图，加速导入导出，已用容量统计
		# 4.  fast-diff 快照时快速比较不同
		# 5. deep-flatten 链接克隆转换为全量克隆
		#journal 日志 保数据 一致性，但是会增加IO
		#data pool 镜像文件一般在副本型存储池, 即可以把rbd放在纠删码池中
		
	op_features: 
	flags: 
	create_timestamp: Thu Apr  1 16:03:08 2021 # 创建时间戳
	access_timestamp: Thu Apr  1 16:03:08 2021
	modify_timestamp: Thu Apr  1 16:03:08 2021

```

### 镜像禁用特性

从J版开始，默认支持以上1-5个特性。Linux加载磁盘，不支持3，4，5

openstack也可以加载磁盘

```diff
[cephadm@ceph-admin01 ceph-cluster]$ rbd feature disable mykernelpool/vol01 object-map fast-diff deep-flatten
[cephadm@ceph-admin01 ceph-cluster]$ rbd info --pool mykernelpool --image vol01
rbd image 'vol01':
	size 2 GiB in 512 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 63f97868939c
	block_name_prefix: rbd_data.63f97868939c
	format: 2
+	features: layering, exclusive-lock
```

### 客户端使用镜像

#### 安装ceph包

提供ceph.ko模块

```bash
rpm -ivh https://mirrors.aliyun.com/ceph/rpm-15.2.9/el7/noarch/ceph-release-1-1.el7.noarch.rpm
wget -O /etc/yum.repos.d/epel-7.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum install -y ceph-common
```

#### 授权mykernel用户 on admin

mykernel用户对mon只读，对osd所有权限

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph auth get-or-create client.mykernel  mon 'allow r' osd 'allow * pool=mykernelpool'
[client.mykernel]
	key = AQDHfWVgIyA+BxAAav2dhHB+up0yX+htCF4bgw==
[cephadm@ceph-admin01 ceph-cluster]$ ceph auth get client.mykernel
exported keyring for client.mykernel
[client.mykernel]
	key = AQDHfWVgIyA+BxAAav2dhHB+up0yX+htCF4bgw==
	caps mon = "allow r"
	caps osd = "allow * pool=mykernelpool"
[cephadm@ceph-admin01 ceph-cluster]$ 

```

#### 分发认证文件

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph auth get client.mykernel -o ceph.client.mykernel.keyring 
exported keyring for client.mykernel

[cephadm@ceph-admin01 ceph-cluster]$ scp ceph.client.mykernel.keyring root@store05:/etc/ceph

```

#### 客户端验证

##### 验证认证文件

```diff
[cephadm@store05 ~]$ 
[cephadm@store05 ~]$ ls /etc/ceph/
ceph.client.kube.keyring  ceph.client.mykernel.keyring  ceph.conf  rbdmap  tmpbuLV3V
[cephadm@store05 ~]$ ls /etc/ceph/ceph.client.mykernel.keyring 
+/etc/ceph/ceph.client.mykernel.keyring

```

##### 验证查看mon状态 权限

```bash
[cephadm@store05 ~]$ ceph --user mykernel -s
  cluster:
    id:     b68bde4e-a661-47d3-b078-4b58f7eae0f5
    health: HEALTH_OK
```

```bash
[cephadm@store05 ~]$ ceph --user mykernel osd pool ls
device_health_metrics
mypool
rbdpool
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
cephfs-metadata
cephfs-data
kube
mykernel
mykernelpool
```

##### 验证仅对mykernelpool有权限

```bash
[cephadm@store05 ~]$ rbd --user mykernel ls -l --pool mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
vol01  2 GiB            2            
vol02  2 GiB            2            
vol03  2 GiB            2            
[cephadm@store05 ~]$ rbd --user mykernel ls -l --pool mykernel
2021-04-01T16:24:20.204+0800 7f178c605f80 -1 librbd::api::Image: list_images: error listing v1 images: (1) Operation not permitted
rbd: listing images failed: (1) Operation not permitted
```



### 客户端加载镜像为本地模块

#### 映射镜像 

当前磁盘

```bash
[cephadm@store05 ~]$ sudo fdisk -l | grep /dev
磁盘 /dev/vda：128.8 GB, 128849018880 字节，251658240 个扇区
/dev/vda1   *        2048     2099199     1048576   83  Linux
/dev/vda2         2099200    23070719    10485760   83  Linux
/dev/vda3        23070720    27267071     2098176   82  Linux swap / Solaris
/dev/vda4        27267072   251658239   112195584    5  Extended
/dev/vda5        27269120   251658239   112194560   83  Linux
磁盘 /dev/vdb：85.9 GB, 85899345920 字节，167772160 个扇区
磁盘 /dev/vdc：128.8 GB, 128849018880 字节，251658240 个扇区
磁盘 /dev/mapper/ceph--7c155bdb--abb2--44dd--bc89--46687793dbde-osd--block--574e5254--660f--4093--96ce--ea6db6cfd612：85.9 GB, 85895151616 字节，167763968 个扇区
磁盘 /dev/mapper/ceph--c4d7fc0a--4570--42ed--8480--75b055bdf8dd-osd--block--40a04263--a8af--49f3--80e1--8c28763d1224：128.8 GB, 128844824576 字节，251650048 个扇区

```

```bash


# 其他用户不可以映射
[cephadm@store05 ~]$ sudo rbd --user kube map  mykernelpool/vol01
rbd: sysfs write failed
2021-04-01T16:30:24.736+0800 7f0c24ff9700 -1 librbd::image::OpenRequest: failed to stat v2 image header: (1) Operation not permitted
2021-04-01T16:30:24.737+0800 7f0c1ffff700 -1 librbd::ImageState: 0x556ea5cca5c0 failed to open image: (1) Operation not permitted
rbd: error opening image vol01: (1) Operation not permitted
In some cases useful info is found in syslog - try "dmesg | tail".
rbd: map failed: (1) Operation not permitted

# mykernel用户
[cephadm@store05 ~]$ sudo rbd --user mykernel map  mykernelpool/vol01
/dev/rbd0

```

验证磁盘

```bash
[cephadm@store05 ~]$ sudo fdisk -l /dev/rbd*
磁盘 /dev/rbd0：2147 MB, 2147483648 字节，4194304 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：4194304 字节 / 4194304 字节

```



#### 格式化挂载

```bash
[cephadm@store05 ~]$ sudo mkfs.xfs /dev/rbd0
meta-data=/dev/rbd0              isize=512    agcount=8, agsize=65536 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

```bash
[cephadm@store05 ~]$ sudo mount /dev/rbd0 /mnt
[cephadm@store05 ~]$ sudo df -TH /mnt
文件系统       类型  容量  已用  可用 已用% 挂载点
/dev/rbd0      xfs   2.2G   34M  2.2G    2% /mnt
```

```diff
# 可以观察到lock
[root@ceph-admin01 ~]# rbd ls -l -p mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
+vol01  2 GiB            2        excl
vol02  2 GiB            2            
vol03  2 GiB            2            
```



#### 卸载

```bash
[cephadm@store05 ~]$ rbd showmapped
id  pool          namespace  image  snap  device   
0   mykernelpool             vol01  -     /dev/rbd0


[cephadm@store05 ~]$ sudo umount /mnt
[cephadm@store05 ~]$ sudo rbd unmap /dev/rbd0


[cephadm@store05 ~]$ rbd showmapped
[cephadm@store05 ~]$ 
```

#### ceph模块

```bash
[cephadm@store05 ~]$ modinfo ceph
filename:       /lib/modules/3.10.0-862.el7.x86_64/kernel/fs/ceph/ceph.ko.xz

```

### 镜像扩容

在线扩容，建议先做全量快照：快照之后保护，克隆，展平。

```bash
# 在线或离线扩容
[cephadm@store05 ~]$ sudo rbd --user mykernel map  mykernelpool/vol01
/dev/rbd0
[cephadm@store05 ~]$ sudo mount /dev/rbd0 /mnt

```

```diff
[root@ceph-admin01 ~]# rbd ls -l -p mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
+vol01  2 GiB            2        excl
vol02  2 GiB            2            
vol03  2 GiB            2            
+[root@ceph-admin01 ~]# rbd resize -s 5G mykernelpool/vol01
Resizing image: 100% complete...done.
 
```

```diff
[root@ceph-admin01 ~]# rbd ls -l -p mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
+vol01  5 GiB            2            
vol02  2 GiB            2            
vol03  2 GiB            2           
```

验证客户端

```bash
[cephadm@store05 ~]$ sudo df -TH /mnt
文件系统       类型  容量  已用  可用 已用% 挂载点
/dev/rbd0      xfs   2.2G   34M  2.2G    2% /mnt


# 拉伸xfs
[cephadm@store05 ~]$ sudo xfs_growfs /mnt
meta-data=/dev/rbd0              isize=512    agcount=8, agsize=65536 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 524288 to 1310720

```

```diff
[cephadm@store05 ~]$ df -TH /mnt
文件系统       类型  容量  已用  可用 已用% 挂载点
+/dev/rbd0      xfs   5.4G   35M  5.4G    1% /mnt
```

### 删除镜像

#### 删除前准备

```bash
# 先卸载
[cephadm@store05 ~]$ rbd showmapped
id  pool          namespace  image  snap  device   
0   mykernelpool             vol01  -     /dev/rbd0
[cephadm@store05 ~]$ sudo umount /mnt
[cephadm@store05 ~]$ sudo rbd unmap /dev/rbd0
[cephadm@store05 ~]$ rbd showmapped
[cephadm@store05 ~]$ 

```

```bash
# 删除前
[root@ceph-admin01 ~]# rbd ls -l -p mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
vol01  5 GiB            2            
vol02  2 GiB            2            
vol03  2 GiB            2      
```

#### 真删除

```bash
[root@ceph-admin01 ~]# rbd rm mykernelpool/vol01
Removing image: 100% complete...done.
[root@ceph-admin01 ~]# rbd ls -l -p mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
vol02  2 GiB            2            
vol03  2 GiB            2            

```

删除不可以恢复，所以建议使用回收站

#### 回收删除

##### 回收

```bash
[root@ceph-admin01 ~]# rbd ls -l -p mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
vol02  2 GiB            2            
vol03  2 GiB            2            
```

回收vol02

```bash
# 查看回收站
[root@ceph-admin01 ~]# rbd trash list -p mykernelpool
[root@ceph-admin01 ~]# 


# 回收vol02
[root@ceph-admin01 ~]# rbd trash move  mykernelpool/vol02
[root@ceph-admin01 ~]# rbd trash list -p mykernelpool
642fee46f26 vol02

# 存储池
[root@ceph-admin01 ~]# rbd ls -l -p mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
vol03  2 GiB            2       
```

##### 回收站恢复

```bash
# 存储池
[root@ceph-admin01 ~]# rbd ls -l -p mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
vol03  2 GiB            2       

# 回收站
[root@ceph-admin01 ~]# rbd trash list -p mykernelpool
642fee46f26 vol02

# 恢复
[root@ceph-admin01 ~]# rbd trash restore -p mykernelpool --image vol02 --image-id 642fee46f26
[root@ceph-admin01 ~]# 

# 检验
[root@ceph-admin01 ~]# rbd ls -l -p mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
vol02  2 GiB            2            
vol03  2 GiB            2            
[root@ceph-admin01 ~]# rbd trash list -p mykernelpool
[root@ceph-admin01 ~]# 
```

##### 回收站中删除

```bash
remove
```

##### 回收站中清空

```bash
purge
```

## 快照管理

### 创建镜像

```bash
[cephadm@store05 ~]$ sudo rbd --user mykernel create --pool mykernelpool --image vol01 --size 2G
[cephadm@store05 ~]$ rbd --user mykernel ls -l --pool mykernelpool
NAME   SIZE   PARENT  FMT  PROT  LOCK
vol01  2 GiB            2            
vol02  2 GiB            2            
vol03  2 GiB            2   

[cephadm@store05 ~]$ rbd --user mykernel info   --pool mykernelpool vol01
rbd image 'vol01':
	size 2 GiB in 512 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 65cd4f8ae945
	block_name_prefix: rbd_data.65cd4f8ae945
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	op_features: 
	flags: 
	create_timestamp: Thu Apr  1 16:49:34 2021
	access_timestamp: Thu Apr  1 16:49:34 2021
	modify_timestamp: Thu Apr  1 16:49:34 2021


[cephadm@store05 ~]$ sudo rbd --user mykernel feature disable mykernelpool/vol01 object-map fast-diff deep-flatten 
[cephadm@store05 ~]$ rbd --user mykernel info   --pool mykernelpool vol01
rbd image 'vol01':
	size 2 GiB in 512 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 65cd4f8ae945
	block_name_prefix: rbd_data.65cd4f8ae945
	format: 2
	features: layering, exclusive-lock

```

### 查看快照

```bash
[root@store05 ~]#  rbd --user mykernel  ls -l mykernelpool
NAME         SIZE   PARENT  FMT  PROT  LOCK
vol01        2 GiB            2        excl
vol01@snap1  2 GiB            2            
vol02        2 GiB            2            
vol03        2 GiB            2            
[root@store05 ~]#  rbd --user mykernel snap ls  mykernelpool/vol01
SNAPID  NAME   SIZE   PROTECTED  TIMESTAMP               
     4  snap1  2 GiB             Thu Apr  1 16:53:11 2021

```



### 创建快照

验证镜像变动，恢复快照数据恢复

```bash
[cephadm@store05 ~]$ sudo rbd --user mykernel map mykernelpool/vol01
/dev/rbd0
[cephadm@store05 ~]$ sudo mkfs.xfs /dev/rbd0

[cephadm@store05 ~]$ sudo mount /dev/rbd0 /mnt
[cephadm@store05 ~]$ sudo cp /etc/fstab  /etc/issue /mnt
[cephadm@store05 ~]$ ls /mnt
fstab  issue

```

创建

```bash
[cephadm@store05 ~]$ sudo rbd  --user mykernel snap create mykernelpool/vol01@snap1
[cephadm@store05 ~]$ sudo rbd  --user mykernel snap list mykernelpool/vol01
SNAPID  NAME   SIZE   PROTECTED  TIMESTAMP               
     4  snap1  2 GiB             Thu Apr  1 16:53:11 2021

```

删除文件

```bash
[cephadm@store05 ~]$ sudo rm -f /mnt/issue 
[cephadm@store05 ~]$ ls /mnt
fstab

```

### 恢复快照

```bash
[cephadm@store05 ~]$ sudo umount /mnt
[cephadm@store05 ~]$ sudo rbd unmap /dev/rbd0

```

```bash
[cephadm@store05 ~]$ sudo rbd --user mykernel snap rollback mykernelpool/vol01@snap1
Rolling back to snapshot: 100% complete...done.

```

重新挂载

```bash
[cephadm@store05 ~]$ sudo rbd --user mykernel map mykernelpool/vol01
/dev/rbd0
[cephadm@store05 ~]$ sudo mount /dev/rbd0 /mnt
[cephadm@store05 ~]$ ls /mnt
fstab  issue
```



### 删除快照

```bash
[root@store05 ~]#  rbd --user mykernel snap rm  mykernelpool/vol01@snap1
Removing snap: 100% complete...done.

```

### 快照限制

#### 添加限制

```bash
[root@store05 ~]#  rbd --user mykernel snap limit set  mykernelpool/vol01 --limit 2
```

验证

```bash
[root@store05 ~]# for i in {1..3}; do sudo rbd  --user mykernel snap create mykernelpool/vol01@snap$i; done
rbd: failed to create snapshot: (122) Disk quota exceeded

[root@store05 ~]# rbd --user mykernel snap ls  mykernelpool/vol01
SNAPID  NAME   SIZE   PROTECTED  TIMESTAMP               
     6  snap1  2 GiB             Fri Apr  2 08:52:23 2021
     7  snap2  2 GiB             Fri Apr  2 08:52:24 2021
# 仅2个快照
```

#### 修改限制

```bash
[root@store05 ~]# rbd --user mykernel snap limit set  mykernelpool/vol01 --limit 3
[root@store05 ~]#  rbd  --user mykernel snap create mykernelpool/vol01@snap3


# 3个快照
[root@store05 ~]#  rbd --user mykernel snap ls  mykernelpool/vol01
SNAPID  NAME   SIZE   PROTECTED  TIMESTAMP               
     6  snap1  2 GiB             Fri Apr  2 08:52:23 2021
     7  snap2  2 GiB             Fri Apr  2 08:52:24 2021
    10  snap3  2 GiB             Fri Apr  2 08:53:22 2021
```

#### 解除限制

```bash
[root@store05 ~]# rbd --user mykernel snap limit clear  mykernelpool/vol01 
```

#### 镜像克隆（链接快照）

原镜像 -> 快照（保护，parent, os基础镜像）->链接快照(clone, children) -> 全量快照(展平)

链接快照：依赖底层快照和原镜像

全量快照: 不依赖底层镜像和原镜像



##### 提供基础镜像

```bash
[root@store05 ~]# rbd --user mykernel create --pool mykernelpool --image vol07 --size 2G
[root@store05 ~]# rbd --user mykernel info   --pool mykernelpool vol07 | grep '\bfeatures:'
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten

[root@store05 ~]# rbd --user mykernel feature disable mykernelpool/vol07 object-map fast-diff deep-flatten
```

映射、格式化、挂载

```bash
[root@store05 ~]#  rbd --user mykernel map  mykernelpool/vol07
/dev/rbd0
[root@store05 ~]#  mkfs.xfs /dev/rbd0
meta-data=/dev/rbd0              isize=512    agcount=8, agsize=65536 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@store05 ~]# mount /dev/rbd0 /mnt
[root@store05 ~]# df -TH /mnt
文件系统       类型  容量  已用  可用 已用% 挂载点
/dev/rbd0      xfs   2.2G   34M  2.2G    2% /mnt

```

准备OS系统

```bash
[root@store05 ~]# cp /etc/issue /etc/passwd /mnt
[root@store05 ~]# 
[root@store05 ~]# ls /mnt
issue  passwd

```

##### 快照（parent)

```bash
# 当模块的快照，先卸载
[root@store05 ~]# umount /mnt
[root@store05 ~]# rbd unmap /dev/rbd0

```

```bash
[root@store05 ~]# rbd --user mykernel snap create mykernelpool/vol07@clonetpl1

[root@store05 ~]# rbd --user mykernel snap ls mykernelpool/vol07
SNAPID  NAME       SIZE   PROTECTED  TIMESTAMP               
    11  clonetpl1  2 GiB             Fri Apr  2 09:06:08 2021
```

###### 保护模式

```bash
[root@store05 ~]# rbd --user mykernel snap protect mykernelpool/vol07@clonetpl1
[root@store05 ~]# rbd --user mykernel snap ls mykernelpool/vol07
SNAPID  NAME       SIZE   PROTECTED  TIMESTAMP               
    11  clonetpl1  2 GiB  yes        Fri Apr  2 09:06:08 2021
```

##### 克隆快照(链接快照)

克隆保护之后，就不再是保护了

###### 克隆myimg01

```bash
[root@store05 ~]# rbd --user mykernel clone mykernelpool/vol07@clonetpl1 mykernelpool/myimg01
```

显示镜像

```bash
[root@store05 ~]# rbd --user mykernel ls -p mykernelpool
myimg01
vol01
vol02
vol03
vol07
```

详细格式

```diff
[root@store05 ~]# rbd --user mykernel ls -l -p mykernelpool
NAME             SIZE   PARENT                        FMT  PROT  LOCK
+myimg01          2 GiB  mykernelpool/vol07@clonetpl1 # 克隆之后会显示父镜像    2            
vol01            2 GiB                                  2            
vol01@snap1      2 GiB                                  2            
vol01@snap2      2 GiB                                  2            
vol01@snap3      2 GiB                                  2            
vol02            2 GiB                                  2            
vol03            2 GiB                                  2            
vol07            2 GiB                                  2            
+vol07@clonetpl1  2 GiB                                  2  yes  # 保护 
```

测试映射 

```bash
[root@store05 ~]# rbd --user mykernel map mykernelpool/myimg01
/dev/rbd0

# 快照克隆的快照，和父快照一致
[root@store05 ~]# mount /dev/rbd0 /mnt
[root@store05 ~]# ls /mnt
issue  passwd

# 添加信息
[root@store05 ~]# echo Hello Ceph Cluster. >> /mnt/issue 
[root@store05 ~]# cat /mnt/issue
\S
Kernel \r on an \m

Hello Ceph Cluster.

```

查看映射

```bash
[root@store05 ~]# rbd showmapped
id  pool          namespace  image    snap  device   
0   mykernelpool             myimg01  -     /dev/rbd0
```



###### 克隆myimg02

```bash
[root@store05 ~]# rbd --user mykernel clone mykernelpool/vol07@clonetpl1 mykernelpool/myimg02

 
```

```diff
# 非保护 
[root@store05 ~]#  rbd --user mykernel ls -l -p mykernelpool
NAME             SIZE   PARENT                        FMT  PROT  LOCK
myimg01          2 GiB  mykernelpool/vol07@clonetpl1    2        excl
+myimg02          2 GiB  mykernelpool/vol07@clonetpl1    2            
vol01            2 GiB                                  2            
vol01@snap1      2 GiB                                  2            
vol01@snap2      2 GiB                                  2            
vol01@snap3      2 GiB                                  2            
vol02            2 GiB                                  2            
vol03            2 GiB                                  2            
vol07            2 GiB                                  2            
vol07@clonetpl1  2 GiB                                  2  yes  
```

映射

```bash
[root@store05 ~]# rbd --user mykernel map mykernelpool/myimg02
/dev/rbd1
[root@store05 ~]# mount /dev/rbd1 /opt
mount: 文件系统类型错误、选项错误、/dev/rbd1 上有坏超级块、
       缺少代码页或助手程序，或其他错误

       有些情况下在 syslog 中可以找到一些有用信息- 请尝试
       dmesg | tail  这样的命令看看。
# 2个ceph块设备不能同时挂载
[root@store05 ~]# umount /mnt
[root@store05 ~]# mount /dev/rbd1 /opt

# 还是原卷内容
[root@store05 ~]# ls /opt
issue  passwd
[root@store05 ~]# cat /opt/issue 
\S
Kernel \r on an \m

[root@store05 ~]# 

```

##### 查看映射

```bash
[root@store05 ~]# rbd showmapped
id  pool          namespace  image    snap  device   
0   mykernelpool             myimg01  -     /dev/rbd0
1   mykernelpool             myimg02  -     /dev/rbd1
```

##### 查看镜像有多少克隆

```bash
# admin角色
[root@ceph-admin01 ~]# rbd children mykernelpool/vol07@clonetpl1
mykernelpool/myimg01
mykernelpool/myimg02

[root@store05 ~]# rbd --user mykernel ls -l -p mykernelpool
NAME             SIZE   PARENT                        FMT  PROT  LOCK
myimg01          2 GiB  mykernelpool/vol07@clonetpl1    2        excl
myimg02          2 GiB  mykernelpool/vol07@clonetpl1    2        excl
```

##### 删除原卷

得需要链接克隆(clone), 变成全量克隆

```bash
[root@store05 ~]# rbd --user mykernel flatten mykernelpool/myimg01
Image flatten: 100% complete...done.

# 可以观察myimg01已经是独立的镜像
[root@store05 ~]# rbd --user mykernel ls -l -p mykernelpool
NAME             SIZE   PARENT                        FMT  PROT  LOCK
myimg01          2 GiB                                  2            
myimg02          2 GiB  mykernelpool/vol07@clonetpl1    2        excl

[root@ceph-admin01 ~]# rbd children mykernelpool/vol07@clonetpl1
mykernelpool/myimg02

```

现在删除原卷和快照

###### 解除保护

```bash
[root@store05 ~]# umount /opt
[root@store05 ~]# rbd showmapped
id  pool          namespace  image    snap  device   
0   mykernelpool             myimg01  -     /dev/rbd0
1   mykernelpool             myimg02  -     /dev/rbd1


[root@store05 ~]# rbd unmap /dev/rbd0
[root@store05 ~]# rbd unmap /dev/rbd1

[root@store05 ~]# rbd showmapped
[root@store05 ~]# 

# 在去掉快照保护前，下面有子镜像，就会报错
[root@store05 ~]#  rbd --user mykernel snap unprotect mykernelpool/vol07@clonetpl1
2021-04-02T09:24:07.102+0800 7f2eef0a8700 -1 librbd::SnapshotUnprotectRequest: cannot get children for pool 'cephf

# 所以先删除镜像
[root@store05 ~]# rbd --user mykernel  rm mykernelpool/myimg02
Removing image: 100% complete...done.


# 再去保护
[root@ceph-admin01 ~]# rbd  snap unprotect mykernelpool/vol07@clonetpl1

```

###### 删除快照及原卷

```bash
[root@ceph-admin01 ~]# rbd  snap rm mykernelpool/vol07@clonetpl1
Removing snap: 100% complete...done.
[root@ceph-admin01 ~]# rbd  rm mykernelpool/vol07
Removing image: 100% complete...done.
```

# ceph为kvm提供存储

以上`3.3.6.4` 创建的myimg01可以作为kvm的镜像。 空镜像，虚拟机安装OS, 快照保护起来，再克隆 。

也可以，下载镜像数据直接导入ceph的镜像中。启动虚拟机直接有操作系统 。

## 外部数据直接导入为镜像

```bash
# 转换格式raw
[root@centos7-iaas template]# qemu-img convert -f qcow2 ubuntu-1804-1C1G.qcow2 -O raw ubuntu-1804-1C1G.raw


[root@centos7-iaas template]# rbd --user mykernel import ./ubuntu-1804-1C1G.raw mykernelpool/ubuntu-1804-raw
Importing image: 4% complete...

Importing image: 100% complete...done.

```

镜像信息

```bash
[root@centos7-iaas template]# rbd --user mykernel info mykernelpool/ubuntu-1804-raw 
rbd image 'ubuntu-1804-raw': 
	size 60 GiB in 15360 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 6b3d7768df94
	block_name_prefix: rbd_data.6b3d7768df94
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	op_features: 
	flags: 
	create_timestamp: Tue Apr  6 10:04:47 2021
	access_timestamp: Tue Apr  6 10:04:47 2021
	modify_timestamp: Tue Apr  6 10:05:48 2021

```

查看镜像

```bash
[root@store05 ~]# rbd --user mykernel ls mykernelpool
cirrors-0.5.1
[root@store05 ~]# rbd --user mykernel ls mykernelpool -l
NAME           SIZE    PARENT  FMT  PROT  LOCK
ubuntu-1804-raw  60 GiB            2           

```

增加镜像大小

```bash
[root@store05 ~]# rbd --user mykernel resize -s 80G mykernelpool/ubuntu-1804-raw 
Resizing image: 100% complete...done.

[root@store05 ~]# rbd --user mykernel ls mykernelpool -l
NAME           SIZE   PARENT  FMT  PROT  LOCK
ubuntu-1804-raw  80 GiB            2                  

```

## kvm加载ceph的用户

```bash
[root@ceph-admin01 ~]# ceph auth get client.mykernel
exported keyring for client.mykernel
[client.mykernel]
	key = AQDHfWVgIyA+BxAAav2dhHB+up0yX+htCF4bgw==
	caps mon = "allow r"
	caps osd = "allow * pool=mykernelpool"
[root@ceph-admin01 ~]# ceph auth caps client.mykernel mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=mykernelpool'
updated caps for client.mykernel
[root@ceph-admin01 ~]# ceph auth get client.mykernel
exported keyring for client.mykernel
[client.mykernel]
	key = AQDHfWVgIyA+BxAAav2dhHB+up0yX+htCF4bgw==
	caps mon = "allow r"
	caps osd = "allow class-read object_prefix rbd_children, allow rwx pool=mykernelpool"



# 官方文档
[root@ceph-admin01 ~]#   ceph auth caps client.mykernel mon 'profile rbd' osd 'profile rbd pool=mykernelpool'
updated caps for client.mykernel
```

导出keyring

```bash
[root@ceph-admin01 ~]# su - cephadm
上一次登录：四 4月  1 15:50:17 CST 2021pts/0 上
[cephadm@ceph-admin01 ~]$ cd ceph-cluster/
[cephadm@ceph-admin01 ceph-cluster]$ ceph auth get client.mykernel -o ceph.client.mykernel.keyring 
exported keyring for client.mykernel
[cephadm@ceph-admin01 ceph-cluster]$ 
```

把keyring文件传送给kvm虚拟机端

```bash
[cephadm@ceph-admin01 ceph-cluster]$ scp ceph.client.mykernel.keyring root@172.18.199.75:/etc/ceph

[root@store05 ~]# ceph --user mykernel -s
  cluster:
    id:     b68bde4e-a661-47d3-b078-4b58f7eae0f5
    health: HEALTH_OK

```

## 准备kvm

libvirt来连接ceph的rbd，所以先安装libvirt

```bash
yum install libvirt -y

完毕！
[root@store05 ~]# systemctl start libvirtd
[root@store05 ~]# systemctl enable libvirtd

```

### 准备ceph的secret

```bash
[root@store05 ~]# cat  client.mykernel.xml
<secret ephemeral='no' private='no'>
    <usage type='ceph'>
        <name>client.mykernel secret</name>
    </usage>
</secret>
[root@store05 ~]# virsh secret-define client.mykernel.xml 
生成 secret 3af314ac-af93-4afc-9b0e-51f155c820b3

```

获取keyring

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph auth print-key client.mykernel
AQDHfWVgIyA+BxAAav2dhHB+up0yX+htCF4bgw==
```

### 配置secret

```bash
[root@store05 ~]# virsh secret-set-value --secret 3af314ac-af93-4afc-9b0e-51f155c820b3 --base64 AQDHfWVgIyA+BxAAav2dhHB+up0yX+htCF4bgw==
secret 值设定

# secret是 virsh secret-define 的值
# base64是 ceph auth print-key 的值
```

获取secret

```bash
[root@store05 ~]# virsh secret-get-value --secret 3af314ac-af93-4afc-9b0e-51f155c820b3
AQDHfWVgIyA+BxAAav2dhHB+up0yX+htCF4bgw==
# 此值也是mykernel用户的keyring.
```

```bash
[root@store05 ~]# virsh secret-list
 UUID                                  用量
--------------------------------------------------------------------------------
 3af314ac-af93-4afc-9b0e-51f155c820b3  ceph client.mykernel secret

```



### 创建虚拟机

https://docs.ceph.com/en/latest/rbd/libvirt/

```bash
yum install virt-install -y
```

#### 磁盘映像启动的kvm

```bash
<domain type='kvm'>
  <name>ceph-vm</name>
  <memory unit='KiB'>2097152</memory> # awk 'BEGIN{print 2097152/1024/1024}' -> 2G
  <vcpu placement='static'>4</vcpu>   # 4核
  <os>
	<type arch='x86_64'>hvm</type> # x86_64
  </os>
  <devices>
	<emulator>/usr/libexec/qemu-kvm</emulator>
	<disk type='file' device='disk'> # file
	  <driver name='qemu' type='qcow2' cache='none' io='native'/> # 优化
	  <source file='/nvme_data/vms/ceph/admin01'/> # 路径 
	  <target dev='vda' bus='virtio'/> # 映射的设备
	</disk>
	<interface type='bridge'> # 网络
	  <source bridge='vmbr2'/> # 哪个物理桥
	  <model type='virtio'/> # virtio
	</interface>
	<interface type='bridge'>
	  <source bridge='vmbr3'/>
	  <model type='virtio'/>
	</interface>
	<graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0'> # vnc
	  <listen type='address' address='0.0.0.0'/> 
	</graphics>
    <console type='pty'>
      <target type='virtio' port='0'/>
    </console>
    <serial type='pty'>
      <target type='isa-serial' port='0'>
        <model name='isa-serial'/>
      </target>
    </serial>

  </devices>
</domain>
```

#### ceph网络启动kvm

```diff
<domain type='kvm'>
  <name>ceph-vm</name>
  <memory unit='KiB'>2097152</memory>
  <vcpu placement='static'>4</vcpu>
  <os>
	<type arch='x86_64'>hvm</type>
  </os>
  <devices>
	<emulator>/usr/libexec/qemu-kvm</emulator>
+	<disk type='network' device='disk'>
+		<driver name='qemu' type='raw' cache='writeback'/>
+		<source protocol='rbd' name='mykernelpool/ubuntu-1804-raw'>
+			<host name='172.18.199.71' port='6789'/>
+		</source>
			
+		<auth username='mykernel'>
+			<secret type='ceph' uuid='3af314ac-af93-4afc-9b0e-51f155c820b3'/>
+		</auth>

+	  <target dev='vda' bus='virtio'/>
	</disk>
	<interface type='bridge'>
	  <source bridge='vmbr2'/>
	  <model type='virtio'/>
	</interface>
	<interface type='bridge'>
	  <source bridge='vmbr3'/>
	  <model type='virtio'/>
	</interface>
	<graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0'>
	  <listen type='address' address='0.0.0.0'/>
	</graphics>
  </devices>
    <console type='pty'>
      <target type='virtio' port='0'/>
    </console>
    <serial type='pty'>
      <target type='isa-serial' port='0'>
        <model name='isa-serial'/>
      </target>
    </serial>

</domain>


```

> ```bash
> --- ceph.xml	2021-04-02 10:45:23.382020496 +0800
> +++ ceph-vm.xml	2021-04-02 10:45:56.676643259 +0800
> @@ -7,9 +7,15 @@
> </os>
> <devices>
> 	<emulator>/usr/libexec/qemu-kvm</emulator>
> -	<disk type='file' device='disk'>
> -	  <driver name='qemu' type='qcow2' cache='none' io='native'/>
> -	  <source file='/nvme_data/vms/ceph/admin01'/>
> +	<disk type='network' device='disk'> # 网络
> +       <driver name='qemu' type='raw' cache='writeback'/> # 此type一定要和镜像类型一致！
> +		<source protocol='rbd' name='mykernelpool/cirrors-0.5.1'> # rbd协议, 存储池和镜像
> +			<host name='172.18.199.71' port='6789'/> # mon01主机名或ip；端口
> +		</source>
> +			
> +		<auth username='mykernel'> # 授权了存储池有权限的用户
> +			<secret type='ceph' uuid='3af314ac-af93-4afc-9b0e-51f155c820b3'/> # secret的uuid 获取命令: virsh secret-list
> +		</auth>
> +
>  	  <target dev='vda' bus='virtio'/> # 目标走virtio, vda设备
>  	</disk>
>  	<interface type='bridge'> 
> ```

```bash
[root@centos7-iaas ~]# virsh define ceph-vm.xml
定义域 ceph-vm（从 ceph-vm.xml）

[root@centos7-iaas ~]# virsh list --all | grep ceph-vm
 -     ceph-vm                        关闭

```

#### 验证虚拟走的ceph

```bash
[root@centos7-iaas ~]# virsh define ceph-vm.xml
定义域 ceph-vm（从 ceph-vm.xml）

```

```bash
[root@centos7-iaas ~]# virsh define ./ceph-vm.xml 
定义域 ceph-vm-1（从 ./ceph-vm.xml）

[root@centos7-iaas ~]# virsh start ceph-vm-1
域 ceph-vm-1 已开始

[root@centos7-iaas ~]# virsh domblklist ceph-vm-1
目标     源
------------------------------------------------
vda        mykernelpool/ubuntu-1804-raw

```

![image-20210406101348744](http://myapp.img.mykernel.cn/image-20210406101348744.png)

![image-20210406101355970](http://myapp.img.mykernel.cn/image-20210406101355970.png)

注意/空间

```bash
root@ubuntu-template:~# df -TH
Filesystem     Type      Size  Used Avail Use% Mounted on
udev           devtmpfs  1.1G     0  1.1G   0% /dev
tmpfs          tmpfs     209M  2.7M  207M   2% /run
/dev/vda1      ext4       64G  4.3G   56G   8% /
tmpfs          tmpfs     1.1G     0  1.1G   0% /dev/shm
tmpfs          tmpfs     5.3M     0  5.3M   0% /run/lock
tmpfs          tmpfs     1.1G     0  1.1G   0% /sys/fs/cgroup
tmpfs          tmpfs     209M     0  209M   0% /run/user/0
root@ubuntu-template:~# 


# 在线扩容
root@ubuntu-template:~# apt install -y cloud-guest-utils
Reading package lists... Done
Building dependency tree       
Reading state information... Done
cloud-guest-utils is already the newest version (0.30-0ubuntu5).
0 upgraded, 0 newly installed, 0 to remove and 179 not upgraded.
root@ubuntu-template:~# growpart /dev/vda 1
CHANGED: partition=1 start=2048 old: size=125825024 end=125827072 new: size=167770079,end=167772127
root@ubuntu-template:~# resize2fs /dev/vda1
resize2fs 1.44.1 (24-Mar-2018)
Filesystem at /dev/vda1 is mounted on /; on-line resizing required
old_desc_blocks = 8, new_desc_blocks = 10
The filesystem on /dev/vda1 is now 20971259 (4k) blocks long.

root@ubuntu-template:~# df -TH
Filesystem     Type      Size  Used Avail Use% Mounted on
udev           devtmpfs  1.1G     0  1.1G   0% /dev
tmpfs          tmpfs     209M  2.7M  207M   2% /run
/dev/vda1      ext4       85G  4.3G   76G   6% /
tmpfs          tmpfs     1.1G     0  1.1G   0% /dev/shm
tmpfs          tmpfs     5.3M     0  5.3M   0% /run/lock
tmpfs          tmpfs     1.1G     0  1.1G   0% /sys/fs/cgroup
tmpfs          tmpfs     209M     0  209M   0% /run/user/0


root@ubuntu-template:~# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    252:0    0  80G  0 disk 
└─vda1 252:1    0  80G  0 part /

```

