---
title: 系统部署
date: 2021-02-24 08:46:27
tags:
---



 

# 部署工具

ceph-deploy https://github.com/ceph/ceph-deploy

> 借助ssh, sudo, python模块完成ceph部署

ceph-ansible https://github.com/ceph/ceph-ansible

ceph-chef 

puppet-ceph



# 原生部署

部署出来可以生产使用，但是功能没有ansible完整

## 网络规划

osd: 至少3个，则3个节点，每个节点3个盘，2个当作osd,1个系统盘.

mon: 集群监控，实验至少1个，生产至少3个+，奇数节点

mgr: 集群实时状态，实验至少1个，生产至少2个+

mds: 兼容posix文件系统，实验至少1个，生产至少2个+

管理节点：，实验至少1个，生产至少2个

使用7个节点

| 外网ip      | 内网          | 角色    | 角色                                       | 角色  | 角色                    | 磁盘1 | 磁盘2 | 描述             | 主机名                   |
| ----------- | ------------- | ------- | ------------------------------------------ | ----- | ----------------------- | ----- | ----- | ---------------- | ------------------------ |
| 172.29.0.77 | 172.18.199.1  | admin   |                                            |       |                         |       |       | 借助ssh,管理ceph | ceph-admin01.mykernel.cn |
| 172.29.0.78 | 172.18.199.2  | admin   |                                            |       |                         |       |       |                  | ceph-admin02.mykernel.cn |
| 172.29.0.11 | 172.18.199.71 | store01 | mon01 paxos协议协作，避免分区，所以至少3个 | mds01 | mgr01 无状态，2个可冗余 | 80G   | 120G  |                  | store01.mykernel.cn      |
| 172.29.0.12 | 172.18.199.72 | store02 | mon02                                      | mds02 | mgr02                   | 80G   | 120G  |                  | store02.mykernel.cn      |
| 172.29.0.13 | 172.18.199.73 | store03 | mon03                                      |       |                         | 80G   | 120G  |                  | store03.mykernel.cn      |
| 172.29.0.14 | 172.18.199.74 | store04 |                                            |       |                         | 80G   | 120G  |                  | store04.mykernel.cn      |
| 172.29.0.15 | 172.18.199.75 | store05 |                                            |       |                         | 80G   | 120G  |                  | store05.mykernel.cn      |

集群网络

