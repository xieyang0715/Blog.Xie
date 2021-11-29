---
title: 高可用Kubernetes
date: 2021-02-05 11:38:29
tags:
---

# 分式布应用协作模型

多个节点之上的应用程序，还需要协作，称为**分布式应用程序协作模型**或**分布式程序高可用方式**

| 分布式应用程序协作模型         | 通信                                     | 工具                                               |
| ------------------------------ | ---------------------------------------- | -------------------------------------------------- |
| 基于协作协议， leader election | 通过程序某个端口，发送心跳，用协议通信。 | etcd, elasticsearch                                |
| 基于分布式锁， leader election | 借助第3方组件(分布式锁)，来选举leader。  | zookeeper, kube-controller-manager, kube-scheduler |

## 协作协议

elasticsearch于9100端口，发送心跳，通过选举协议(leader election)选举主节点(leader)，其他节点工作于备用模式或观察者模式或学习者模式。leader出现故障，剩下的人一而上进行下一轮选举，选举新的leader。

etcd高可用

## 分布式锁

zookeeper通过分布式锁来选举leader，区别谁是主动，谁是被动。

![image-20210206091042845](http://myapp.img.mykernel.cn/image-20210206091042845.png)

存储系统宕机？

> 存储系统自已需要高可用

## 分布式应用面临的问题

- network partition 网络分区

**CAP理论**，**早期的参考理论模型**，任何一个分布式系统只能满足CAP三种特性中的2种。 即分式系统出现故障后，可以满足的特性

```bash
C 一致性：  故障后数据不一致，到底哪个是真哪个是假？

A 可用性：故障后必然是可用的。

P 分区容错性：        经常可能网络抖动就出现分区。其中2个节点联系不到另一个节点。
```

**BASE理论**：从另一个角度，确保CAP3理论满足。**针对ACID来说的**。从强一致转换为弱一致性，刚开始从某个节点访问数据不一样，，但过一个时间窗口后数据就一致了。

强一致：复制同步 

弱一致：复制异步，最终一致性

https://zhuanlan.zhihu.com/p/67949045

```bash
A 原子
C 一致
I 隔离
D 持久

acid 酸：尖酸房东
base 碱： https://zhuanlan.zhihu.com/p/102875436
  ba Basically Avaliable基本可用
  s  soft state 软状态、柔性事物 
  e  Eventually consistent 最终一致性
```

所以现在很多系统基于base理论构建，但是etcd是强一致产品，必须遵循CAP，但是也是base理论的实践



**脑裂**：发生分区后，一个节点联系节点1，另一个节点联系节点2/3，两边存储数据，过段时间etcd3个节点又见面了，数据不一样，就是脑裂。

![image-20210206095743522](http://myapp.img.mykernel.cn/image-20210206095743522.png)

分布式系统发生分区，一定要避免脑裂，一旦发生分区，只能有一部分(多数方)，代表原集群工作。少数方拒绝向客户端提供服务。



判断机制: quorum ,法定票数

with quorum: 多数方，继续工作。大于半数

without quorum: 少数方，拒绝工作。少于半数





# 生产部署要点

测试

- 单：master/etcd
- node主机数量按需
- nfs/gluster

生产环境

- 高可用etcd, 奇数节点
- 高可用master
  - kube-apiserver：http/https 无状态，1) keepalived vip流动实现冗余。2) 前端通过haproxy+keepalived，反代api server
  - kube-scheduler及kube-controller-manager各自只能有一个活动实例，但可以有多个备用。各自有自带leader选举功能，默认启用。
- 多node, 数量越多，冗余能力越强
- ceph/glusterfs/iscsi/fc san及云存储



api server: 存储CR, 特定格式

controller: 创建

scheduler: 预选、优选

kubelet：启动





## etcd 高可用

- 高可用
  - 生产过程中，单etcd故障时，整个集群数据（用户定义的期望状态和实例当前状态）都丢失了，因此必须对etcd高可用。
- 周期性数据备份
  - 每个etcd节点之上误删除资源定义时，每一个高可用节点所有数据都是同步的，要想恢复，得周期性的备份

etcd不再是etcd，所有组件与etcd通信，是经由api server，api server将存储接口封装，只能定义成特定的格式(yaml格式)，才可以往etcd中存储。所以外在看来api server就是一个存储系统，只是api server存储数据在etcd中。



### 高可用

etcd本身就是存储，不需要高外依赖，etcd运行多个实例时，所以直接基于raft协议，完成leader election, 完成数据强一致性。

**每一个节点均可以读写**，但是你写到任意一个节点的**数据**，都要**同步**到同一个集群的另外的节点，而且确保数据是**强一致**的





go语言开发分布式应用，完成协作，使用raft协议

raft协议是简装版的paxos协议，今天各种分布式协作逻辑中，出现很多协议，他们都沿用paxos协议的思想，或简化、或另辟蹊径，都或多或少受到paxos协议的影响。作者穷10年之功，才设计出paxos。

特性

- leader election
- 数据强一致

raft对比paxos

| raft         | paxos |
| ------------ | ----- |
| 轻量化       | 重量  |
| 功能不比不弱 |       |

java开发分布式程序，就会使用paxos协议或 google研发的paxos变种zab协议(Zookeeper Atomic Broadcast)



协议工作逻辑：https://www.bilibili.com/video/av77388641/



### 部署etcd高可用

部署多个etcd节点，激活raft协议工作逻辑

- 每个节点可读写

  - 读：所有节点数据一样。读负载均衡
  - 写：不能均衡，写之后异步复制。

- 节点数量奇数个（3，5，7) 确保分布式应用发生网络分区后，至少存在半数以上的节点得到法定票数(with quorum), 让集群继续工作。

  - 只需要3-7个节点，过多没有正向价值，因为每个节点数据一样，只要一个节点在，数据就在。任何节点均可以读，但是写，你在任何一个节点写入后，需要在其他节点同步一份，每多一个节点就多同步一次。

  - k8s集群规模 小规模 3. 如果是2节点，是脑裂状态，一个宕机集群不可用。
  - k8s集群规模 中规模 5
  - k8s集群规模 大规模 7



## master节点高可用

- api server
  - stateless
- controller-manager
  - endpoint
- scheduler
  - endpoint

## kube-apiserver高可用

- 集群网关：所有对k8s访问，经由api server，因此应该高可用。

- 不高可用，一旦master节点的api server故障，各节点的kubelet不能周期性访问apiserver, 获取状态信息，也不能接收api server通知信息。

