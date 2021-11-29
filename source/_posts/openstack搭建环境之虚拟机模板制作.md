---
title: openstack搭建环境之虚拟机模板制作
date: 2020-12-25 02:00:41
tags:
toc: true
---



openstack是（ infrastructure as a service ，基础设置即服务 IAAS 架构的实现， OpenStack 是一个由 NASA （美国国家航空航天局）和 Rackspace 合作研发并发起的，以 Apache 许可证授权的自由软件和开放源代码项目。

OpenStack是一个旨在为公共及私有云的建设与管理提供软件的开源项目。它的社区拥有超过 130 家企业及 1350 位开发者，这些机构与个人都将 OpenStack 作为基础设施即服务（ IaaS资源的通用前端。 OpenStack 项目的首要任务是简化云的部署过程并为其带来良好的可扩展性。

openstack的核心组件是计算、网络和存储，底层是KVM。

[openstack](http://docs.openstack.org)每半年更新一次新版本，遵循一个一年两次的开发及发布的周期，在春末提供一个发布，秋季第二个版本。使用版本的代号按字母顺序排列

openstack最简安装， identity(keystone), glance, nova( controller, compute), neutron( controller, compute), dashboard(Horizon)

| 组件      | 作用                                             |
| --------- | ------------------------------------------------ |
| Keystone  | 提供账户登录安全认证                             |
| Glance    | 提供虚拟镜像的注册和存储管理                     |
| Placement | 收集node的资源使用情况，供controller使用         |
| nova      | 通过虚拟化技术提供虚拟机计算资源池               |
| neutron   | 实现了虚拟机的网络资源管理，即虚拟机网络         |
| dashboard | 基予openstack API接口使用django开发的web管理服务 |

以上这些组件先规划一下ip

| ip           | 作用           | 描述                                                         |
| ------------ | -------------- | ------------------------------------------------------------ |
| 172.16.0.100 | 制作虚拟机镜像 | 使用100的优势，在配置各个主机的网络时可以快速配置，仅需要修改一个数字。 |
| 172.16.0.101 | controller1    |                                                              |
| 172.16.0.102 | controller2    |                                                              |
| 172.16.0.103 | mysql1         |                                                              |
| 172.16.0.104 | mysql2         |                                                              |
| 172.16.0.105 | haproxy1       | 也做nfs master(rsync+inotify)                                |
| 172.16.0.106 | haproxy2       | 做nfs slave.                                                 |
| 172.16.0.107 | node1          |                                                              |
| 172.16.0.108 | node2          |                                                              |
| 172.16.0.109 | node3          |                                                              |
| 172.16.0.110 | node4          |                                                              |



前期部署为了搭建起来, 仅做最简单的架构, 所有节点单点。仅看蓝色区域

架构的高可用，controller, mysql, nfs, rabbitmq在后期在做。

![image-20201225104216105](http://myapp.img.mykernel.cn/image-20201225104216105.png)

1. 运维连接haproxy到达crontroller
2. controller将集群状态数据写入mysql
3. controller通过rabbitmq与node通信
4. glance组件将数据放在共享存储。



由于所有端口在haproxy上管理真正数据流转图如下

![image-20201225105256532](http://myapp.img.mykernel.cn/image-20201225105256532.png)



# 部署前提

- 所有主机2个网卡
  - eth0 外网通信，安装openstack组件
  - eth1 内网通信
- 所有主机使用相同的发行版 centos 7+ 3.10.x+/ubuntu 1804+
- compute节点必须打开虚拟化支持. windows如图，linux: [打开cpu虚拟化](http://blog.mykernel.cn/2020/11/11/%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AEkvm%E8%99%9A%E6%8B%9F%E6%9C%BA%EF%BC%8C%E5%B9%B6%E5%88%9B%E5%BB%BA%E8%99%9A%E6%8B%9F%E6%9C%BA/#9-2-%E5%AE%89%E8%A3%85kvm)
- 不能对openstack做内核优化操作，在后期优化
- 必须2个网卡，否则openstack启动不了
- controller在虚拟机上VMware/KVM/VMware EXSI. 方便快照和回滚。node节点在高配置的物理机上。

# 部署顺序

- [x] 虚拟机制作
- [x] haproxy
- [x] mysql、rabbitmq、memcached
- [x] controller keystone
- [x] controller glance
- [x] nova controller
- [x] nova compute
- [x] nova check
- [x] neutron controller
- [x] neutron compute
- [x] neutron check
- [x] dashboard



<!--more-->

## 1.1 环境准备

![image-20201225100433009](http://myapp.img.mykernel.cn/image-20201225100433009.png)



- [ ] 4核cpu,4G内存，安装openstack更快，运行虚拟机也更多一点
- [ ] 120GB，一个虚拟机使用10GB磁盘。至少10个虚拟机
- [x] **满足**2网卡
- [x] **满足**统一发行版: centos7.7 1908
- [x] **满足**虚拟化
- [x] 没有优化内核参数
- [x] **满足**VMware



NAT模式：外网通信

![image-20201225100859198](http://myapp.img.mykernel.cn/image-20201225100859198.png)

添加网桥，内网通信

![image-20201225100912719](http://myapp.img.mykernel.cn/image-20201225100912719.png)

## 1.2 初始化虚拟机

卸载NetworkManager, firewalld

```bash
yum remove -y NetworkManager* firewalld*
```

禁用selinux

```bash
[root@localhost ~]# cat /etc/sysconfig/selinux
7	SELINUX=disabled
```

安装常用命令和桥接命令

```bash
yum install -y vim wget tree  lrzsz gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop ntpdate lsof bridge-utils
```

时间同步 

```bash
# crontab -e
*/10 * * * * /usr/sbin/ntpdate time1.aliyun.com && hwclock -w
```



```bash
[root@localhost ~]# crontab -l
*/10 * * * * /usr/sbin/ntpdate time1.aliyun.com && hwclock -w
```

主机名

```bash
hostnamectl set-hostname "openstack-template.magedu.local"
```

验证

```bash
~]# cat /etc/hostname 
openstack-template.magedu.local
```



时区

```bash
timedatectl set-timezone Asia/Shanghai
```

验证

```bash
~]# timedatectl 
      Local time: Fri 2020-12-25 10:15:08 CST
  Universal time: Fri 2020-12-25 02:15:08 UTC
        RTC time: Fri 2020-12-25 02:15:08
       Time zone: Asia/Shanghai (CST, +0800)
     NTP enabled: n/a
NTP synchronized: no
 RTC in local TZ: no
      DST active: n/a
```



配置网络eth0, eth1方便克隆时，快速配置网络地址

```bash
[root@openstack-template ~]# cd /etc/sysconfig/network-scripts/
[root@openstack-template network-scripts]# cat ifcfg-eth0
TYPE="Ethernet" 
BOOTPROTO="static"    
DEFROUTE="yes"
DEVICE="eth0"
ONBOOT="yes"
IPADDR=172.16.0.100
NETMASK=255.255.0.0
GATEWAY=172.16.0.1
# 不配置DNS, 配置DNS后network会在/etc/resolv.conf文件加domain, k8s会有问题
[root@openstack-template network-scripts]# cat ifcfg-eth1
TYPE=Ethernet
BOOTPROTO=none
DEVICE=eth1
ONBOOT=yes
IPADDR=10.0.0.100
NETMASK=255.255.0.0
```

> 1. 配置100，后期的静态ip全是101,102,...所以方便配置
> 2. eth1内网不需要网关，仅需要走静态路由，就可以与其他网络通信

配置dns

```bash
# cat /etc/resolv.conf 
nameserver 223.6.6.6
```



重启验证

```bash
reboot
```

> 在重启的过程中，xshell添加模板地址

![image-20201225102257733](http://myapp.img.mykernel.cn/image-20201225102257733.png)

![image-20201225102243132](http://myapp.img.mykernel.cn/image-20201225102243132.png)



验证iptables, selinux, hostname, dns, ip

```bash
[root@openstack-template ~]# iptables -vnL
Chain INPUT (policy ACCEPT 0 packets, 0 bytes) # ACCEPT, 规则空
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)# ACCEPT, 规则空
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes) # ACCEPT, 规则空
 pkts bytes target     prot opt in     out     source               destination         
[root@openstack-template ~]# getenforce 
Disabled # 表示已经禁用
[root@openstack-template ~]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.0.100  netmask 255.255.0.0  broadcast 172.16.255.255 # IP, NETMASK
        inet6 fe80::20c:29ff:fe66:c350  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:66:c3:50  txqueuelen 1000  (Ethernet)
        RX packets 64  bytes 6901 (6.7 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 73  bytes 9741 (9.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.0.100  netmask 255.255.0.0  broadcast 10.0.255.255# IP, NETMASK
        inet6 fe80::20c:29ff:fe66:c35a  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:66:c3:5a  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 14  bytes 1016 (1016.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@openstack-template ~]# cat /etc/resolv.conf 
nameserver 223.6.6.6 # DNS没有问题
[root@openstack-template ~]# hostname
openstack-template.magedu.local # 主机名没有问题

[root@openstack-template ~]# ping www.baidu.com
PING www.a.shifen.com (14.215.177.38) 56(84) bytes of data.
64 bytes from 14.215.177.38 (14.215.177.38): icmp_seq=1 ttl=128 time=40.2 ms
^C              # 按CTRL + C可以终止ping. linux不手工终止会一直ping.
--- www.a.shifen.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 40.282/40.282/40.282/0.000 ms
[root@openstack-template ~]# 
```



关机快照

```bash
[root@openstack-template ~]# poweroff
```

![image-20201225102732743](http://myapp.img.mykernel.cn/image-20201225102732743.png)