![image-20210224171838799](http://myapp.img.mykernel.cn/image-20210224171838799.png)



## 准备环境

准备这个网络拓扑，当前实验环境是在kvm主机之上

`/etc/rc.local`

```bash
# vmbr2 internal for ceph cluster
# 添加桥接接口
brctl addbr vmbr2
# 配置桥接接口地址，此即为内部通信的网关
ifconfig vmbr2 172.18.0.1/16 up
# 不需要转发，就是需要其不能上外网


# vmbr3 public for ceph cluster
# 添加桥接接口
brctl addbr vmbr3
# 配置桥接接口地址，此即为内部通信的网关
ifconfig vmbr3 172.29.0.1/16 up
# 需要内部通信完成SNAT转换源地址
iptables -t nat -A POSTROUTING -s 172.29.0.0/16 -j MASQUERADE
```

准备主机

```bash
# 准备store 1-5
[root@centos7-iaas kvm]# for i in {01..05}; do bash safe_clone_kvm.sh -i /VMs/template/centos7.5-1804-2C2G.qcow2 -d /VMs/ceph/store${i} -t kvm -n store${i} -r 2048 -v 4 -b vmbr2 -f vmbr3; done


# 准备admin 1-2
[root@centos7-iaas kvm]# for i in {01..02}; do bash safe_clone_kvm.sh -i /VMs/template/centos7.5-1804-2C2G.qcow2 -d /VMs/ceph/admin${i} -t kvm -n admin${i} -r 2048 -v 4 -b vmbr2 -f vmbr3; done
```

配置各主机的ip

```bash
[root@centos7-iaas kvm]# virsh console admin01
localhost login: root
Password: 
Last login: Thu Jan 14 13:43:37 on ttyS0
[root@localhost ~]# 

[root@localhost ~]# cd /etc/sysconfig/network-scripts/
[root@localhost network-scripts]# cat > ifcfg-eth0 <<EOF
TYPE="Ethernet" 
BOOTPROTO="static"    
DEFROUTE="yes"
DEVICE="eth0"
ONBOOT="yes"
IPADDR=172.18.199.1
NETMASK=255.255.0.0
EOF
[root@localhost network-scripts]# cat > ifcfg-eth1 <<EOF
TYPE="Ethernet" 
BOOTPROTO="static"    
DEFROUTE="yes"
DEVICE="eth1"
ONBOOT="yes"
IPADDR=172.29.0.77
NETMASK=255.255.0.0
GATEWAY=172.29.0.1
EOF
[root@localhost network-scripts]# systemctl restart network

reboot

# 测试网络
[root@localhost network-scripts]# ping www.baidu.com
PING www.a.shifen.com (14.215.177.39) 56(84) bytes of data.
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=1 ttl=53 time=32.6 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=2 ttl=53 time=32.5 ms
 
--- www.a.shifen.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms

```

以上admin01示例，完成所有的ip配置

对所有主机添加快照

```bash
# sotre
[root@centos7-iaas ~]# for i in {01..05}; do virsh snapshot-create-as --name NewOS store$i ; done

# admin
[root@centos7-iaas ~]# for i in {01..02}; do virsh snapshot-create-as --name NewOS admin$i ; done
```



启动主机

```bash
[root@centos7-iaas ~]# for i in {01..05}; do virsh start admin$i ; done
域 admin01 已开始

域 admin02 已开始

[root@centos7-iaas ~]# for i in {01..05}; do virsh start store$i ; done
域 store01 已开始

域 store02 已开始

域 store03 已开始

域 store04 已开始

域 store05 已开始

```

添加硬盘

![image-20210224175225118](http://myapp.img.mykernel.cn/image-20210224175225118.png)

![image-20210224175401909](http://myapp.img.mykernel.cn/image-20210224175401909.png)

依次创建出5对磁盘，每对80G/120G

为了确保crush调制，磁盘越多越好，6个更佳。3个太少了。

![image-20210224175630578](http://myapp.img.mykernel.cn/image-20210224175630578.png)

选择disk01

![image-20210224175938015](http://myapp.img.mykernel.cn/image-20210224175938015.png)

![image-20210224175956888](http://myapp.img.mykernel.cn/image-20210224175956888.png)

验证磁盘

![image-20210224180043496](http://myapp.img.mykernel.cn/image-20210224180043496.png)

使用类似方式，添加其他store的磁盘。

各节点配置好环境, firewalld, selinux, hostname, 时间

```bash
# all nodes, 每个节点的主机名不同
# bash linux-temp.sh --hostname=ceph-admin01.mykernel.cn --author=songliangcheng --qq=2192383945 --desc="A test toy" --resourceslimit=1 --kernelparams=1 --basepkgs=1 --chinese=1 --eth0=0 --umirror=1

# all nodes 
# cat > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

172.29.0.11   mon01.mykernel.cn mon01 store01.mykernel.cn store01 mds01 mgr01
172.29.0.12   mon02.mykernel.cn mon02 store02.mykernel.cn store02 mds02 mgr02
172.29.0.13   mon03.mykernel.cn mon03 store03.mykernel.cn store03
172.29.0.14   store04.mykernel.cn store04
172.29.0.15   store05.mykernel.cn store05
172.29.0.16   store06.mykernel.cn store06
EOF
```

> 这个是示例，其他主机的hosts略
>
> 注意，里面是连接外网的ip

各节点配置普通用户及sudo权限

```bash
# all nodes

[root@ceph-admin01 ~]# useradd cephadm && echo magedu | passwd --stdin cephadm
[root@ceph-admin01 ~]# cat > /etc/sudoers.d/cephadm <<EOF
cephadm ALL=(root) NOPASSWD:ALL
EOF
```

> cephadm用户
>
> ALL 所有主机
>
> root 以root用户
>
> NOPASSWD 无密码
>
> ALL 所有命令

准备cephadm免密认证, 仅管理节点

```bash
# 所有节点
[root@ceph-admin01 ~]# su - cephadm
最后一次失败的登录：四 2月 25 08:52:40 CST 2021从 localhostssh:notty 上
最有一次成功登录后有 1 次失败的登录尝试。

# only admi01
[cephadm@ceph-admin01 ~]$ ssh-keygen -t rsa -b 2048 -P '' -f ~/.ssh/id_rsa
[cephadm@ceph-admin01 ~]$ ssh-copy-id cephadm@localhost
```

> ```bash
> # only admin01
> 
> # 复制.ssh到其他主机
> [cephadm@ceph-admin01 ~]$ sudo yum install sshpass -y
> 
> [cephadm@ceph-admin01 ~]$ cat sshpass.sh
> hosts=(
> 172.18.199.1
> 172.18.199.2
> 172.18.199.71
> 172.18.199.72
> 172.18.199.73
> 172.18.199.74
> 172.18.199.75
> )
> for host in ${hosts[@]}; do
> 	sshpass -p "magedu" ssh-copy-id  cephadm@$host  -o StrictHostKeyChecking=no
> 	[ $? -eq 0 ] && echo -e "\033[1;32m$host add success.\033[0m" ||  echo  -e "\033[1;31m$host add failure.\033[0m"
> done
> 
> 
> [cephadm@ceph-admin01 ~]$ bash sshpass.sh 
> 
> ```

## 安装ceph

ceph版本 https://docs.ceph.com/en/latest/releases/

```bash
ACTIVE RELEASES
v15.2.8 Octopus
v15.2.7 Octopus
v15.2.6 Octopus
v15.2.5 Octopus
v15.2.4 Octopus
v15.2.3 Octopus
v15.2.2 Octopus
v15.2.1 Octopus
v15.2.0 Octopus
v14.2.16 Nautilus
v14.2.15 Nautilus
v14.2.14 Nautilus
v14.2.13 Nautilus
v14.2.12 Nautilus
v14.2.11 Nautilus
v14.2.10 Nautilus
v14.2.9 Nautilus
v14.2.8 Nautilus
v14.2.7 Nautilus
v14.2.6 Nautilus
v14.2.5 Nautilus
v14.2.4 Nautilus
v14.2.3 Nautilus
v14.2.2 Nautilus
v14.2.1 Nautilus
v14.2.0 Nautilus
```

aliyun的 ceph仓库https://mirrors.aliyun.com/ceph

```bash
rpm-15.1.0/                                        13-Mar-2020 22:21                   -
rpm-15.1.1/                                        13-Mar-2020 22:23                   -
rpm-15.2.0/                                        24-Mar-2020 14:43                   -
rpm-15.2.1/                                        09-Apr-2020 16:21                   -
rpm-15.2.2/                                        18-May-2020 19:57                   -
rpm-15.2.3/                                        30-May-2020 00:30                   -
rpm-15.2.4/                                        30-Jun-2020 20:32                   -
rpm-15.2.5/                                        16-Sep-2020 13:52                   -
rpm-15.2.6/                                        19-Nov-2020 02:48                   -
rpm-15.2.7/                                        01-Dec-2020 00:53                   -
rpm-15.2.8/                                        17-Dec-2020 01:42                   -
rpm-15.2.9/                                        23-Feb-2021 22:03                   -
rpm-giant/                                         15-Sep-2015 18:58                   -
rpm-hammer/                                        21-Jun-2016 18:21                   -
rpm-infernalis/                                    06-Jan-2016 18:19                   -
rpm-jewel/                                         07-Mar-2017 12:49                   -
rpm-kraken/                                        13-Oct-2016 12:29                   -
rpm-luminous/                                      31-Jan-2020 15:20                   -  # 成熟版本
```

repodata地址 https://mirrors.aliyun.com/ceph/rpm-15.2.9/el7/x86_64/

仅需要安装admin端，其他端通过ceph-deploy来完成

admin端应该是普通用户操作连接每个节点，每个节点应该是普通用户有sudo权限，仅对某些命令有权限



安装ceph源 https://mirrors.aliyun.com/ceph/rpm-15.2.9/el7/noarch/

```bash
# 所有节点安装
sudo rpm -ivh https://mirrors.aliyun.com/ceph/rpm-15.2.9/el7/noarch/ceph-release-1-1.el7.noarch.rpm
sudo wget -O /etc/yum.repos.d/epel-7.repo http://mirrors.aliyun.com/repo/epel-7.repo
```



### 安装ceph-deploy

在管理节点安装ceph-deploy

```bash
[cephadm@ceph-admin01 ~]$ sudo yum install ceph-deploy python-setuptools python2-subprocess32
```

## 部署rados集群

管理节点以**cephadm用户**准备集群配置文件目录

```bash
[cephadm@ceph-admin01 ~]$ mkdir ceph-cluster
[cephadm@ceph-admin01 ~]$ cd ceph-cluster
```

### 添加mon节点

初始化**第一个mon节点**，指定的名称与主机名保持一致

```bash
# only admin01

[cephadm@ceph-admin01 ceph-cluster]$ ls

[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy new --help
usage: ceph-deploy new [-h] [--no-ssh-copykey] [--fsid FSID]
                       [--cluster-network CLUSTER_NETWORK]
                       [--public-network PUBLIC_NETWORK]
                       MON [MON ...]

[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy new  --cluster-network 172.18.0.0/16 --public-network 172.29.0.0/16  store01.mykernel.cn
 
 
[cephadm@ceph-admin01 ceph-cluster]$ tree
.
|-- ceph.conf                         # 配置文件
|-- ceph-deploy-ceph.log
-- ceph.mon.keyring                  # 密钥环

0 directories, 3 files
[cephadm@ceph-admin01 ceph-cluster]$ cat ceph.conf 
[global]
fsid = 2f2d297c-a337-40de-bff9-13d3ea8aae31
public_network = 172.29.0.0/16                # 公网
cluster_network = 172.18.0.0/16               # 集群网络，内网
mon_initial_members = store01                 # 初始化节点，可以逗号分隔多个名称
mon_host = 172.29.0.11                        # store01对应的ip
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx


```

为各节点安装ceph包

```bash
# 此步骤跳过
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy install mon01 mon02 mon03 store04 store05


# 观察结果
[mon03][DEBUG ] ceph version 15.2.9 (357616cbf726abb779ca75a551e8d02568e15b17) octopus (stable)

```

> 为ceph的store节点安装ceph, ceph-radosgw程序包(可以忽略, 在以上步骤自动完成)
>
> ```bash
> # 所有store节点
> sudo rpm -ivh https://mirrors.aliyun.com/ceph/rpm-15.2.9/el7/noarch/ceph-release-1-1.el7.noarch.rpm
> sudo wget -O /etc/yum.repos.d/epel-7.repo http://mirrors.aliyun.com/repo/epel-7.repo
> sudo yum install ceph ceph-radosgw -y
> 
> # admin01
> #ceph-deploy install mon01 mon02 mon03 store04 store05 --no-adjust-repos
> ```

真正初始mon节点, 激活监控器

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy mon create-initial
# create-initial 初始Monitor cluster的第一个节点

# 生成的配置
[cephadm@ceph-admin01 ceph-cluster]$ ll
总用量 80
-rw-------. 1 cephadm cephadm   113 2月  25 11:52 ceph.bootstrap-mds.keyring     # 引导启动mds
-rw-------. 1 cephadm cephadm   113 2月  25 11:52 ceph.bootstrap-mgr.keyring     # ... mgr
-rw-------. 1 cephadm cephadm   113 2月  25 11:52 ceph.bootstrap-osd.keyring     # ... osd
-rw-------. 1 cephadm cephadm   113 2月  25 11:52 ceph.bootstrap-rgw.keyring     # ... rgw
-rw-------. 1 cephadm cephadm   151 2月  25 11:52 ceph.client.admin.keyring      # 客户端的集群管理员密钥
-rw-rw-r--. 1 cephadm cephadm   259 2月  25 10:33 ceph.conf                      # 配置
-rw-rw-r--. 1 cephadm cephadm 50182 2月  25 11:52 ceph-deploy-ceph.log
-rw-------. 1 cephadm cephadm    73 2月  25 10:33 ceph.mon.keyring


# 验证mon
[root@store01 ~]# ps -ef | grep ceph-mon
ceph       15738       1  0 11:52 ?        00:00:00 /usr/bin/ceph-mon -f --cluster ceph --id store01 --setuser ceph --setgroup ceph

# 验证ceph配置
[root@store01 ~]# ls /etc/ceph/
ceph.conf  rbdmap  tmpjXrmUx

# 其他ceph
[root@store02 ~]# ls /etc/ceph/
rbdmap

```

### 准备ceph配置

复制ceph配置文件

```bash
ceph-deploy -h
	admin               Push configuration and client.admin key to a remote host.    # 复制配置、密钥，打算在此处管理集群，才有必要复制。
    config              Copy ceph.conf to/from remote host(s)                        # 复制配置


# 复制admin keyring给每个节点
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy admin mon01 mon02 mon03 store04 store05

# 验证
[root@store01 ~]# ls -l /etc/ceph/
总用量 12
-rw-------. 1 root root 151 2月  25 11:57 ceph.client.admin.keyring                   # 管理员密钥，打算在此处管理集群，才有必要复制。
-rw-r--r--. 1 root root 259 2月  25 11:57 ceph.conf
-rw-r--r--. 1 root root  92 2月  23 22:41 rbdmap
-rw-------. 1 root root   0 2月  25 11:52 tmpjXrmUx
```

在目标节点上配置 cephadm用户对ceph.client.admin.keyring有权限读取

```bash
# 所有store节点
[root@store01 ~]# setfacl -m u:cephadm:r /etc/ceph/ceph.client.admin.keyring 
setfacl -m u:cephadm:r /etc/ceph/ceph.client.admin.keyring  # 2
setfacl -m u:cephadm:r /etc/ceph/ceph.client.admin.keyring  # 3
setfacl -m u:cephadm:r /etc/ceph/ceph.client.admin.keyring  # 4
setfacl -m u:cephadm:r /etc/ceph/ceph.client.admin.keyring  # 5
```

### 添加mgr

配置mgr，(luminious+版本)

集群规模大时，使用专用的节点跑mgr

```bash
# mgr无状态
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy mgr create mgr01 mgr02

# 验证mgr，在71，72
[root@store01 ~]# ps -ef | grep mgr
postfix     1292    1284  0 2月24 ?       00:00:00 qmgr -l -t unix -u
ceph       16185       1 14 12:01 ?        00:00:03 /usr/bin/ceph-mgr -f --cluster ceph --id mgr01 --setuser ceph --setgroup ceph

```

### 集群状态

集群还没有OSD, 但是集群框架有了，检验ceph集群状态，需要ceph程序

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph
-bash: ceph: 未找到命令

# admin01,admin02安装
[cephadm@ceph-admin01 ceph-cluster]$ sudo yum install ceph-common -y
# ceph-common包提供ceph命令，此命令通过  /etc/ceph/ceph.client.admin.keyring 密钥完成管理ceph集群

# 准备ceph密钥
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy admin localhost
[cephadm@ceph-admin01 ceph-cluster]$ sudo setfacl -m u:cephadm:r /etc/ceph/ceph.client.admin.keyring
# 查看集群状态
[cephadm@ceph-admin01 ceph-cluster]$ ceph -s
  cluster:
    id:     2f2d297c-a337-40de-bff9-13d3ea8aae31 # 配置文件中的fsid
    health: HEALTH_WARN # 当前集群是否健康
            Module 'restful' has failed dependency: No module named 'pecan'
            OSD count 0 < osd_pool_default_size 3
 
  services: # 集群部署的服务
    mon: 1 daemons, quorum store01 (age 54m) # paxos协议的quorum，一个mon, 所以就是1个
    mgr: mgr01(active, since 44m), standbys: mgr02 # mgr 2个
    osd: 0 osds: 0 up, 0 in  # 没有osd
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     


# 所有节点安装
[cephadm@ceph-admin01 ~]$  sudo yum -y install python-pecan python-werkzeug
[cephadm@ceph-admin01 ~]$  sudo pip3 install pecan werkzeug cherrypy
[cephadm@ceph-admin01 ~]$ sudo reboot



# vim ceph.conf 
# 添加如下信息：时间同步
mon clock drift allowed = 2    
mon clock drift warn backoff = 30

# 分发所有配置
ceph-deploy --overwrite-conf config push  store{01..05}


# 所有mon上，重启monitor
systemctl restart ceph-mon.target

# 查看ceph状态
[cephadm@ceph-admin01 ceph-cluster]$ ceph -s
  cluster:
    id:     2f2d297c-a337-40de-bff9-13d3ea8aae31
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum store01,store02,store03 (age 2s)
    mgr: mgr01(active, since 5m), standbys: mgr02, store03
    osd: 10 osds: 10 up (since 5m), 10 in (since 53m)
 
  data:
    pools:   2 pools, 33 pgs
    objects: 0 objects, 0 B
    usage:   10 GiB used, 990 GiB / 1000 GiB avail
    pgs:     33 active+clean
```

### 向rados集群添加osd

filestore

blustore, 性能比filestore性能快一倍，L版本之后ceph默认是bluestore

![image-20210225125744201](http://myapp.img.mykernel.cn/image-20210225125744201.png)

> 三类数据，同盘
>
> 元数据 fluefs db wal 
>
> 数据 data blobs
>
> rockdb元数据 flubefsdb



管理远程主机的磁盘

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy disk -h
usage: ceph-deploy disk [-h] {zap,list} ...

Manage disks on a remote host.

positional arguments:
  {zap,list}
    zap       destroy existing data and filesystem on LV or partition
    list      List disk info from remote host(s) # 列出
    
    
# 列出mon01的磁盘
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy disk list mon01


# shell完成擦除所有主机
[cephadm@ceph-admin01 ceph-cluster]$  for i in {01..05}; do ceph-deploy disk zap store${i} /dev/vdc; done
[cephadm@ceph-admin01 ceph-cluster]$  for i in {01..05}; do ceph-deploy disk zap store${i} /dev/vdb; done


```

添加OSD

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy osd -h
usage: ceph-deploy osd [-h] {list,create} ...

Create OSDs from a data disk on a remote host:

    ceph-deploy osd create {node} --data /path/to/device

For bluestore, optional devices can be used::
									# 数据             # 块数据 rocksdb, 元数据
    ceph-deploy osd create {node} --data /path/to/data --block-db /path/to/db-device
													  # wal    数据二进制日志位置
	ceph-deploy osd create {node} --data /path/to/data --block-wal /path/to/wal-device
    ceph-deploy osd create {node} --data /path/to/data --block-db /path/to/db-device --block-wal /path/to/wal-device

For filestore, the journal must be specified, as well as the objectstore::
												# 数据                  # 文件系统日志
    ceph-deploy osd create {node} --filestore --data /path/to/data --journal /path/to/journal

For data devices, it can be an existing logical volume in the format of:
vg/lv, or a device. For other OSD components like wal, db, and journal, it
can be logical volume (in vg/lv format) or it must be a GPT partition.

positional arguments:
  {list,create}
    list         List OSD info from remote host(s)
    create       Create new Ceph OSD daemon by preparing and activating a device
```

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy osd create --data /dev/vdb  store01

# 检验
[cephadm@ceph-admin01 ceph-cluster]$ ceph -s

    osd: 1 osds: 1 up (since 10s), 1 in (since 10s) # 一个osd ，当前up, 可以在线读写数据
 
  data:
    pools:   1 pools, 1 pgs # 一个Pools, pgs
    objects: 0 objects, 0 B
    usage:   1.0 GiB used, 79 GiB / 80 GiB avail
    pgs:     100.000% pgs not active
             1 undersized+peered


# shell 添加其他osd
[cephadm@ceph-admin01 ceph-cluster]$ for i in {02..05}; do ceph-deploy osd create --data /dev/vdb  store${i}; done

[cephadm@ceph-admin01 ceph-cluster]$ for i in {01..05}; do ceph-deploy osd create --data /dev/vdc  store${i}; done

# 查看ceph状态
[cephadm@ceph-admin01 ceph-cluster]$ ceph -s
  cluster:
    id:     b68bde4e-a661-47d3-b078-4b58f7eae0f5
    health: HEALTH_OK # 正常
 
  services:
    mon: 1 daemons, quorum store01 (age 18m) # rest协议，奇数节点，拥有合法票数在mon01
    mgr: mgr02(active, since 40m), standbys: mgr01 # 无状态的mgr, 主是mgr02
    osd: 10 osds: 10 up (since 4m), 10 in (since 4m) # 一共10个OSD
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   10 GiB used, 990 GiB / 1000 GiB avail
    pgs:     1 active+clean
```

#### 检验osd

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy osd list store{01..05}
```

#### 移除osd

```bash
# 停用
ceph osd out {osd-num}

# 停止进程
sudo system stop ceph-osd@{osd-num}

# 移除
osd purge {id} --yes-i-relly-mean-it

# 清理配置段
[osd.1]
	host = {hostname}
```

Luminous之前版本, ....

### 测试上传和下载

```bash
# 存储池
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool -h
osd pool create <pool> [<pg_num:int>] [<pgp_num:int>] [replicated|erasure] [<erasure_code_profile>] [<rule>] [<expected_num_objects:int>] [<size:int>] [<pg_  create pool
 num_min:int>] [on|off|warn] [<target_size_bytes:int>] [<target_size_ratio:float>]  
 
# ceph是高级命令，rados更底层的命令
[cephadm@ceph-admin01 ceph-cluster]$ rados -h
POOL COMMANDS
   lspools                          list pools
   cppool <pool-name> <dest-pool>   copy content of a pool
   purge <pool-name> --yes-i-really-really-mean-it
                                    remove all objects from pool <pool-name> without removing it
   df                               show per-pool and total usage
   ls                               list objects in pool


```

创建存储池

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool create mypool 32 32
pool 'mypool' created

pg数量 32
pgp数量一般与pg数量一致 32
```

显示存储池

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool ls
device_health_metrics #内置
mypool

[cephadm@ceph-admin01 ceph-cluster]$ rados lspools
device_health_metrics
mypool

```

现在没有rdb, radosgw, cephfs接口，只能原始接口

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rados put day.jpg /usr/share/backgrounds/day.jpg --pool mypool
[cephadm@ceph-admin01 ceph-cluster]$ 
```

> `put <id> 路径 -p|--pool 存储池名` 

列出存储池中的文件

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rados ls -p mypool
day.jpg
```

下载

```bash

```

显示元数据

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd map mypool day.jpg
osdmap e68 pool 'mypool' (2) object 'day.jpg' -> pg 2.a15eeea3 (2.3) -> up ([9,8,1], p9) acting ([9,8,1], p9)
												   #存储池编号.pg位图           #数据3份，分别在9号/8号/1号的OSD上。     #acting 即所有OSD上活动
```

删除文件

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rados rm day.jpg -p mypool

# 检验
rados ls -p mypool
```

删除存储池

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool rm mypool mypool --yes-i-really-really-mean-it
Error EPERM: pool deletion is disabled; you must first set the mon_allow_pool_delete config option to true before you can destroy a pool

```

## 扩展ceph集群

### 添加监控节点

遵循paxos协议，奇数节点

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy mon add  store02 
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy mon add  store03


# 结果显示监视器
[store03][DEBUG ]     "mons": [
[store03][DEBUG ]       {
[store03][DEBUG ]         "addr": "172.29.0.11:6789/0", 
[store03][DEBUG ]         "name": "store01", 
[store03][DEBUG ]         "priority": 0, 
[store03][DEBUG ]         "public_addr": "172.29.0.11:6789/0", 
[store03][DEBUG ]         "public_addrs": {
[store03][DEBUG ]           "addrvec": [
[store03][DEBUG ]             {
[store03][DEBUG ]               "addr": "172.29.0.11:3300", 
[store03][DEBUG ]               "nonce": 0, 
[store03][DEBUG ]               "type": "v2"
[store03][DEBUG ]             }, 
[store03][DEBUG ]             {
[store03][DEBUG ]               "addr": "172.29.0.11:6789", 
[store03][DEBUG ]               "nonce": 0, 
[store03][DEBUG ]               "type": "v1"
[store03][DEBUG ]             }
[store03][DEBUG ]           ]
[store03][DEBUG ]         }, 
[store03][DEBUG ]         "rank": 0, 
[store03][DEBUG ]         "weight": 0
[store03][DEBUG ]       }, 
[store03][DEBUG ]       {
[store03][DEBUG ]         "addr": "172.29.0.12:6789/0", 
[store03][DEBUG ]         "name": "store02", 
[store03][DEBUG ]         "priority": 0, 
[store03][DEBUG ]         "public_addr": "172.29.0.12:6789/0", 
[store03][DEBUG ]         "public_addrs": {
[store03][DEBUG ]           "addrvec": [
[store03][DEBUG ]             {
[store03][DEBUG ]               "addr": "172.29.0.12:3300", 
[store03][DEBUG ]               "nonce": 0, 
[store03][DEBUG ]               "type": "v2"
[store03][DEBUG ]             }, 
[store03][DEBUG ]             {
[store03][DEBUG ]               "addr": "172.29.0.12:6789", 
[store03][DEBUG ]               "nonce": 0, 
[store03][DEBUG ]               "type": "v1"
[store03][DEBUG ]             }
[store03][DEBUG ]           ]
[store03][DEBUG ]         }, 
[store03][DEBUG ]         "rank": 1, 
[store03][DEBUG ]         "weight": 0
[store03][DEBUG ]       }, 
[store03][DEBUG ]       {
[store03][DEBUG ]         "addr": "172.29.0.13:6789/0", 
[store03][DEBUG ]         "name": "store03", 
[store03][DEBUG ]         "priority": 0, 
[store03][DEBUG ]         "public_addr": "172.29.0.13:6789/0", 
[store03][DEBUG ]         "public_addrs": {
[store03][DEBUG ]           "addrvec": [
[store03][DEBUG ]             {
[store03][DEBUG ]               "addr": "172.29.0.13:3300", 
[store03][DEBUG ]               "nonce": 0, 
[store03][DEBUG ]               "type": "v2"
[store03][DEBUG ]             }, 
[store03][DEBUG ]             {
[store03][DEBUG ]               "addr": "172.29.0.13:6789", 
[store03][DEBUG ]               "nonce": 0, 
[store03][DEBUG ]               "type": "v1"
[store03][DEBUG ]             }
[store03][DEBUG ]           ]
[store03][DEBUG ]         }, 
[store03][DEBUG ]         "rank": 2, 
[store03][DEBUG ]         "weight": 0
[store03][DEBUG ]       }
[store03][DEBUG ]     ]

```

查看mon选举结果

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph quorum_status --format json-pretty
"quorum_names": [
"store01",
"store02",
"store03"
],
"quorum_leader_name": "store01",

```

显示集群状态

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph -s
  cluster:
    id:     b68bde4e-a661-47d3-b078-4b58f7eae0f5
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum store01,store02,store03 (age 35s) # 3个mon
    mgr: mgr02(active, since 47m), standbys: mgr01  # 2个mgr
    osd: 10 osds: 10 up (since 11m), 10 in (since 11m) # 10个Osd
 
  task status:
 
  data:
    pools:   2 pools, 33 pgs
    objects: 0 objects, 0 B
    usage:   10 GiB used, 990 GiB / 1000 GiB avail
    pgs:     33 active+clean

```



### 添加mgr

无状态的web服务，所以2个即可

有性能压力，可以给3个

```bash
[cephadm@ceph-admin01 ceph-cluster]$  ceph-deploy mgr create store03
```

验证mgr

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph -s
  cluster:
    id:     2f2d297c-a337-40de-bff9-13d3ea8aae31
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum store01,store02,store03 (age 2s)
    mgr: mgr01(active, since 5m), standbys: mgr02, store03
    osd: 10 osds: 10 up (since 5m), 10 in (since 53m)
 
  data:
    pools:   2 pools, 33 pgs
    objects: 0 objects, 0 B
    usage:   10 GiB used, 990 GiB / 1000 GiB avail
    pgs:     33 active+clean

```



### ceph上层访问接口

ceph有3类接口

rados客户端抽象成：radosgw, rbd, cephfs

无论哪种都需要与存储池打交道

#### 提供rbd接口

```bash
# 客户端需要配置

# 服务端只需要存储池，块设备在存储池中表现为Image, 即存储空间
```

创建存储池

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool create rbdpool 64 64
pool 'rbdpool' created

```

> rbdpool 存储池
>
> pg数量小，报警
>
> pgp和pg一般相同

存储池当作rbd使用，需要初始化

```bash
# 启用
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool application enable rbdpool rbd
enabled application 'rbd' on pool 'rbdpool'
# 禁用
#ceph osd pool application disable rbdpool rbd
```

初始化

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rbd -h

[cephadm@ceph-admin01 ceph-cluster]$ rbd pool init -p rbdpool
```

> rbdpool是存储池，是空间，可以其他创建许多镜像，每个镜像就是一个rbd

创建镜像

```bash
[cephadm@ceph-admin01 ceph-cluster]$ rbd create  --size 2G rbdpool/myimg


# rbd craete --pool kube --image vol01 --size 2G
```

> 大小
>
> 池/镜像名

```bash
# 列出镜像
[cephadm@ceph-admin01 ceph-cluster]$ rbd ls -p rbdpool
myimg

# 1. 镜像文件可以在客户端通过内核访问，识别为/dev/sdx
# 2. 基于用户空间文件系统挂载

# 查看img信息
[cephadm@ceph-admin01 ceph-cluster]$ rbd info rbdpool/myimg
rbd image 'myimg':
	size 2 GiB in 512 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: ac5bcd74041b
	block_name_prefix: rbd_data.ac5bcd74041b
	format: 2
				分层      排他                           
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	op_features: 
	flags: 
	create_timestamp: Thu Feb 25 14:24:17 2021
	access_timestamp: Thu Feb 25 14:24:17 2021
	modify_timestamp: Thu Feb 25 14:24:17 2021
```

删除镜像

```bash
rbd rm -h

# 删除
rbd trash move kube/vol01
rbd ls -p kube -l
# 列出回收站
rbd trash list -p kube
-000000000 vol01

# 恢复
rbd trash restore  -p kube --image vol01 --image-id -000000000

# 移除
rbd trash remove
rbd trash purge
```



快照

```bash
snap create
```

克隆

```bash

```



#### 提供radosgw接口

一般生产使用哪个接口，才有必要启用

需要自已的存储池

自动初始化存储池，但是需要一个专用的守护进程rgw, 是客户端组件。

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy rgw create store01
[ceph_deploy.rgw][INFO  ] The Ceph Object Gateway (RGW) is now running on host store01 and default port 7480 # 提供web


# 检验 only store01有
[root@store01 ~]# ps -ef | grep rgw
root        2025       1  0 14:28 ?        00:00:00 /usr/bin/radosgw -f --cluster ceph --name client.rgw.store01 --setuser ceph --setgroup ceph

```

浏览器直接访问 http://172.29.0.11:7480/

public network访问的

```bash
<ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"> # 模拟亚马逊的对象存储
<Owner>
<ID>anonymous</ID>
<DisplayName/>
</Owner>
<Buckets/>
</ListAllMyBucketsResult>
```

查看集群状态

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph -s
  cluster:
    id:     b68bde4e-a661-47d3-b078-4b58f7eae0f5
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum store01,store02,store03 (age 5m)
    mgr: mgr02(active, since 52m), standbys: mgr01, store03
    osd: 10 osds: 10 up (since 16m), 10 in (since 16m)
    rgw: 1 daemon active (store01)  # 一个daemon, 冗余，至少2个
 
  task status:
 
  data:
    pools:   7 pools, 213 pgs
    objects: 192 objects, 4.9 KiB
    usage:   10 GiB used, 990 GiB / 1000 GiB avail
    pgs:     1.408% pgs not active
             210 active+clean
             2   peering
             1   activating
 
  progress:
    PG autoscaler decreasing pool 7 PGs from 32 to 8 (60s)
      [===========.................] (remaining: 84s)



```

会自动创建一系列存储池

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool ls
device_health_metrics
mypool
rbdpool
################ 以下自动创建的 ################
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
```

##### 高可用radosgw

一个成为单点，所以步骤2个

```bash
# 添加一个daemon
ceph-deploy rgw create store02

# 2个daemon
[cephadm@ceph-admin01 ceph-cluster]$ ceph -s
    rgw: 2 daemons active (store01, store02)

```

验证进程

```bash
[root@store01 ~]# lsof -ni:7480
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
radosgw 2898 ceph   61u  IPv4  30887      0t0  TCP *:7480 (LISTEN)
radosgw 2898 ceph   62u  IPv6  30889      0t0  TCP *:7480 (LISTEN)
```

```bash
[root@store01 ~]# curl 172.29.0.11:7480
<?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Owner><ID>anonymous</ID><DisplayName></DisplayName></Owner><Buckets></Buckets></ListAllMyBucketsResult>[root@store01 ~]# 


[root@store01 ~]# curl 172.29.0.12:7480
<?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Owner><ID>anonymous</ID><DisplayName></DisplayName></Owner><Buckets></Buckets></ListAllMyBucketsResult>[root@store01 ~]# 
```

##### 修改监听端口

```bash
[cephadm@ceph-admin01 ceph-cluster]$ cat ceph.conf
[global]
fsid = b68bde4e-a661-47d3-b078-4b58f7eae0f5
public_network = 172.29.0.0/16
cluster_network = 172.18.0.0/16
mon_initial_members = store01
mon_host = 172.29.0.11
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

mon clock drift allowed = 2    
mon clock drift warn backoff = 30

[mds.store03]
mds_standby_for_name = mds01 # store03的备用为mds01
mds_standby_replay = true    # 打开replay模式，当store03写入内存元数据后，会立即同步至mds02, 这样在store03宕机时，mds01可以立即顶替。否则此选项为false, 则需要等待一段时间。
[mds.store02]
mds_standby_for_name = mds02 # store02的备用为mds02
mds_standby_replay = true



[client.rgw.store01]
rgw_host = store01
rgw_frontends = "civetweb port=8080"

[client.rgw.store02]
rgw_host = store02
rgw_frontends = "civetweb port=8080"
```

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy --overwrite-conf config   push localhost store{01..05}
```

```bash
# 在store01, store02上重启
[root@store01 ~]# systemctl restart ceph-radosgw@rgw.store01
[root@store02 ~]#  systemctl restart ceph-radosgw@rgw.store02
[root@store02 ~]# lsof -ni:8080
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
radosgw 39981 ceph   58u  IPv4 366054      0t0  TCP *:webcache (LISTEN)
```

##### 添加https协议

```bash
install -dv /etc/ceph/ssl/
cd /etc/ceph/ssl/
openssl genrsa -out  civetweb.key 2048
openssl req -new -x509 -key civetweb.key -out civetweb.crt -days 3650 -subj "/CN=store01.mykernel.io"
cat civetweb.key civetweb.crt > civetweb.pem

```

```diff
[root@store01 ssl]# cat /etc/ceph/ceph.conf
[global]
fsid = b68bde4e-a661-47d3-b078-4b58f7eae0f5
public_network = 172.29.0.0/16
cluster_network = 172.18.0.0/16
mon_initial_members = store01
mon_host = 172.29.0.11
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

mon clock drift allowed = 2    
mon clock drift warn backoff = 30

[mds.store03]
mds_standby_for_name = mds01 # store03的备用为mds01
mds_standby_replay = true    # 打开replay模式，当store03写入内存元数据后，会立即同步至mds02, 这样在store03宕机时，mds01可以立即顶替。否则此选项为false, 则需要等待一段时间。
[mds.store02]
mds_standby_for_name = mds02 # store02的备用为mds02
mds_standby_replay = true



[client.rgw.store01]
rgw_host = store01
+rgw_frontends = "civetweb port=8443s ssl_certificate=/etc/ceph/ssl/civetweb.pem" 

[client.rgw.store02]
rgw_host = store02
rgw_frontends = "civetweb port=8080"

```

```bash
[root@store01 ssl]# systemctl restart ceph-radosgw@rgw.store01
[root@store01 ssl]# lsof -ni:8443
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
radosgw 27116 ceph   58u  IPv4 313084      0t0  TCP *:pcsync-https (LISTEN)
[root@store01 ssl]# curl -k https://172.29.0.11:8443
<?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Owner><ID>anonymous</ID><DisplayName></DisplayName></Owner><Buckets></Buckets></ListAllMyBucketsResult>[root@store01 ssl]# 

```

##### http+https

```diff
[root@store01 ssl]# cat /etc/ceph/ceph.conf
[global]
fsid = b68bde4e-a661-47d3-b078-4b58f7eae0f5
public_network = 172.29.0.0/16
cluster_network = 172.18.0.0/16
mon_initial_members = store01
mon_host = 172.29.0.11
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

mon clock drift allowed = 2    
mon clock drift warn backoff = 30

[mds.store03]
mds_standby_for_name = mds01 # store03的备用为mds01
mds_standby_replay = true    # 打开replay模式，当store03写入内存元数据后，会立即同步至mds02, 这样在store03宕机时，mds01可以立即顶替。否则此选项为false, 则需要等待一段时间。
[mds.store02]
mds_standby_for_name = mds02 # store02的备用为mds02
mds_standby_replay = true



[client.rgw.store01]
rgw_host = store01
+rgw_frontends = "civetweb port=7480+8443s ssl_certificate=/etc/ceph/ssl/civetweb.pem" 

[client.rgw.store02]
rgw_host = store02
rgw_frontends = "civetweb port=8080"

```

```bash
[root@store01 ssl]# systemctl restart ceph-radosgw@rgw.store01

[root@store01 ssl]# lsof -ni:8443 -ni:7480
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
radosgw 27745 ceph   58u  IPv4 316060      0t0  TCP *:7480 (LISTEN)
radosgw 27745 ceph   59u  IPv4 316061      0t0  TCP *:pcsync-https (LISTEN)

```

##### 调整并发

```diff
"/etc/ceph/ceph.conf" 30L, 941C 已写入                                                                                                                                                                                                                                                                   
[root@store01 ssl]# cat /etc/ceph/ceph.conf
[global]
fsid = b68bde4e-a661-47d3-b078-4b58f7eae0f5
public_network = 172.29.0.0/16
cluster_network = 172.18.0.0/16
mon_initial_members = store01
mon_host = 172.29.0.11
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

mon clock drift allowed = 2    
mon clock drift warn backoff = 30

[mds.store03]
mds_standby_for_name = mds01 # store03的备用为mds01
mds_standby_replay = true    # 打开replay模式，当store03写入内存元数据后，会立即同步至mds02, 这样在store03宕机时，mds01可以立即顶替。否则此选项为false, 则需要等待一段时间。
[mds.store02]
mds_standby_for_name = mds02 # store02的备用为mds02
mds_standby_replay = true



[client.rgw.store01]
rgw_host = store01
+rgw_frontends = "civetweb port=7480+8443s ssl_certificate=/etc/ceph/ssl/civetweb.pem num_threads=2000"

[client.rgw.store02]
rgw_host = store02
rgw_frontends = "civetweb port=8080"
```

通过压测`http(s)://bucket-name.rgw_host:rgw_port/images/obj1.jpg` 测试最大多少响应正常`fio`命令即可完成。

images/obj1.jpg就是对象标识, images解析为目录



##### 泛域名解析

由于bucket-name为rgw的前缀，所以需要配置泛域名解析, 此处假设在`172.29.0.11`主机上搭建dns

```bash
[root@store01 ~]# yum install bind-utils bind
```

配置文件`/etc/named.conf`

```diff
-	listen-on port 53 { 127.0.0.1; };
-	listen-on-v6 port 53 { ::1; };
+	listen-on port 53 { any; };
+	//listen-on-v6 port 53 { ::1; };
 	directory 	"/var/named";
 	dump-file 	"/var/named/data/cache_dump.db";
 	statistics-file "/var/named/data/named_stats.txt";
 	memstatistics-file "/var/named/data/named_mem_stats.txt";
 	recursing-file  "/var/named/data/named.recursing";
 	secroots-file   "/var/named/data/named.secroots";
-	allow-query     { localhost; };
+	allow-query     { any; };
-	dnssec-enable yes;
-	dnssec-validation yes;
+	//dnssec-enable yes;
+	//dnssec-validation yes;
 
 	/* Path to ISC DLV key */
-	bindkeys-file "/etc/named.root.key";
+	//bindkeys-file "/etc/named.root.key";
 
-	managed-keys-directory "/var/named/dynamic";
+	//managed-keys-directory "/var/named/dynamic";
```

`/etc/named.rfc1912.zones`

```bash
zone "mykernel.io" IN {
	type master;
	file "mykernel.io.zone";
};
```

```bash
[root@store01 ~]# named-checkconf
```

`/var/named/mykernel.io.zone` 配置文件

```bash
[root@store01 ~]# touch /var/named/mykernel.io.zone
[root@store01 ~]# chown root.named /var/named/mykernel.io.zone

```

```bash
$TTL 1D
@   IN  SOA ns.mykernel.io. admin.mykernel.io.  (
    0   ;serial
    1D  ;refresh
    1H  ;retry
    1W  ;expire
    3H  ;minimum
    )
            	IN  NS  	ns
ns          	IN  A   	172.29.0.11
store01     	IN  A   	172.29.0.11
store02     	IN  A   	172.29.0.12
*.store01       IN  CNAME   store01
*.store02       IN  CNAME   store02
```

```bash
[root@store01 ~]# named-checkzone mykernel.io /var/named/mykernel.io.zone
zone mykernel.io/IN: loaded serial 0
OK
```

```bash
[root@store01 ~]# systemctl start named
```

客户端测试

```bash
root@ceph-client:~# host -t A images.store01.mykernel.io 
images.store01.mykernel.io is an alias for store01.mykernel.io.
store01.mykernel.io has address 172.29.0.11
```

配置ceph.conf

```diff
[cephadm@ceph-admin01 ceph-cluster]$ cat ceph.conf
[global]
fsid = b68bde4e-a661-47d3-b078-4b58f7eae0f5
public_network = 172.29.0.0/16
cluster_network = 172.18.0.0/16
mon_initial_members = store01
mon_host = 172.29.0.11
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

mon clock drift allowed = 2    
mon clock drift warn backoff = 30

[mds.store03]
mds_standby_for_name = mds01 # store03的备用为mds01
mds_standby_replay = true    # 打开replay模式，当store03写入内存元数据后，会立即同步至mds02, 这样在store03宕机时，mds01可以立即顶替。否则此选项为false, 则需要等待一段时间。
[mds.store02]
mds_standby_for_name = mds02 # store02的备用为mds02
mds_standby_replay = true



[client.rgw.store01]
rgw_host = store01
+rgw_frontends = "civetweb port=7480 num_threads=500 request_timeout_ms=60000"
+rgw_dns_name = store01.mykernel.io

[client.rgw.store02]
rgw_host = store02
+rgw_frontends = "civetweb port=7480 num_threads=500 request_timeout_ms=60000"
+rgw_dns_name = store02.mykernel.io
```

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy --overwrite-conf config   push localhost store{01..05}
```

```bash
# on store01, rgw守护进程运行在其上
systemctl restart ceph-radosgw@rgw.store01
# on store02
systemctl restart ceph-radosgw@rgw.store02
```











#### 提供cephfs接口

额外mds守护进程：缓存+存储池(metadata pool)

兼容posix文件系统接口，用户访问文件时，先联系mds守护进程查询文件元数据。数据是另一个存储池 data pool

mds一个成为瓶颈时，需要提升多个

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy mds create store02

# 检验
[root@store02 ~]# ps -ef | grep mds
ceph        1981       1  0 14:36 ?        00:00:00 /usr/bin/ceph-mds -f --cluster ceph --id store02 --setuser ceph --setgroup ceph


# 检验
[cephadm@ceph-admin01 ceph-cluster]$ ceph -s
  cluster:
    id:     2f2d297c-a337-40de-bff9-13d3ea8aae31
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum store01,store02,store03 (age 22m)
    mgr: mgr01(active, since 27m), standbys: mgr02, store03
    mds:  1 up:standby               # 一个mds
    osd: 10 osds: 10 up (since 27m), 10 in (since 75m)
    rgw: 1 daemon active (store01)
 
  task status:
 
  data:
    pools:   7 pools, 201 pgs
    objects: 192 objects, 4.9 KiB
    usage:   10 GiB used, 990 GiB / 1000 GiB avail
    pgs:     201 active+clean

# 至少2个mds
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy mds create store03


[cephadm@ceph-admin01 ceph-cluster]$ ceph mds stat
 2 up:standby
```

创建元数据池, 创建数据池

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool create cephfs-metadata 32 32
pool 'cephfs-metadata' created
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool create cephfs-data 64 64
pool 'cephfs-data' created

# 状态
[cephadm@ceph-admin01 ceph-cluster]$ ceph -s
  cluster:
    id:     2f2d297c-a337-40de-bff9-13d3ea8aae31
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum store01,store02,store03 (age 24m)
    mgr: mgr01(active, since 29m), standbys: mgr02, store03
    mds:  1 up:standby
    osd: 10 osds: 10 up (since 29m), 10 in (since 77m)
    rgw: 1 daemon active (store01)
 
  task status:
 
  data:
    pools:   9 pools, 297 pgs # 9个pool

```

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph fs new cephfs cephfs-metadata cephfs-data
new fs with metadata pool 8 and data pool 9
```

```bash
# 状态
[cephadm@ceph-admin01 ceph-cluster]$ ceph fs status cephfs
cephfs - 0 clients
======
RANK  STATE     MDS       ACTIVITY     DNS    INOS  
 0    active  store02  Reqs:    0 /s    10     13   
      POOL         TYPE     USED  AVAIL  
cephfs-metadata  metadata  1536k   312G  
  cephfs-data      data       0    312G  
MDS version: ceph version 15.2.9 (357616cbf726abb779ca75a551e8d02568e15b17) octopus (stable)
```

##### 高可用mds

rank 2个。各一reply从。

```diff
+# 查看cephfs状态
+[root@ceph-admin01 ~]# ceph fs status cephfs
cephfs - 0 clients
======
RANK  STATE     MDS       ACTIVITY     DNS    INOS  
+ 0    active  store03  Reqs:    0 /s    10     13    # 1个rank, 一个活动store03
      POOL         TYPE     USED  AVAIL  
cephfs-metadata  metadata  1555k   307G  
  cephfs-data      data       0    307G  
STANDBY MDS  
+  store02    # 一个备用store02.    
MDS version: ceph version 15.2.10 (27917a557cca91e4da407489bbaa64ad4352cc02) octopus (stable)
[root@ceph-admin01 ~]# ceph fs get cephfs
Filesystem 'cephfs' (1)
fs_name	cephfs
epoch	5
flags	12
created	2021-03-30T15:28:26.771832+0800
modified	2021-03-30T15:28:27.781302+0800
tableserver	0
root	0
session_timeout	60
session_autoclose	300
max_file_size	1099511627776
min_compat_client	0 (unknown)
last_failure	0
last_failure_osd_epoch	0
compat	compat={},rocompat={},incompat={1=base v0.20,2=client writeable ranges,3=default file layouts on dirs,4=dir inode in separate object,5=mds uses versioned encoding,6=dirfrag is stored in omap,8=no anchor table,9=file layout v2,10=snaprealm v2}
max_mds	1
in	0
up	{0=44195}
failed	
damaged	
stopped	
data_pools	[9]
metadata_pool	8
inline_data	disabled
balancer	
standby_count_wanted	1
[mds.store03{0:44195} state up:active seq 13 addr [v2:172.29.0.13:6808/728588292,v1:172.29.0.13:6809/728588292]]

[root@ceph-admin01 ~]# ceph mds stat
cephfs:1 {0=store03=up:active} 1 up:standby

```

配置rank为2个: 0 1

```diff
# 配置rank
+# ceph fs set cephfs max_mds 2

# 验证
[root@ceph-admin01 ~]# ceph fs status
cephfs - 0 clients
======
RANK  STATE     MDS       ACTIVITY     DNS    INOS  
+ 0    active  store03  Reqs:    0 /s    10     13     # 活跃
+ 1    active  store02  Reqs:    0 /s    10     13     # 活跃
      POOL         TYPE     USED  AVAIL  
cephfs-metadata  metadata  2707k   307G  
  cephfs-data      data       0    307G  
MDS version: ceph version 15.2.10 (27917a557cca91e4da407489bbaa64ad4352cc02) octopus (stable)

```

配置2个rank各有一个备用mds

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy --overwrite-conf config   push  mds01 mds02 store03 store04

[cephadm@ceph-admin01 ceph-cluster]$ ceph-deploy mds create mds01 mds02 store03 store04

# 查看mds节点
[cephadm@ceph-admin01 ceph-cluster]$ ceph fs status
cephfs - 0 clients
======
RANK  STATE     MDS       ACTIVITY     DNS    INOS  
 0    active  store03  Reqs:    0 /s    10     13   
 1    active  store02  Reqs:    0 /s    10     13   
      POOL         TYPE     USED  AVAIL  
cephfs-metadata  metadata  2707k   307G  
  cephfs-data      data       0    307G  
STANDBY MDS  
   mds01     
   mds02     
MDS version: ceph version 15.2.10 (27917a557cca91e4da407489bbaa64ad4352cc02) octopus (stable)

```

```bash
/home/cephadm/ceph-cluster/ceph.conf
[cephadm@ceph-admin01 ceph-cluster]$ cat ceph.conf
[global]
fsid = b68bde4e-a661-47d3-b078-4b58f7eae0f5
public_network = 172.29.0.0/16
cluster_network = 172.18.0.0/16
mon_initial_members = store01
mon_host = 172.29.0.11
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

mon clock drift allowed = 2    
mon clock drift warn backoff = 30

[mds.store03]
mds_standby_for_name = mds01 # store03的备用为mds01
mds_standby_replay = true    # 打开replay模式，当store03写入内存元数据后，会立即同步至mds02, 这样在store03宕机时，mds01可以立即顶替。否则此选项为false, 则需要等待一段时间。
[mds.store02]
mds_standby_for_name = mds02 # store02的备用为mds02
mds_standby_replay = true


# 推送
ceph-deploy --overwrite-conf config   push  mds01 mds02 store03 store04
```



## ceph常用命令

### 集群状态

```bash
 [cephadm@ceph-admin01 ceph-cluster]$ ceph -s
  cluster:
    id:     2f2d297c-a337-40de-bff9-13d3ea8aae31                  # 集群ID
    health: HEALTH_OK                                             # 集群运行状态
 
  services:
    mon: 3 daemons, quorum store01,store02,store03 (age 28m)      # 监控器版本，仲裁状态
    mgr: mgr01(active, since 33m), standbys: mgr02, store03       # 
    mds: cephfs:1 {0=store02=up:active}
    osd: 10 osds: 10 up (since 33m), 10 in (since 81m)            # osd版本，状态
    rgw: 1 daemon active (store01)
 
  task status:
 
  data:
    pools:   9 pools, 297 pgs                                     # 归置组，存储池
    objects: 214 objects, 7.1 KiB
    usage:   10 GiB used, 990 GiB / 1000 GiB avail
    pgs:     297 active+clean

```

### pg状态  

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph pg stat
297 pgs: 297 active+clean; 7.1 KiB data, 314 MiB used, 990 GiB / 1000 GiB avail
总pg      pg状态               一共数据       使用的数据，是bluestore的rocksdb占用， 总空间
```

### 存储池状态 

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool stats
pool device_health_metrics id 1
  nothing is going on

pool mypool id 2
  nothing is going on

pool rbdpool id 3
  nothing is going on

pool .rgw.root id 4
  nothing is going on

pool default.rgw.log id 5
  nothing is going on

pool default.rgw.control id 6
  nothing is going on

pool default.rgw.meta id 7
  nothing is going on

pool cephfs-metadata id 8
  nothing is going on

pool cephfs-data id 9
  nothing is going on
  
  
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd pool stats cephfs-data
pool cephfs-data id 9
  nothing is going on

```

### 显示ceph存储空间

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph df
--- RAW STORAGE --- # 全局
CLASS  SIZE      AVAIL    USED     RAW USED  %RAW USED
hdd    1000 GiB  990 GiB  314 MiB    10 GiB       1.03
TOTAL  1000 GiB  990 GiB  314 MiB    10 GiB       1.03
 
--- POOLS ---
POOL                   ID  PGS  STORED   OBJECTS  USED     %USED  MAX AVAIL
device_health_metrics   1    1      0 B        0      0 B      0    312 GiB
mypool                  2   32      0 B        0      0 B      0    312 GiB
rbdpool                 3   64    197 B        5  576 KiB      0    312 GiB
.rgw.root               4   32  1.3 KiB        4  768 KiB      0    312 GiB
default.rgw.log         5   32  3.4 KiB      175    6 MiB      0    312 GiB
default.rgw.control     6   32      0 B        8      0 B      0    312 GiB
default.rgw.meta        7    8      0 B        0      0 B      0    312 GiB
cephfs-metadata         8   32  2.2 KiB       22  1.5 MiB      0    312 GiB
cephfs-data             9   64      0 B        0      0 B      0    312 GiB
```

> RAW USED         已用原始存储容量
>
> %RAW USED      已用原始存储量百分比

### osd 和mon状态

osd

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd stat
10 osds: 10 up (since 39m), 10 in (since 87m); epoch: e243

[cephadm@ceph-admin01 ceph-cluster]$ ceph osd dump


# CRUSH 地址中的位置查看OSD
# 节点对应的osd
[cephadm@ceph-admin01 ceph-cluster]$ ceph osd tree
ID   CLASS  WEIGHT   TYPE NAME         STATUS  REWEIGHT  PRI-AFF
 -1         0.97641  root default                               
 -3         0.19528      host store01                           
  0    hdd  0.07809          osd.0         up   1.00000  1.00000
  3    hdd  0.11719          osd.3         up   1.00000  1.00000
 -5         0.19528      host store02                           
  1    hdd  0.07809          osd.1         up   1.00000  1.00000
  4    hdd  0.11719          osd.4         up   1.00000  1.00000
 -7         0.19528      host store03                           
  2    hdd  0.07809          osd.2         up   1.00000  1.00000
  5    hdd  0.11719          osd.5         up   1.00000  1.00000
 -9         0.19528      host store04                           
  6    hdd  0.07809          osd.6         up   1.00000  1.00000
  8    hdd  0.11719          osd.8         up   1.00000  1.00000
-11         0.19528      host store05                           
  7    hdd  0.07809          osd.7         up   1.00000  1.00000
  9    hdd  0.11719          osd.9         up   1.00000  1.00000

```

### Mon

```bash
[cephadm@ceph-admin01 ceph-cluster]$ ceph mon stat
e3: 3 mons at {store01=[v2:172.29.0.11:3300/0,v1:172.29.0.11:6789/0],store02=[v2:172.29.0.12:3300/0,v1:172.29.0.12:6789/0],store03=[v2:172.29.0.13:3300/0,v1:172.29.0.13:6789/0]}, election epoch 22, leader 0 store01, quorum 0,1,2 store01,store02,store03
[cephadm@ceph-admin01 ceph-cluster]$ ceph mon dump
dumped monmap epoch 3
epoch 3
fsid 2f2d297c-a337-40de-bff9-13d3ea8aae31
last_changed 2021-02-25T13:39:21.993960+0800
created 2021-02-25T11:52:07.329170+0800
min_mon_release 15 (octopus)
0: [v2:172.29.0.11:3300/0,v1:172.29.0.11:6789/0] mon.store01
1: [v2:172.29.0.12:3300/0,v1:172.29.0.12:6789/0] mon.store02
2: [v2:172.29.0.13:3300/0,v1:172.29.0.13:6789/0] mon.store03


# 仲裁结果
[cephadm@ceph-admin01 ceph-cluster]$ ceph quorum_status
```

### 管理套接字

只能在节点本地运行

```bash
[root@store01 ~]#  ls /var/run/ceph/
ceph-client.rgw.store01.2025.94026773340728.asok  ceph-mgr.mgr01.asok  ceph-mon.store01.asok  ceph-osd.0.asok  ceph-osd.3.asok


# mon接口
[root@store01 ~]# ceph --admin-daemon /var/run/ceph/ceph-mon.store01.asok help


# osd接口
[root@store01 ~]# ceph --admin-daemon /var/run/ceph/ceph-osd.0.asok help

[root@store01 ~]# ceph --admin-daemon /var/run/ceph/ceph-osd.0.asok version
{
    "version": "15.2.9",
    "release": "octopus",
    "release_type": "stable"
}
[root@store01 ~]# ceph --admin-daemon /var/run/ceph/ceph-osd.0.asok status
{
    "cluster_fsid": "2f2d297c-a337-40de-bff9-13d3ea8aae31",
    "osd_fsid": "f83e6ddf-f892-4443-bed0-84734a8a3c23",
    "whoami": 0,
    "state": "active",
    "oldest_map": 1,
    "newest_map": 243,
    "num_pgs": 61
}
```

### 停止或重启ceph集群

异常操作会数据丢失

停止

```bash
[root@store01 ~]# ceph osd set noout
# 停客户端 radosgw, mds, 
# osd, manager, monitor
```



启动

```bash
mon
mgr
osd

mds
radosgw
访问
删除noout： ceph osd unset noout
```

## ceph配置文件

```bash
[root@ceph-admin01 ~]# su - cephadm
上一次登录：四 2月 25 14:11:50 CST 2021pts/0 上
[cephadm@ceph-admin01 ~]$ cd ceph-cluster/
[cephadm@ceph-admin01 ceph-cluster]$ cat ceph.conf
[global]
fsid = 2f2d297c-a337-40de-bff9-13d3ea8aae31
public_network = 172.29.0.0/16
cluster_network = 172.18.0.0/16
mon_initial_members = store01
mon_host = 172.29.0.11
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

# 时间
mon clock drift allowed = 2    
mon clock drift warn backoff = 30
[cephadm@ceph-admin01 ceph-cluster]$ 

```

#或;开头为注释 

配置段

```bash
[global]
[osd] 所有osd生效, 指定osd, osd.标识         #ceph osd tree   [osd.4]定义指定osd
[mon] 所有mon生效        #ceph mon stat                      [mon.store01]定义指定mon
[client] 所有client
```

ceph加载配置文件, 类型mysql

$CEPH_CONF变量指定的路径 

ceph -c path/path 命令行参数

/etc/ceph/ceph.conf

~/.ceph/config

./ceph.conf   工作目录

后面配置文件的选项和前面配置相同时，后面会覆盖前端配置

元参数：

```bash
$cluster ceph集群名
$type    当前 服务类型名 osd/mon
$id      进程标签符. osd.0就是0
$host    守护进程主机名
$name     $type.$id
```

运行时配置

```bash
ceph daemon {type}.{id} config show


# 配置
ceph tell {type}.{id} intectargs -- {name} {value} [-- {name} {value} ] ...
	# ceph tell osd.0 intectargs '--debug_osd 0/5'
	
ceph daemon {type}.{id}  set name value
	# ceph daemon osd.0 config set debug_osd 0/5
```



