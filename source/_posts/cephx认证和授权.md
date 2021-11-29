---
title: cephx认证和授权
date: 2021-03-31 03:04:26
---



# 为什么使用cephx认证和授权

ceph分布式集群各组件

1. monitor, 整个集群的元数据服务器, 维护整个集群的运行图包含PG图
2. osd，每个磁盘有专用的OSD守护进程，来管理这个磁盘的任务。OSD复制任务，报告OSD的状态，与monitor联络
3. manager, 抽取monitor当中的状态和统计数据，是无状态的http/https服务，内部有很多模块(也称为插件)，可以单独启用或禁用，也需要与mon通信

ceph客户端

1. mds守护进程，给用户提供cephfs文件系统 。用户从mds请求元数据,从osd请求数据。
2. rbd内核库。用户只与rbd通信。
3. rgw守护进程，提供对象存储。用户只与rgw通信。
4. 用户自行构建客户端，访问mds文件系统客户端。或以块形式访问RBD。以对象形式访问RGW。

# MON的cephx协议工作流程

![image-20210331111254795](http://myapp.img.mykernel.cn/image-20210331111254795.png)

1. mon创建用户, 携带keyring，发给client.
2. client请求mon, mon使用keyring加密SessonKey(SK),发送给client, client使用keyring解密。
3. client拿着SK请求MON的ticket, MON检查SK，使用SK加密Ticket,发送给client, client使用SK解密出ticket.
4. mon会ticket与mds和osd共享。
5. client拿着ticket直接请求mds和osd。

# cephx权限

用户、授权、使能

## 用户

TYPE.ID格式

个人：现实的人  client类型 例如：client.admin

参与者： pod    osd与monitor打交道 , 使用cephx协议，账号不是client账号，而是内部账号。

## 授权使能

使能：描述用户可针对MON, OSD, MDS使用的权限范围或级别

通用语法格式 `daemon-type ‘allow caps’ […] `显式授权。不给没有权限

### Mon使能

​	包括 r w x或 allow profile cap

​		读写执行，允许MON资料分享和评估

​	例如：mon ‘allow rwx’ 以及 mon ‘allow profile osd’

 

### Osd使能

​	R w x class-read class-write 和 profile osd

​	OSD允许存储池和名称空间设置权限

​	限定特定存储池  replicated和纠删编码池

​	限定特定存储池的某个名称空间

 ### MDS使能

​	只有allow,或留空

​	元数据只有MDS写，所以无论哪种设定用户把操作发给MDS

 



## 权限

allow

​	MDS上就是rw，如果 在mon/osd上，allow什么权限，需要明确定义

r 

​	访问mon以检索CRUSH 

w 写入

x 调用类方法的能力

class-read ：x子集，读取类

class-write ：x子集，修改类

*表示通配

 

Profile osd 授予用户OSD身份连接到其他OSD或监视器的权限。	授于OSD权限，使OSD能够处理复制检测信号流量和状态报告。

Profile MDS	授权用户以某个MDS身份连接到其他MDS或监视器的权限

Profile bootstrap-osd

​		授予用户引导OSD加入集群的权限

​		授权给部署工具，引导OSD时有权添加密钥

Profile bootstrap-mds

​		授权用户引导元数据服务器权限

​		授权部署工具，使其在引导元数据服务器时有权添加密钥

 

# cephx管理

## 用户管理

cephx有2种用户，真实用户`client.<user-name>`, ceph内部使用的用户`TYPE.ID`

Keyring文件存放多个用户及其密钥信息或1个用户信息。

Monitoring server将多个用户信息放在一个文件。

客户端一个用户信息在一个文件。

### 帮助

```bash
[root@ceph-admin01 ~]# ceph auth -h 
```

### 列出用户

```bash
[root@ceph-admin01 ~]# ceph auth list
mgr.store03 # 用户标识：TYPE.ID
	key: AQBE0WJg/okeCxAAa/A0KnkV6IQt8tHPJQk1Bg==
	caps: [mds] allow *
	caps: [mon] allow profile mgr
	caps: [osd] allow *
```

### 查看用户

```bash
[root@ceph-admin01 ~]# ceph auth get client.admin
exported keyring for client.admin
[client.admin]
	key = AQCpwWJgPik7CBAAImzqb3155idqs6rQcyL9ew==
	caps mds = "allow *"
	caps mgr = "allow *"
	caps mon = "allow *"
	caps osd = "allow *"

```

### 添加用户

```bash
# 添加client.testuser账号，对mon有读权限。对OSD的rbdpool有读写权限
[root@ceph-admin01 ~]# ceph auth add client.testuser mon 'allow r' osd 'allow rw pool=rbdpool'
added key for client.testuser

# ceph auth get-or-create client.testuser mon 'allow r' osd 'allow rw pool=rbdpool'

# ceph auth get-or-create-key client.testuser  mon 'allow r' osd 'allow rw pool=rbdpool'

# 检验用户
ceph auth get client.testuser
```

### 列出用户密钥

```bash
#获取用户信息
[root@ceph-admin01 ~]# ceph auth get client.testuser
exported keyring for client.testuser
[client.testuser]
	key = AQBp4GNgjhclCxAAVP3MquUIGmAB3xGyRmdbeg==
	caps mon = "allow r"
	caps osd = "allow rw pool=rbdpool"


#不存在则创建用户， 存在返回密钥 get-or-create
[root@ceph-admin01 ~]# ceph auth get-or-create client.testuser
[client.testuser]
	key = AQBp4GNgjhclCxAAVP3MquUIGmAB3xGyRmdbeg==

#用户存在，额外创建密钥 get-or-create-key
[root@ceph-admin01 ~]# ceph auth get-or-create-key client.testuser  mon 'allow r' osd 'allow rw pool=rbdpool'
AQBp4GNgjhclCxAAVP3MquUIGmAB3xGyRmdbeg==



#列出用户密钥
[root@ceph-admin01 ~]# ceph auth print-key client.testuser
AQBp4GNgjhclCxAAVP3MquUIGmAB3xGyRmdbeg==[root@ceph-admin01 ~]# 
```

### 导入用户

```bash
导入用户：从别的ceph集群拿到key文件, 有相同的账号和权限
ceph auth import
```

### 修改caps

会覆盖之前的权限

```diff
#testuser用户，修改mon权限有rw
+[root@ceph-admin01 ~]# ceph auth caps client.testuser mon 'allow rw' osd 'allow rw pool=rbdpool'
updated caps for client.testuser

[root@ceph-admin01 ~]# ceph auth get client.testuser
exported keyring for client.testuser
[client.testuser]
	key = AQCv62Ngaf0eOhAACMnOaK5ZhAvuObaLC9AxWQ==
+	caps mon = "allow rw"
	caps osd = "allow rw pool=rbdpool"
```

### 删除用户

```bash
root@ceph-admin01 ~]# ceph auth del client.testuser

[root@ceph-admin01 ~]# ceph auth get client.testuser
Error ENOENT: failed to find client.testuser in keyring
```

## keyring管理

任何一个client访问MON, 都先查本地keyring文件(单个用户->多个用户)，以便认证到MON。mon提供加密的sessionkey, client使用keyring解密

单个用户的keyring cluster-name.user-name.keyring. 即ceph.client.admin.keyring其中ceph是集群名。Client.admin是用户名

多个用户的keyring. Cluster.keyring或keyring或keyring.bin

keyring一般在/etc/ceph目录中，否则命令行指定

### 生成keyring

```bash
# 创建client.kube用户 mon读，osd的kube池所有权限，即读写执行权限。
[root@ceph-admin01 ~]# ceph auth get-or-create-key client.kube  mon 'allow r' osd 'allow * pool=kube'
AQCe5GNgvWxrJRAAXXNv75OcSk83X5V70+J39A==
[root@ceph-admin01 ~]# ceph auth get client.kube
exported keyring for client.kube
[client.kube]
	key = AQCe5GNgvWxrJRAAXXNv75OcSk83X5V70+J39A==
	caps mon = "allow r"
	caps osd = "allow * pool=kube"

```

```bash
#生成keyring
[cephadm@ceph-admin01 ceph-cluster]$ ceph auth get client.kube -o ceph.client.kube.keyring
[cephadm@ceph-admin01 ceph-cluster]$ ls ceph.client.kube.keyring 
ceph.client.kube.keyring
[cephadm@ceph-admin01 ceph-cluster]$  cat ceph.client.kube.keyring
[client.kube]
	key = AQCe5GNgvWxrJRAAXXNv75OcSk83X5V70+J39A==
	caps mon = "allow r"
	caps osd = "allow * pool=kube"

```

将来这个文件在k8s每个节点有这个文件，就可以访问ceph的kube存储池的 mon r, osd的kube存储池的rwx

### 合并keyring

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rm -f cluster.keyring 
# 创建keyring
[cephadm@ceph-admin01 ceph-cluster]$ ceph-authtool --create-keyring cluster.keyring
creating cluster.keyring

# 导入keyring
[cephadm@ceph-admin01 ceph-cluster]$ ceph-authtool cluster.keyring --import-keyring ./ceph.client.kube.keyring
importing contents of ./ceph.client.kube.keyring into cluster.keyring
[cephadm@ceph-admin01 ceph-cluster]$ ceph-authtool cluster.keyring --import-keyring ./ceph.client.admin.keyring
importing contents of ./ceph.client.admin.keyring into cluster.keyring

# 查看keyring
[cephadm@ceph-admin01 ceph-cluster]$  ceph-authtool -l cluster.keyring
[client.admin]
	key = AQCpwWJgPik7CBAAImzqb3155idqs6rQcyL9ew==
	caps mds = "allow *"
	caps mgr = "allow *"
	caps mon = "allow *"
	caps osd = "allow *"
[client.kube]
	key = AQCe5GNgvWxrJRAAXXNv75OcSk83X5V70+J39A==
	caps mon = "allow r"
	caps osd = "allow * pool=kube"
```

