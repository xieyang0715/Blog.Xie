---
title: 'Ubuntu双网卡绑定bond0, 双网卡桥接'
date: 2020-09-29 16:06:13
tags: plan
toc: true
---

基于上篇的Ubuntu克隆主机完成双网卡绑定bond0, 双网卡配置桥接



# 1. 绑定和桥接使用场景 

虚拟化、openstack：桥接

k8s、openstack多服务场景, 带宽太低: 采用网卡绑定

## 1.1 七种bond模式说明：

>  注意：需要两个网卡在同一个vlan, 且是相同的工作模式, vmware: 桥接: vmnet0.      

以下通过centos 7和ubuntu示例

### 1.1.1 mode=0, balance-rr

第⼀种模式：mod=0，即：(balance-rr) Round-robin policy（平衡轮循环策略）
特点：传输数据包顺序是依次传输（即：第1个包⾛eth0，下⼀个包就⾛eth1….⼀直循环下去，直到最后⼀个传输完
毕），此模式提供负载平衡和容错能⼒。

![image-20201118102747447](http://myapp.img.mykernel.cn/image-20201118102747447.png)

#### 1.1.1.1 vmware的网卡配置 需要两个网卡在同一个vlan, 且是相同的工作模式

![image-20201118102513825](http://myapp.img.mykernel.cn/image-20201118102513825.png)

#### 1.1.1.2 centos网卡配置bond(mode=0)

```bash
#pwd
/etc/sysconfig/network-scripts/
#ifcfg-eth0
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes
MASTER=bond0
SLAVE=yes

#ifcfg-eth1
BOOTPROTO=static
DEVICE=eth1
ONBOOT=yes
MASTER=bond0
SLAVE=yes

#ifcfg-bond0
TYPE=Ethernet
BOOTPROTO=static
DEVICE=bond0
ONBOOT=yes

IPADDR=192.168.1.7
NETMASK=255.255.255.0
DNS1=223.6.6.6
DNS2=114.114.114.114
GATEWAY=192.168.1.1

#/etc/modprobe.d/bond.conf
alias bond0 bonding
options bond0  miimon=100 mode=0
#options bond0 miimon=100 mode=1 use_carrier=0 fail_over_mac=1 primary=eth0 #vmware
#options bond0 miimon=100 mode=1 use_carrier=0 #physical machine
#options bond0  miimon=100 mode=6


#重启
#reboot
```



#### 1.1.1.3 ubuntu网卡配置bond(mode=0)

```bash
#/etc/netplan/01-netcfg.yaml 
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
    eth1:
      dhcp4: no
  bonds:
    bond0:
      interfaces:
      - eth0
      - eth1
      addresses: [192.168.0.196/24]
      gateway4: 192.168.0.1
      nameservers:
        addresses: [223.6.6.6]
      parameters:
        mode: 0
        mii-monitor-interval: 100

#重启
#reboot
```

### 1.1.2 mode=1, active-backup

第⼆种模式：mod=1，即： (active-backup) Active-backup policy（主-备份策略）
特点：只有⼀个设备处于活动状态，当⼀个宕掉另⼀个⻢上由备份转换为主设备。mac地址是外部可⻅得，从外⾯看
来，bond的MAC地址是唯⼀的，以避免switch(交换机)发⽣混乱。此模式只提供了容错能⼒；由此可⻅此算法的优点
是可以提供⾼⽹络连接的可⽤性，但是它的资源利⽤率较低，只有⼀个接⼝处于⼯作状态，在有 N 个⽹络接⼝的情况
下，资源利⽤率为1/N。

![image-20201118104101521](http://myapp.img.mykernel.cn/image-20201118104101521.png)

在另一个主机ping这个主机的bond0网卡过程中，停止这个主机一个网卡，不会影响ping.



#### 1.1.2.1 vmware配置 需要两个网卡在同一个vlan, 且是相同的工作模式

![image-20201118102513825](http://myapp.img.mykernel.cn/image-20201118102513825.png)

#### 1.1.2.2 centos网卡配置 bond(mode=1)

```bash
#pwd
/etc/sysconfig/network-scripts/
#ifcfg-eth0
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes
MASTER=bond0
SLAVE=yes

#ifcfg-eth1
BOOTPROTO=static
DEVICE=eth1
ONBOOT=yes
MASTER=bond0
SLAVE=yes

#ifcfg-bond0
TYPE=Ethernet
BOOTPROTO=static
DEVICE=bond0
ONBOOT=yes

IPADDR=192.168.1.7
NETMASK=255.255.255.0
DNS1=223.6.6.6
DNS2=114.114.114.114
GATEWAY=192.168.1.1

#/etc/modprobe.d/bond.conf
alias bond0 bonding
options bond0 miimon=100 mode=1 use_carrier=0 fail_over_mac=1 primary=eth0
# options bond0 miimon=100 mode=1 use_carrier=0 #这个是公司实际物理器配置.



#重启
#reboot
```

> #在vmware中需要fail_over_mac=1, 在实际物理服务器上并不需要这个选项.

#### 1.1.2.3 ubuntu网卡配置bond(mode=1)

```bash
#/etc/netplan/01-netcfg.yaml 
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
    eth1:
      dhcp4: no
  bonds:
    bond0:
      interfaces:
      - eth0
      - eth1
      addresses: [192.168.0.196/24]
      gateway4: 192.168.0.1
      nameservers:
        addresses: [223.6.6.6]
      parameters:
        mode: 1
        mii-monitor-interval: 100

#重启
#reboot
```



### 1.1.3 其他模式

第三种模式：mod=2，即：(balance-xor) XOR policy（平衡策略）
特点：基于指定的传输HASH策略传输数据包。缺省的策略是：(源MAC地址 XOR ⽬标MAC地址) % slave数量。其他
的传输策略可以通过xmit_hash_policy选项指定，此模式提供负载平衡和容错能⼒。



第四种模式：mod=3，即：broadcast（⼴播策略）
特点：在每个slave接⼝上传输每个数据包，此模式提供了容错能⼒。



第五种模式：mod=4，即：(802.3ad) IEEE 802.3adDynamic link aggregation（IEEE 802.3ad 动态链接
聚合）
特点：创建⼀个聚合组，它们共享同样的速率和双⼯设定。根据802.3ad规范将多个slave⼯作在同⼀个激活的聚合体
下。
必要条件：
条件1：ethtool⽀持获取每个slave的速率和双⼯设定。
条件2：switch(交换机)⽀持IEEE 802.3ad Dynamic link aggregation。
条件3：⼤多数switch(交换机)需要经过特定配置才能⽀持802.3ad模式。
第六种模式：mod=5，即：(balance-tlb) Adaptive transmit load balancing（适配器传输负载均衡）
特点：不需要任何特别的switch(交换机)⽀持的通道bonding。在每个slave上根据当前的负载（根据速度计算）分
配外出流量。如果正在接受数据的slave出故障了，另⼀个slave接管失败的slave的MAC地址。
该模式的必要条件：
ethtool⽀持获取每个slave的速率

第七种模式：mod=6，即：(balance-alb) Adaptive load balancing（适配器适应性负载均衡）
特点：该模式包含了balance-tlb模式，同时加上针对IPV4流量的接收负载均衡(receive load balance,
rlb)，⽽且不需要任何switch(交换机)的⽀持。



## 1.2 桥接bridge配置

### 1.2.1 ubuntu单网卡桥接br0

```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
  bridges:
    br0:
      interfaces:
      - eth0
      addresses: [192.168.0.196/24]
      gateway4: 192.168.0.1
      nameservers:
        addresses: [223.6.6.6]
        
#重启
#reboot
```

### 1.2.2 centos 单网卡桥接br0

```bash
[root@centos-mq01 network-scripts]# cat ifcfg-eth0 
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
DEFROUTE=yes
#
DEVICE=eth0
BRIDGE=br0
[root@centos-mq01 network-scripts]# cat ifcfg-br0 
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.0.167
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
#
DEVICE=br0
TYPE=Bridge

[root@centos-mq01 network-scripts]# systemctl restart network
[root@centos-mq01 network-scripts]# systemctl enable network

```

### 



# 2. 高性能高可用配置

## 2.1 cenots/ubuntu双网卡桥接

| 网络   | 接口 | 绑定 | 桥接 |
| ------ | ---- | ---- | ---- |
| vmnet0 | eth0 |      | br0  |
| vmnet1 | eth1 |      | br1  |

![image-20201118152038821](http://myapp.img.mykernel.cn/image-20201118152038821.png)

### 2.1.1 centos双网卡桥接br0,br1

> #外网需要网关,DNS，内网不需要网关,DNS，并走静态路由。

```bash
[root@centos-mq01 network-scripts]# cat ifcfg-eth0 
BOOTPROTO=static
ONBOOT=yes
#
DEVICE=eth0
BRIDGE=br0
TYPE=Ethernet

[root@centos-mq01 network-scripts]# cat ifcfg-br0 
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.0.167
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
#
DEVICE=br0
TYPE=Bridge


[root@centos-mq01 network-scripts]# cat ifcfg-eth1 
BOOTPROTO=static
ONBOOT=yes
#
DEVICE=eth1
BRIDGE=br1
TYPE=Ethernet

[root@centos-mq01 network-scripts]# cat ifcfg-br1 
BOOTPROTO=static
ONBOOT=yes
IPADDR=10.20.10.2
NETMASK=255.255.0.0
#
DEVICE=br1
TYPE=Bridge


#配置内网的静态路由
#route-br1
10.20.0.0/255.255.0.0 via 10.20.0.1 dev br1
172.16.0.0/255.255.0.0 via 10.20.0.1 dev br1
192.168.100.0/255.255.255.0 via 10.20.0.1 dev br1


#重启
#reboot
```

### 2.1.2 ubuntu配置双网卡桥接br0,br1

eth0在桥接vmnet0, 表示直接连连通公网的路由。

eth1在vmnet1，是仅主机, 仅主机表示连接了没有接公网的路由器, 

> #外网需要网关,DNS，内网不需要网关,DNS，并走静态路由。

```bash
#/etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
    eth1:
      dhcp4: no
    eth2:
      dhcp4: no
    eth3:
      dhcp4: no
  bridges:
    br0:
      interfaces:
      - eth0
      addresses: [192.168.0.196/24]
      gateway4: 192.168.0.1
      nameservers:
        addresses: [223.6.6.6]
    br1:
      interfaces:
      - eth2
      addresses: [10.20.10.1/16]
	  routes:
	  - to: 10.20.0.0/16
	    via: 10.20.0.1
	  - to: 192.168.100.0/24
	    via: 10.20.0.1
	  - to: 172.16.0.0/16
	    via: 10.20.0.1
#重启
#reboot

#ping
#ping www.baidu.com -I br0 # 可通公网
#ping www.baidu.com -I br1 # 不通公网
```



## 2.2 centos/ubuntu 冗余bond+桥接

| 网络   | 接口 | 绑定  | 桥接 |
| ------ | ---- | ----- | ---- |
| vmnet0 | eth0 | bond0 | br0  |
|        | eth1 |       |      |
| vmnet1 | eth2 | bond1 | br1  |
|        | eth3 |       |      |



![Linux运维架构](http://myapp.img.mykernel.cn/Linux运维架构.png)





### 2.2.1 centos冗余bond+桥接

```bash
#pwd
/etc/sysconfig/network-scripts/
#ifcfg-eth0
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes
MASTER=bond0
SLAVE=yes

#ifcfg-eth1
BOOTPROTO=static
DEVICE=eth1
ONBOOT=yes
MASTER=bond0
SLAVE=yes

#ifcfg-bond0 
TYPE=Ethernet
BOOTPROTO=static
DEVICE=bond0
ONBOOT=yes
#
BRIDGE=br0

#ifcfg-br0
TYPE=Bridge
#
BOOTPROTO=static
DEVICE=br0
ONBOOT=yes

IPADDR=192.168.1.7
NETMASK=255.255.255.0
DNS1=223.6.6.6
DNS2=114.114.114.114
GATEWAY=192.168.1.1



#ifcfg-eth2
BOOTPROTO=static
DEVICE=eth2
ONBOOT=yes
MASTER=bond1
SLAVE=yes

#ifcfg-eth3
BOOTPROTO=static
DEVICE=eth3
ONBOOT=yes
MASTER=bond1
SLAVE=yes

#ifcfg-bond1 
TYPE=Ethernet
BOOTPROTO=static
DEVICE=bond1
ONBOOT=yes
#
BRIDGE=br1

#ifcfg-br1
TYPE=Bridge
#
BOOTPROTO=static
DEVICE=br1
ONBOOT=yes

IPADDR=10.20.10.2
NETMASK=255.255.0.0

#route-br1
10.20.0.0/255.255.0.0 via 10.20.0.1 dev br1
172.16.0.0/255.255.0.0 via 10.20.0.1 dev br1
192.168.100.0/255.255.255.0 via 10.20.0.1 dev br1

#/etc/modprobe.d/bond.conf
alias bond0 bonding
options bond0 miimon=100 mode=1 use_carrier=0 fail_over_mac=1 primary=eth0
# options bond0 miimon=100 mode=1 use_carrier=0 #这个是公司实际物理器配置.
alias bond1 bonding
options bond1 miimon=100 mode=1 use_carrier=0 fail_over_mac=1 primary=eth0
# options bond0 miimon=100 mode=1 use_carrier=0 #这个是公司实际物理器配置.

# systemctl restart network

# 另一台主机ping 192.168.1.7, ping不通就重启
#cat /proc/net/bonding/bond1
eth0

#ifconfig eth0 down
# 发现ping不会中断

#cat /proc/net/bonding/bond1
eth1



# 另一台主机ping 10.20.10.2 , ping不通就重启
#cat /proc/net/bonding/bond1
eth2

#ifconfig eth2 down
# 发现ping不会中断

#cat /proc/net/bonding/bond1
eth3

```



配置过程如下：

```bash
#pwd
/etc/sysconfig/network-scripts/
#ifcfg-eth0
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes
MASTER=bond0
SLAVE=yes

#ifcfg-eth1
BOOTPROTO=static
DEVICE=eth1
ONBOOT=yes
MASTER=bond0
SLAVE=yes

#ifcfg-bond0
TYPE=Ethernet
BOOTPROTO=static
DEVICE=bond0
ONBOOT=yes

IPADDR=192.168.1.7
NETMASK=255.255.255.0
DNS1=223.6.6.6
DNS2=114.114.114.114
GATEWAY=192.168.1.1

#/etc/modprobe.d/bond.conf
alias bond0 bonding
options bond0 miimon=100 mode=1 use_carrier=0 fail_over_mac=1 primary=eth0
# options bond0 miimon=100 mode=1 use_carrier=0 #这个是公司实际物理器配置.


# systemctl restart network


# 另一台主机ping 192.168.1.7, ping不通就重启
#cat /proc/net/bonding/bond1
eth0

#ifconfig eth0 down
# 发现ping不会中断

#cat /proc/net/bonding/bond1
eth1


#pwd
/etc/sysconfig/network-scripts/
#ifcfg-eth2
BOOTPROTO=static
DEVICE=eth2
ONBOOT=yes
MASTER=bond1
SLAVE=yes

#ifcfg-eth3
BOOTPROTO=static
DEVICE=eth3
ONBOOT=yes
MASTER=bond1
SLAVE=yes

#ifcfg-bond1
TYPE=Ethernet
BOOTPROTO=static
DEVICE=bond1
ONBOOT=yes

IPADDR=10.20.10.2
NETMASK=255.255.0.0

#route-bond1
10.20.0.0/255.255.0.0 via 10.20.0.1 dev bond1
172.16.0.0/255.255.0.0 via 10.20.0.1 dev bond1
192.168.100.0/255.255.255.0 via 10.20.0.1 dev bond1

#/etc/modprobe.d/bond.conf
alias bond0 bonding
options bond0 miimon=100 mode=1 use_carrier=0 fail_over_mac=1 primary=eth0
# options bond0 miimon=100 mode=1 use_carrier=0 #这个是公司实际物理器配置.
alias bond1 bonding
options bond1 miimon=100 mode=1 use_carrier=0 fail_over_mac=1 primary=eth0
# options bond0 miimon=100 mode=1 use_carrier=0 #这个是公司实际物理器配置.

# systemctl restart network

# 另一台主机ping 10.20.10.2 , ping不通就重启
#cat /proc/net/bonding/bond1
eth2

#ifconfig eth2 down
# 发现ping不会中断

#cat /proc/net/bonding/bond1
eth3

```

```bash
# cp ifcfg-bond0 ifcfg-br0

#ifcfg-bond0 
TYPE=Ethernet
BOOTPROTO=static
DEVICE=bond0
ONBOOT=yes
#
BRIDGE=br0

#ifcfg-br0
TYPE=Bridge
#
BOOTPROTO=static
DEVICE=br0
ONBOOT=yes

IPADDR=192.168.1.7
NETMASK=255.255.255.0
DNS1=223.6.6.6
DNS2=114.114.114.114
GATEWAY=192.168.1.1

#systemctl restart network

# 另一台主机ping 192.168.1.7, ping不通就重启
#cat /proc/net/bonding/bond1
eth0

#ifconfig eth0 down
# 发现ping不会中断

#cat /proc/net/bonding/bond1
eth1


# cp ifcfg-bond1 ifcfg-br1
#ifcfg-bond1 
TYPE=Ethernet
BOOTPROTO=static
DEVICE=bond1
ONBOOT=yes
#
BRIDGE=br1

#ifcfg-br1
TYPE=Bridge
#
BOOTPROTO=static
DEVICE=br1
ONBOOT=yes

IPADDR=10.20.10.2
NETMASK=255.255.0.0


# mv route-bond1 route-br1
#route-br1
10.20.0.0/255.255.0.0 via 10.20.0.1 dev br1
172.16.0.0/255.255.0.0 via 10.20.0.1 dev br1
192.168.100.0/255.255.255.0 via 10.20.0.1 dev br1

# 另一台主机ping 10.20.10.2 , ping不通就重启
#cat /proc/net/bonding/bond1
eth2

#ifconfig eth2 down
# 发现ping不会中断

#cat /proc/net/bonding/bond1
eth3
```



### 2.2.2 ubuntu冗余bond+桥接

```bash
#/etc/netplan/01-netcfg.yaml
# This file describes the network interfaces available on your system
# For more information, see netplan(5).
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      dhcp6: no
    eth1:
      dhcp4: no
      dhcp6: no


    eth2:
      dhcp4: no
      dhcp6: no
    eth3:
      dhcp4: no
      dhcp6: no

  bonds:
    bond0:
      dhcp4: no
      dhcp6: no
#      addresses: [192.168.1.100/24]
#      gateway4: 192.168.1.1
#      nameservers:
#        addresses: [223.6.6.6,223.5.5.5]
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
      interfaces:
      - eth0
      - eth1

    bond1:
      dhcp4: no
      dhcp6: no
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
      interfaces:
      - eth2
      - eth3


  bridges:
    br0:
      dhcp4: no
      dhcp6: no
      addresses: [192.168.1.100/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [223.6.6.6,223.5.5.5]
      interfaces:
      - bond0
    
      
      
    br1:
      dhcp4: no
      dhcp6: no
      interfaces:
      - bond1
      addresses: [10.20.0.188/16]
      routes:
      - to: 192.168.1.0/24
        via: 10.20.0.1
      - to: 10.20.0.0/16
        via: 10.20.0.1



# netplan apply
# reboot

# 另一台主机ping 192.168.1.100
#cat /proc/net/bonding/bond0
eth1

#ifconfig eth1 down
# 发现ping不会中断

#cat /proc/net/bonding/bond1
eth0

```



## 2.3 centos/ubuntu 高可用(mode=1)+高性能(mode=0)+桥接

![Linux 架构](http://myapp.img.mykernel.cn/Linux_架构.png)

> #根据老师上课, 讲的, 只是还没有实现出来，以下以mode=6代替。

### 2.3.1 centos 高可用(mode=1)+高性能(mode=0)+桥接

```bash
#ifcfg-eth0
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes

MASTER=bond0
SLAVE=yes


#ifcfg-eth1
BOOTPROTO=static
DEVICE=eth1
ONBOOT=yes

MASTER=bond0
SLAVE=yes


#ifcfg-eth2
BOOTPROTO=static
DEVICE=eth2
ONBOOT=yes

MASTER=bond1
SLAVE=yes


#ifcfg-eth3
BOOTPROTO=static
DEVICE=eth3
ONBOOT=yes

MASTER=bond1
SLAVE=yes



#ifcfg-bond0 
BOOTPROTO=static
ONBOOT=yes

DEVICE=bond0
BRIDGE=br0


#ifcfg-bond1
BOOTPROTO=static

ONBOOT=yes
DEVICE=bond1
BRIDGE=br1


#ifcfg-br0 
BOOTPROTO=static
DEVICE=br0
ONBOOT=yes
TYPE=Bridge

IPADDR=192.168.1.7
NETMASK=255.255.255.0
DNS1=114.114.114.114
GATEWAY=192.168.1.1

#ifcfg-br1
BOOTPROTO=static
DEVICE=br1
ONBOOT=yes
TYPE=Bridge

IPADDR=10.20.10.2
NETMASK=255.255.0.0

#route-br1
10.20.0.0/16 via 10.20.0.1 dev br1
172.16.0.0/16 via 10.20.0.1 dev br1
192.168.100.0/24 via 10.20.0.1 dev br1

#/etc/modprobe.d/bond.conf 
alias bond0 bonding
options bond0 mode=6 miimon=200
alias bond1 bonding
options bond1 mode=6 miimon=200

# systemctl restart network

#查看带宽
# ethtool bond0
	Speed: 2000Mb/s 


# ethtool bond1
	Speed: 2000Mb/s

#查看bond状态
# cat /proc/net/bonding/bond0
Bonding Mode: adaptive load balancing
Currently Active Slave: eth0

# cat /proc/net/bonding/bond1
Bonding Mode: adaptive load balancing
Currently Active Slave: eth2

#在Ping过程中，停止eth0, eth2可以发现带宽自动降低，发现ping不中断
# ifconfig eth0 down
# ethtool bond0
	Speed: 1000Mb/s

#但是在启动eth0, 也就是eth0网卡恢复时，需要重启网卡，才可以恢复网络
# ifconfig eth0 up #网络中断
#一会又恢复, ping出现DUP

#systemctl restart network
#恢复


# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    0      0        0 br0
10.20.0.0       0.0.0.0         255.255.0.0     U     0      0        0 br1
172.16.0.0      10.20.0.1       255.255.0.0     UG    0      0        0 br1
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 br0
192.168.100.0   10.20.0.1       255.255.255.0   UG    0      0        0 br1
```



### 2.3.2  ubuntu 高可用(mode=1)+高性能(mode=0)+桥接

```bash
#/etc/netplan/01-netcfg.yaml
# This file describes the network interfaces available on your system
# For more information, see netplan(5).
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      dhcp6: no
    eth1:
      dhcp4: no
      dhcp6: no


    eth2:
      dhcp4: no
      dhcp6: no
    eth3:
      dhcp4: no
      dhcp6: no

  bonds:
    bond0:
      dhcp4: no
      dhcp6: no
#      addresses: [192.168.1.100/24]
#      gateway4: 192.168.1.1
#      nameservers:
#        addresses: [223.6.6.6,223.5.5.5]
      parameters:
        mode: 6
        mii-monitor-interval: 100
      interfaces:
      - eth0
      - eth1

    bond1:
      dhcp4: no
      dhcp6: no
      parameters:
        mode: 6
        mii-monitor-interval: 100
      interfaces:
      - eth2
      - eth3


  bridges:
    br0:
      dhcp4: no
      dhcp6: no
      addresses: [192.168.1.100/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [223.6.6.6,223.5.5.5]
      interfaces:
      - bond0
    
      
      
    br1:
      dhcp4: no
      dhcp6: no
      interfaces:
      - bond1
      addresses: [10.20.0.188/16]
      routes:
      - to: 192.168.1.0/24
        via: 10.20.0.1
      - to: 10.20.0.0/16
        via: 10.20.0.1



# netplan apply
# reboot

# 当一个网卡故障，带宽会减半
# 当恢复时，需要重启机器
```



