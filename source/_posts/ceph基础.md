---
title: ceph基础
date: 2021-02-23 08:19:22
tags:
---



# 为什么使用ceph

数据：元数据、数据

存储设备

- DAS: 直接接在主板总线，ide, sata, scsi, sas, usb

- NAS: 通过网络附加当前主机文件系统, nfs, cifs(samba)

- SAN: 存储区域网络，提供块接口。scsi协议，底层协议更换：FCSAN(光纤传输scsi协议)，ISCSI（inter）



单个硬盘空间容量有限

硬盘IOPS有上限

传统解决方案，通过RAID组合构建大型存储系统，但是iops扩展有限。和一些专用的存储设备：EMC, NetAPP, IBM

现在，分布式存储，有状态应用。通过元数据来路由





## HDFS

hadoop distributed filesystem(java), 山寨了google filesystem.

taobao对hdfs二次开发，TFS，为淘宝存储海量图片.

特点：读写接口受限，可以随机读、顺序读。只能顺序写，至今只能顺序写。



![image-20210223164747358](http://myapp.img.mykernel.cn/image-20210223164747358.png)

namenode 划分数据为小块，并路由到各节点

datanode 存放数据



对数据修改，记录下来。类似redis的aof(appendonlyfile) 追加着写，只能重放记录的指令来恢复。

名称服务可以将修改放在zookeeper中，namenode一个宕机，另一个连接zookeeper即可

![image-20210223165309196](http://myapp.img.mykernel.cn/image-20210223165309196.png)

数据节点高可用：

1. 节点级冗余。每个节点一个主从复制集群

2. 数据级冗余。每个节点存一个，冗余的块放在其他节点。 elasticsearch就是数据块级别。

   > 每个块为一个shard, 块也有主从，1个primary shard和2个replica shard，通常同机架一个副本，跨机架一个副本。



元数据节点很可能成为分布式存储的瓶颈



存储有3类：块存储、文件系统、对象存储

区别

- 块级别，裸设备：没有被组织过的存储空间。kvm虚拟机直接访问磁盘，而不是文件模拟成磁盘，这样性能才更好。
- 文件系统：只能以文件形式存储数据。建构在块级别存储空间之上。分布式存储常见接口
  - 存储文件，数据区和元数据区
- 对象存储：存储对象，对象自带元数据。
  - 对象大小不固定



# ceph

一种分布式存储，软件定义的存储平台，

一个对象式存储系统

ceph中将每个hdfs的分块叫"对象"，对象自身自带元数据。将待管理的数据流切分成固定大小的对象数据，并以其为原子单元完成数据存取。

对象数据由rados集群，多个主机组成存储集群。librados是rados的api, 支持c/c++/java/python/ruby/php编程语言。



ceph从设计上，就解决了分布式存储的元数据节点的瓶颈。对对象名称哈希计算对节点数量取模，就得出在哪个节点。取模法有缺陷，所以使用一致性哈希算法。CRUSH算法，通过计算方式完成对文件路由，是一个混合的算法，其中包含一致性哈希算法。

![image-20210223172725183](http://myapp.img.mykernel.cn/image-20210223172725183.png)

> 适用于不同的应用场景：
>
> 使用ceph, 可以通过librados开发
>
> cephfs 直接类似nfs/cifs挂载使用
>
> rbd      将存储空间模拟成块设备，每个块设备就是image
>
> radosgw      更抽象的跨互联网使用的对象存储，restful接口的云存储。内部对象是固定大小块。



filestore, 起初ceph集群每个节点，是文件系统存储。最初又是元数据+数据，有性能问题。

bluestore, 后来ceph直接使用裸设备。 rocksdb

![image-20210224170320408](http://myapp.img.mykernel.cn/image-20210224170320408.png)

> filestore: 文件系统（数据+元数据）+ Leveldb(数据的元数据)
>
> Bluestore: 裸盘中的数据+元数据(rocksdb) + 元数据的元数据(wal即redis的aof文件或binlog)，**新版本默认**
>
> > Bluestore支持3种存储设备：rocksdb和bluefs放在固态硬盘。数据在机械硬盘。元数据的元数据在nvme固态之上

# rados

![image-20210223174637998](http://myapp.img.mykernel.cn/image-20210223174637998.png)

> 访问RADOS Cluster的接口
>
> 1. cephfs 集成集群中.        使用最少。
> 2. librados ，是api接口，基于此的接口有rbd, radosgw。最为流行。
>
> osd: object storage device 就是磁盘。**至少3个OSD**
>
> mon: 不是元数据服务器，而是监控器，来管理整个集群。“集群元数据”，内部使用**paxos协议**实现数据冗余。 实时查询，不适合一直采集数据。**奇数个，建议3个。避免网络分区**
>
> mgr:  manager简写，新版引入的组件，把**高频查询操作缓存下来**，一旦有人监控，就及时反应。**一个就够了，为了冗余，一般2个。** 没有协同协议，是无状态的。
>
> MDSs(medata servers) ：使用cephfs才有必要部署

![image-20210224165901144](http://myapp.img.mykernel.cn/image-20210224165901144.png)

需要把一个文件存放到rados集群的方式

1. client: radosgw, rdb, cephfs
2. 把文件切割成固定大小的data object
3. ceph把提供的存储空间划分成多个分区，每个分区叫pool(存储池)，存储池的大小取决于底层的存储空间。存储池中可以进下一步划分成名称空间。
4. 每一个pool内部有一个pg存储，即归置组，都是逻辑概念。
5. 数据对象存进rados, 先请求pool, 对对象进行crush计算，会落在pool中某个pg上。一个pg中有100个对象，crush完成对pg数据第一次写在主osd中，然后由主osd同步给从osd中。

ceph可以无限制的扩展

部署时，看生产需要，才部署指定的接口：radosgw/rdb/cephfs



# ceph的存储池、pg、crush协同工作

## 存储池pool

- 不同应用的数据放在不同的存储池中。cephfs在2个存储池中。radosgw在4个存储池中。rbd在1个存储池中。
- 存储池中可以划分名称空间
- 数据在存储池中的pg中，pg与名称空间无关系
- 客户端连接ceph, 需要认证。认证后一直维护与存储池连接

存储池类型

- 副本池 replicated, 默认存储池类型。任何对象在集群中存储多个副本，默认3个副本。 **rbd仅支持 **          更占空间

- 纠删码池 erasure code, 类似RAID5，每个数据过来切成4个块+2个纠删码，允许最多丢失2个块，块越多，浪费空间多，冗余能力高。通常8+4。有效利用率8/12就是3/2。**radosgw支持**         更占CPU

```bash
# 创建池
ceph osd pool create <pool> [<pg_num:int>] [<pgp_num:int>] [replicated|erasure] [<erasure_code_profile>] [<rule>] [<expected_num_objects:int>] [<size:int>] [<pg_  create pool num_min:int>] [on|off|warn] [<target_size_bytes:int>] [<target_size_ratio:float>]

[replicated|erasure] 默认是副本池，指定erasure为纠删码池
rule 指定 crush-ruleset-name CRUSH规则集名称


# 创建副本池
ceph osd pool create <pool> [<pg_num:int>] [<pgp_num:int>] replicated
# 创建纠删码池，本身用的不多，而且radosgw会自动创建
ceph osd pool create <pool> [<pg_num:int>] [<pgp_num:int>] erasure [<erasure_code_profile>]
	ceph osd erasure-code-profile ls
	ceph osd erasure-code-profile get default
	ceph osd erasure-code-profile set myprofile k=4 m=2 crush-failure-domain=osd
		K= 数据指定切分几个
		M= 冗余几个 m/k+m
		
		k+m即pg映射的OSD数量
```

```bash
# 列出存储池
[root@ceph-admin01 ~]# ceph osd pool ls
device_health_metrics
mypool
rbdpool
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
cephfs-metadata
cephfs-data

# 详细信息
[root@ceph-admin01 ~]# ceph osd pool ls detail
```

```bash
# 删除存储池
[root@ceph-admin01 ~]# ceph tell mon.* injectargs --mon-allow-pool-delete=true
mon.store01: mon_allow_pool_delete = 'true' 
mon.store01: {}
mon.store02: mon_allow_pool_delete = 'true' 
mon.store02: {}
mon.store03: mon_allow_pool_delete = 'true' 
mon.store03: {}
[root@ceph-admin01 ~]# ceph osd pool rm mypool mypool --yes-i-really-really-mean-it
[root@ceph-admin01 ~]# ceph osd pool ls
device_health_metrics
rbdpool
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
cephfs-metadata
cephfs-data


## 避免其他人误删除，还需要设定false
ceph tell mon.* injectargs --mon-allow-pool-delete=false
```

```bash
# 存储池配额
#1. 数量
#2. 空间

[root@ceph-admin01 ~]# ceph osd pool get-quota rbdpool
quotas for pool 'rbdpool':
  max objects: N/A
  max bytes  : N/A
	
[root@ceph-admin01 ~]# ceph osd pool set-quota rbdpool max_objects 10000
set-quota max_objects = 10000 for pool rbdpool
[root@ceph-admin01 ~]# ceph osd pool get-quota rbdpool
quotas for pool 'rbdpool':
  max objects: 10k objects  (current num objects: 5 objects)
  max bytes  : N/A

# 一般公共的空间应限制
```

```bash
# 存储池快照

创建
ceph osd pool mksnap <pool> <snap-name>

列出
rados -s <pool> lssnap

回滚
rados -p <pool> rollback <snap-name>

删除快照
ceph osd pool rmsnap <pool> <snap-name>
```

```bash
# 数据压缩

即时压缩
ceph osd pool set <pool> compression_algorithm snappy
	none
	zlib 不建议使用
	lz4
	zstd 压缩比好，消耗CPU
	snappy    压缩比没有zstd, 但是CPU占用较低
	
ceph osd pool set <pool> compression_mode aggressive
	none          始终不压缩
	passive       提示COMPRESSIBLE，则压缩。默认不压缩
	aggressive    默认压缩，提供INCOMPRESSIBLE，则不压缩
	force         始终压缩
	
	
# 全局压缩选项，对所有存储池有效
```



副本池IO工作模型

cleint选择一个PG，会放在3个OSD上。一个主OSD，PG直接写入，其他OSD从主OSD复制数据。在3个OSD存储完了，才返回给客户端响应



纠删码池IO

3+2, 数据切成3个块。2个纠删码块。



## 归置组

归置组PG， object -> pg -> osd, CRUSH算法中存在。归置组数量需要定义。

​	例如：256 PG，每个PG包含1/256的存储池数据。移动PG时，PG过少，移动太多数据。 PG太多，需要RAM来追踪数据。

​	一般为性能最大化的最小数量。 50个OSD -> 50-100PG。            

​	8个OSD, 每个OSD上大约100个PG，总共800个PG. 创建10个pool, 每个存储池只能使用80个PG。如果定义一个存储池，直接写800个PG,那后面就不能创建存储池了。 并且PG数量一般2^N。



```bash
总PG数量
(总OSD * 每OSD的PG数量)/复制因子 => 总PG
8*100/3 并且将此值，四舍五入到2^n
>>> 800 / 3
266
所以PG应该是256个


[root@ceph-admin01 ~]# ceph -s
  cluster:
    id:     2f2d297c-a337-40de-bff9-13d3ea8aae31
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum store01,store02,store03 (age 79m)
    mgr: mgr01(active, since 84m), standbys: mgr02, store03
    mds: cephfs:1 {0=store02=up:active}
    osd: 10 osds: 10 up (since 84m), 10 in (since 2h)
    rgw: 1 daemon active (store01)
 
  task status:
 
  data:
    pools:   9 pools, 297 pgs
    objects: 214 objects, 7.1 KiB
    usage:   10 GiB used, 990 GiB / 1000 GiB avail
    pgs:     297 active+clean

>>> 10*100 / 3
333
>>> 2**8
256
>>> 2**9
512

所以总PG是256个，想更多的PG， 就加节点
默认情况下ceph的OSD不能超256PG
https://ceph.io/pgcalc/
size复制因子
OSD数量
每个OSD上有多少个PG
可以直接求出合理的PG数量
```

PG选择的OSD中，一个坏了，CRUSH会再找一个OSD, 复制数据至新OSD。

PG状态

- active 正常
- clean 主osd和辅助osd 正常
  - 活动集：正常IO的OSD
  - 上行集：异常OSD和正常OSD
- peering 
  - 正在复制 

- degraded 降级

  - osd标记为down时，所有映射到此OSD的PG转入 此状态

  - osd重新启动，
  - osd标记为down时间超过5分钟，ceph对pg恢复，找新OSD
  - 内部OSD上对象不可用，PG也降级 

- stale

  - 主OSD工作异常时，主OSD的PG标记为stale

- undersized (under sized 低于数量)

  - PG副本少于其存储池定义个数时转入此状态

- scrubbing

  - 各OSD检查数据完整性，检查过程中标记为此状态
    - light      轻，随时
    - shallow
    - simply 
    - deep     按位检查，周期

- recovering

  - 加新OSD或某OSD宕掉

- backfilling

  - 新OSD加入时，数据对象会移动到新OSD，即“回填”





## CRUSH

obj直接映射到OSD之上，不可避免OSD变动对集群产生扰动

CRUSH OBJ -> PG -> OSD

均以实时计算方式完成

controller replication under scalable hashing, CRUSH类似一致性哈希

存储对象时，请求mon的集群状态，绑定指定存储池，对pool上pg内对象执行IO操作