- stateless**无状态服务**: api server只是http/**https服务**，**自已不需要保存任何数据**，因此只需要确保kube-apiserver可以做到多实例就可以了。

- 高可用：2个以上含2个都可以。

- 无状态的，2种方案完成可用性设定

  1. 多个api server, 每个节点上部署一个keepalived, keepalived可以流动ip地址，假设仅流动单一IP，因此被流动的IP在哪个节点上，哪个节点就能接收用户请求。随后我们访问api server不使用物理地址，而是使用keepalived 流动的VIP

     > 缺陷：处理请求的节点1个。其他2个节点空闲状态。
     >
     > 也可以做3组VIP，正常每个节点1个VIP, 一旦某节点故障，节点VIP暂时寄存在另外2个节点的某个节点之上。

     <img src="http://myapp.img.mykernel.cn/image-20210206083313934.png" alt="image-20210206083313934" style="zoom:50%;" />

  2. 在多个api server前端使用Nginx做一个反向代理服务器，做负载均衡，让nginx对后端每一个api server做健康状态检测。并且正常负载客户端请求到3个api server中任何一个节点之上。需要每个客户端：controller/scheduler/kubelet/... 它们指向的地址不是特定api server的地址，而是前端调度器的地址。

     但是调度器是单点，因此需要对他做高可用，使用KeepAlived + Nginx(双主)

     > 优点：
     >
     > 1. 每个api server可以发挥出作用来。
     > 2. 在客户端较多时，这种方式能很好负载用户的请求。
     > 3. 无状态的 https服务，自已不保存任何数据，可以任意使用rr, wrr(常用), wls算法。将请求向任何节点调度。
     > 4. 客户端提交的数据放在etcd存储中，所以api server自已就变成了无状态

     ![image-20210206084019014](http://myapp.img.mykernel.cn/image-20210206084019014.png)





不使用协作协议，而是使用分布式锁

毕竟有一个etcd了，由于api server看起来就是一个存储系统，有自已抽象的存储接口。

所以多个scheduler/controller-manager可以抢占 api server之上的某一个固定资源，多个scheduler谁先抢到，并持续更新，谁就是leader(活动的)，余下的是被动。如果在指定周期内不再更新api server, 那么剩下scheduler在抢占资源，成为leader。

多个kube-controller-manager 抢占 api server的endpoint资源

多个kube-scheduler 抢占 api server的endpoint资源





# 规划



## 最多的规模

2个节点专门跑api server. 

2个节点专门跑 controller manager

2个节点专门跑 scheduler

3个节点专门跑 etcd

![image-20210206101000779](http://myapp.img.mykernel.cn/image-20210206101000779.png)





## 较小规模

controller-manager/scheduler 分布式锁，2个节点

api server无状态，至少2个节点

etcd集群，避免脑裂，至少3个节点。



![image-20210206101130645](http://myapp.img.mykernel.cn/image-20210206101130645.png)



## 更小规模 

![image-20210206101534349](http://myapp.img.mykernel.cn/image-20210206101534349.png)

> 甚至可以将nginx + keepalived均运行为3个节点之上，有3个虚拟VIP位于各个节点之上



或者node节点之上部署nginx 4层反代至3个api server. master是3个节点即可，不需要nginx + keepalived.



## 基于k8s官方二进制程序部署k8s集群

https://github.com/kubernetes/kubernetes

<img src="http://myapp.img.mykernel.cn/image-20210206102821416.png" alt="image-20210206102821416" style="zoom: 50%;" />

<img src="http://myapp.img.mykernel.cn/image-20210206102907376.png" alt="image-20210206102907376" style="zoom:50%;" />

[v1.20.2](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#v1202)

- Downloads for v1.20.2
  - [Source Code](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#source-code)
  - [Client binaries](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#client-binaries) # kubectl
  - [Server binaries ](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#server-binaries) # kube-apiserver, kubeadm, kube-scheduler, kube-controller-manager
  - [Node binaries](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#node-binaries) # kubelet, kube-proxy, kubeadm
- [Changelog since v1.20.1](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#changelog-since-v1201)
- Changes by Kind
  - [Bug or Regression](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#bug-or-regression)
- Dependencies
  - [Added](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#added)
  - [Changed](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#changed)
  - [Removed](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#removed)



部署流程

- [x] 高可用etcd

- [x] 1个节点的master

  ```bash
  # master01
  wget https://dl.k8s.io/v1.20.2/kubernetes-server-linux-amd64.tar.gz
  ```

- [x] 2个node加入master

  ```bash
  # node01
  wget https://dl.k8s.io/v1.20.2/kubernetes-node-linux-amd64.tar.gz
  ```

- [x] 高可用master组件





- [x] 6个节点

  ```bash
  [root@centos7-iaas ~]# for i in {101..106}; do virsh start 172.16.0.$i; done
  ```

  各节点环境准备参考 [静态pod环境准备](http://blog.mykernel.cn/2021/01/13/Kubernetes%E9%9B%86%E7%BE%A4%E9%83%A8%E7%BD%B2%E5%8F%8A%E9%99%88%E8%BF%B0%E5%BC%8F%E5%91%BD%E4%BB%A4%E7%AE%A1%E7%90%86/#%E9%9D%99%E6%80%81pod%E9%83%A8%E7%BD%B2-kubernetes)

  | 节点名称 | ip           | 命令                                           | 角色 |
  | -------- | ------------ | ---------------------------------------------- | ---- |
  | master01 | 172.16.0.101 | `hostnamectl set-hostname master01.magedu.com` | etcd |
  | master02 | 172.16.0.102 | `hostnamectl set-hostname master02.magedu.com` | etcd |
  | master03 | 172.16.0.103 | `hostnamectl set-hostname master03.magedu.com` | etcd |
  | node01   | 172.16.0.104 | `hostnamectl set-hostname node01.magedu.com`   |      |
  | node02   | 172.16.0.105 | `hostnamectl set-hostname node02.magedu.com`   |      |
  | node03   | 172.16.0.106 | `hostnamectl set-hostname node03.magedu.com`   |      |

  各节点时间同步 

  ```bash
   sed -i -e 's@us.archive.ubuntu.com@mirrors.aliyun.com@g' -e 's@security.ubuntu.com@mirrors.aliyun.com@g' /etc/apt/sources.list
  
  apt update
  
  apt -y install chrony && systemctl enable chronyd && systemctl restart chronyd
  ```

  主机名解析

  ```bash
  root@master01:~/k8s-certs-generator# cat /etc/hosts
  127.0.0.1	localhost
  127.0.1.1	ubuntu
  
  # The following lines are desirable for IPv6 capable hosts
  ::1     localhost ip6-localhost ip6-loopback
  ff02::1 ip6-allnodes
  ff02::2 ip6-allrouters
  
  
  172.16.0.101 master01.magedu.com etcd01.magedu.com etcd01
  172.16.0.102 master02.magedu.com etcd02.magedu.com etcd02
  172.16.0.103 master03.magedu.com etcd03.magedu.com etcd03
  172.16.0.104 node01.magedu.com
  172.16.0.105 node02.magedu.com
  172.16.0.106 node03.magedu.com
  
  
  
  # etcd
  scp /etc/hosts master02.magedu.com:/etc/hosts
  scp /etc/hosts master03.magedu.com:/etc/hosts
  
  # node
  scp /etc/hosts node01.magedu.com:/etc/hosts
  ```

  关闭iptables, firewalld, NetworkManager

  ubuntu 略

  ```bash
  systemctl disable iptables  && systemctl stop iptables 
  systemctl disable firewalld && systemctl stop  firewalld
  systemctl disable NetworkManager --now
  ```

  禁用selinux

  ```
  ubuntu 略
  ```

  必须完成内核参数优化和资源限制优化

  参考：
  
  [资源限制优化](http://blog.mykernel.cn/2020/09/29/Ubuntu-Server-18-04%E7%9A%84%E5%AE%89%E8%A3%85-%E4%BC%98%E5%8C%96%E7%B3%BB%E7%BB%9F/#3-8-%E7%B3%BB%E7%BB%9F%E8%B5%84%E6%BA%90%E9%99%90%E5%88%B6%E4%BC%98%E5%8C%96)
  
  [内核参数优化](http://blog.mykernel.cn/2020/09/29/Ubuntu-Server-18-04%E7%9A%84%E5%AE%89%E8%A3%85-%E4%BC%98%E5%8C%96%E7%B3%BB%E7%BB%9F/#3-9-%E5%86%85%E6%A0%B8%E5%8F%82%E6%95%B0%E4%BC%98%E5%8C%96)
  

重启

```bash
  # 时间
root@master01:~# date
  Sun Feb  7 08:19:04 CST 2021
  
```

  

  

  

## 高可用etcd

etcd存放所有资源，不允许任何人访问，而且etcd基于restful(http/https)风格API提供服务，所以etcd最好使用数字证书认证。

1. etcd彼此间通信：2380, 证书认证。每个节点有一个证书完成对等通信。

2. 任何客户端访问etcd: 2379端口，证书认证

   - 客户端提供证书
   - 每个etcd有一个服务端证书。

3. etcd专用的证书

   

| 证书                                                         | 通信     |      |
| ------------------------------------------------------------ | -------- | ---- |
| etcd-ca                                                      | 颁发证书 |      |
| etcd1-peer，etcd2-peer,etcd3-peer                            | 2380通信 |      |
| etcd1-server, etcd2-server, etcd3-server.        client(每个api server) | 2379通信 |      |

简化逻辑：

一个ca

一个peer证书：全部一样

一个server证书（全部一样），一个client证书（全部一样）



### 部署etcd

- [x] extras/apt
- [x] 二进制部署

etcd版本 etcd v2. etcdv3, kubernetes新版使用v3

```bash
# 缓存
root@master01:~# apt update

# 查看版本
root@master01:~# apt-cache  madison etcd
      etcd | 3.2.17+dfsg-1 | http://mirrors.aliyun.com/ubuntu bionic/universe amd64 Packages
      etcd | 3.2.17+dfsg-1 | http://mirrors.aliyun.com/ubuntu bionic/universe i386 Packages
      etcd | 3.2.17+dfsg-1 | http://mirrors.aliyun.com/ubuntu bionic/universe Sources

# 安装etcd
root@master01:~# apt install etcd -y
Reading package lists... Done
Building dependency tree       
Reading state information... Done
etcd is already the newest version (3.2.17+dfsg-1).
0 upgraded, 0 newly installed, 0 to remove and 207 not upgraded.


# 版本
root@master01:~# etcd --version
etcd Version: 3.2.17          # V3版本，新版Kubernetes使用V3版本
Git SHA: Not provided (use ./build instead of go build)
Go Version: go1.10
Go OS/Arch: linux/amd64

```

### etcd配置文件

- [x] 不使用证书调度通
- [x] 使用证书认证

客户端

```bash
root@master01:~# dpkg -L etcd-client
/usr/bin/etcd-dump-db
/usr/bin/etcd-dump-logs
/usr/bin/etcd2-backup-coreos
/usr/bin/etcdctl
```



服务端 

```bash
root@master01:~# dpkg -L etcd-server
/lib/systemd/system/etcd.service

root@master01:~# cat /lib/systemd/system/etcd.service
[Unit]
Description=etcd - highly-available key value store
Documentation=https://github.com/coreos/etcd
Documentation=man:etcd
After=network.target
Wants=network-online.target

[Service]
Environment=DAEMON_ARGS=
Environment=ETCD_NAME=%H
Environment=ETCD_DATA_DIR=/var/lib/etcd/default  # 数据目录
EnvironmentFile=-/etc/default/%p       # 配置文件
Type=notify
User=etcd
PermissionsStartOnly=true
#ExecStart=/bin/sh -c "GOMAXPROCS=$(nproc) /usr/bin/etcd $DAEMON_ARGS"
ExecStart=/usr/bin/etcd $DAEMON_ARGS
Restart=on-abnormal
#RestartSec=10s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd2.service


# 配置文件
root@master01:~# cat /etc/default/etcd 
ETCD_DATA_DIR="/var/lib/etcd/k8s.etcd"       # 数据存放位置
ETCD_LISTEN_PEER_URLS="https://172.16.0.101:2380"    # 监听只能是ip     # “监听” 3个etcd raft协议心跳地址 error verifying flags, expected IP in URL for binding (https://master01.magedu.com:2380). See 'etcd --help'.
ETCD_LISTEN_CLIENT_URLS="https://172.16.0.101:2379"       # “监听” client 与etcd通信的接口 # 暴露给客户端添加127.0.0.1
ETCD_NAME="etcd02.magedu.com"                             # etcd集群的节点名 # 和INITIAL_CLUSTER的KEY一样
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://etcd02.magedu.com:2380"  # 工作为集群时，对外“通告”自已协议接口
ETCD_ADVERTISE_CLIENT_URLS="https://etcd02.magedu.com:2379"        # 工作为集群时，对外“通告”自已客户访问接口
ETCD_INITIAL_CLUSTER="etcd01.magedu.com=https://etcd01.magedu.com:2380,etcd02.magedu.com=https://etcd02.magedu.com:2380,etcd03.magedu.com=https://etcd03.magedu.com:2380"        # 集群成员
	# 静态初始化，写进来
	# 动态初始化，通过另一个集群发现自已的节点
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster" 

# client访问etcd 2379使用https通信
ETCD_CERT_FILE="/etc/etcd/pki/server.crt" # 提供给client证书
ETCD_KEY_FILE="/etc/etcd/pki/server.key"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/pki/ca.crt" # 检验client的证书
ETCD_AUTO_TLS="false"    # 自动生成etcd server证书


# 各节点间通信也叫peer通信 # 2380端口通信
ETCD_PEER_CERT_FILE="/etc/etcd/pki/peer.crt"  # 简单使用相同证书，检验对端的CN得吻合
ETCD_PEER_KEY_FILE="/etc/etcd/pki/peer.key"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/pki/ca.crt"  # 签发etcd证书和验证peer证书的ca.
ETCD_PEER_AUTO_TLS="false"
```

### 配置https *

至少2个节点存在才有法定票数，至少配置2个节点才可以启动etcd

#### 生成证书

http://blog.mykernel.cn/2021/01/29/%E8%AE%A4%E8%AF%81%E3%80%81%E6%8E%88%E6%9D%83%E5%8F%8A%E5%87%86%E5%85%A5%E6%8E%A7%E5%88%B6/#%E5%88%B6%E4%BD%9C%E8%B4%A6%E6%88%B7



https://github.com/iKubernetes/k8s-certs-generator

证书要求 

https://kubernetes.io/zh/docs/setup/best-practices/certificates/#%E6%89%8B%E5%8A%A8%E9%85%8D%E7%BD%AE%E8%AF%81%E4%B9%A6

CA

| 路径                   | 默认 CN                   | 描述                                                         |
| ---------------------- | ------------------------- | ------------------------------------------------------------ |
| ca.crt,key             | kubernetes-ca             | Kubernetes 通用 CA                                           |
| etcd/ca.crt,key        | etcd-ca                   | 与 etcd 相关的所有功能                                       |
| front-proxy-ca.crt,key | kubernetes-front-proxy-ca | 用于 [前端代理](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/configure-aggregation-layer/) |

```bash
# 制作ca
# https://kubernetes.io/zh/docs/concepts/cluster-administration/certificates/
```

CSR

##### 准备3个ca csr，并生成ca证书

```bash
curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o cfssl
chmod +x cfssl
curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o cfssljson
chmod +x cfssljson
curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o cfssl-certinfo
chmod +x cfssl-certinfo

mkdir cert
cd cert
```

k8s-ca csr CN=kubernetes-ca

```diff
root@ubuntu-template:~/cert# cat k8s-ca-csr.json 
{
+  "CN": "kubernetes-ca",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
+    "C": "China",
+    "ST": "SiChuan",
+    "L": "ChengDu",
+    "O": "system:HuaYang",
+    "OU": "system"
  }],
+    "hosts": []
}
```

> CN 
>
> C  国家
>
> ST 省
>
> L  城市
>
> O 组织
>
> OU 组织
>
> K8S中使用CN作用户名和O作为组，非常关键
>
> hosts即下表中的SAN， 但是从证书要求中的CA并没有names中的要求，只有CN要求，所以下面随意
>
> | 默认 CN                       | 父级 CA                   | O (位于 Subject 中) | 类型           | 主机 (SAN)                                          |
> | ----------------------------- | ------------------------- | ------------------- | -------------- | --------------------------------------------------- |
> | kube-etcd                     | etcd-ca                   |                     | server, client | `localhost`, `127.0.0.1`                            |
> | kube-etcd-peer                | etcd-ca                   |                     | server, client | `<hostname>`, `<Host_IP>`, `localhost`, `127.0.0.1` |
> | kube-etcd-healthcheck-client  | etcd-ca                   |                     | client         |                                                     |
> | kube-apiserver-etcd-client    | etcd-ca                   | system:masters      | client         |                                                     |
> | kube-apiserver                | kubernetes-ca             |                     | server         | `<hostname>`, `<Host_IP>`, `<advertise_IP>`, `[1]`  |
> | kube-apiserver-kubelet-client | kubernetes-ca             | system:masters      | client         |                                                     |
> | front-proxy-client            | kubernetes-front-proxy-ca |                     | client         |                                                     |
>
> [1]: 用来连接到集群的不同 IP 或 DNS 名 （就像 [kubeadm](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm/) 为负载均衡所使用的固定 IP 或 DNS 名，`kubernetes`、`kubernetes.default`、`kubernetes.default.svc`、 `kubernetes.default.svc.cluster`、`kubernetes.default.svc.cluster.local`）。
>
> 其中，`kind` 对应一种或多种类型的 [x509 密钥用途](https://godoc.org/k8s.io/api/certificates/v1beta1#KeyUsage)：
>
> | kind   | 密钥用途                       |
> | ------ | ------------------------------ |
> | server | 数字签名、密钥加密、服务端认证 |
> | client | 数字签名、密钥加密、客户端认证 |
>
> > **说明：**
> >
> > 上面列出的 Hosts/SAN 是推荐的配置方式；如果需要特殊安装，则可以在所有服务器证书上添加其他 SAN。
>
> > **说明：**
> >
> > 对于 kubeadm 用户：
> >
> > - 不使用私钥，将证书复制到集群 CA 的方案，在 kubeadm 文档中将这种方案称为外部 CA。
> > - 如果将以上列表与 kubeadm 生成的 PKI 进行比较，你会注意到，如果使用外部 etcd，则不会生成 `kube-etcd`、`kube-etcd-peer` 和 `kube-etcd-healthcheck-client` 证书



生成ca证书

```bash
root@ubuntu-template:~/cert# ../cfssl gencert -initca k8s-ca-csr.json  | ../cfssljson -bare k8s-ca
2021/02/08 10:07:43 [INFO] generating a new CA key and certificate from CSR
2021/02/08 10:07:43 [INFO] generate received request
2021/02/08 10:07:43 [INFO] received CSR
2021/02/08 10:07:43 [INFO] generating key: rsa-2048
2021/02/08 10:07:43 [INFO] encoded CSR
2021/02/08 10:07:43 [INFO] signed certificate with serial number 376173122527285936707056740923555246283312145708
```

etcd-ca csr CN=etcd-ca

```diff
 cp k8s-ca-csr.json etcd-ca-csr.json 
root@ubuntu-template:~/cert# cat etcd-ca-csr.json
{
+  "CN": "etcd-ca",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
    "O": "system:HuaYang",
    "OU": "system"
  }],
+    "hosts": []
}

root@ubuntu-template:~/cert# ../cfssl gencert -initca etcd-ca-csr.json  | ../cfssljson -bare etcd-ca
```

proxy-front-ca CN=kubernetes-front-proxy-ca

```diff
root@ubuntu-template:~/cert# cp etcd-ca-csr.json kubernetes-front-proxy-ca.json
root@ubuntu-template:~/cert# cat kubernetes-front-proxy-ca.json
{
+  "CN": "kubernetes-front-proxy-ca",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
    "O": "system:HuaYang",
    "OU": "system"
  }],
+    "hosts": []
}

root@ubuntu-template:~/cert# ../cfssl gencert -initca   kubernetes-front-proxy-ca.json | ../cfssljson -bare  kubernetes-front-proxy-ca
2021/02/08 10:10:23 [INFO] generating a new CA key and certificate from CSR
2021/02/08 10:10:23 [INFO] generate received request
2021/02/08 10:10:23 [INFO] received CSR
2021/02/08 10:10:23 [INFO] generating key: rsa-2048
2021/02/08 10:10:24 [INFO] encoded CSR
2021/02/08 10:10:24 [INFO] signed certificate with serial number 528994211403294494357426848889182897326724284674

```

验证

```bash
root@ubuntu-template:~/cert# ls *.pem
etcd-ca-key.pem  etcd-ca.pem  k8s-ca-key.pem  k8s-ca.pem  kubernetes-front-proxy-ca-key.pem  kubernetes-front-proxy-ca.pem
```



##### 准备所有ca的配置文件

```diff
root@ubuntu-template:~/cert# cat common-config.json 
{
    "signing": {
        "default": {
+            "expiry": "876000h" 
        },
        "profiles": {
            "kubernetes-ca": { 
+                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
+                    "server auth",
+                    "client auth"
                ]
            },
            "etcd-ca": {
+                "expiry": "87600h",  
                "usages": [
                    "signing",
                    "key encipherment",
 +                   "server auth",
 +                   "client auth"
                ]
            },
            "kubernetes-front-proxy-ca": {
+                "expiry": "87600h", 
                "usages": [
                    "signing",
                    "key encipherment",
+                    "server auth",
+                    "client auth"
                ]
            }
        }
    }
}
```

> profiles 一级键为上面ca的CSR的CN名 profiles kubernetes-front-proxy-ca etcd-ca kubernetes-ca
>
> expiry 过期时间，此处100年
>
> server auth 完成服务端认证, 签发服务端证书
>
> client auth  完成客户端认证，签发客户端证书
>
> 多个默认CN，../cfssl gencert -profile=kubernetes-ca调用哪个父级CA配置
>
> | 默认 CN                       | 父级 CA                   | O (位于 Subject 中) | 类型           | 主机 (SAN)                                          |
> | ----------------------------- | ------------------------- | ------------------- | -------------- | --------------------------------------------------- |
> | kube-etcd                     | etcd-ca                   |                     | server, client | `localhost`, `127.0.0.1`                            |
> | kube-etcd-peer                | etcd-ca                   |                     | server, client | `<hostname>`, `<Host_IP>`, `localhost`, `127.0.0.1` |
> | kube-etcd-healthcheck-client  | etcd-ca                   |                     | client         |                                                     |
> | kube-apiserver-etcd-client    | etcd-ca                   | system:masters      | client         |                                                     |
> | kube-apiserver                | kubernetes-ca             |                     | server         | `<hostname>`, `<Host_IP>`, `<advertise_IP>`, `[1]`  |
> | kube-apiserver-kubelet-client | kubernetes-ca             | system:masters      | client         |                                                     |
> | front-proxy-client            | kubernetes-front-proxy-ca |                     | client         |                                                     |

##### 准备etcd相关的证书

| 默认 CN                      | 建议的密钥路径              | 建议的证书路径              | 命令    | 密钥参数        | 证书参数         |
| ---------------------------- | --------------------------- | --------------------------- | ------- | --------------- | ---------------- |
| kube-etcd                    | etcd/server.key             | etcd/server.crt             | etcd    | --key-file      | --cert-file      |
| kube-etcd-peer               | etcd/peer.key               | etcd/peer.crt               | etcd    | --peer-key-file | --peer-cert-file |
| etcd-ca                      |                             | etcd/ca.crt                 | etcdctl |                 | --cacert         |
| kube-etcd-healthcheck-client | etcd/healthcheck-client.key | etcd/healthcheck-client.crt | etcdctl | --key           | --cert           |

准备etcd server证书 

```diff
root@ubuntu-template:~/cert# cat etcd-server-csr.json
{
+  "CN": "kube-etcd",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
    "O": "system:HuaYang",
    "OU": "system"
  }],
    "hosts": [
+	"172.16.0.101",
+	"172.16.0.102",
+	"172.16.0.103",
+	"127.0.0.1"
	]
}
```

> 由以上表格可知，CN的值
>
> hosts表示连接提供ssl服务的端点。也就是连接https server的可用地址：
>
> 172.16.0.101, 因为etcd运行在这个节点上，其他节点来连接这个主机的etcd，会使用 https://172.16.0.101:2379
>
> 172.16.0.102, 因为etcd运行在这个节点上，其他节点来连接这个主机的etcd，会使用 https://172.16.0.102:2379
>
> 172.16.0.103, 因为etcd运行在这个节点上，其他节点来连接这个主机的etcd，会使用 https://172.16.0.103:2379
>
> 连接本地，https://127.0.0.1:2379
>
> 由于这个证书在3个节点上使用，所以访问端点是多个。

签发证书

```bash
root@ubuntu-template:~/cert# ../cfssl gencert -ca=etcd-ca.pem -ca-key=etcd-ca-key.pem      --config=common-config.json -profile=etcd-ca      etcd-server-csr.json | ../cfssljson -bare etcd-server
2021/02/08 10:28:27 [INFO] generate received request
2021/02/08 10:28:27 [INFO] received CSR
2021/02/08 10:28:27 [INFO] generating key: rsa-2048
2021/02/08 10:28:27 [INFO] encoded CSR
2021/02/08 10:28:27 [INFO] signed certificate with serial number 167952533688731313993659908226917742879839208285
2021/02/08 10:28:27 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

```bash
root@ubuntu-template:~/cert# find . -mmin -1
.
./etcd-server.pem
./etcd-server-key.pem
./etcd-server.csr
```



准备etcd对等证书

```bash
root@ubuntu-template:~/cert# cp etcd-server-csr.json etcd-peer-csr.json
```

```diff
root@ubuntu-template:~/cert# cat etcd-peer-csr.json
{
+  "CN": "kube-etcd-peer",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
    "O": "system:HuaYang",
    "OU": "system"
  }],
    "hosts": [
+	"172.16.0.101",
+	"172.16.0.102",
+	"172.16.0.103",
	"127.0.0.1"
	]
}
```

> CN
>
> hosts 因为是通用的，所以连接客户端可以是任意etcd节点

```bash
root@ubuntu-template:~/cert# ../cfssl gencert -ca=etcd-ca.pem -ca-key=etcd-ca-key.pem      --config=common-config.json -profile=etcd-ca      etcd-peer-csr.json | ../cfssljson -bare etcd-peer
2021/02/08 10:43:29 [INFO] generate received request
2021/02/08 10:43:29 [INFO] received CSR
2021/02/08 10:43:29 [INFO] generating key: rsa-2048
2021/02/08 10:43:30 [INFO] encoded CSR
2021/02/08 10:43:30 [INFO] signed certificate with serial number 583768040850632027323548043304335924205882310223
2021/02/08 10:43:30 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

```





客户端连接etcd的证书

```diff
root@ubuntu-template:~/cert# cp etcd-server-csr.json kube-etcd-healthcheck-client-csr.json
root@ubuntu-template:~/cert# cat kube-etcd-healthcheck-client-csr.json
    {
    +  "CN": "kube-etcd-healthcheck-client",
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names":[{
        "C": "China",
        "ST": "SiChuan",
        "L": "ChengDu",
        "O": "system:HuaYang",
        "OU": "system"
      }],
    +    "hosts": []
    }

```

> CN 就是上面的CN
>
> 因为自已不提供服务，所以主机不用写

```bash
root@ubuntu-template:~/cert# ../cfssl gencert -ca=etcd-ca.pem -ca-key=etcd-ca-key.pem      --config=common-config.json -profile=etcd-ca      kube-etcd-healthcheck-client-csr.json | ../cfssljson -bare kube-etcd-healthcheck-client
2021/02/08 10:44:05 [INFO] generate received request
2021/02/08 10:44:05 [INFO] received CSR
2021/02/08 10:44:05 [INFO] generating key: rsa-2048
2021/02/08 10:44:05 [INFO] encoded CSR
2021/02/08 10:44:05 [INFO] signed certificate with serial number 675296885400368548825413968121995364077353707954
2021/02/08 10:44:05 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```



api 连接etcd的证书

```diff
root@ubuntu-template:~/cert# cp etcd-server-csr.json kube-apiserver-etcd-client-csr.json
root@ubuntu-template:~/cert# cat kube-apiserver-etcd-client-csr.json
{
+  "CN": "kube-apiserver-etcd-client",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
+    "O": "system:masters",
    "OU": "system"
  }],
+    "hosts": []
}

```

> CN 就是上面的CN
>
> 因为自已不提供服务，所以主机不用写
>
> O 需要k8s集群管理员

```bash
root@ubuntu-template:~/cert# ../cfssl gencert -ca=etcd-ca.pem -ca-key=etcd-ca-key.pem      --config=common-config.json -profile=etcd-ca      kube-apiserver-etcd-client-csr.json | ../cfssljson -bare kube-apiserver-etcd-client
2021/02/08 10:44:39 [INFO] generate received request
2021/02/08 10:44:39 [INFO] received CSR
2021/02/08 10:44:39 [INFO] generating key: rsa-2048
2021/02/08 10:44:39 [INFO] encoded CSR
2021/02/08 10:44:39 [INFO] signed certificate with serial number 571214478398291787239257328707475471018501778705
2021/02/08 10:44:39 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```



将证书准备成上面建议的文件名

```bash
install -dv /etc/etcd/pki/
#server
cp etcd-server.pem /etc/etcd/pki/server.crt 
cp etcd-server-key.pem /etc/etcd/pki/server.key
#ca
cp etcd-ca.pem /etc/etcd/pki/ca.crt
cp etcd-ca-key.pem /etc/etcd/pki/ca.key
#peer
cp etcd-peer.pem /etc/etcd/pki/peer.crt
cp etcd-peer-key.pem /etc/etcd/pki/peer.key
# client
cp kube-etcd-healthcheck-client.pem /etc/etcd/pki/client.crt
cp kube-etcd-healthcheck-client-key.pem /etc/etcd/pki/client.key

# api client
install -dv /etc/kubernetes/pki/
cp kube-apiserver-etcd-client.pem /etc/kubernetes/pki/apiserver-etcd-client.crt
cp kube-apiserver-etcd-client-key.pem /etc/kubernetes/pki/apiserver-etcd-client.key
```

2380 peer证书一样

2379 server证书一样

```bash
ssh-keygen  -t rsa -b 4096 -P '' -f ~/.ssh/id_rsa
ssh-copy-id 172.16.0.101:
ssh-copy-id 172.16.0.102:
ssh-copy-id 172.16.0.103:
scp -rp /etc/{etcd,kubernetes}  172.16.0.101:/etc
scp -rp /etc/{etcd,kubernetes}  172.16.0.102:/etc
scp -rp /etc/{etcd,kubernetes}  172.16.0.103:/etc


# 检验
tree /etc/etcd
tree /etc/kubernetes
```

配置节点初始化`deploy.sh`

```bash
root@ubuntu-template:~# cat deploy.sh
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-07
#FileName：             deploy.sh
#URL:                   http://www.${DOMAIN}
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************
 pkill apt-get
 pkill apt-get


DOMAIN=${DOMAIN:-magedu.com}
hostnamectl set-hostname $1
sed -i -e 's@us.archive.ubuntu.com@mirrors.aliyun.com@g' -e 's@security.ubuntu.com@mirrors.aliyun.com@g' /etc/apt/sources.list

apt update

apt -y install chrony && systemctl enable chronyd && systemctl restart chronyd
cat > /etc/hosts <<EOF
127.0.0.1	localhost
127.0.1.1	ubuntu

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters


172.16.0.101 master01.${DOMAIN} etcd01.${DOMAIN} etcd01
172.16.0.102 master02.${DOMAIN} etcd02.${DOMAIN} etcd02
172.16.0.103 master03.${DOMAIN} etcd03.${DOMAIN} etcd03
172.16.0.104 node01.${DOMAIN}
172.16.0.105 node02.${DOMAIN}
172.16.0.106 node03.${DOMAIN}
EOF
```



```bash
# 172.16.0.101
export DOMAIN=mykernel.cn
bash deploy.sh master01.mykernel.cn
# 172.16.0.102
export DOMAIN=mykernel.cn
bash deploy.sh master02.mykernel.cn
# 172.16.0.103
export DOMAIN=mykernel.cn
bash deploy.sh master03.mykernel.cn

# 各节点安装etcd
apt install -y etcd
```

如果以上方法配置证书，直接跳到配置etcd01



```bash
# ca
root@master01:~# git clone https://github.com/iKubernetes/k8s-certs-generator.git

# 生成证书 
bash  gencerts.sh etcd
magedu.com


# 注意如果使用 10.96.0.0/16 K8S配置的首个集群IP是10.64.0.1，暂时不清楚原因
# 10.96.0.0/16 K8S配置的首个集群IP是10.96.0.1，暂时不清楚原因。状态这个又不在公网使用，直接使用默认值
bash  gencerts.sh k8s
magedu.com
kubernetes
Enter the IP Address in default namespace  
	of the Kubernetes API Server[10.96.0.1]: 10.96.0.1 # 这里填
master01 master02 master03

root@master01:~/k8s-certs-generator# ls etcd/pki/
ca.crt  ca.key    # CA

apiserver-etcd-client.crt  apiserver-etcd-client.key   # 连接etcd客户端
server.crt  server.key # client连接etcd, etcd提供的证书 
client.crt  client.key  # 其他客户端
peer.crt  peer.key     # etcd各节点间提供的证书  

# 配置免密认证
root@master01:~/k8s-certs-generator# ssh-keygen -t rsa -b 4096 -P '' -f ~/.ssh/id_rsa
ssh-copy-id master01
ssh-copy-id master02
ssh-copy-id master03

# 配置公钥
scp -rp etcd/ etcd01.magedu.com:/etc/
scp -rp etcd/ etcd02.magedu.com:/etc/
scp -rp etcd/ etcd03.magedu.com:/etc/

# 验证公钥，有时复制是复制的内容
root@master01:~/k8s-certs-generator# ls /etc/etcd
pki
root@master02:~# ls /etc/etcd/
pki
root@master03:~# ls /etc/etcd
pki

```



#### 配置etcd01 *

走的协议https

```bash
chown etcd.etcd /etc/etcd/pki/*.key
root@master01:/etc/etcd/pki# cat /etc/default/etcd 
ETCD_DATA_DIR="/var/lib/etcd/k8s-etcd"
ETCD_LISTEN_PEER_URLS="https://172.16.0.101:2380"   
ETCD_LISTEN_CLIENT_URLS="https://172.16.0.101:2379"  
ETCD_NAME="etcd01.magedu.com"  
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://etcd01.magedu.com:2380"  
ETCD_ADVERTISE_CLIENT_URLS="https://etcd01.magedu.com:2379" 
ETCD_INITIAL_CLUSTER="etcd01.magedu.com=https://etcd01.magedu.com:2380,etcd02.magedu.com=https://etcd02.magedu.com:2380,etcd03.magedu.com=https://etcd03.magedu.com:2380"     
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"

ETCD_CERT_FILE="/etc/etcd/pki/server.crt"
ETCD_KEY_FILE="/etc/etcd/pki/server.key"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/pki/ca.crt" 
ETCD_AUTO_TLS="false"   
ETCD_PEER_CERT_FILE="/etc/etcd/pki/peer.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/pki/peer.key"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/pki/ca.crt"
ETCD_PEER_AUTO_TLS="false"


rm -fr /var/lib/etcd/k8s-etcd
```

配置unit

```bash
# /lib/systemd/system/etcd.service
Restart=on-failure
```



后期脚本`etcd1.sh`

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-07
#FileName：             etcd1.sh
#URL:                   http://www.${DOMAIN}
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************
if [ -z $1 ]; then
echo error
echo "$0 1|2|3"
exit 1
fi

DOMAIN=${DOMAIN:-magedu.com}

chown etcd.etcd /etc/etcd/pki/*.key
cat > /etc/default/etcd  <<EOF
ETCD_DATA_DIR="/var/lib/etcd/k8s-etcd"
ETCD_LISTEN_PEER_URLS="https://172.16.0.10$1:2380"   
ETCD_LISTEN_CLIENT_URLS="https://172.16.0.10$1:2379"  
ETCD_NAME="etcd0$1.${DOMAIN}"  
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://etcd0$1.${DOMAIN}:2380"  
ETCD_ADVERTISE_CLIENT_URLS="https://etcd0$1.${DOMAIN}:2379" 
ETCD_INITIAL_CLUSTER="etcd01.${DOMAIN}=https://etcd01.${DOMAIN}:2380,etcd02.${DOMAIN}=https://etcd02.${DOMAIN}:2380,etcd03.${DOMAIN}=https://etcd03.${DOMAIN}:2380"     
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"

ETCD_CERT_FILE="/etc/etcd/pki/server.crt"
ETCD_KEY_FILE="/etc/etcd/pki/server.key"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/pki/ca.crt" 
ETCD_AUTO_TLS="false"   
ETCD_PEER_CERT_FILE="/etc/etcd/pki/peer.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/pki/peer.key"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/pki/ca.crt"
ETCD_PEER_AUTO_TLS="false"
EOF


rm -fr /var/lib/etcd/k8s-etcd



cat > /lib/systemd/system/etcd.service <<'EOF'
[Unit]
Description=etcd - highly-available key value store
Documentation=https://github.com/coreos/etcd
Documentation=man:etcd
After=network.target
Wants=network-online.target

[Service]
Environment=DAEMON_ARGS=
Environment=ETCD_NAME=%H
Environment=ETCD_DATA_DIR=/var/lib/etcd/default
EnvironmentFile=-/etc/default/%p
Type=notify
User=etcd
PermissionsStartOnly=true
#ExecStart=/bin/sh -c "GOMAXPROCS=$(nproc) /usr/bin/etcd $DAEMON_ARGS"
ExecStart=/usr/bin/etcd $DAEMON_ARGS
Restart=on-failure
#RestartSec=10s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd2.service
EOF

systemctl daemon-reload
```

```bash
root@ubuntu-template:~# export DOMAIN=mykernel.cn
bash etcd1.sh 1

# 证书写的可以通信的是IP
# hosts
 sed -i -e 's@etcd02.mykernel.cn@172.16.0.102@g' -e 's@etcd01.mykernel.cn@172.16.0.101@g' -e 's@etcd03.mykernel.cn@172.16.0.103@g' /etc/default/etcd 
```



#### 配置etcd02*

```diff


root@master01:/etc/etcd/pki# scp /etc/default/etcd  etcd02:/etc/default/etcd
chown etcd.etcd /etc/etcd/pki/*.key

root@master02:~# cat /etc/default/etcd
ETCD_DATA_DIR="/var/lib/etcd/k8s-etcd"
+ETCD_LISTEN_PEER_URLS="https://172.16.0.102:2380"   
+ETCD_LISTEN_CLIENT_URLS="https://172.16.0.102:2379"  # 暴露给客户端添加127.0.0.1
+ETCD_NAME="etcd02.magedu.com"  
+ETCD_INITIAL_ADVERTISE_PEER_URLS="https://etcd02.magedu.com:2380"  
+ETCD_ADVERTISE_CLIENT_URLS="https://etcd02.magedu.com:2379" 
ETCD_INITIAL_CLUSTER="etcd01.magedu.com=https://etcd01.magedu.com:2380,etcd02.magedu.com=https://etcd02.magedu.com:2380,etcd03.magedu.com=https://etcd03.magedu.com:2380"     
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"

ETCD_CERT_FILE="/etc/etcd/pki/server.crt"
ETCD_KEY_FILE="/etc/etcd/pki/server.key"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/pki/ca.crt" 
ETCD_AUTO_TLS="false"   
ETCD_PEER_CERT_FILE="/etc/etcd/pki/peer.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/pki/peer.key"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/pki/ca.crt"
ETCD_PEER_AUTO_TLS="false"



rm -fr /var/lib/etcd/k8s-etcd
```

配置unit

```bash
# /lib/systemd/system/etcd.service
Restart=on-failure
```



启动etcd

```bash
# master01
systemctl daemon-reload 
systemctl restart etcd
systemctl enable etcd

root@master01:~/k8s-certs-generator# systemctl enable etcd

# ubuntu安装etcd之后，默认开机启动和启动


# master02 
systemctl daemon-reload 
systemctl restart etcd
systemctl enable etcd

root@master02:~# systemctl enable etcd

```

现在集群已经是**脑裂状态**, 大于半数节点存在，可以接收服务

```bash
root@master01:~# etcdctl   --endpoints=https://172.16.0.103:2379 --cert-file=/etc/etcd/pki/client.crt --key-file=/etc/etcd/pki/client.key --ca-file=/etc/etcd/pki/ca.crt  member list 
root@master01:~# etcdctl   --endpoints=https://172.16.0.102:2379 --cert-file=/etc/etcd/pki/client.crt --key-file=/etc/etcd/pki/client.key --ca-file=/etc/etcd/pki/ca.crt  member list 
root@master01:~# etcdctl   --endpoints=https://172.16.0.101:2379 --cert-file=/etc/etcd/pki/client.crt --key-file=/etc/etcd/pki/client.key --ca-file=/etc/etcd/pki/ca.crt  member list 


```



#### 配置etcd03*

```diff
# pwd
/root/k8s-certs-generator
root@master01:~/k8s-certs-generator#  scp -rp etcd/pki/ etcd03.magedu.com:/etc/etcd/
root@master03:~# chown etcd.etcd /etc/etcd/pki/*.key


root@master01:/etc/etcd/pki# scp /etc/default/etcd  etcd03:/etc/default/etcd

root@master03:~# cat /etc/default/etcd 
ETCD_DATA_DIR="/var/lib/etcd/k8s-etcd"
+ETCD_LISTEN_PEER_URLS="https://172.16.0.103:2380"  
+ETCD_LISTEN_CLIENT_URLS="https://172.16.0.103:2379"  # 暴露给客户端添加127.0.0.1
+ETCD_NAME="etcd03.magedu.com" 
+ETCD_INITIAL_ADVERTISE_PEER_URLS="https://etcd03.magedu.com:2380"  
+ETCD_ADVERTISE_CLIENT_URLS="https://etcd03.magedu.com:2379" 
ETCD_INITIAL_CLUSTER="etcd01.magedu.com=https://etcd01.magedu.com:2380,etcd02.magedu.com=https://etcd02.magedu.com:2380,etcd03.magedu.com=https://etcd03.magedu.com:2380"     
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"

ETCD_CERT_FILE="/etc/etcd/pki/server.crt"
ETCD_KEY_FILE="/etc/etcd/pki/server.key"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/pki/ca.crt" 
ETCD_AUTO_TLS="false"   
ETCD_PEER_CERT_FILE="/etc/etcd/pki/peer.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/pki/peer.key"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/pki/ca.crt"
ETCD_PEER_AUTO_TLS="false"

```

配置unit

```bash
# /lib/systemd/system/etcd.service
Restart=on-failure
```



```bash
systemctl daemon-reload 
systemctl restart etcd

root@master03:~# systemctl status etcd
Feb 07 08:36:49 master03.magedu.com etcd[3150]: published {Name:etcd03.magedu.com ClientURLs:[https://etcd03.magedu.com:2379]} to cluster e9df2ed0e9b3d8cd
Feb 07 08:36:49 master03.magedu.com etcd[3150]: ready to serve client requests
Feb 07 08:36:49 master03.magedu.com systemd[1]: Started etcd - highly-available key value store.
Feb 07 08:36:49 master03.magedu.com etcd[3150]: serving client requests on 172.16.0.103:2379


root@master03:~# systemctl enable etcd

```

```diff
root@master03:~# etcdctl   --endpoints=https://etcd01.magedu.com:2379 --cert-file=/etc/etcd/pki/client.crt --key-file=/etc/etcd/pki/client.key --ca-file=/etc/etcd/pki/ca.crt  member list 
																				 # 添加URL
+12ac5d248ff0c431: name=etcd03.magedu.com peerURLs=https://etcd03.magedu.com:2380 clientURLs=https://etcd03.magedu.com:2379 isLeader=false
42443bc12f6e4e47: name=etcd01.magedu.com peerURLs=https://etcd01.magedu.com:2380 clientURLs=https://etcd01.magedu.com:2379 isLeader=true
d233e11fea5ece18: name=etcd02.magedu.com peerURLs=https://etcd02.magedu.com:2380 clientURLs=https://etcd02.magedu.com:2379 isLeader=false

```



## 备份etcd

https://www.cnblogs.com/chenqionghe/p/10622859.html



## 部署k8s

## 生成k8s证书及配置 *

| 默认 CN        | 父级 CA       | O (位于 Subject 中) | 类型   | 主机 (SAN)                                         |
| -------------- | ------------- | ------------------- | ------ | -------------------------------------------------- |
| kube-apiserver | kubernetes-ca |                     | server | `<hostname>`, `<Host_IP>`, `<advertise_IP>`, `[1]` |

| 默认 CN                       | 建议的密钥路径               | 建议的证书路径               | 命令           | 密钥参数                | 证书参数                       |
| ----------------------------- | ---------------------------- | ---------------------------- | -------------- | ----------------------- | ------------------------------ |
| etcd-ca                       | etcd/ca.key                  | etcd/ca.crt                  | kube-apiserver |                         | --etcd-cafile                  |
| kube-apiserver-etcd-client    | apiserver-etcd-client.key    | apiserver-etcd-client.crt    | kube-apiserver | --etcd-keyfile          | --etcd-certfile                |
| kubernetes-ca                 | ca.key                       | ca.crt                       | kube-apiserver |                         | --client-ca-file               |
| kube-apiserver                | apiserver.key                | apiserver.crt                | kube-apiserver | --tls-private-key-file  | --tls-cert-file                |
| kube-apiserver-kubelet-client | apiserver-kubelet-client.key | apiserver-kubelet-client.crt | kube-apiserver | --kubelet-client-key    | --kubelet-client-certificate   |
| front-proxy-ca                | front-proxy-ca.key           | front-proxy-ca.crt           | kube-apiserver |                         | --requestheader-client-ca-file |
| front-proxy-client            | front-proxy-client.key       | front-proxy-client.crt       | kube-apiserver | --proxy-client-key-file | --proxy-client-cert-file       |

生成kube-apiserver的server证书

```diff
root@master01:~/cert# cat kube-apiserver-csr.json
{
+  "CN": "kube-apiserver",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
    "O": "system:HuaYang",
    "OU": "system"
  }],
    "hosts": [
+	"172.16.0.101",
+	"172.16.0.102",
+	"172.16.0.103",
+    "10.64.0.1",
+    "kubernetes-api.mykernel.cn",
+    "kubernetes",
+    "kubernetes.default",
+    "kubernetes.default.svc",
+    "kubernetes.default.svc.cluster",
+    "kubernetes.default.svc.cluster.local",
+	"127.0.0.1"
	]
}

```

> master集群地址
>
> 10.64.0.1 集群内部api service地址
>
> ​	我打算使用10.76.0.0/12网络，所以集群的首个地址是10.64.0.1/12
>
> ​	10.<0100 1100>.0.0
>
> ​     全1.<1111 0000>.0.0 掩码
>
> ​      10.<0100 0000>.0.0 运算结果：10.64.0.0/12 ,所以第一个是10.64.0.1
>
> ​    参考：http://blog.mykernel.cn/2021/02/08/IP%E5%9C%B0%E5%9D%80%E6%8D%A2%E7%AE%97/
>
> 访问https的端点
>
> https://kubernetes-api.mykernel.cn 集群外部入口的域名
>
> https://172.16.0.101:6443 https://172.16.0.102:6443  https://172.16.0.103:6443 由于api server在3个节点上，证书用于3个位置，所以有3个地址。
>
>  https://10.64.0.1 表示用户访问过来的地址，是10.64.0.1, 别人提供的证书的hosts包含这个地址，证书才有效。
>
> hosts表示，这个证书可以工作的位置并提供的端点。
>
> 如果给这个证书放在3个api server上，也给nginx使用，并且nginx提供的域名是www.magedu.com, 所以hosts添加www.magedu.com

```bash
root@master01:~/cert# ../cfssl gencert -ca=k8s-ca.pem -ca-key=k8s-ca-key.pem      --config=common-config.json -profile=kubernetes-ca      kube-apiserver-csr.json | ../cfssljson -bare kube-apiserver
2021/02/08 12:34:18 [INFO] generate received request
2021/02/08 12:34:18 [INFO] received CSR
2021/02/08 12:34:18 [INFO] generating key: rsa-2048
2021/02/08 12:34:19 [INFO] encoded CSR
2021/02/08 12:34:19 [INFO] signed certificate with serial number 229420910407236101093538869581356375943239359495
2021/02/08 12:34:19 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

root@master01:~/cert# find . -mmin -1
.
./kube-apiserver.pem
./kube-apiserver-key.pem
./kube-apiserver.csr

```



api 连接kubelet，发出命令调用docker引擎启动pod

| 默认 CN                       | 父级 CA       | O (位于 Subject 中) | 类型   | **主机 (SAN)** |
| ----------------------------- | ------------- | ------------------- | ------ | -------------- |
| kube-apiserver-kubelet-client | kubernetes-ca | system:masters      | client |                |
|                               |               |                     |        |                |

| 默认 CN                       | 建议的密钥路径               | 建议的证书路径               | 命令           | 密钥参数             | 证书参数                     |
| ----------------------------- | ---------------------------- | ---------------------------- | -------------- | -------------------- | ---------------------------- |
| kube-apiserver-kubelet-client | apiserver-kubelet-client.key | apiserver-kubelet-client.crt | kube-apiserver | --kubelet-client-key | --kubelet-client-certificate |

```diff
root@master01:~/cert# cp kube-apiserver-csr.json kube-apiserver-kubelet-client-csr.json
root@master01:~/cert# cat kube-apiserver-kubelet-client-csr.json
{
+  "CN": "kube-apiserver-kubelet-client",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
+    "O": "system:masters",
    "OU": "system"
  }],
+    "hosts": []
}

```

> cn, o, hosts

```bash
root@master01:~/cert# ../cfssl gencert -ca=k8s-ca.pem -ca-key=k8s-ca-key.pem      --config=common-config.json -profile=kubernetes-ca      kube-apiserver-kubelet-client-csr.json | ../cfssljson -bare kube-apiserver-kubelet-client
2021/02/08 12:41:52 [INFO] generate received request
2021/02/08 12:41:52 [INFO] received CSR
2021/02/08 12:41:52 [INFO] generating key: rsa-2048
2021/02/08 12:41:52 [INFO] encoded CSR
2021/02/08 12:41:52 [INFO] signed certificate with serial number 488149578516609332130142288698031832913016853353
2021/02/08 12:41:52 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

root@master01:~/cert# find . -mmin -1
.
./kube-apiserver-kubelet-client-csr.json
./kube-apiserver-kubelet-client-key.pem
./kube-apiserver-kubelet-client.pem
./kube-apiserver-kubelet-client.csr

```



| 默认 CN            | 建议的密钥路径         | 建议的证书路径         | 命令                    | 密钥参数                | 证书参数                       |
| ------------------ | ---------------------- | ---------------------- | ----------------------- | ----------------------- | ------------------------------ |
| front-proxy-client | front-proxy-client.key | front-proxy-client.crt | kube-apiserver          | --proxy-client-key-file | --proxy-client-cert-file       |
| front-proxy-ca     | front-proxy-ca.key     | front-proxy-ca.crt     | kube-controller-manager |                         | --requestheader-client-ca-file |
| front-proxy-ca     | front-proxy-ca.key     | front-proxy-ca.crt     | kube-apiserver          |                         | --requestheader-client-ca-file |

| 默认 CN            | 父级 CA                   | O (位于 Subject 中) | 类型   | 主机 (SAN) |
| ------------------ | ------------------------- | ------------------- | ------ | ---------- |
| front-proxy-client | kubernetes-front-proxy-ca |                     | client |            |
|                    |                           |                     |        |            |

api server里是aggregator 反代内部的api server, 如果用户自定义api server, 需要使用自定义资源时，就需要在api server反代到aggregated pod, 所以也需要证书

```diff
root@master01:~/cert# cp kube-apiserver-csr.json front-proxy-client-csr.json
root@master01:~/cert# cat front-proxy-client-csr.json
{
+  "CN": "front-proxy-client",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
    "O": "system:HuaYang",
    "OU": "system"
  }],
+    "hosts": []
}

```

```bash
root@master01:~/cert# ../cfssl gencert -ca=k8s-ca.pem -ca-key=k8s-ca-key.pem      --config=common-config.json -profile=kubernetes-ca      front-proxy-client-csr.json | ../cfssljson -bare front-proxy-client
2021/02/08 12:48:46 [INFO] generate received request
2021/02/08 12:48:46 [INFO] received CSR
2021/02/08 12:48:46 [INFO] generating key: rsa-2048
2021/02/08 12:48:47 [INFO] encoded CSR
2021/02/08 12:48:47 [INFO] signed certificate with serial number 235607205505692462747874116469767403063567420592
2021/02/08 12:48:47 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

root@master01:~/cert# find . -mmin -1
.
./kube-apiserver-kubelet-client-key.pem
./kube-apiserver-kubelet-client.pem
./kube-apiserver-kubelet-client.csr
./front-proxy-client-csr.json

```









准备目录结构

```bash
# ca
cp k8s-ca.pem /etc/kubernetes/pki/ca.crt
cp k8s-ca-key.pem /etc/kubernetes/pki/ca.key
# api -> kubelet
cp  kube-apiserver-kubelet-client.pem /etc/kubernetes/pki/apiserver-kubelet-client.crt
cp  kube-apiserver-kubelet-client-key.pem /etc/kubernetes/pki/apiserver-kubelet-client.key
# api的 aggregator -> aggregated pod
cp front-proxy-client.pem  /etc/kubernetes/pki/front-proxy-client.crt
cp front-proxy-client-key.pem  /etc/kubernetes/pki/front-proxy-client.key

# api的 aggregator 与 aggregated通信的ca
cp kubernetes-front-proxy-ca.pem /etc/kubernetes/pki/front-proxy-ca.crt
cp kubernetes-front-proxy-ca-key.pem /etc/kubernetes/pki/front-proxy-ca.key

# service account key
cp k8s-ca.pem /etc/kubernetes/pki/sa.pub
cp k8s-ca-key.pem /etc/kubernetes/pki/sa.key

# 生成token
BOOTSTRAP_TOKEN="$(head -c 6 /dev/urandom | md5sum | head -c 6).$(head -c 16 /dev/urandom | md5sum | head -c 16)"
echo "$BOOTSTRAP_TOKEN,\"system:bootstrapper\",10001,\"system:bootstrappers\"" > /etc/kubernetes/token.csv

# api server的证书
cp kube-apiserver.pem /etc/kubernetes/pki/apiserver.crt
cp kube-apiserver-key.pem /etc/kubernetes/pki/apiserver.key

```

以上生成证书后，略过以下生成证书直接配置api server组件



```bash
root@master01:~/k8s-certs-generator# pwd
/root/k8s-certs-generator
root@master01:~/k8s-certs-generator# ls
etcd  etcd-certs-gen.sh  gencerts.sh  k8s-certs-gen.sh  openssl.conf  README.md
root@master01:~/k8s-certs-generator# bash gencerts.sh k8s
Enter Domain Name [ilinux.io]: magedu.com              # 主机名后缀
Enter Kubernetes Cluster Name [kubernetes]: kubernetes # 集群名无所谓
Enter the IP Address in default namespace 
  of the Kubernetes API Server[10.96.0.1]:  10.96.0.1           # api地址暴露在默认名称空间中，这个应该使用service网段的第1个地址
Enter Master servers name[master01 master02 master03]: master01 master02 master03       # 主机名master01.magedu.com 但是之前domain name已经写了，所以会自动补后缀      


root@master01:~/k8s-certs-generator# tree kubernetes/
kubernetes/ # 公用证书
├── CA # k8s CA证书
│   ├── ca.crt
│   └── ca.key
├── front-proxy # 用户 -> aggregator -> 自定义APIservice: api http-> 自定义pod; 自定义pod -> api可以使用 front-proxy-client.crt
│   ├── front-proxy-ca.crt
│   ├── front-proxy-ca.key
│   ├── front-proxy-client.crt
│   └── front-proxy-client.key
├── ingress
│   ├── ingress-server.crt
│   ├── ingress-server.key
│   └── patches
│       └── ingress-tls.patch
├── kubelet
│   ├── auth
│   │   ├── bootstrap.conf
│   │   └── kube-proxy.conf
│   └── pki
│       ├── ca.crt
│       ├── kube-proxy.crt
│       └── kube-proxy.key
├── master01 # master01专用目录
│   ├── auth
│   │   ├── admin.conf
│   │   ├── controller-manager.conf
│   │   └── scheduler.conf
│   ├── pki
│   │   ├── apiserver.crt
│   │   ├── apiserver-etcd-client.crt # etcd生成的给api server客户端证书
│   │   ├── apiserver-etcd-client.key
│   │   ├── apiserver.key
│   │   ├── apiserver-kubelet-client.crt
│   │   ├── apiserver-kubelet-client.key
│   │   ├── ca.crt
│   │   ├── ca.key
│   │   ├── front-proxy-ca.crt #
│   │   ├── front-proxy-ca.key
│   │   ├── front-proxy-client.crt
│   │   ├── front-proxy-client.key
│   │   ├── kube-controller-manager.crt
│   │   ├── kube-controller-manager.key
│   │   ├── kube-scheduler.crt
│   │   ├── kube-scheduler.key
│   │   ├── sa.key
│   │   └── sa.pub
│   └── token.csv # 基于token认证的配置文件
```

准备证书

```bash
scp -rp kubernetes/master01 master01:/etc/kubernetes
scp -rp kubernetes/master02 master02:/etc/kubernetes
scp -rp kubernetes/master03 master03:/etc/kubernetes

# 检验
root@master01:~/k8s-certs-generator# ls /etc/kubernetes/
auth  pki  token.csv

root@master02:~# ls /etc/kubernetes/
auth  pki  token.csv

root@master03:~# ls /etc/kubernetes/
auth  pki  token.csv
root@master03:~# tree /etc/kubernetes/
/etc/kubernetes/
├── auth
│   ├── admin.conf                # root权限
│   ├── controller-manager.conf   # kubeconfig,用户为system:kube-controller-manager, 已经集群内建绑定应有的权限
│   └── scheduler.conf            # kubeconfig,用户为system:kube-scheduler, 已经集群内建绑定应有的权限
├── pki                 # 证书
│   ├── apiserver.crt
│   ├── apiserver-etcd-client.crt
│   ├── apiserver-etcd-client.key
│   ├── apiserver.key
│   ├── apiserver-kubelet-client.crt
│   ├── apiserver-kubelet-client.key
│   ├── ca.crt
│   ├── ca.key
│   ├── front-proxy-ca.crt
│   ├── front-proxy-ca.key
│   ├── front-proxy-client.crt
│   ├── front-proxy-client.key
│   ├── kube-controller-manager.crt
│   ├── kube-controller-manager.key
│   ├── kube-scheduler.crt
│   ├── kube-scheduler.key
│   ├── sa.key
│   └── sa.pub
└── token.csv # 完成bootstrap的token和用户和组保存位置，可以用来对用户授权

```

## 配置master01 *

```bash
root@master01:~# tar xvf kubernetes-server-linux-amd64.tar.gz -C /usr/local/

root@master01:/usr/local# tree kubernetes/server/
kubernetes/server/
└── bin
    ├── apiextensions-apiserver
    ├── kubeadm
    ├── kube-aggregator
    ├── kube-apiserver
    ├── kube-apiserver.docker_tag
    ├── kube-apiserver.tar
    ├── kube-controller-manager
    ├── kube-controller-manager.docker_tag
    ├── kube-controller-manager.tar
    ├── kubectl
    ├── kubelet
    ├── kube-proxy
    ├── kube-proxy.docker_tag
    ├── kube-proxy.tar
    ├── kube-scheduler
    ├── kube-scheduler.docker_tag
    ├── kube-scheduler.tar
    └── mounter

```

各个程序的配置文件写起来麻烦，有现成的

https://github.com/iKubernetes/k8s-bin-inst

```bash
root@master01:~# git clone https://github.com/iKubernetes/k8s-bin-inst
```





### api server配置介绍

```bash
k8s-bin-inst/master/etc/kubernetes/config
# 所有应用程序公共配置
KUBE_LOGTOSTDERR="--logtostderr=true" # 日志

# journal message level, 0 is debug
# 级别，生产不应该是debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
# 允许特权容器
KUBE_ALLOW_PRIV="--allow-privileged=true"
```

api sever

```bash
# 监听地址
KUBE_API_ADDRESS="--advertise-address=0.0.0.0"

# 端口 insecure-port非0就是8080
KUBE_API_PORT="--secure-port=6443 --insecure-port=0"

# 修改为etcd集群地址
KUBE_ETCD_SERVERS="--etcd-servers=https://k8s-master01.ilinux.io:2379,https://k8s-master02.ilinux.io:2379,https://k8s-master03.ilinux.io:2379"

# 集群地址，要和生产证书的默认地址在同一个网段
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.96.0.0/12"

# 启动准入控制器
KUBE_ADMISSION_CONTROL="--enable-admission-plugins=NodeRestriction"

# authorization-mode启动认证插件
# client-ca-file 客户端认证ca
# enable-bootstrap-token-auth node引导时基于引导令牌认证. 那么多kubelet生成证书麻烦
	# 1. api server内部有自动签署工具：--controllers=*,bootstrapsigner,tokencleanerr, 拿着令牌的人属于system:bootstrappers 组
	# 2. kubelet在node节点起动时，自动生成一个key, csr.
	# 3. node的kubelet 第一次连接api server时, 先认证共享密钥，共享密钥认证通过，kubelet才可以将csr发给api server
	# 4. api server签署证书，将证书返回给node
	
	# server端支持的token  --token-auth-file=/etc/kubernetes/token.csv 
	
	
# Api -> ETCD
# etcd-* apiserver连接etcd的ca/证书


# Api -> kubelet
# kubelet-* k8s api server需要通知kubelet, 连接https

# Api aggregator -> aggregated pod
# proxy* api server需要连接aggregator. 
# requestheader* aggregated的pod, 允许哪个证书来认证。 aggregator向后端代理时添加的首部。


# service-account-key-file 每个pod有默认的serviceaccount, 而且这个serviceaccount会使用密钥连接api server, 需要确定pod的service身份。
## 是公钥加密的 sa.pub公钥，sa.key私钥在 apiserver服务端。所以pod使用公钥签名，apiserver 使用私钥解密确认pod身份发送的信息没有被篡改。

# ApiServer 自已的证书
# tls* api server作为server端的证书


KUBE_API_ARGS="--authorization-mode=Node,RBAC \ 
    --client-ca-file=/etc/kubernetes/pki/ca.crt \
    --enable-bootstrap-token-auth=true \
    --token-auth-file=/etc/kubernetes/token.csv \
    --etcd-cafile=/etc/etcd/pki/ca.crt \
    --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt \
    --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key \
    --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt \
    --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key \
    --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \
    --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt \
    --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key \
    --requestheader-allowed-names=front-proxy-client \
    --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt \
    --requestheader-extra-headers-prefix=X-Remote-Extra- \
    --requestheader-group-headers=X-Remote-Group \
    --requestheader-username-headers=X-Remote-User\
    --service-account-key-file=/etc/kubernetes/pki/sa.pub \
    --tls-cert-file=/etc/kubernetes/pki/apiserver.crt \
    --tls-private-key-file=/etc/kubernetes/pki/apiserver.key"


```

unit-file

```bash
root@master01:~# cat k8s-bin-inst/master/unit-files/kube-apiserver.service 
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/config 
EnvironmentFile=-/etc/kubernetes/apiserver 
User=kube # 启动服务的用户
ExecStart=/usr/local/kubernetes/server/bin/kube-apiserver \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBE_ETCD_SERVERS \
	    $KUBE_API_ADDRESS \
	    $KUBE_API_PORT \
	    $KUBELET_PORT \
	    $KUBE_ALLOW_PRIV \
	    $KUBE_SERVICE_ADDRESSES \
	    $KUBE_ADMISSION_CONTROL \
	    $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

### 配置api server组件 *

```bash

git clone https://github.com/iKubernetes/k8s-bin-inst
cp -a k8s-bin-inst/master/etc/kubernetes/* /etc/kubernetes/
root@master01:~# cp -a k8s-bin-inst/master/unit-files/* /lib/systemd/system/


# 检验 master端的配置，kubeconfig,证书
root@master01:~# tree /etc/kubernetes/
/etc/kubernetes/
├── apiserver # 配置
├── auth # kubeconfig
│   ├── admin.conf
│   ├── controller-manager.conf
│   └── scheduler.conf
├── config # 配置
├── controller-manager # 配置
├── pki # 证书
│   ├── apiserver.crt
│   ├── apiserver-etcd-client.crt
│   ├── apiserver-etcd-client.key
│   ├── apiserver.key
│   ├── apiserver-kubelet-client.crt
│   ├── apiserver-kubelet-client.key
│   ├── ca.crt
│   ├── ca.key
│   ├── front-proxy-ca.crt
│   ├── front-proxy-ca.key
│   ├── front-proxy-client.crt
│   ├── front-proxy-client.key
│   ├── kube-controller-manager.crt
│   ├── kube-controller-manager.key
│   ├── kube-scheduler.crt
│   ├── kube-scheduler.key
│   ├── sa.key
│   └── sa.pub
├── scheduler # 配置
└── token.csv # bootstrapper


# unit文件
```

修改config文件

> 日志级别，0调度方便，生产中为1-2甚至更高



修改api server文件

> etcd
>
> service cluster ip range 和生成证书的网络一样

```diff
# /etc/kubernetes/apiserver 
#:s@k8s-master@master@g
#:s@ilinux.io@magedu.com@g

KUBE_API_ADDRESS="--advertise-address=0.0.0.0"

# The port on the local server to listen on.
KUBE_API_PORT="--secure-port=6443 --insecure-port=0"

# Comma separated list of nodes in the etcd cluster
+KUBE_ETCD_SERVERS="--etcd-servers=https://etcd01.magedu.com:2379,https://etcd02.magedu.com:2379,https://etcd03.magedu.com:2379"

# Address range to use for services
+KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.96.0.0/12"

# default admission control policies
KUBE_ADMISSION_CONTROL="--enable-admission-plugins=NodeRestriction"

# Add your own!
KUBE_API_ARGS="--authorization-mode=Node,RBAC \
    --client-ca-file=/etc/kubernetes/pki/ca.crt \
    --enable-bootstrap-token-auth=true \
    --etcd-cafile=/etc/etcd/pki/ca.crt \
    --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt \
    --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key \
    --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt \
    --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key \
    --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \
    --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt \
    --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key \
    --requestheader-allowed-names=front-proxy-client \
    --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt \
    --requestheader-extra-headers-prefix=X-Remote-Extra- \
    --requestheader-group-headers=X-Remote-Group \
    --requestheader-username-headers=X-Remote-User\
    --service-account-key-file=/etc/kubernetes/pki/sa.pub \
+	--service-account-signing-key-file=/etc/kubernetes/pki/sa.key \
+	--service-account-issuer=api \
+   --service-node-port-range=20000-50000 \
+   --enable-aggregator-routing=true \
    --tls-cert-file=/etc/kubernetes/pki/apiserver.crt \
    --tls-private-key-file=/etc/kubernetes/pki/apiserver.key \
    --token-auth-file=/etc/kubernetes/token.csv  "
```

> 10.96.0.0/12 要和生成证书的service API同网段
>
> --enable-aggregator-routing=true 在master没有配置kube-proxy就不会将service变为ipvs/iptables规则，所以需要指定此选项

> https://jpweber.io/blog/a-look-at-tokenrequest-api/



修改unit文件 

> 准备kube用户

k8s临时状态数据

```bash
useradd -r kube
root@master01:~# chown kube.kube /etc/kubernetes/pki/*.key
```

token文件

```bash
root@master01:~# cat /etc/kubernetes/token.csv
6f51ec.24b8a00891db8d35,"system:bootstrapper",10001,"system:bootstrappers"

# 第1个字段是kubelet认证的token
# 2 认证之后的用户
# 4 认证之后的组
```



启动

配置日志级别

```diff
root@master01:~# cat /etc/kubernetes/config 
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
+KUBE_LOG_LEVEL="--v=10"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

```



```bash

root@master01:~# systemctl start kube-apiserver
systemctl status kube-apiserver

# 切换终端查看日志
journalctl -u kube-apiserver --since="$(date -d"-10 second" +"%F %T")"


# 避免重启主机集群异常
root@master01:~# systemctl enable kube-apiserver

# 启动慢，连接etcd需要初始化
```



### 配置kubectl命令

生成管理员账号, 由于上面的kubelet-client属于system:masters组，直接使用这个证书生成master即可

```bash
# 生成配置文件
/usr/local/kubernetes/server/bin/kubectl config set-cluster kubernetes --server=https://172.16.0.101:6443  --certificate-authority=/etc/kubernetes/pki/ca.crt  --embed-certs=true  --kubeconfig=/etc/kubernetes/auth/admin.conf

# 设定集群用户
/usr/local/kubernetes/server/bin/kubectl config set-credentials admin --client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --username=admin --embed-certs=true  --kubeconfig=/etc/kubernetes/auth/admin.conf
	#--username=admin 集群用户，无所谓，重点是证书的用户
	


```

```diff
./cfssl-certinfo -cert /etc/kubernetes/pki/apiserver-kubelet-client.crt 
  "subject": {
+    "common_name": "kube-apiserver-kubelet-client", # 这个不重要， 因为基于管理员授权
    "country": "China",
    "organization": "system:masters",
    "organizational_unit": "system",
    "locality": "ChengDu",
    "province": "SiChuan",
    "names": [
      "China",
      "SiChuan",
      "ChengDu",
+      "system:masters", # 集群管理员
      "system",
      "kube-apiserver-kubelet-client"
    ]
  },

```

```bash
# 设定上下文
root@master01:~# /usr/local/kubernetes/server/bin/kubectl config set-context  admin@kubernetes  --cluster=kubernetes --user=admin --kubeconfig=/etc/kubernetes/auth/admin.conf
Context "admin@kubernetes" created.

# 切换当前上下文
 /usr/local/kubernetes/server/bin/kubectl config use-context  admin@kubernetes  --kubeconfig=/etc/kubernetes/auth/admin.conf
 
 
 # 查看配置 
 root@master01:~# /usr/local/kubernetes/server/bin/kubectl config view --kubeconfig=/etc/kubernetes/auth/admin.conf
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.16.0.101:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: admin
  name: admin@kubernetes
current-context: admin@kubernetes
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
    username: admin


```





脚本已经生成连接当前集群的管理账号

```bash
root@master01:~# install -dv ~/.kube
root@master01:~# cp /etc/kubernetes/auth/admin.conf ~/.kube/config

root@master01:~# ln -sv /usr/local/kubernetes/server/bin/kubectl /usr/bin/

root@master01:~# kubectl config view 
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://master01.magedu.com:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: k8s-admin
  name: k8s-admin@kubernetes
current-context: k8s-admin@kubernetes
kind: Config
preferences: {}
users:
- name: k8s-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED


```

查看主节点的端口

```bash
root@master01:~# ss -tnl | grep 6443
LISTEN   0         20480                     *:6443                   *:*       

root@master01:~# kubectl get all -A
NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.64.0.1    <none>        443/TCP   12m



# service是iptables/ipvs规则所以只有在kube-proxy启动才有用。
# 生成证书是10.64.0.1 加入证书，这个IP将不能在集群内通信，需要重建集群，使用这个IP作为入口
```

### 允许node加入集群

添加system:bootstraper 用户 绑定 system:node-bootstrapper 角色，即用户就有了可以拉起一个node的权限

```bash
# api server通过这个 token文件，来认证kubelet
root@master01:~# cat /etc/kubernetes/token.csv 
6f51ec.24b8a00891db8d35,"system:bootstrapper",10001,"system:bootstrappers"

# kubelet的bootstrapper选项对应的Kubeconfig是token生成的文件，用户只要携带token(bootstrap kubeconfig中的credential 并不是用户)， apiserver只要识别到token和master一致，就当作第2和第4个字段对应的用户和组。

# 所以只要授权到用户和组，拿着token来的人就可以有对应的权限。
```

- 角色 - 用户

  ```bash
  root@master01:~# kubectl create clusterrolebinding system:bootstrapper --user=system:bootstrapper --clusterrole=system:node-bootstrapper
  ```

  

- 角色 - 组

  ```bash
  root@master01:~# kubectl create clusterrolebinding system:bootstrapper --group=system:bootstrappers --clusterrole=system:node-bootstrapper
  ```

  

```bash
root@master01:~# kubectl get clusterrolebinding system:bootstrapper
NAME                  ROLE                                   AGE
system:bootstrapper   ClusterRole/system:node-bootstrapper   25s

```



因为api server端启动的token认证信息的这个用户(第2个字段)，这个组(4字段)，默认不允许bootstrapper, 所以以上就授权

```bash
root@master01:~# cat /etc/kubernetes/token.csv 
cdde8f.75e1739dd28b5499,"system:bootstrapper",10001,"system:bootstrappers"
```

然后api server启动指定这个 token, node节点启动kubelet指定的配置文件用户

```bash
kubelet --bootstrap-kubeconfig=/etc/kubernetes/auth/bootstrap.conf"
```

node的kubelet 第一次连接api server时, 先认证共享密钥（cdde8f.75e1739dd28b5499，共享密钥认证通过，api 就识别为2/4字段的用户或组，并且用户有拉集群的权限，kubelet才可以将csr发给api server, kube-controller才签发证书



查看kubelet与kube-apiserver通信

```bash
root@master01:~# systemctl status kube-apiserver
```

接下来配置kube-controller，就可以完成签发证书

### controller manager配置介绍

```bash
# 通用配置
root@master01:~# cat /etc/kubernetes/config
略，参考 api server配置介绍

# controller manager配置
root@master01:~# cat /etc/kubernetes/controller-manager 
# --bind-address=127.0.0.1  自已监听地址127.0.0.1 因为api server绑定在0.0.0.0 所以可以直接与api server通信
# --cluster-cidr=10.244.0.0/16 # pod网络插件的网段; flannel 10.244.0.0/16; calico 192.168.0.0/16;
# --node-cidr-mask-size=24  # flannel会给每个节点划分一个子网，这个子网的掩码是多长？ 24位就是划分255个子网
#  --service-cluster-ip-range=10.96.0.0/12  # service的网段

# cluster-signing* controller-manager给kubelet签发证书的ca

# authentication* 连接api server的kubeconfig文件
    # kubectl config view --kubeconfig=/etc/kubernetes/auth/controller-manager.conf
    #users:
    #- name: system:kube-controller-manager # controller-manager用户的用户名, 需要k8s内置的用户名，api server启动时将这个用户绑定至了controller-manager应该有的角色上。system:kube-controller-manager

# --controllers=*,bootstrapsigner,tokencleaner 
	# kubelet提交 证书签署请求 csr. signer来签署
	# tokencleaner 会清理token. 
	# 为了避免node盗用，应该api server指定的token定期换，node加入时就需要新的token
	
#  --leader-elect=true 默认controller启动分布式锁，抢占endpoint资源


KUBE_CONTROLLER_MANAGER_ARGS="--bind-address=127.0.0.1 \
    --allocate-node-cidrs=true \
    --service-cluster-ip-range=10.96.0.0/12 \
    --authentication-kubeconfig=/etc/kubernetes/auth/controller-manager.conf \
    --authorization-kubeconfig=/etc/kubernetes/auth/controller-manager.conf \
    --client-ca-file=/etc/kubernetes/pki/ca.crt \
    --cluster-cidr=10.244.0.0/16 \
    --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt \
    --cluster-signing-key-file=/etc/kubernetes/pki/ca.key \
    --controllers=*,bootstrapsigner,tokencleaner \
    --kubeconfig=/etc/kubernetes/auth/controller-manager.conf \
    --leader-elect=true \
    --node-cidr-mask-size=24 \
    --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt \
    --root-ca-file=/etc/kubernetes/pki/ca.crt \
    --service-account-private-key-file=/etc/kubernetes/pki/sa.key \
    --use-service-account-credentials=true"

```

### 配置controller manager组件 *

先生成证书 https://kubernetes.io/zh/docs/setup/best-practices/certificates/#%E6%89%8B%E5%8A%A8%E9%85%8D%E7%BD%AE%E8%AF%81%E4%B9%A6

| 文件名                  | 凭据名称                   | 默认 CN                               | O (位于 Subject 中) |
| ----------------------- | -------------------------- | ------------------------------------- | ------------------- |
| admin.conf              | default-admin              | kubernetes-admin                      | system:masters      |
| kubelet.conf            | default-auth               | system:node:`<nodeName>` （参阅注释） | system:nodes        |
| controller-manager.conf | default-controller-manager | system:kube-controller-manager        |                     |
| scheduler.conf          | default-scheduler          | system:kube-scheduler                 |                     |

```diff
root@master01:~/cert# cp kube-apiserver-csr.json kube-controller-manager-csr.json
root@master01:~/cert# cat kube-controller-manager-csr.json 
{
+  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
    "O": "system:HuaYang",
    "OU": "system"
  }],
+    "hosts": []
}

```

> 因为是controller连接kube-apiserver,所以仅是client.

```bash
root@master01:~/cert# ../cfssl gencert -ca=k8s-ca.pem -ca-key=k8s-ca-key.pem      --config=common-config.json -profile=kubernetes-ca     kube-controller-manager-csr.json | ../cfssljson -bare kube-controller-manager
2021/02/08 16:29:48 [INFO] generate received request
2021/02/08 16:29:48 [INFO] received CSR
2021/02/08 16:29:48 [INFO] generating key: rsa-2048
2021/02/08 16:29:49 [INFO] encoded CSR
2021/02/08 16:29:49 [INFO] signed certificate with serial number 537691745436972979906472672552534554649055901244
2021/02/08 16:29:49 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
root@master01:~/cert# find . -mmin -1
.
./kube-controller-manager.csr
./kube-controller-manager-key.pem
./kube-controller-manager.pem
```

准备一些证书文件

```bash
export KUBECONFIG=/etc/kubernetes/auth/controller-manager.conf
export ADDR=https://172.16.0.101:6443
export USERNAME=default-controller-manager
export CERT=kube-controller-manager.pem
export KEY=kube-controller-manager-key.pem

kubectl config set-cluster default-cluster --server=$ADDR --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs
kubectl config set-credentials $USERNAME --client-certificate=$CERT --client-key=$KEY --username=$USERNAME --embed-certs=true 
kubectl config set-context default-system --cluster default-cluster --user $USERNAME
kubectl config use-context default-system
export KUBECONFIG=
chown kube.kube /etc/kubernetes/auth/controller-manager.conf

```





```diff
# cat  /etc/kubernetes/controller-manager 

KUBE_CONTROLLER_MANAGER_ARGS="--bind-address=127.0.0.1 \
    --allocate-node-cidrs=true \
+    --service-cluster-ip-range=10.96.0.0/12 \
    --authentication-kubeconfig=/etc/kubernetes/auth/controller-manager.conf \
    --authorization-kubeconfig=/etc/kubernetes/auth/controller-manager.conf \
    --client-ca-file=/etc/kubernetes/pki/ca.crt \
+    --cluster-cidr=10.244.0.0/16 \
    --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt \
    --cluster-signing-key-file=/etc/kubernetes/pki/ca.key \
    --controllers=*,bootstrapsigner,tokencleaner \
    --kubeconfig=/etc/kubernetes/auth/controller-manager.conf \
    --leader-elect=true \
+    --node-cidr-mask-size=24 \
    --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt \
    --root-ca-file=/etc/kubernetes/pki/ca.crt \
    --service-account-private-key-file=/etc/kubernetes/pki/sa.key \
+    --horizontal-pod-autoscaler-sync-period=5s \
+     --node-monitor-grace-period 20s \
+      --leader-elect-resource-lock=endpoints \
     --use-service-account-credentials=true"
```

> service-cluster-ip-range 证书会指定，api server会指定
>
> cluster-cidr 网络插件的网段 
>
> 新版默认锁的leases资源，我们也可以锁endpoints资源，保证抢占到资源的组件是主组件

```bash
root@master01:~# systemctl start kube-controller-manager
root@master01:~# systemctl status kube-controller-manager
● kube-controller-manager.service - Kubernetes Controller Manager
   Loaded: loaded (/lib/systemd/system/kube-controller-manager.service; disabled; vendor preset: enabled)
   Active: active (running) since Sat 2021-02-06 14:30:51 CST; 2s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 6284 (kube-controller)
    Tasks: 8 (limit: 2318)
   CGroup: /system.slice/kube-controller-manager.service
           └─6284 /usr/local/kubernetes/server/bin/kube-controller-manager --logtostderr=true --v=0 --bind-address=127.0.0.1 --allocate-node-cidrs=true --authentication-kubeconfig=/etc/kubernetes/auth/controller-manager.conf --authorization-kubeconfig=/etc/kubernetes/auth/controller-manager.conf --client-ca-file=/

Feb 06 14:30:51 master01.magedu.com systemd[1]: Started Kubernetes Controller Manager.
Feb 06 14:30:52 master01.magedu.com kube-controller-manager[6284]: I0206 14:30:52.483592    6284 serving.go:331] Generated self-signed cert in-memory



root@master01:~# systemctl enable kube-controller-manager

root@master01:~/cert# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
controller-manager   Healthy     ok                                                                                            
etcd-1               Healthy     {"health": "true"}                                                                            
etcd-2               Healthy     {"health": "true"}                                                                            
etcd-0               Healthy     {"health": "true"}                     


# 确保有权限，这是
root@master01:~/cert# kubectl get    leases --kubeconfig=/etc/kubernetes/auth/controller-manager.conf   -A
NAMESPACE     NAME                      HOLDER                                                      AGE
kube-system   kube-controller-manager   master01.mykernel.cn_0ba4eb7f-40af-4f88-b2d5-a41792c85a86   15m

root@master01:~/cert# kubectl get    endpoints --kubeconfig=/etc/kubernetes/auth/controller-manager.conf   -n kube-system
NAME                      ENDPOINTS   AGE
kube-controller-manager   <none>      109s

```

### kube-scheduler配置介绍

```bash
root@master01:~# cat /etc/kubernetes/scheduler 
# kubeconfig 连接api server
# root@master01:~# kubectl config view --kubeconfig=/etc/kubernetes/auth/scheduler.conf
#users:
#- name: system:kube-scheduler # 必须这个用户名，因为默认集群binding了这个用户至scheduler应有的权限

# leader-elect 分布式高可用，必须启用

KUBE_SCHEDULER_ARGS="--address=127.0.0.1 \
    --kubeconfig=/etc/kubernetes/auth/scheduler.conf \
    --leader-elect=true"

```

### 配置kube-scheduler组件 *

先生成证书 https://kubernetes.io/zh/docs/setup/best-practices/certificates/#%E6%89%8B%E5%8A%A8%E9%85%8D%E7%BD%AE%E8%AF%81%E4%B9%A6

| 文件名                  | 凭据名称                   | 默认 CN                               | O (位于 Subject 中) |
| ----------------------- | -------------------------- | ------------------------------------- | ------------------- |
| admin.conf              | default-admin              | kubernetes-admin                      | system:masters      |
| kubelet.conf            | default-auth               | system:node:`<nodeName>` （参阅注释） | system:nodes        |
| controller-manager.conf | default-controller-manager | system:kube-controller-manager        |                     |
| scheduler.conf          | default-scheduler          | system:kube-scheduler                 |                     |

```diff
root@master01:~/cert# cat kube-scheduler-csr.json 
{
+  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
    "O": "system:HuaYang",
    "OU": "system"
  }],
+    "hosts": []
}

```

> 因为是kube-scheduler连接kube-apiserver,所以仅是client.

```bash
root@master01:~/cert# ../cfssl gencert -ca=k8s-ca.pem -ca-key=k8s-ca-key.pem      --config=common-config.json -profile=kubernetes-ca     kube-scheduler-csr.json | ../cfssljson -bare kube-scheduler
2021/02/08 17:00:43 [INFO] generate received request
2021/02/08 17:00:43 [INFO] received CSR
2021/02/08 17:00:43 [INFO] generating key: rsa-2048
2021/02/08 17:00:43 [INFO] encoded CSR
2021/02/08 17:00:43 [INFO] signed certificate with serial number 435725102947513299471981934859586414569861423441
2021/02/08 17:00:43 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").


root@master01:~/cert# find . -mmin -1
.
./kube-scheduler.pem
./kube-scheduler-csr.json
./kube-scheduler.csr
./kube-scheduler-key.pem

```

准备一些证书文件

```bash
export KUBECONFIG=/etc/kubernetes/auth/scheduler.conf # 配置文件中指定这个位置
export ADDR=https://172.16.0.101:6443 # 运行kube-scheduler节点的api server, 如果都指向 一个位置，api server压力大
export USERNAME=default-scheduler
export CERT=kube-scheduler.pem
export KEY=kube-scheduler-key.pem

kubectl config set-cluster default-cluster --server=$ADDR --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs
kubectl config set-credentials $USERNAME --client-certificate=$CERT --client-key=$KEY --username=$USERNAME --embed-certs=true 
kubectl config set-context default-system --cluster default-cluster --user $USERNAME
kubectl config use-context default-system

chown kube.kube $KUBECONFIG
export KUBECONFIG=
```





```bash
systemctl start kube-scheduler
root@master01:~# systemctl enable kube-scheduler

root@master01:~# systemctl status kube-scheduler


# 查看抢占资源
root@master01:~/cert# kubectl get leases -A
NAMESPACE     NAME                      HOLDER                                                      AGE
kube-system   kube-controller-manager   master01.mykernel.cn_0ba4eb7f-40af-4f88-b2d5-a41792c85a86   22m
kube-system   kube-scheduler            master01.mykernel.cn_73fff66d-9912-4df0-98a1-e1e672f07ba2   69s

```

```bash
root@master01:~/k8s-certs-generator# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}   
```

```bash
root@master01:~/cert# kubectl get all -A
NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.64.0.1    <none>        443/TCP   52m

```



## 配置node01添加master01 *



node节点需要运行Pod: (kubelet + docker)

期望service 工作 ipvs模式，需要装入ipvs模块



初始化，参考:

http://blog.mykernel.cn/2021/02/05/%E9%AB%98%E5%8F%AF%E7%94%A8Kubernetes/

### 安装部署docker容器引擎

```bash
# docker 和 二进制下载版本一样
```

https://developer.aliyun.com/mirror/docker-ce?spm=a2c6h.13651102.0.0.3e221b11JBwFyb

```bash
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"


# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# apt-cache madison docker-ce
#   docker-ce | 17.03.1~ce-0~ubuntu-xenial | https://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
#   docker-ce | 17.03.0~ce-0~ubuntu-xenial | https://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
# Step 2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.1~ce-0~ubuntu-xenial)
# sudo apt-get -y install docker-ce=[VERSION]
```

由于官方二进制是 1.20.2, 所以我就选择docker版本 

```bash
root@node01:~# apt install docker-ce=5:20.10.2~3-0~ubuntu-bionic -y
```

需要Pod的底层架构镜像，在gcr.io上，所以需要配置docker代理

`/lib/systemd/system/docker.service`

```diff
[Service]
+Environment=HTTPS_PROXY="192.168.0.33:808"
+Environment=NO_PROXY="127.0.0.0/8,172.16.0.0/16,10.244.0.0/16,10.96.0.0/12"
+ExecStartPost="/sbin/iptables -P FORWARD ACCEPT                                 
```

> pod  10.244.0.0/16 打算使用flannel
>
> service 10.96.0.0/12



此处也可以不加代理

```bash
root@node01:~# docker save  k8s.gcr.io/pause:3.2 coredns/coredns:1.8.0 quay.io/coreos/flannel:v0.13.1-rc2 -o node.tar
root@node01:~# scp node.tar node02.mykernel.cn:
root@ubuntu-template:~# docker load -i node.tar 
ba0dae6243cc: Loading layer [==================================================>]  684.5kB/684.5kB
Loaded image: k8s.gcr.io/pause:3.2
225df95e717c: Loading layer [==================================================>]  336.4kB/336.4kB
69ae2fbf419f: Loading layer [==================================================>]  42.24MB/42.24MB
Loaded image: coredns/coredns:1.8.0
50644c29ef5a: Loading layer [==================================================>]  5.845MB/5.845MB
0be670d27a91: Loading layer [==================================================>]  11.42MB/11.42MB
90679e912622: Loading layer [==================================================>]  2.267MB/2.267MB
6db5e246b16d: Loading layer [==================================================>]  45.69MB/45.69MB
97320fed8db7: Loading layer [==================================================>]   5.12kB/5.12kB
8a984b390686: Loading layer [==================================================>]  9.216kB/9.216kB
3b729894a01f: Loading layer [==================================================>]   7.68kB/7.68kB
Loaded image: quay.io/coreos/flannel:v0.13.1-rc2

```



```bash
systemctl daemon-reload
systemctl restart docker
```

```diff
root@node01:~# docker info

+HTTPS Proxy: 192.168.0.33:808
+No Proxy: 127.0.0.0/8,172.16.0.0/16,10.244.0.0/16,10.96.0.0/12
+WARNING: No swap limit support 
```

> https://www.k2zone.cn/?p=2356
>
> 解决WARNING, 不过WARNING可以不管

### kubelet配置介绍

v 1.12版本之后，kubelet/kube-proxy支持配置文件加载参数

```bash
root@node01:~# tar xvf kubernetes-node-linux-amd64.tar.gz  -C /usr/local
kubernetes/node/
kubernetes/node/bin/
kubernetes/node/bin/kube-proxy
kubernetes/node/bin/kubeadm
kubernetes/node/bin/kubectl
kubernetes/node/bin/kubelet
```

配置文件

```bash
root@master01:~# tree k8s-bin-inst/nodes/var/lib/
k8s-bin-inst/nodes/var/lib/
├── kubelet
│   └── config.yaml
└── kube-proxy
    └── config.yaml
```

kubelet配置

```diff
root@master01:~# cat k8s-bin-inst/nodes/var/lib/kubelet/config.yaml 
+address: 0.0.0.0 # 监听地址
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
+    clientCAFile: /etc/kubernetes/pki/ca.crt # kubelet的CA
+authorization: # 认证方式
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
+cgroupDriver: cgroupfs # 得和docker引擎保持一致。需要调用docker引擎启动Pod # docker info | grep 'Cgroup Driver'
cgroupsPerQOS: true
+clusterDNS: # 指定DNS的service地址用于Pod访问DNS.
+- 10.64.0.10
+clusterDomain: cluster.local # 指定集群域名后缀
configMapAndSecretChangeDetectionStrategy: Watch
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuCFSQuotaPeriod: 100ms
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
+failSwapOn: false              # swap启用状态下是否报错
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kind: KubeletConfiguration
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeLeaseDurationSeconds: 40
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
port: 10250
registryBurst: 10
registryPullQPS: 5
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
+staticPodPath: /etc/kubernetes/manifests 
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s

```

> 集群dns: 10.96.0.10

配置文件



 /etc/kubernetes配置文件

```bash
root@master01:~# cat k8s-bin-inst/nodes/etc/kubernetes/kubelet 

KUBELET_ADDRESS="--address=0.0.0.0"

# KUBELET_PORT="--port=10250"


# network-plugin k8s节点两个网络插件
 # 1. cni, 手工部署不会生成二进制程序
 # 2. kubenet

# config kubelet的配置

# kubeconfig 指定kubelet接入api server.
	# 平时通信使用
	# 在启动kubelet时生成kubeconfig文件
	
# bootstrap-kubeconfig 以bootstrapper方式加入集群。加入使用



KUBELET_ARGS="--network-plugin=cni \
    --config=/var/lib/kubelet/config.yaml \
    --kubeconfig=/etc/kubernetes/auth/kubelet.conf \
    --bootstrap-kubeconfig=/etc/kubernetes/auth/bootstrap.conf"
```



查看 unit文件

```bash
root@master01:~# cat k8s-bin-inst/nodes/unit-files/kubelet.service 
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/local/kubernetes/node/bin/kubelet \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBELET_API_SERVER \
	    $KUBELET_ADDRESS \
	    $KUBELET_PORT \
	    $KUBELET_HOSTNAME \
	    $KUBE_ALLOW_PRIV \
	    $KUBELET_ARGS
Restart=on-failure
KillMode=process
RestartSec=10

[Install]
WantedBy=multi-user.target

```

> 注意没有写明默认用户，则是管理员运行



查看bootstrapper的证书

```diff
root@master01:~# kubectl config view --kubeconfig=k8s-certs-generator/kubernetes/kubelet/auth/bootstrap.conf 
apiVersion: v1
clusters:
#- cluster:
+    certificate-authority-data: DATA+OMITTED # kubelet会获取这个公钥来签署证书，生成kubelet连接apiserver的kubeconfig
+    server: https://kubernetes-api.magedu.com:6443 # kubelet会获取这个地址作为api 入口
  name: kubernetes
contexts:
#- context:
    cluster: kubernetes
    user: system:bootstrapper
  name: system:bootstrapper@kubernetes
current-context: system:bootstrapper@kubernetes
kind: Config
preferences: {}
users:
#- name: system:bootstrapper
  user:
    token: REDACTED
```



### 配置kubelet组件 *

基于bootstrapper方式加入集群，

1. 基于token认证，签发证书
2. 随后认证，基于证书认证

或者直接加入集群

| 文件名       | 凭据名称     | 默认 CN                               | O (位于 Subject 中) |
| ------------ | ------------ | ------------------------------------- | ------------------- |
| kubelet.conf | default-auth | system:node:`<nodeName>` （参阅注释） | system:nodes        |



k8s api server是高可用的，所以node节点的组件`kubelet`, `kube-proxy`是连接高可用节点的位置，是https认证连接，所以连接的地址是在api server的hosts列表中

```diff
root@master01:~# ./cfssl-certinfo -cert cert/kube-apiserver.pem 
{
  "subject": {
    "common_name": "kube-apiserver",
    "country": "China",
    "organization": "system:HuaYang",
    "organizational_unit": "system",
    "locality": "ChengDu",
    "province": "SiChuan",
    "names": [
      "China",
      "SiChuan",
      "ChengDu",
      "system:HuaYang",
      "system",
      "kube-apiserver"
    ]
  },
  "sans": [
+    "kubernetes-api.mykernel.cn", # 集群入口
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local",
    "172.16.0.101",
    "172.16.0.102",
    "172.16.0.103",
    "10.64.0.1",
    "127.0.0.1"
  ],
```

由于Kubelet启动时，也是使用bootstrap的kubeconfig文件

```bash
export KUBECONFIG=/etc/kubernetes/auth/bootstrap.conf  # 配置文件中指定这个位置
export ADDR=https://kubernetes-api.mykernel.cn:6443 # 运行kube-scheduler节点的api server, 如果都指向 一个位置，api server压力大
export USERNAME=system:bootstrapper # 使用token认证时，不需要用户名，但是需要指定集群用户名，这个与认证无关，可以写system:bootstrapper，方便记忆
export TOKEN=3971b0.22aa4688dc55837b

kubectl config set-cluster default-cluster --server=$ADDR --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs
kubectl config set-credentials $USERNAME --token=$TOKEN
kubectl config set-context default-system --cluster default-cluster --user $USERNAME
kubectl config use-context default-system

chown kube.kube $KUBECONFIG
export KUBECONFIG=
```

生成bootstrap配置

```bash
export DOMAIN=mykernel.cn
bash deploy.sh node01.mykernel.cn
root@node01:~# install -dv /etc/kubernetes/{auth,pki}

root@master01:~# ssh-copy-id node01

root@master01:~# scp /etc/kubernetes/auth/bootstrap.conf node01.mykernel.cn:/etc/kubernetes/auth/
root@master01:~# scp -rp k8s-bin-inst/nodes/var/lib/* node01.mykernel.cn:/var/lib/
# unit file
root@master01:~# scp k8s-bin-inst/nodes/unit-files/* node01.mykernel.cn:/lib/systemd/system/
# 配置文件
root@master01:~# scp k8s-bin-inst/nodes/etc/kubernetes/{config,kubelet} node01.mykernel.cn:/etc/kubernetes/


# CA证书
root@master01:~# scp cert/k8s-ca.pem  node01.mykernel.cn:/etc/kubernetes/pki/ca.crt


tar xvf kubernetes-node-linux-amd64.tar.gz  -C /usr/local
```

如果cfssl生成证书跳过以下代码配置证书环节，直接查看集群入口

```bash


tar xvf kubernetes-node-linux-amd64.tar.gz  -C /usr/local
ssh-copy-id node01.magedu.com

root@master01:~# scp -rp k8s-bin-inst/nodes/etc/kubernetes/ node01.magedu.com:/etc/
root@node01:~# ls /etc/kubernetes/ # 全是配置文件
config  kubelet  proxy

root@master01:~# scp -rp k8s-bin-inst/nodes/var/lib/* node01.magedu.com:/var/lib/
# 验证
root@node01:~# ls /var/lib/kube* -d # 全是配置文件
/var/lib/kubelet  /var/lib/kube-proxy

#证书
root@master01:~# scp -rp k8s-certs-generator/kubernetes/kubelet/* node01.magedu.com:/etc/kubernetes/
root@master01:~# tree  k8s-certs-generator/kubernetes/kubelet/
k8s-certs-generator/kubernetes/kubelet/
├── auth
│   ├── bootstrap.conf  # kubelet加入集群
│   └── kube-proxy.conf 
└── pki
    ├── ca.crt 
    ├── kube-proxy.crt
    └── kube-proxy.key

# 验证auth, pki目录
root@node01:~# ls /etc/kubernetes/auth/ # 拉集群的bootstrap, 专为kubelet自动生成证书。而kube-proxy是静态生成证书
bootstrap.conf  kube-proxy.conf
root@node01:~# ls /etc/kubernetes/pki/
ca.crt  kube-proxy.crt  kube-proxy.key

# unit file
root@master01:~# scp k8s-bin-inst/nodes/unit-files/* node01.magedu.com:/lib/systemd/system/

```

配置集群入口

```diff
root@node01:~# cat /etc/kubernetes/auth/bootstrap.conf 
#- name: kubernetes
  cluster:
+    server: https://kubernetes-api.magedu.com:6443 
```

> 此地址的修改会在生成kubelet的配置也会是对应的入口的域名
>
> 这个域名也是生成证书时，加入的域名 集群名-api.域名

```diff
root@master01:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters


+172.16.0.101 master01.mykernel.cn etcd01.mykernel.cn etcd01 kubernetes-api.mykernel.cn

```

```bash
root@master01:~# ping kubernetes-api.mykernel.cn
PING kubernetes-api.mykernel.cn (172.16.0.101) 56(84) bytes of data.
64 bytes from kubernetes-api.mykernel.cn (172.16.0.101): icmp_seq=1 ttl=64 time=0.034 ms

```



提供cni插件

https://github.com/containernetworking/plugins/releases

![image-20210206152305669](http://myapp.img.mykernel.cn/image-20210206152305669.png)

找到符合平台的插件

```bash
root@node01:~# export https_proxy=http://192.168.0.33:808
root@node01:~# wget https://github.com/containernetworking/plugins/releases/download/v0.9.0/cni-plugins-linux-amd64-v0.9.0.tgz

```

要求插件位置必须在/opt/cni/bin 目录, 以下两个选项传递给kubelet可以指定cni插件目录，和对接网络的配置

  --cni-bin-dir=/usr/bin  
  --cni-conf-dir=/etc/cni/net.d 

指定Pod的基础容器

  --pod-infra-container-image= 

```bash
root@node01:~# install -dv /opt/cni/bin
install: creating directory '/opt/cni'
install: creating directory '/opt/cni/bin'
root@node01:~# tar xvf cni-plugins-linux-amd64-v0.9.0.tgz -C /opt/cni/bin
./
./macvlan
./flannel # flannel插件
./static
./vlan
./portmap
./host-local
./vrf
./bridge
./tuning
./firewall
./host-device
./sbr
./loopback
./dhcp
./ptp
./ipvlan
./bandwidth

```



去掉KUBE_ALLOW_PRIV, 此flag已经 废弃

https://github.com/cloudnativelabs/kube-router/issues/761

```diff
root@node01:~# cat /lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/local/kubernetes/node/bin/kubelet \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBELET_API_SERVER \
	    $KUBELET_ADDRESS \
	    $KUBELET_PORT \
	    $KUBELET_HOSTNAME \
-	    $KUBE_ALLOW_PRIV \
	    $KUBELET_ARGS
Restart=on-failure
KillMode=process
RestartSec=10

[Install]
WantedBy=multi-user.target

```

> KUBE_ALLOW_PRIV 就是config通用配置的allow-privileged=true



```diff
root@node01:~# grep -n server /etc/kubernetes/auth/*.conf 
/etc/kubernetes/auth/bootstrap.conf:6:    server: https://kubernetes-api.magedu.com:6443
/etc/kubernetes/auth/kube-proxy.conf:6:    server: https://kubernetes-api.magedu.com:6443
```

> api server高可用主要给kubelet使用，所以这个域名应该是api server的入口

kubelet启动, 会根据/etc/kubernetes/auth/bootstrap.conf文件自动生成kubeconfig文件，所以api server是一致

kubeconfig文件中包含kubelet的证书

```diff
+root@node01:~# ls /etc/kubernetes/auth/kubelet.conf
ls: cannot access '/etc/kubernetes/auth/kubelet.conf': No such file or directory

+root@node01:~# systemctl start kubelet
+root@node01:~# systemctl enable kubelet
+root@node01:~# grep -n server /etc/kubernetes/auth/*.conf 
/etc/kubernetes/auth/bootstrap.conf:6:    server: https://kubernetes-api.magedu.com:6443
+/etc/kubernetes/auth/kubelet.conf:5:    server: https://kubernetes-api.magedu.com:6443
/etc/kubernetes/auth/kube-proxy.conf:6:    server: https://kubernetes-api.magedu.com:6443

root@node01:~# cat /etc/kubernetes/auth/kubelet.conf
users:
+- name: default-auth
  user:
+    client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
+    client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
```

直接签发的证书的是没有任何权限的，需要授权。controller 完成签发证书和分配 pod的子网 

查看kube-apiserver和kube-controller的状态

```bash
root@master01:~# systemctl status kube-controller-manager
● kube-controller-manager.service - Kubernetes Controller Manager
   Loaded: loaded (/lib/systemd/system/kube-controller-manager.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: https://github.com/GoogleCloudPlatform/kubernetes

# 启动kube-controller
root@master01:~# systemctl start kube-controller-manager
root@master01:~# systemctl enable kube-controller-manager


# 查看api server状态
 systemctl status kube-apiserver

```

验证kube-scheduler状态

```bash
systemctl status kube-scheduler
```



解析api 域名至其中一个api server, api作了高可用后，再dns解析指向 高可用的入口：keepalived vip或 nginx+keepalived的vip或dns3个A记录至3个Api Server.

> 当前使用hosts文件解析

```diff
root@node01:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters


+172.16.0.101 master01.magedu.com etcd01.magedu.com etcd01 kubernetes-api.magedu.com
172.16.0.102 master02.magedu.com etcd02.magedu.com etcd02
172.16.0.103 master03.magedu.com etcd03.magedu.com etcd03
172.16.0.104 node01.magedu.com
172.16.0.105 node02.magedu.com
172.16.0.106 node03.magedu.com

```

测试Ping

```bash
root@node02:~# ping kubernetes-api.magedu.com
PING master01.magedu.com (172.16.0.101) 56(84) bytes of data.
64 bytes from master01.magedu.com (172.16.0.101): icmp_seq=1 ttl=64 time=0.732 ms

```

配置dns文件位置

```diff
root@ubuntu-template:~# cat /var/lib/kubelet/config.yaml 
address: 0.0.0.0
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: cgroupfs
cgroupsPerQOS: true
+clusterDNS:
+- 10.64.0.10
clusterDomain: cluster.local
configMapAndSecretChangeDetectionStrategy: Watch
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuCFSQuotaPeriod: 100ms
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: false
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kind: KubeletConfiguration
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeLeaseDurationSeconds: 40
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
port: 10250
registryBurst: 10
registryPullQPS: 5
+resolvConf: /run/systemd/resolve/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s

```

> ubuntu需要
>
> 并且dns是配置容器中的dns

```bash
systemctl restart kubelet
```



#### master接入node *

```bash
root@master01:~# kubectl get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR             CONDITION
csr-4wmvb   2m25s   kubernetes.io/kube-apiserver-client-kubelet   system:bootstrapper   Pending

# 签署证书
root@master01:~# kubectl certificate approve csr-4wmvb
certificatesigningrequest.certificates.k8s.io/csr-4wmvb approved
root@master01:~# kubectl get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR             CONDITION
csr-4wmvb   2m40s   kubernetes.io/kube-apiserver-client-kubelet   system:bootstrapper   Approved,Issued


root@master01:~# kubectl describe csr
Name:               csr-4wmvb
Labels:             <none>
Annotations:        <none>
CreationTimestamp:  Sun, 07 Feb 2021 09:39:57 +0800
Requesting User:    system:bootstrapper
Signer:             kubernetes.io/kube-apiserver-client-kubelet
Status:             Approved,Issued
Subject:
         Common Name:    system:node:node01.magedu.com
         Serial Number:  
         Organization:   system:nodes
Events:  <none>

```

```bash
root@master01:~# kubectl get node
NAME                STATUS     ROLES    AGE   VERSION
node01.magedu.com   NotReady   <none>   14s   v1.20.2

```

是NotReady，还需要启动kube-proxy

```diff
root@master01:~# systemctl status kube-controller-manager

+Feb 07 09:42:43 master01.magedu.com kube-controller-manager[3279]: I0207 09:42:43.836658    3279 range_allocator.go:373] Set node node01.magedu.com PodCIDR to [10.244.0.0/24]
Feb 07 09:42:44 master01.magedu.com kube-controller-manager[3279]: I0207 09:42:44.898306    3279 node_lifecycle_controller.go:1429] Initializing eviction metric for zone:
Feb 07 09:42:44 master01.magedu.com kube-controller-manager[3279]: W0207 09:42:44.898481    3279 node_lifecycle_controller.go:1044] Missing timestamp for Node node01.magedu.com. Assuming now as a timestamp.
Feb 07 09:42:44 master01.magedu.com kube-controller-manager[3279]: I0207 09:42:44.898598    3279 node_lifecycle_controller.go:1195] Controller detected that all Nodes are not-Ready. Entering master disruption mode.
+279 event.go:291] "Event occurred" object="node01.magedu.com" kind="Node" apiVersion="v1" type="Normal" reason="RegisteredNode" message="Node node01.magedu.com event: Registered Node node01.magedu.com in Controller"
```

### kube-proxy配置文件

也支持--config加载配置文件, 默认在/var/lib/kube-proxy目录下

```diff
root@node01:~# cat /var/lib/kube-proxy/config.yaml 
apiVersion: kubeproxy.config.k8s.io/v1alpha1
+bindAddress: 0.0.0.0 # 自已监听地址
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/auth/kube-proxy.conf
  qps: 5
+clusterCIDR: 10.244.0.0/16 # pod网络
configSyncPeriod: 15m0s
conntrack:
  max: null
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
+healthzBindAddress: 0.0.0.0:10256 # 健康状态检测的地址和端口
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
+ipvs: # lvs
  excludeCIDRs: null
  minSyncPeriod: 0s
+  scheduler: ""      # 调度算法
  syncPeriod: 30s
kind: KubeProxyConfiguration
+metricsBindAddress: 127.0.0.1:10249 # 指标采集的端口
+mode: ipvs            # 定义ipvs规则的service. 需要系统加载ipvs模块
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
```

配置node节点的ipvs规则



配置文件

```bash
root@master01:~# tail -n 100000 k8s-bin-inst/nodes/etc/kubernetes/{config,proxy}
==> k8s-bin-inst/nodes/etc/kubernetes/config <== # 公共配置
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

==> k8s-bin-inst/nodes/etc/kubernetes/proxy <==  # 配置文件，直接加载
###
# kubernetes proxy config

# Add your own!
KUBE_PROXY_ARGS="--config=/var/lib/kube-proxy/config.yaml"

```



### 配置kube-proxy组件 *

请发kube-proxy证书，这个需要手动签发

```diff
root@master01:~/cert# cp kube-scheduler-csr.json kube-proxy-csr.json

root@master01:~/cert# cat kube-proxy-csr.json
{
+  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
    "O": "system:HuaYang",
    "OU": "system"
  }],
+    "hosts": []
}

```

> 连接api 即可 hosts": []

```bash
root@master01:~/cert# ../cfssl gencert -ca=k8s-ca.pem -ca-key=k8s-ca-key.pem      --config=common-config.json -profile=kubernetes-ca     kube-proxy-csr.json | ../cfssljson -bare kube-proxy
2021/02/08 17:47:09 [INFO] generate received request
2021/02/08 17:47:09 [INFO] received CSR
2021/02/08 17:47:09 [INFO] generating key: rsa-2048
2021/02/08 17:47:09 [INFO] encoded CSR
2021/02/08 17:47:09 [INFO] signed certificate with serial number 107316077825516598461351800247438979126718842335
2021/02/08 17:47:09 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

```



```bash
root@master01:~# scp k8s-bin-inst/nodes/etc/kubernetes/{config,proxy} node01.mykernel.cn:/etc/kubernetes/

root@master01:~# scp -rp k8s-bin-inst/nodes/var/lib/kube-proxy/ node01.mykernel.cn:/var/lib/


# 生成证书
#/etc/kubernetes/auth/kube-proxy.conf 
export KUBECONFIG=/etc/kubernetes/auth/kube-proxy.conf  # 配置文件中指定这个位置
export ADDR=https://kubernetes-api.mykernel.cn:6443 # 运行kube-scheduler节点的api server, 如果都指向 一个位置，api server压力大
export USERNAME=default-scheduler
export CERT=kube-proxy.pem
export KEY=kube-proxy-key.pem

kubectl config set-cluster default-cluster --server=$ADDR --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs
kubectl config set-credentials $USERNAME --client-certificate=$CERT --client-key=$KEY --username=$USERNAME --embed-certs=true 
kubectl config set-context default-system --cluster default-cluster --user $USERNAME
kubectl config use-context default-system

chown kube.kube $KUBECONFIG
 kubectl config view

export KUBECONFIG=


root@master01:~/cert# scp /etc/kubernetes/auth/kube-proxy.conf node01.mykernel.cn:/etc/kubernetes/auth/kube-proxy.conf
```



如果使用以上cfssl生成证书，跳过这一步

```bash
root@master01:~# scp k8s-bin-inst/nodes/etc/kubernetes/{config,proxy} node01.magedu.com:/etc/kubernetes/
#验证
root@node01:~# ls /etc/kubernetes/
auth  config  kubelet  pki  proxy # config, proxy

root@master01:~# scp -rp k8s-bin-inst/nodes/var/lib/kube-proxy/ node01.magedu.com:/var/lib/
# 验证
root@node01:~# ls /var/lib/kube-proxy/
config.yaml


root@master01:~# scp k8s-certs-generator/kubernetes/kubelet/auth/kube-proxy.conf  node01.magedu.com:/etc/kubernetes/auth/
# 验证
root@node01:~# ls /etc/kubernetes/auth/
bootstrap.conf  kubelet.conf  kube-proxy.conf # kube-proxy 证书认证连接api server kubeconfig 



```

```diff
root@node01:~# cat /etc/kubernetes/auth/kube-proxy.conf 
apiVersion: v1
kind: Config
clusters:
 - name: kubernetes
  cluster:
    server: https://kubernetes-api.magedu.com:6443
    certificate-authority-data: 
users:
 - name: system:kube-proxy
  user:
    client-certificate-data: 
    client-key-data: 
contexts:
 - context:
    cluster: kubernetes
+    user: system:kube-proxy # 证书中必须是这个用户名，kubeconfig文件中的user这个无所谓
  name: system:kube-proxy@kubernetes
current-context: system:kube-proxy@kubernetes
```

这个system:kube-proxy用户默认绑定在集群内建的角色上

```diff

+root@master01:~# kubectl describe clusterrolebinding system:node-proxier
Name:         system:node-proxier
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  system:node-proxier
Subjects:
  Kind  Name               Namespace
  ----  ----               ---------
+  User  system:kube-proxy  

```

配置kube-proxy的pod网络

```diff
root@node01:~# cat /var/lib/kube-proxy/config.yaml 
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/auth/kube-proxy.conf
  qps: 5
clusterCIDR: 10.244.0.0/16
configSyncPeriod: 15m0s
conntrack:
  max: null
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
+  scheduler: "rr"
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
+mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""


```

> 由于我们使用flannel就直接使用10.244.0.0/16

配置ipvs模块

```bash
root@node02:~# cat > /etc/modules-load.d/10-k8s-modules.conf <<EOF
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
EOF
for i in br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack_ipv4; do modprobe $i; done

# 验证
lsmod | grep ip_vs
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 147456  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          131072  8 xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_ipv4,nf_nat,ipt_MASQUERADE,nf_nat_ipv4,nf_conntrack_netlink,ip_vs
libcrc32c              16384  4 nf_conntrack,nf_nat,raid456,ip_vs
```

安装ipset命令

```bash
apt install -y ipset ipvsadm

# 后期ipvsadm -Ln 查看规则
```



```bash
root@node01:~# systemctl enable kube-proxy
root@node01:~# systemctl start kube-proxy

root@node01:~# systemctl status kube-proxy

```





### 配置网络插件 *

controller, kube-proxy使用

- [x] 二进制部署flannel/calico
- [x] pod运行flannel

https://github.com/coreos/flannel



```bash
For Kubernetes v1.17+ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

```bash
root@master01:~# export https_proxy=http://192.168.0.33:808
root@master01:~# export no_proxy=127.0.0.0/8,172.16.0.0/16,10.244.0.0/16,10.76.0.0/12,*.mykernel.cn
root@master01:~# wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

编辑 kube-flannel.yml

```diff
126   net-conf.json: |
127     {
128       "Network": "10.244.0.0/16",
129       "Backend": {
+130         "Type": "vxlan",
+131         "DirectRouting": true
132       }
133     }
```

> 注意"DirectRouting": true 前面是空格，用space键补上，不要使用TAB

```bash
root@master01:~# kubectl apply -f kube-flannel.yml
podsecuritypolicy.policy/psp.flannel.unprivileged created # pod的psp策略
clusterrole.rbac.authorization.k8s.io/flannel created    # clusterrole
clusterrolebinding.rbac.authorization.k8s.io/flannel created # 绑定
serviceaccount/flannel created # sa, pod中进程的权限
configmap/kube-flannel-cfg created # cm就是上面编辑的部署
daemonset.apps/kube-flannel-ds created # daemonset, 每个节点一个Pod
```

在过程中查看node节点

```bash
root@node01:~# docker ps
CONTAINER ID   IMAGE                  COMMAND    CREATED              STATUS          PORTS     NAMES
93203a5e3c79   k8s.gcr.io/pause:3.2   "/pause"   About a minute ago   Up 58 seconds             k8s_POD_kube-flannel-ds-fm9kn_kube-system_514f095d-cfc9-42be-8e40-200c80aa667d_0

# Pod基础架构容器
```

```bash
root@master01:~# kubectl get pods -n kube-system   kube-flannel-ds-fm9kn
NAME                    READY   STATUS    RESTARTS   AGE
kube-flannel-ds-fm9kn   1/1     Running   1          2m25s
```



查看集群状态

```bash
root@master01:~# kubectl get node
NAME                STATUS   ROLES    AGE     VERSION
# 节点自动签发证书，所以后面的node节点直接以3.10步骤添加即可
node01.magedu.com   Ready    <none>   5m17s   v1.20.2
```



现在集群正常了，将master组件的日志级别调低

```bash
/etc/kubernetes/config
16 KUBE_LOG_LEVEL="--v=0"
```

```bash
root@master01:~# systemctl restart kube-apiserver.service kube-controller-manager.service kube-scheduler.service
```



### 配置coredns *

https://github.com/coredns/deployment/tree/master/kubernetes

```bash
git clone https://github.com/coredns/deployment.git

root@master01:~/deployment/kubernetes# ./deploy.sh -i 10.64.0.10 -d cluster.local | kubectl apply -f -

```

```bash
root@master01:~# kubectl get pods -n kube-system    -o wide
NAME                       READY   STATUS    RESTARTS   AGE     IP             NODE                 NOMINATED NODE   READINESS GATES
coredns-6ccb5d565f-tk4rk   1/1     Running   0          2m10s   10.244.0.3     node01.mykernel.cn   <none>           <none>
kube-flannel-ds-ggxg8      1/1     Running   0          14m     172.16.0.104   node01.mykernel.cn   <none>           <none>

```

```bash
root@ubuntu-template:~#  ping 10.244.0.3
PING 10.244.0.3 (10.244.0.3) 56(84) bytes of data.
64 bytes from 10.244.0.3: icmp_seq=1 ttl=64 time=0.090 ms

64 bytes from 10.244.0.3: icmp_seq=2 ttl=64 time=0.077 ms
64 bytes from 10.244.0.3: icmp_seq=3 ttl=64 time=0.091 ms

```

> node节点可达
>
> master没安装kube-proxy

测试dns

```bash
root@master01:~# kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.64.0.10   <none>        53/UDP,53/TCP,9153/TCP   8m32s

```

```diff
root@master01:~# kubectl describe svc -n kube-system
Name:              kube-dns
Namespace:         kube-system
Labels:            k8s-app=kube-dns
                   kubernetes.io/cluster-service=true
                   kubernetes.io/name=CoreDNS
Annotations:       prometheus.io/port: 9153
                   prometheus.io/scrape: true
Selector:          k8s-app=kube-dns
Type:              ClusterIP
IP Families:       <none>
IP:                10.64.0.10
IPs:               10.64.0.10
Port:              dns  53/UDP
TargetPort:        53/UDP
Endpoints:         10.244.0.3:53
Port:              dns-tcp  53/TCP
TargetPort:        53/TCP
+Endpoints:         10.244.0.3:53

```

```bash
root@ubuntu-template:~# host -t A kube-dns.kube-system.svc.cluster.local  10.244.0.3
Using domain server:
Name: 10.244.0.3
Address: 10.244.0.3#53
Aliases: 

kube-dns.kube-system.svc.cluster.local has address 10.64.0.10
```



### 测试pod *

```bash
root@master01:~# kubectl create deployment myapp --replicas=2 --image=ikubernetes/myapp:v1 
root@master01:~# kubectl create service clusterip myapp --tcp=80:80


root@master01:~# kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
myapp-7d4b7b84b-8rslx   1/1     Running   0          55s   10.244.0.5   node01.mykernel.cn   <none>           <none>
myapp-7d4b7b84b-ssnwl   1/1     Running   0          55s   10.244.0.4   node01.mykernel.cn   <none>           <none>


root@master01:~# kubectl describe service myapp
Endpoints:         10.244.0.4:80,10.244.0.5:80


root@master01:~/deployment/kubernetes# kubectl edit svc myapp
 27   type: NodePort     
 root@master01:~/deployment/kubernetes# kubectl get svc -A
default       myapp        NodePort    10.69.202.62   <none>        80:32953/TCP             114s

```

> 外部访问 node节点的 172.16.0.104:32953

![image-20210208182042057](http://myapp.img.mykernel.cn/image-20210208182042057.png)



node节点查看ipvs规则

```bash
root@node01:~# apt install ipvsadm
root@node01:~# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn

```



```bash
root@master02:~# kubectl exec -it  -n default       myapp-7d4b7b84b-jx5gq -- sh
/ # cat /etc/resolv.conf 
nameserver 10.64.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
/ # ping www.baidu.com
PING www.baidu.com (14.215.177.39): 56 data bytes
64 bytes from 14.215.177.39: seq=0 ttl=52 time=30.306 ms
64 bytes from 14.215.177.39: seq=1 ttl=52 time=30.510 ms
^C
--- www.baidu.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 30.306/30.408/30.510 ms

```



## 高可用master *

- [x] 复制配置

- [x] 启动



### 复制配置和证书

```bash
root@master03:~# tar xvf kubernetes-server-linux-amd64.tar.gz -C /usr/local
# 配置 和 master一致
root@master01:~# scp -rp /etc/kubernetes/  master03:/etc/

# 证书
# 先清理从master01复制过来的证书信息
root@master03:~# rm -fr /etc/kubernetes/{auth,pki,token.csv}
root@master01:~# scp -rp k8s-certs-generator/kubernetes/master03/* master03:/etc/kubernetes

# 验证
root@master03:~# ls /etc/kubernetes/
apiserver  auth  config  controller-manager  pki  scheduler  token.csv

root@master01:~# scp -rp k8s-bin-inst/master/unit-files/* master03:/lib/systemd/system/
```

### 启动

```bash
useradd -r kube
root@master03:~# chown kube.kube /etc/kubernetes/pki/*.key

root@master03:~# systemctl enable kube-apiserver kube-controller-manager kube-scheduler
root@master03:~# systemctl restart kube-apiserver kube-controller-manager kube-scheduler

```

### 验证kubelet命令

```bash
mkdir ~/.kube
cp /etc/kubernetes/auth/admin.conf .kube/config
ln -sv /usr/local/kubernetes/server/bin/kubectl /usr/bin/
```

```diff
root@master03:~# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
+    server: https://kubernetes-api.mykernel.cn:6443
```

配置域名解析

```diff
root@master02:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters


172.16.0.101 master01.mykernel.cn etcd01.mykernel.cn etcd01
+172.16.0.102 master02.mykernel.cn etcd02.mykernel.cn etcd02 kubernetes-api.mykernel.cn

```



并将此节点的controller, kube-scheduler指向高可用的api server

```diff
vim /etc/kubernetes/auth/controller-manager.conf
+    server: https://kubernetes-api.mykernel.cn:6443

vim /etc/kubernetes/auth/scheduler.conf 
+    server: https://kubernetes-api.mykernel.cn:6443
```

重启

```bash
root@master02:~# chown kube.kube kube-controller-manager

systemctl restart kube-apiserver kube-controller-manager kube-scheduler
```

获取当前锁位置

```bash
root@master02:~# kubectl get leases -n kube-system
NAME                      HOLDER                                                      AGE
kube-controller-manager   master01.mykernel.cn_0ba4eb7f-40af-4f88-b2d5-a41792c85a86   131m
kube-scheduler            master01.mykernel.cn_72fcfb59-5365-4ec1-a741-36c9820d7cce   110m
```

停止master01

```bash
root@master01:~# poweroff

```

配置node01指向master02

```diff
root@node01:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters


172.16.0.101 master01.magedu.com etcd01.magedu.com etcd01 

+172.16.0.102 master02.magedu.com etcd02.magedu.com etcd02 kubernetes-api.mykernel.cn
```



查看锁资源

```bash
root@master02:~# kubectl get leases -n kube-system
NAME                      HOLDER                                                      AGE
kube-controller-manager   master01.mykernel.cn_0ba4eb7f-40af-4f88-b2d5-a41792c85a86   3h19m
kube-scheduler            master02.mykernel.cn_ee3e140e-2316-4700-8d09-b2bfd9634dd8   177m

# controller-manager因为我调整了选项，所以在endpoints上
root@master02:~# kubectl describe endpoints -n kube-system kube-controller-manager
  Normal  LeaderElection  58m   kube-controller-manager  master02.mykernel.cn_e7a78440-3001-4871-866c-99662b667e54 became leader
```



现在已经完成api/controller/scheduler高可用，高可用方式：

- node节点添加nginx -> 3个api server.
- 3 api 前端添加 4层负载。所有节点使用/etc/hosts文件指向这个外部的vip
- 3 api + keepalived VIP漂移
- 3 api地址直接在DNS上添加3个A记录 

阿里云：

- keepalived的VIP：haip
- 内网SLB

## node节点快速添加 *

```bash
root@node01:~# docker save  k8s.gcr.io/pause:3.2 coredns/coredns:1.8.0 quay.io/coreos/flannel:v0.13.1-rc2 -o node.tar
root@node01:~# scp node.tar node02.mykernel.cn:
root@ubuntu-template:~# docker load -i node.tar 
ba0dae6243cc: Loading layer [==================================================>]  684.5kB/684.5kB
Loaded image: k8s.gcr.io/pause:3.2
225df95e717c: Loading layer [==================================================>]  336.4kB/336.4kB
69ae2fbf419f: Loading layer [==================================================>]  42.24MB/42.24MB
Loaded image: coredns/coredns:1.8.0
50644c29ef5a: Loading layer [==================================================>]  5.845MB/5.845MB
0be670d27a91: Loading layer [==================================================>]  11.42MB/11.42MB
90679e912622: Loading layer [==================================================>]  2.267MB/2.267MB
6db5e246b16d: Loading layer [==================================================>]  45.69MB/45.69MB
97320fed8db7: Loading layer [==================================================>]   5.12kB/5.12kB
8a984b390686: Loading layer [==================================================>]  9.216kB/9.216kB
3b729894a01f: Loading layer [==================================================>]   7.68kB/7.68kB
Loaded image: quay.io/coreos/flannel:v0.13.1-rc2
```

在node01上完成以下脚本制作脚本

```bash
mkdir node
mv kubernetes-node-linux-amd64.tar.gz cni-plugins-linux-amd64-v0.9.0.tgz node.tar node
cd node
# 配置和证书
cp -a /etc/kubernetes/  .
# 清理kubelet
rm -f kubernetes/auth/kubelet.conf 
# 配置
cp -a /var/lib/kube* .
# unit
cp /lib/systemd/system/kubelet.service /lib/systemd/system/kube-proxy.service .
# hosts
cp /etc/hosts .

```

编辑脚本

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-08
#FileName：             install.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************
pkill apt-get
pkill apt-get

hostnamectl set-hostname $1
sed -i -e 's@us.archive.ubuntu.com@mirrors.aliyun.com@g' -e 's@security.ubuntu.com@mirrors.aliyun.com@g' /etc/apt/sources.list

apt update

apt -y install chrony && systemctl enable chronyd && systemctl restart chronyd
cp hosts /etc/hosts

# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

apt install docker-ce=5:20.10.2~3-0~ubuntu-bionic -y
systemctl start docker
systemctl enable docker


# pod infra, flannel, dns
docker load -i node.tar 

# cni plugins
install -dv /opt/cni/bin
tar xvf cni-plugins-linux-amd64-v0.9.0.tgz -C /opt/cni/bin


# kubelet and kube-proxy
install -dv /etc/kubernetes/{auth,pki}
# config and certificate
cp -a kubernetes /etc/
rm -f kubernetes/auth/kubelet.conf 
# config
cp -a  kubelet kube-proxy /var/lib/

# unit
cp kubelet.service kube-proxy.service /lib/systemd/system/






# ipvs
cat > /etc/modules-load.d/10-k8s-modules.conf <<EOF
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
EOF
for i in br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack_ipv4; do modprobe $i; done


# start
systemctl enable kube-proxy kubelet
systemctl restart kube-proxy kubelet


root@node01:~/node# ls
cni-plugins-linux-amd64-v0.9.0.tgz  hosts  install.sh  kubelet  kubelet.service  kube-proxy  kube-proxy.service  kubernetes  kubernetes-node-linux-amd64.tar.gz  kubernetes-v1.20.2.tar  node.tar
```

node02试试

```bash
install -dv node
root@ubuntu-template:~# tar xvf kubernetes-v1.20.2.tar -C node
root@ubuntu-template:~/node# bash install.sh node02.mykernel.cn

```

master节点查看

```bash
root@master02:~# kubectl get csr
NAME        AGE    SIGNERNAME                                    REQUESTOR                        CONDITION
csr-74vwv   64s    kubernetes.io/kube-apiserver-client-kubelet   system:node:node01.mykernel.cn   Pending


root@master02:~# kubectl certificate approve csr-74vwv 
certificatesigningrequest.certificates.k8s.io/csr-74vwv approved


root@master02:~# kubectl get node
NAME                 STATUS     ROLES    AGE    VERSION
node01.mykernel.cn   Ready      <none>   137m   v1.20.2
node02.mykernel.cn   NotReady   <none>   5s     v1.20.2


#等待flannel启动
root@master02:~# kubectl get node
NAME                 STATUS   ROLES    AGE    VERSION
node01.mykernel.cn   Ready    <none>   138m   v1.20.2
node02.mykernel.cn   Ready    <none>   35s    v1.20.2 # Ready

```

测试将pod规模扩容，看是否到node02

```bash
root@master02:~# kubectl scale deploy/myapp --replicas=5
root@master02:~# kubectl get pods -o wide -w
NAME                    READY   STATUS              RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
myapp-7d4b7b84b-fnm9d   0/1     ContainerCreating   0          6s    <none>       node02.mykernel.cn   <none>           <none>
myapp-7d4b7b84b-hphjq   0/1     Pending             0          6s    <none>       node02.mykernel.cn   <none>           <none>
myapp-7d4b7b84b-jx5gq   1/1     Running             0          43m   10.244.0.8   node01.mykernel.cn   <none>           <none>
myapp-7d4b7b84b-mbfxp   0/1     ContainerCreating   0          6s    <none>       node02.mykernel.cn   <none>           <none>
myapp-7d4b7b84b-sh646   1/1     Running             0          43m   10.244.0.9   node01.mykernel.cn   <none>           <none>
myapp-7d4b7b84b-hphjq   0/1     ContainerCreating   0          6s    <none>       node02.mykernel.cn   <none>           <none>

# 已经自动调度到node02
```



测试跨主机的两个pod通信，依赖node节点有ipv4.ip_forward规则

![image-20210209092351687](http://myapp.img.mykernel.cn/image-20210209092351687.png)

## 部署容器监控及告警

Prometheus + 钉钉 + grafana 

告警：PromQL 

展示：PromQL

可以参考 http://blog.mykernel.cn/2021/02/03/%E8%B5%84%E6%BA%90%E6%8C%87%E6%A0%87%E4%B8%8EHPA%E6%8E%A7%E5%88%B6%E5%99%A8/#hpa

- adaptor内置核心指标
- cumstom metrics完成基于pod指标自动伸缩



## 部署ELK

直接裸机部署

参考：http://blog.mykernel.cn/2020/12/23/%E9%83%A8%E7%BD%B2elasticsearch/

参考：http://blog.mykernel.cn/2020/11/19/Kibana%E6%9F%A5%E8%AF%A2%E6%85%A2/

es和kibana和logstash和fluentd或filebeat 版本尽可能一样

redis

访问链路：Pod（fluentd + jar/nginx) -> redis svc -> redis pod -> logstash .... ->  elasticsearch



