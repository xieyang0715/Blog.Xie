---
title: 安装配置openstack(Train)，并创建虚拟机
date: 2020-11-11 06:55:04
tags: plan
toc: true
---



官网：https://docs.openstack.org

# 1. openstack组件及部署顺序

| 名称                                                         | 作用                                         | 工作模式 |
| ------------------------------------------------------------ | -------------------------------------------- | -------- |
| [Compute service (Nova)](https://docs.openstack.org/nova/victoria/install/) | 计算                                         | C/S      |
| [Image service (Glance)](https://docs.openstack.org/glance/victoria/install/) | 镜像服务                                     |          |
| [Identity service (Keystone)](https://docs.openstack.org/keystone/train/install/) | 认证，所有服务都通过公共身份服务进行身份验证 |          |
| [Networking service (Neutron)](https://docs.openstack.org/neutron/train/install/) | 网络，桥接使用的多。自服务不用。             | C/S      |
| [Dashboard (Horizon)](https://docs.openstack.org/horizon/train/install/) | web UI, 可有可无                             |          |

安装顺序

1. 环境 https://docs.openstack.org/install-guide/environment.html,  ==配置controller节点和node节点==	
2. 包(所有节点需要安装源) https://docs.openstack.org/install-guide/environment-packages.html, 发行版发布OpenStack包作为发行版的一部分，或者使用其他方法，因为发布计划不同。在==所有节点上, 只要属于openstack的节点==执行这些过程。
3. 认证：所有服务都通过公共身份服务进行身份验证
4. 镜像，上传镜像
5. 计算服务的控制节点和compute node
6. 网络服务的控制节点和compute node
7. dashboard前端



# 2. openstack版本选择

| 版本    | 描述                 |
| ------- | -------------------- |
| alpha   | 内部测试，不使用     |
| dev     | 开发                 |
| beta    | 比dev稳定            |
| rc      | 不加新功能，主要排错 |
| release | 标准版               |
| ga      | 正式版               |



# 3. openstack部署架构示例



![image-20201116214330430](http://myapp.img.mykernel.cn/image-20201116214330430.png)

**controller节点**：controller节点运行the Identity service, Image service, Placement service, management portions of Compute, management portion of Networking, various Networking agents, and the Dashboard. 它还包括支持服务，例如 an SQL database, [message queue](https://docs.openstack.org/install-guide/common/glossary.html#term-message-queue), and [NTP](https://docs.openstack.org/install-guide/common/glossary.html#term-Network-Time-Protocol-NTP).

可选地，控制器节点运行the Block Storage, Object Storage, Orchestration, and Telemetry services.

控制器节点至少需要两个网络接口。

 

**计算节点：**运行操作实例的Compute的[管理程序](https://docs.openstack.org/install-guide/common/glossary.html#term-hypervisor)部分。默认情况下，Compute使用 [KVM](https://docs.openstack.org/install-guide/common/glossary.html#term-kernel-based-VM-KVM)管理程序。计算节点还运行网络服务代理，该服务将实例连接到虚拟网络，并通过[安全组](https://docs.openstack.org/install-guide/common/glossary.html#term-security-group)为实例提供防火墙服务 。

您可以部署多个计算节点。每个节点至少需要两个网络接口。





**网络**

1. **provider networks选项**：主要通过第2层（桥接/交换）服务和网络的VLAN分段，以最简单的方式部署OpenStack网络服务。本质上，它将虚拟网络桥接到物理网络，并依赖于物理网络基础结构来进行第3层（路由）服务。另外，[DHCP](https://docs.openstack.org/install-guide/common/glossary.html#term-Dynamic-Host-Configuration-Protocol-DHCP)服务向实例提供IP地址信息。

OpenStack用户需要有关基础网络基础结构的更多信息，以创建与基础结构完全匹配的虚拟网络。

![network1-services](http://myapp.img.mykernel.cn/network1-services.png)



<!--more-->



# 4. Openstack(Train)单机部署

https://docs.openstack.org/install-guide/environment.html



大多数环境包括 Identity, Image service, Compute, at least one networking service, and the Dashboard

以下是最低要求，来验证以上概念，并运行核心的服务和一些cirros镜像

- ==controller node== 控制器节点：==1个处理器，4 GB内存==和5 GB存储
- ==compute node== 计算节点：==1个处理器，2 GB内存==和10 GB存储

每个节点上安装==64位版本==的发行版

==统一的Linux发行版==, centos/ubuntu

==控制节点安装过程使用虚拟机==，好处是可以快照，支持回滚到任意时刻的配置。VM一定要打开嵌套的虚拟化支持。

所有节点必须==2个网络接口==，并==支持网络访问==。

![networklayout](http://myapp.img.mykernel.cn/networklayout.png)

此次部署的网络：

- 在网关10.0.0.1是10.0.0.0/24的网络里的完成网络管理。

  通过仅主机或虚拟网桥完成一个隔离的网络，提供给openstack来管理网络

- Provider在网关是172.16.0.1的172.16.0.0/16网络里。==与图有差异==

  通过NAT或桥接完成给内部主机提供网络访问
  
  这个网络需要一个网关来提供对OpenStack环境中实例的Internet访问。
  
  此网络要求网关为所有节点提供Internet访问，以便进行管理，例如包安装、安全更新、[dns](https://docs.openstack.org/install-guide/common/glossary.html#term-Domain-Name-System-DNS)，和[NTP](https://docs.openstack.org/install-guide/common/glossary.html#term-Network-Time-Protocol-NTP).
  
  

| ip           | 域名                                | 描述   | VIP          |
| ------------ | ----------------------------------- | ------ | ------------ |
| 172.16.0.101 | openstack-controller1.magedu.local  |        |              |
| 172.16.0.102 | openstack-controller2.magedu.local  | option |              |
| 172.16.0.103 | openstack-mysql-master.magedu.local |        |              |
| 172.16.0.104 | openstack-mysql-slave.magedu.local  | option |              |
| 172.16.0.105 | openstack-haproxy1.magedu.local     |        | 172.16.0.248 |
| 172.16.0.106 | openstack-haproxy2.magedu.local     | option | 172.16.0.249 |
| 172.16.0.107 | openstack-node1.magedu.local        |        |              |
| 172.16.0.108 | openstack-node2.magedu.local        |        |              |
| 172.16.0.109 | openstack-node3.magedu.local        | option |              |

## 4.1 环境准备：安装和配置组件 

### 4.1.1 准备provider网络，由kvm配置桥接接口

启动的controller只需要指向这个网关就可以完成外网通信

```bash
# 添加桥接接口
brctl addbr vmbr0
# 配置桥接接口地址，此即为provider网络的网关
ifconfig vmbr0 172.16.0.1/16 up
# 需要内部通信完成SNAT转换源地址
iptables -t nat -A POSTROUTING -s 172.16.0.0/16 -j MASQUERADE
# 需要网关打开转发
# grep 'net\.ipv4\.ip_forward' /etc/sysctl.conf 
net.ipv4.ip_forward = 1
```

加入开机启动, 编辑/etc/rc.local

```bash
# vmbr0 provider
# 添加桥接接口
brctl addbr vmbr0
# 配置桥接接口地址，此即为provider网络的网关
ifconfig vmbr0 172.16.0.1/16 up
# 需要内部通信完成SNAT转换源地址
iptables -t nat -A POSTROUTING -s 172.16.0.0/16 -j MASQUERADE

```



准备内部网络，完成网络管理

```bash
# vmbr1 internal
# 添加桥接接口
brctl addbr vmbr1
# 配置桥接接口地址，此即为内部通信的网关
ifconfig vmbr1 10.0.0.1/24 up
# 不需要转发，就是需要其不能上外网
```

加入/etc/rc.local



### 4.1.2 创建虚拟机

准备kvm模板

> [kvm创建模板](http://blog.mykernel.cn/2020/11/11/%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AEkvm%E8%99%9A%E6%8B%9F%E6%9C%BA%EF%BC%8C%E5%B9%B6%E5%88%9B%E5%BB%BA%E8%99%9A%E6%8B%9F%E6%9C%BA/#9-3-%E5%88%9B%E5%BB%BAkvm%E6%A8%A1%E6%9D%BF)
>
> 注意，只使用centos/ubuntu来完成创建openstack.



创建controller1

```bash
[root@centos7-iaas ~]# cd /tools/downloads/linux_scripts/kvm/
[root@centos7-iaas kvm]# NAME=openstack-controller1;  ./safe_clone_kvm.sh -i /VMs/template/centos7-1804-1C1G.qcow2 -d /VMs/openstack/${NAME}.qcow2  -t kvm  -n ${NAME} -r 1024  -v 1 -b vmbr0 -f vmbr1
# NAME是kvm创建名称，因为在virt-manager界面不能分目录，就加个openstack前缀方便区分
# -i /VMs/template/centos7-1804-1C1G.qcow2 在官网支持系统有centos/ubuntu/SLS，suse. https://docs.openstack.org/install-guide/preface.html#operating-systems
# -r 1024  -v 1  1024G内存，要求是1C4G，但是我的环境太小了。
# =============openstack所有主机必须2个接口==================================
# -b # 脚本中指定连接到虚拟机的第一个接口
# -f # 脚本中指定连接到虚拟机的第二个接口
```

> 脚本来源请参考[kvm创建虚拟机](http://blog.mykernel.cn/2020/11/11/%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AEkvm%E8%99%9A%E6%8B%9F%E6%9C%BA%EF%BC%8C%E5%B9%B6%E5%88%9B%E5%BB%BA%E8%99%9A%E6%8B%9F%E6%9C%BA/#9-4-%E5%88%9B%E5%BB%BA%E8%99%9A%E6%8B%9F%E6%9C%BA)

启动后需要配置网络，参考[配置单接口的网络](http://blog.mykernel.cn/2020/12/18/ubuntu-1804%E5%92%8Ccentos-1804%E9%85%8D%E7%BD%AE%E5%8D%95%E7%BD%91%E5%8D%A1%E7%BD%91%E7%BB%9C/)

1. 配置eth0，验证网络通了

2. 配置eth1, 不需要网关。

3. 主机名

4. 重启，验证网络通了，route, 主机名。 

   如果不重启验证，直接快照，假如配置问题，可能创建了一堆虚拟机都有问题。

5. poweroff, 快照，添加xshell

> 先ifconfig eth0 172.16.0.101/16 up, 在xshell中连接后配置网络。

```bash
[root@centos-template network-scripts]# cat ifcfg-eth0 
TYPE=Ethernet
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes
IPADDR=172.16.0.101
NETMASK=255.255.0.0
GATEWAY=172.16.0.1
#DNS1=192.168.0.1

[root@centos-template network-scripts]# cat ifcfg-eth1
TYPE=Ethernet
BOOTPROTO=static
DEVICE=eth1
ONBOOT=yes
IPADDR=10.0.0.101
NETMASK=255.255.255.0
[root@centos-template network-scripts]# systemctl restart network
# 测试内网是通的
[root@centos7-2-1511 network-scripts]# ping 10.0.0.1
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.629 ms

[root@centos-template network-scripts]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.0.1      0.0.0.0         UG    0      0        0 eth0
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 eth1
172.16.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
[root@centos-template network-scripts]# ping www.baidu.com
PING www.a.shifen.com (110.242.68.4) 56(84) bytes of data.
64 bytes from 110.242.68.4 (110.242.68.4): icmp_seq=1 ttl=51 time=31.9 ms
[root@centos-template network-scripts]# cat /etc/resolv.conf
nameserver 223.6.6.6
[root@centos-template network-scripts]# echo 'openstack-controller1.magedu.local' > /etc/hostname 
[root@centos-template network-scripts]# reboot

# 验证
[root@centos-template network-scripts]# ping www.baidu.com
PING www.a.shifen.com (110.242.68.4) 56(84) bytes of data.
64 bytes from 110.242.68.4 (110.242.68.4): icmp_seq=1 ttl=51 time=31.9 ms
[root@centos-template network-scripts]# cat /etc/resolv.conf
nameserver 223.6.6.6

# 快照 
[root@centos-template network-scripts]# poweroff

```



以下如同controller1的方式，克隆安装出mysql-master,  haproxy1, haproxy2,  node1, node2, node3

```bash
# NAME=openstack-mysql1;  ./safe_clone_kvm.sh -i /VMs/template/centos7-1804-1C1G.qcow2 -d /VMs/openstack/${NAME}.qcow2  -t kvm  -n ${NAME} -r 1024  -v 1 -b vmbr0 -f vmbr1
# ifconfig eth0 172.16.0.103/16 up
# echo 'openstack-mysql-master.magedu.local' > /etc/hostname 

# NAME=openstack-haproxy1;  ./safe_clone_kvm.sh -i /VMs/template/centos7-1804-1C1G.qcow2 -d /VMs/openstack/${NAME}.qcow2  -t kvm  -n ${NAME} -r 1024  -v 1 -b vmbr0 -f vmbr1
# ifconfig eth0 172.16.0.105/16 up
# echo 'openstack-haproxy1.magedu.local' > /etc/hostname 

# NAME=openstack-node1;  ./safe_clone_kvm.sh -i /VMs/template/centos7-1804-1C1G.qcow2 -d /VMs/openstack/${NAME}.qcow2  -t kvm  -n ${NAME} -r 1024  -v 1 -b vmbr0 -f vmbr1
# ifconfig eth0 172.16.0.107/16 up
# echo 'openstack-node1.magedu.local' > /etc/hostname 


# 其他可以后期做高可用
```

>  安装keepalived + haproxy 可以参考[haproxy+keepalived高可用](http://blog.mykernel.cn/2020/10/26/%E5%AE%9E%E7%8E%B0haproxy-keepalived%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4%E8%BD%AC%E5%8F%91/)



### 4.1.3 时间同步

controller和所有node完成时间同步  

```bash
#时间同步
# crontab -l
*/10 * * * * /sbin/ntpdate time1.aliyun.com && /usr/sbin/hwclock -w &> /dev/null
```





> 约定数据库相关密码为: openstack123
>
> 服务账号和密码同名: nova, nova123

## 4.2 openstack 包安装

https://docs.openstack.org/install-guide/environment-packages.html在最下面可以选择centos版本

![image-20201116221031851](http://myapp.img.mykernel.cn/image-20201116221031851.png)



在CentOS上，`extras`存储库提供了启用OpenStack存储库的RPM。CentOS包括`extras`默认情况下存储库，因此您可以简单地安装包来启用OpenStack存储库。对于CentOS 8，还需要启用PowerTools存储库。

==查看发行版提供的openstack版本==

```bash
[root@openstack-controller1 ~]# yum makecache fast
[root@openstack-controller1 ~]# yum list openstack*
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.163.com
 * extras: mirrors.163.com
 * updates: mirror.bit.edu.cn
错误：没有匹配的软件包可以列出
[root@openstack-controller1 ~]# yum list *openstack*
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.163.com
 * extras: mirrors.163.com
 * updates: mirror.bit.edu.cn
可安装的软件包
centos-release-openstack-queens.noarch                                                      1-2.el7.centos                                                       extras
centos-release-openstack-rocky.noarch                                                       1-1.el7.centos                                                       extras
centos-release-openstack-stein.noarch                                                       1-1.el7.centos                                                       extras
centos-release-openstack-train.noarch                                                       1-1.el7.centos                                                       extras
```

> 注意，来自官方源，过段时间openstack的版本更新了，官方源更新，之前版本就安装不了了，所以需要将镜像openstack源。https://mirrors.aliyun.com/centos-vault/7.5.1804/cloud/x86_64/
>
> https://mirrors.aliyun.com/centos/7/cloud/x86_64/



==需要在所有节点上安装以下包==

```bash
yum install centos-release-openstack-train

# rdo源
yum install https://rdoproject.org/repos/rdo-release.rpm
```

> 官方还执行yum upgrade, 此步骤并不需要

 

==以下仅在控制节点安装==

```bash
# 安装openstack客户端
yum install python-openstackclient
# centos/rhel自动启动selinux, 安装包用于自动管理安全策略。
yum install openstack-selinux
# 安装mysql, memcache客户端
yum install -y  python-memcached python2-PyMySQL
```



## 4.3 [mariadb数据库安装](https://docs.openstack.org/install-guide/environment-sql-database.html)

安装组件

数据库通常在控制器节点上运行，本次是独立的一个节点运行（rabbitmq, memcached, mysql）

```bash
# yum install centos-release-openstack-train -y
# yum install https://rdoproject.org/repos/rdo-release.rpm -y
# yum install mariadb mariadb-server 

#/etc/my.cnf.d/openstack.cnf
[mysqld]
bind-address = 0.0.0.0

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

```bash
# systemctl enable mariadb.service
# systemctl start mariadb.service
```

>  配置root密码, openstack123
>
> ```bash
> # mysql_secure_installation 
> ```

```bash
#检验
[root@openstack-mysql-master ~]# mysql -popenstack123
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 21
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> exit
Bye
```



haproxy暴露mysql端口

```bash
#  /etc/haproxy/haproxy.cfg 

listen mysql-3306
    bind 172.16.0.248:3306
    mode tcp
    server 172.16.0.103 172.16.0.103:3306  check port 3306 inter 3s fall 2 rise 5
    #server 172.16.0.103 172.16.0.103:3306  
  

  
# systemctl restart haproxy
```

打开haproxy状态页查看mysql tcp检测是否已经通了

http://172.16.0.248:9999/haproxy

![image-20201218172213560](http://myapp.img.mykernel.cn/image-20201218172213560.png)



## 4.5 [消息队列安装](https://docs.openstack.org/install-guide/environment-messaging.html)

OpenStack使用[消息队列](https://docs.openstack.org/install-guide/common/glossary.html#term-message-queue)协调服务之间的操作和状态信息。消息队列服务通常在控制器节点上运行。

```bash

# 安装软件包
# yum install rabbitmq-server
# systemctl enable rabbitmq-server.service
# systemctl start rabbitmq-server.service

#添加openstack用户：openstack123是密码
# rabbitmqctl add_user openstack openstack123

#允许openstack用户配置、写入和读取访问权限。
# rabbitmqctl set_permissions openstack ".*" ".*" ".*"

```



haproxy暴露openstack端口

```bash

listen rabbitmq-5672
  bind 172.16.0.248:5672
  mode tcp
  server 172.16.0.103 172.16.0.103:5672  check port 5672 inter 3s fall 2 rise 5

  
# systemctl restart haproxy keepalived
```

http://172.16.0.248:9999/haproxy

![image-20201218173954551](http://myapp.img.mykernel.cn/image-20201218173954551.png)

## 4.6 [memcached安装](https://docs.openstack.org/install-guide/environment-memcached.html)

session共享, 用于认证，保持会话。

```bash
# yum install memcached 

#编辑/etc/sysconfig/memcached归档并完成以下操作：
#配置服务以使用控制器节点的管理IP地址。这是为了使其他节点能够通过管理网络访问：
OPTIONS="-l 0.0.0.0,::1,controller"


# systemctl enable memcached.service
# systemctl start memcached.service
```

haproxy配置

```bash

listen memcached-11211
  bind 172.16.0.248:11211
  mode tcp
  server 172.16.0.103 172.16.0.103:11211  check port 11211 inter 3s fall 2 rise 5

# systemctl restart haproxy keepalived
```

http://172.16.0.248:9999/haproxy

![image-20201218174212720](http://myapp.img.mykernel.cn/image-20201218174212720.png)

## 4.7 [安装openstack服务](https://docs.openstack.org/install-guide/openstack-services.html)

![image-20201116222520832](http://myapp.img.mykernel.cn/image-20201116222520832.png)

在控制器节点上安装和配置OpenStack标识服务(代号为Keystone)。为了实现可伸缩性，此配置部署Fernet令牌和ApacheHTTP服务器来处理请求

### 4.7.1 [安装认证服务 keystone](https://docs.openstack.org/keystone/train/install/index.html)

https://docs.openstack.org/keystone/train/install/keystone-install-rdo.html

```bash
# mysql主机
[root@openstack-mysql ~]# mysql -u root -p

CREATE DATABASE keystone;

GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'keystone123';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone123';

# 验证
[root@openstack-mysql-master ~]# mysql -ukeystone -pkeystone123 -h172.16.0.248
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 28
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> ^DBye

```



在controller1

```bash
# 添加vip解析
[root@openstack-controller1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.0.248 openstack-vip.magedu.local

```

安装包

```bash
# yum install openstack-keystone httpd mod_wsgi
```

```bash
#编辑/etc/keystone/keystone.conf归档并完成以下操作：
#在[database]节，配置数据库访问：
connection = mysql+pymysql://keystone:keystone123@openstack-vip.magedu.local/keystone


#在[token]节中，配置Fernet令牌提供程序
[token]
# ...
provider = fernet

```

填充身份服务数据库：

```bash
#
su -s /bin/sh -c "keystone-manage db_sync" keystone

# 验证
[root@openstack-mysql-master ~]# mysql -ukeystone -pkeystone123 -h172.16.0.248
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 30
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
+--------------------+
2 rows in set (0.003 sec)

MariaDB [(none)]> use keystone
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [keystone]> show tables;
+-----------------------+
| Tables_in_keystone    |
+-----------------------+
| assignment            |
| credential            |
| domain                |
| endpoint              |
| group                 |
| id_mapping            |
| migrate_version       |
| policy                |
| project               |
| region                |
| role                  |
| sensitive_config      |
| service               |
| token                 |
| trust                 |
| trust_role            |
| user                  |
| user_group_membership |
| whitelisted_config    |
+-----------------------+
19 rows in set (0.002 sec)

# 观察有表输出
```

初始化Fernet密钥存储库：

```bash
#
# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

引导身份服务：

```bash

keystone-manage bootstrap --bootstrap-password admin \
--bootstrap-admin-url http://openstack-vip.magedu.local:5000/v3/ \
--bootstrap-internal-url http://openstack-vip.magedu.local:5000/v3/ \
--bootstrap-public-url http://openstack-vip.magedu.local:5000/v3/ \
--bootstrap-region-id RegionOne
# 创建默认域
# 验证
MariaDB [keystone]> select * from project;
+----------------------------------+--------------------------+-------+-----------------------------------------------+---------+--------------------------+-----------+-----------+
| id                               | name                     | extra | description                                   | enabled | domain_id                | parent_id | is_domain |
+----------------------------------+--------------------------+-------+-----------------------------------------------+---------+--------------------------+-----------+-----------+
| 5c77b1ffc4124e17be58a9f9944cff56 | admin                    | {}    | Bootstrap project for initializing the cloud. |       1 | default                  | default   |         0 |
| <<keystone.domain.root>>         | <<keystone.domain.root>> | {}    |                                               |       0 | <<keystone.domain.root>> | NULL      |         1 |
| default                          | Default                  | {}    | The default domain                            |       1 | <<keystone.domain.root>> | NULL      |         1 |
+----------------------------------+--------------------------+-------+-----------------------------------------------+---------+--------------------------+-----------+-----------+
3 rows in set (0.003 sec)

MariaDB [keystone]> select * from user;
+----------------------------------+-------+---------+--------------------+---------------------+----------------+-----------+
| id                               | extra | enabled | default_project_id | created_at          | last_active_at | domain_id |
+----------------------------------+-------+---------+--------------------+---------------------+----------------+-----------+
| 492f498e5d9941d0817de60f50d4a64b | {}    |       1 | NULL               | 2020-12-18 10:17:35 | NULL           | default   |
+----------------------------------+-------+---------+--------------------+---------------------+----------------+-----------+
1 row in set (0.003 sec)

MariaDB [keystone]> select * from role;
+----------------------------------+--------+-------+-----------+-------------+
| id                               | name   | extra | domain_id | description |
+----------------------------------+--------+-------+-----------+-------------+
| 1f291c3ca3ae457f8c5590e5e8c0439a | reader | {}    | <<null>>  | NULL        |
| 3632943d76e44eb78281a4a785f9d1ae | admin  | {}    | <<null>>  | NULL        |
| dd757bc2ed1b4fd2bca01b389be14cf6 | member | {}    | <<null>>  | NULL        |
+----------------------------------+--------+-------+-----------+-------------+
3 rows in set (0.003 sec)

MariaDB [keystone]> select * from region;
+-----------+-------------+------------------+-------+
| id        | description | parent_region_id | extra |
+-----------+-------------+------------------+-------+
| RegionOne |             | NULL             | {}    |
+-----------+-------------+------------------+-------+
1 row in set (0.003 sec)


# admin为管理用户的密码

  admin:管理网络
    192.168.0.0/16
  internal：内部网络
    10.20.0.0/16
  public：共有网络
    172.16.0.0/16
   
  region：地区、区域
  domain：机房级别
  project：项目
```

配置apache服务器

```bash
# /etc/httpd/conf/httpd.conf
ServerName openstack-vip.magedu.local

# ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

# systemctl enable httpd.service
# systemctl start httpd.service


# 验证
[root@openstack-controller1 ~]# lsof -ni :80
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
httpd   14812   root    4u  IPv6  34490      0t0  TCP *:http (LISTEN)
httpd   14818 apache    4u  IPv6  34490      0t0  TCP *:http (LISTEN)
httpd   14819 apache    4u  IPv6  34490      0t0  TCP *:http (LISTEN)
httpd   14820 apache    4u  IPv6  34490      0t0  TCP *:http (LISTEN)
httpd   14821 apache    4u  IPv6  34490      0t0  TCP *:http (LISTEN)
httpd   14822 apache    4u  IPv6  34490      0t0  TCP *:http (LISTEN)
```



配置变量

```bash
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://openstack-vip.magedu.local:5000/v3
export OS_IDENTITY_API_VERSION=3
```



添加haproxy配置

```bash

listen keystone-api-5000
  bind 172.16.0.248:5000
  mode tcp
  server 172.16.0.101 172.16.0.101:5000  check port 5000 inter 3s fall 2 rise 5


# systemctl restart haproxy keepalived
```

![image-20201218183102306](http://myapp.img.mykernel.cn/image-20201218183102306.png)



身份服务为每个OpenStack服务提供身份验证服务。身份验证服务使用域、项目、用户和角色的组合。

创建新的域

```bash
[root@openstack-controller1 ~]# openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | 974dc857f718456291cfd1e77b2c51e2 |
| name        | example                          |
| options     | {}                               |
| tags        | []                               |
+-------------+----------------------------------+

```

使用以上admin用户来添加一个服务 

```bash
[root@openstack-controller1 ~]# openstack project create --domain default   --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  | 
| domain_id   | default                          | 服务所属域
| enabled     | True                             |
| id          | 510d4911d2574412bb07fb6c79d300dc |
| is_domain   | False                            |
| name        | service                          | 服务
| options     | {}                               |
| parent_id   | default                          | 默认域
| tags        | []                               |
+-------------+----------------------------------+

```

常规(非管理)任务应该使用非特权项目和用户。

例如，本指南创建`myproject`项目和`myuser`用户。

> 您可以重复此过程来创建其他项目和用户。

```bash
# 创建myproject项目
[root@openstack-controller1 ~]# openstack project create --domain default   --description "Demo Project" myproject
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     | 
| domain_id   | default                          | 
| enabled     | True                             |
| id          | 7231f3a71b7e4db190ef5e8632988aca |
| is_domain   | False                            |
| name        | myproject                        | 项目
| options     | {}                               |
| parent_id   | default                          | 机房
| tags        | []                               |
+-------------+----------------------------------+

  region：地区、区域
  domain：机房级别
  project：项目
```

> 在为此项目创建其他用户时，不要重复此步骤

```bash
# 创建myuser用户
[root@openstack-controller1 ~]# openstack user create --domain default   --password-prompt myuser
User Password: myuser
Repeat User Password: myuser
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | fec0dd14c4ba4352aa464e578ae04657 |
| name                | myuser                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

#创建myrole角色
[root@openstack-controller1 ~]# openstack role create myrole
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| domain_id   | None                             |
| id          | be2fec9575ea40f58d8ab41ccaa9a9cf |
| name        | myrole                           |
| options     | {}                               |
+-------------+----------------------------------+

#添加myrole角色到myproject项目和myuser用户
[root@openstack-controller1 ~]# openstack role add --project myproject --user myuser myrole

```



在安装其他服务之前验证身份服务的操作。

> 在控制器节点上执行这些命令。

```bash
# 取消临时设置OS_AUTH_URL和OS_PASSWORD环境变量：
unset OS_AUTH_URL OS_PASSWORD

# admin用户，请求身份验证令牌
[root@openstack-controller1 ~]# openstack --os-auth-url http://openstack-vip.magedu.local:5000/v3   --os-project-domain-name Default --os-user-domain-name Default   --os-project-name admin --os-username admin token issue
Password: 
Password: 
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-12-19T09:11:07+0000                                                                                                                                                                |
| id         | gAAAAABf3bWb9dLS6ADPjDm1_SdzbkI_oudAD077cHl2D3sAjZd2mqvlR8pTFWtgfGUoY4O40bUBLZnFudp30lShXZJnWwmfvAw6RCd3jOJtMvTW819oHwu_vF3bTBh1lmYHCfyzpDR18x_GdmLr1yQCRM6EZsavNUk4MI-TSpPRQvRMhv0rEdU |
| project_id | 5c77b1ffc4124e17be58a9f9944cff56                                                                                                                                                        |
| user_id    | 492f498e5d9941d0817de60f50d4a64b                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

```



创建OpenStack客户端环境脚本



前面的部分使用了环境变量和命令选项的组合，通过OpenStack客户端与标识服务进行交互。为了提高客户端操作的效率，OpenStack支持简单的客户机环境脚本，也称为OpenRC文件。这些脚本通常包含所有客户端的公共选项，但也支持唯一的选项。

```bash
# 创建和编辑admin-openrc文件并添加以下内容
[root@openstack-controller1 ~]# cat admin-openrc 
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
# admin用户的密码
export OS_PASSWORD=admin
# httpd提供的identify服务暴露VIP暴露
export OS_AUTH_URL=http://openstack-vip.magedu.local:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2


# 创建和编辑demo-openrc文件并添加以下内容：
[root@openstack-controller1 ~]# cat demo-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=myuser
export OS_AUTH_URL=http://openstack-vip.magedu.local:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2



#以特定项目和用户的身份运行客户端，只需在运行客户端环境脚本之前加载相关的客户端环境脚本即可
#加载admin-openrc文件来填充环境变量，其中包含标识服务的位置和admin项目和用户凭据
[root@openstack-controller1 ~]# source admin-openrc 
[root@openstack-controller1 ~]# openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-12-19T09:44:19+0000                                                                                                                                                                |
| id         | gAAAAABf3b1jKCx01hDzw7SR7ZUdHpQf2sub5FCsv2Lncx7FG85bi4fUIi1OGgaGK8TmJleNmhJBy4kZUKkEeE-cRqfmiKlUyy3MVZ3j8EmlHSe8rieBBk8ft8T1Y0-oiikFE7ANRC6Qi2idOzNiXB-W0-Y2B5WKrsP6mqqJ7PKh0Bdxv_t96xc |
| project_id | 5c77b1ffc4124e17be58a9f9944cff56                                                                                                                                                        |
| user_id    | 492f498e5d9941d0817de60f50d4a64b                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
[root@openstack-controller1 ~]# source demo-openrc 
[root@openstack-controller1 ~]# openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-12-19T09:44:36+0000                                                                                                                                                                |
| id         | gAAAAABf3b10wK5lW13mx6q_w0PLCc5ltxEsY44Gywg3f5QN5PGdvZVqhYWaLrpesYP-0lcUhVPD8Npb7f5eoJGevSkAXMcrOrTYaNBHr_WPkQk_3TBx-7EgaZ7tnoHauH9MTPjWh0IRX6-P-9rP5V5HlwkC22EQ7Gl8p5BwcpuOMCJdMO4Xr2A |
| project_id | 7231f3a71b7e4db190ef5e8632988aca                                                                                                                                                        |
| user_id    | fec0dd14c4ba4352aa464e578ae04657                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+


```



### 4.7.2 安装glance镜像服务

The Image service (glance) 使用户能够发现、注册和检索虚拟机映像。它提供了一个REST API 使您能够查询虚拟机映像元数据并检索实际映像的API。 镜像在存储在多种存储上：nfs、对象存储ceph、...。



> 为简单，本指南将glance镜像存储使用**file**后端，上传镜像存储在glance服务所在节点的**/var/lib/glance/images/**目录中。

The OpenStack Image service是基础设施即服务(IaaS)的核心。它接受 disk 或 server images 和metadata definitions from end users  或 OpenStack Compute components 的API请求。它还支持在各种存储库类型上存储磁盘或服务映像，包括OpenStack对象存储。

在OpenStackImage服务上运行许多定期进程以支持缓存。复制服务通过集群确保一致性和可用性。其他定期过程包括 auditors, updaters, and reapers.

OpenStack图像服务包括以下组件：

- **glance-api**

  接受用于image 发现、检索和存储的image API调用。

- **glance-registry**

  存储、处理和检索有关image的元数据。元数据包括大小和类型等项。

  > registry是供OpenStack image service使用的私有内部服务。不要向用户公开此服务。
  >
  > The Glance Registry Service及其API已在Queens release中被废弃，并将在“S”开发周期开始时被移除。[OpenStack标准弃用策略](https://governance.openstack.org/reference/tags/assert_follows-standard-deprecation.html).

- **Database**

  存储image元数据，您可以根据喜好选择数据库。大多数部署使用MySQL或SQLite。

- **image文件存储库**

  支持各种存储库类型，包括普通的文件系统(或挂载在the glance-api controller node的任何文件系统)、对象存储、RADOS块设备、VMware数据存储和HTTP。请注意，一些存储库只支持只读使用。

- **元数据定义服务**

  一个通用API，供vendors, admins, services, and users 有意义地定义自己的custom metadata。这种元数据可以用于不同类型的资源，如 images, artifacts, volumes, flavors, and aggregates。定义包括新属性的key, description, constraints,以及可与其关联的  resource types。 



####　在Python 3下运行Glance

您应该始终在OpenStack发行版指定的Python版本下运行Glance 

如果您正在从源代码构建OpenStack，则当前支持Glance在Python 2(特别是Python2.7或更高版本)下运行。

如果希望在Python 3下运行Glance，则需要进行一些部署配置。使用运行Python3.5的单元测试和功能测试测试。但是，Glance运行的基于事件的服务器目前受到一个错误的影响，该错误阻止ssl握手完成(请参阅[Bug#1482633](https://bugs.launchpad.net/glance/+bug/1482633))。因此，如果您希望在Python3.5下运行Glance，则必须以这样的方式部署Glance，即在调用到达Glance之前，SSL终端由类似HAProxy之类的东西处理。



#### 先决条件

本节描述如何在控制器节点上安装和配置image服务，代码名为glance.为了简单起见，此配置将image存储在本地文件系统上。

在安装和配置Image服务之前，必须**创建数据库、服务凭据和API端点** 

```bash
# 使用数据库访问客户端连接到数据库服务器，作为root用户
[root@openstack-mysql-master ~]# mysql -uroot -popenstack123
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 29
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE DATABASE glance;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'glance123';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%'  IDENTIFIED BY 'glance123';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> \q
Bye
##　验证
[root@openstack-mysql-master ~]# mysql -uglance -pglance123 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 40
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> ^DBye



# controller节点 创建服务凭据, admin用户
[root@openstack-controller1 ~]# source admin-openrc 
## 创建glance用户
[root@openstack-controller1 ~]# openstack user create --domain default --password-prompt glance
User Password: glance
Repeat User Password: glance
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 548f489ce792457cbc27629fbbce152c |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
## 添加admin角色到glance用户和service项目：
[root@openstack-controller1 ~]# openstack role add --project service --user glance admin
## 创建glance服务entity：
[root@openstack-controller1 ~]# openstack service create --name glance \
>   --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 5480fb87a69049239a63b139cec9ac8f |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+


# api端点
[root@openstack-controller1 ~]# openstack endpoint create --region RegionOne image public http://openstack-vip.magedu.local:9292
+--------------+----------------------------------------+
| Field        | Value                                  |
+--------------+----------------------------------------+
| enabled      | True                                   |
| id           | be250b328bb34c99af0f5950f84037ad       |
| interface    | public                                 |
| region       | RegionOne                              |
| region_id    | RegionOne                              |
| service_id   | 5480fb87a69049239a63b139cec9ac8f       |
| service_name | glance                                 |
| service_type | image                                  |
| url          | http://openstack-vip.magedu.local:9292 |
+--------------+----------------------------------------+
[root@openstack-controller1 ~]# openstack endpoint create --region RegionOne image internal http://openstack-vip.magedu.local:9292
+--------------+----------------------------------------+
| Field        | Value                                  |
+--------------+----------------------------------------+
| enabled      | True                                   |
| id           | eeb61a9be396441396c64875230a6696       |
| interface    | internal                               |
| region       | RegionOne                              |
| region_id    | RegionOne                              |
| service_id   | 5480fb87a69049239a63b139cec9ac8f       |
| service_name | glance                                 |
| service_type | image                                  |
| url          | http://openstack-vip.magedu.local:9292 |
+--------------+----------------------------------------+
[root@openstack-controller1 ~]# openstack endpoint create --region RegionOne image admin http://openstack-vip.magedu.local:9292
+--------------+----------------------------------------+
| Field        | Value                                  |
+--------------+----------------------------------------+
| enabled      | True                                   |
| id           | 8f046417150f430487acb26946a03722       |
| interface    | admin                                  |
| region       | RegionOne                              |
| region_id    | RegionOne                              |
| service_id   | 5480fb87a69049239a63b139cec9ac8f       |
| service_name | glance                                 |
| service_type | image                                  |
| url          | http://openstack-vip.magedu.local:9292 |
+--------------+----------------------------------------+
[root@openstack-controller1 ~]# 

```

#### 安装配置组件

> 默认配置文件随发行版而异。您可能需要添加这些section和选项，而不是修改现有的section和选项。另外，省略号(`...`)在配置片段中，指示应该保留的潜在默认配置选项。

```bash
# 安装软件包：
[root@openstack-controller1 ~]#  yum install openstack-glance -y


# /etc/glance/glance-api.conf
[database]
connection = mysql+pymysql://glance:glance123@openstack-vip.magedu.local/glance            

[keystone_authtoken]
# ...
# kestone
www_authenticate_uri  = http://openstack-vip.magedu.local:5000
auth_url = http://openstack-vip.magedu.local:5000
# memcached
memcached_servers = openstack-vip.magedu.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
# glance password for mysql
password = glance

[paste_deploy]
# ...
flavor = keystone

[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/


# install -dv -o glance -g glance /var/lib/glance/images/


# 填充image服务数据库：
[root@openstack-controller1 ~]# su -s /bin/sh -c "glance-manage db_sync" glance
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
/usr/lib/python2.7/site-packages/pymysql/cursors.py:170: Warning: (1280, u"Name 'alembic_version_pkc' ignored for PRIMARY key.")
  result = self._query(query)

## mysql验证
[root@openstack-mysql-master ~]# mysql -uglance -pglance123 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 41
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use glance
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [glance]> show tables;
+------------------+
| Tables_in_glance |
+------------------+
| alembic_version  |
| image_locations  |
| image_members    |
| image_properties |
| image_tags       |
| images           |
| migrate_version  |
+------------------+
7 rows in set (0.000 sec)

MariaDB [glance]> select * from migrate_version;
+-------------------+--------------------------------------------------------------------+---------+
| repository_id     | repository_path                                                    | version |
+-------------------+--------------------------------------------------------------------+---------+
| Glance Migrations | /usr/lib/python2.7/site-packages/glance/db/sqlalchemy/migrate_repo |       0 |
+-------------------+--------------------------------------------------------------------+---------+
1 row in set (0.001 sec)


# 启动Image服务并将其配置为在系统启动时启动：
# systemctl enable openstack-glance-api.service
# systemctl start openstack-glance-api.service

## 生成glance管理脚本
[root@openstack-controller1 ~]# cat glance-restart.sh
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-19
#FileName：             glance-restart.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************
systemctl restart openstack-glance-api.service
[root@openstack-controller1 ~]# chmod +x glance-restart.sh


# 当查看到监听9292表示成功
[root@openstack-controller1 ~]# tail -n 20 /var/log/glance/api.log 
2020-12-19 17:42:02.629 3022 WARNING keystonemiddleware.auth_token [-] AuthToken middleware is set with keystone_authtoken.service_token_roles_required set to False. This is backwards compatible but deprecated behaviour. Please set this to True.
2020-12-19 17:42:03.208 3022 WARNING oslo_config.cfg [-] Deprecated: Option "stores" from group "glance_store" is deprecated for removal (
This option is deprecated against new config option
``enabled_backends`` which helps to configure multiple backend stores
of different schemes.

This option is scheduled for removal in the U development
cycle.
).  Its value may be silently ignored in the future.
2020-12-19 17:42:03.210 3022 WARNING oslo_config.cfg [-] Deprecated: Option "default_store" from group "glance_store" is deprecated for removal (
This option is deprecated against new config option
``default_backend`` which acts similar to ``default_store`` config
option.

This option is scheduled for removal in the U development
cycle.
).  Its value may be silently ignored in the future.
2020-12-19 17:42:03.213 3022 INFO glance.common.wsgi [-] Starting 1 workers
2020-12-19 17:42:03.216 3022 INFO glance.common.wsgi [-] Started child 3035
2020-12-19 17:42:03.223 3035 INFO eventlet.wsgi.server [-] (3035) wsgi starting up on http://0.0.0.0:9292

```



haproxy暴露glance api

```bash
listen glance-api-9292
  bind 172.16.0.248:9292
  mode tcp
  server 172.16.0.101 172.16.0.101:9292  check port 9292 inter 3s fall 2 rise 5

[root@openstack-haproxy1 ~]# systemctl restart haproxy
```

![image-20201219174927504](http://myapp.img.mykernel.cn/image-20201219174927504.png)



#### 验证操作

使用[CirrOS](http://launchpad.net/cirros)检验image service，cirros这是一个小型Linux映像，可以帮助您测试OpenStack部署。

有关如何下载和构建映像的详细信息，请参阅[OpenStack虚拟机image指南](https://docs.openstack.org/image-guide/)。有关如何管理image的信息，请参阅[OpenStack终端用户指南](https://docs.openstack.org/user-guide/common/cli-manage-images.html).



下载cirrors

[CirrOS下载页面](http://download.cirros-cloud.net/).

如果您的部署使用QEMU或KVM，我们建议使用qcow2格式的映像。在撰写本文时，最新的 64-bit qcow2 image是 [cirros-0.5.1-x86_64-disk.img](http://download.cirros-cloud.net/0.5.1/cirros-0.5.1-x86_64-disk.img).

```bash
[root@openstack-controller1 ~]# wget http://download.cirros-cloud.net/0.5.1/cirros-0.5.1-x86_64-disk.img
```

> 在控制器节点上执行这些命令。

```bash
[root@openstack-controller1 ~]# glance image-create --name "cirros-0.5.1"   --file cirros-0.5.1-x86_64-disk.img   --disk-format qcow2 --container-format bare   --visibility=public
Error finding address for http://openstack-vip.magedu.local:9292/v2/schemas/image: Unable to establish connection to http://openstack-vip.magedu.local:9292/v2/schemas/image: ('Connection aborted.', BadStatusLine("''",))
# 表示执行SELECT 1报错，haproxy的mysql配置了check, 只要网络不稳定mysql就可能被haproxy下线。所以修正配置
[root@openstack-haproxy1 ~]# cat /etc/haproxy/haproxy.cfg
listen mysql
  bind 172.16.0.248:3306
  mode tcp
  server 172.16.0.103 172.16.0.103:3306  
[root@openstack-haproxy1 ~]# systemctl restart haproxy


[root@openstack-controller1 ~]# glance image-create --name "cirros-0.5.1"   --file cirros-0.5.1-x86_64-disk.img   --disk-format qcow2 --container-format bare   --visibility=public
# --name <NAME> 图像的描述性名称
# --file <FILE> 包含要在创建过程中上载的磁盘映像的本地文件。或者，图像数据可以通过stdin传递到客户端。
# --progress 显示上传进度条。
# --disk-format <DISK_FORMAT> 磁盘有效值的格式： None, ami, ari, aki, vhd, vhdx, vmdk, raw, qcow2, vdi, iso, ploop
# --container-format <CONTAINER_FORMAT> 容器有效值的格式： None, ami, ari, aki, bare, ovf, ova, docker
# --visibility <VISIBILITY> 图像可访问性范围-有效值：public, private, community, shared
# --progress Show upload progress bar.
+------------------+----------------------------------------------------------------------------------+
| Property         | Value                                                                            |
+------------------+----------------------------------------------------------------------------------+
| checksum         | 1d3062cd89af34e419f7100277f38b2b                                                 |
| container_format | bare                                                                             |
| created_at       | 2020-12-19T09:58:05Z                                                             |
| disk_format      | qcow2                                                                            |
| id               | a0b5e9a5-e3ae-44b5-952e-d02cf13bcaa1   OpenStack动态生成ID，因此您将在示例命令输出中看到不同的值。                   
| min_disk         | 0                                                                                |
| min_ram          | 0                                                                                |
| name             | cirros-0.5.1                                                                     |
| os_hash_algo     | sha512                                                                           |
| os_hash_value    | 553d220ed58cfee7dafe003c446a9f197ab5edf8ffc09396c74187cf83873c877e7ae041cb80f3b9 |
|                  | 1489acf687183adcd689b53b38e3ddd22e627e7f98a09c46                                 |
| os_hidden        | False                                                                            |
| owner            | 5c77b1ffc4124e17be58a9f9944cff56                                                 |
| protected        | False                                                                            |
| size             | 16338944                                                                         |
| status           | active                                                                           |
| tags             | []                                                                               |
| updated_at       | 2020-12-19T09:58:09Z                                                             |
| virtual_size     | Not available                                                                    |
| visibility       | public                                                                           |
+------------------+----------------------------------------------------------------------------------+
[root@openstack-controller1 ~]# 
[root@openstack-controller1 ~]# ll /var/lib/glance/images/ -h
总用量 16M
-rw-r----- 1 glance glance 16M 12月 19 17:58 a0b5e9a5-e3ae-44b5-952e-d02cf13bcaa1

```

> 获取有关glance参数，见 `OpenStack User Guide`中的 [Image service (glance) command-line client](https://docs.openstack.org/python-glanceclient/latest/cli/details.html) 。
>
> 有关映像的磁盘和容器格式的信息，请参阅 `OpenStackVirtual Machine Image Guide`中的[Disk and container formats for images](https://docs.openstack.org/image-guide/image-formats.html)。

 



确认上传图像并验证属性：

```bash
[root@openstack-controller1 ~]# glance image-list
+--------------------------------------+--------------+
| ID                                   | Name         |
+--------------------------------------+--------------+
| a0b5e9a5-e3ae-44b5-952e-d02cf13bcaa1 | cirros-0.5.1 |
+--------------------------------------+--------------+
```





### 4.7.3  [placement](https://docs.openstack.org/placement/train/install/)服务

The placement API service是在nova存储库中的14.0.0 Newton release中引入的，并在19.0.0 Stein发行版中提取到了 [placement repository](https://opendev.org/openstack/placement)。这是一个a REST API stack and data model,**用于跟踪资源provider的库存和使用情况**，以及不同资源classes 。例如，资源提供程序可以是计算节点、共享存储池或IP分配池。The placement service跟踪每个provider的库存和使用情况。例如，在计算节点上创建的实例可以是来自计算节点资源提供者的资源(如RAM和CPU)的使用者, 资源也可以是来自外部共享存储池资源提供者的磁盘以及来自外部IP池资源提供者的IP地址。

所使用的资源类型被跟踪为**classes**。服务提供了一组标准资源类(例如`DISK_GB`, `MEMORY_MB`，和`VCPU`)并提供根据需要定义自定义资源类的能力。

每个资源提供者也可能具有描述资源提供者的定性方面的一组特征。特性描述了资源提供者的一个方面。资源提供者本身不能被消耗，但是可能需要指定工作量。例如，可用磁盘可能是固态驱动器(SSD)



#### 部署api



placement提供`placement-api`用于使用Apache、nginx或其他支持WSGI的Web服务器运行服务的WSGI脚本。根据用于部署OpenStack的打包解决方案，WSGI脚本可能位于`/usr/bin`或`/usr/local/bin`.

`placement-api`，作为标准的wsgi脚本，placement-api提供了一个大多数WSGI服务器希望找到的模块级别`application`属性， 这意味着可以在许多不同的服务器上运行placement-api，在不同的部署场景下提供灵活性。常见情况包括：

- [apache2](http://httpd.apache.org/) with [mod_wsgi](https://modwsgi.readthedocs.io/)
- apache2 with [mod_proxy_uwsgi](http://uwsgi-docs.readthedocs.io/en/latest/Apache.html)
- [nginx](http://nginx.org/) with [uwsgi](http://uwsgi-docs.readthedocs.io/en/latest/Nginx.html)
- nginx with [gunicorn](http://gunicorn.org/)

在所有这些场景中，应用程序的主机、端口和挂载路径(或前缀)控制在Web服务器的配置中，而不是在placement-api配置(`placement.conf`)中的申请。

当placement 用`mod_wsgi`风格添加至 [first added to DevStack](https://review.opendev.org/#/c/342362/)，后来它使用[mod_proxy_uwsgi](http://uwsgi-docs.readthedocs.io/en/latest/Apache.html)更新，查看这些更改对于理解相关选项是有用的。

DevStack配置`在http或https的默认端口上(`80`或`443`)的/placement。使用默认端口比较好。

默认情况下，Place应用程序将从`/etc/placement/placement.conf`获取配置如数据库连接。配置文件所在的目录可以在启动应用程序的进程的环境中通过设置`OS_PLACEMENT_CONFIG_DIR`更改。最近发布的`oslo.config`, ，配置选项也可以在环境变量中。

> 当使用带有前端(例如apache2或nginx)的uwsgi时，需要一些东西来确保uwsgi进程正在运行。在DevStack中，这是用[系统d](https://review.opendev.org/#/c/448323/)。这是管理uwsgi的许多不同方法之一。

本文档没有为placement service声明一组安装说明。这是因为拥有WSGI应用程序的主要目的是使部署尽可能灵活。因为 placement API service 本身是无状态的(所有状态都在数据库中)，因此可以部署尽可能多的服务器，并且在负载平衡解决方案的后面，以实现健壮和简单的扩展。如果您熟悉安装通用WSGI应用程序(使用上面常见场景列表中的链接)，这些技术将适用于这里。



#### 同步数据库

placement service使用自己的数据库，定义在[`placement_database`](https://docs.openstack.org/placement/train/configuration/config.html#placement_database)配置部分。这个[`placement_database.connection`](https://docs.openstack.org/placement/train/configuration/config.html#placement_database.connection)选项必须设置，否则服务不会启动。命令行工具[placement-manage](https://docs.openstack.org/placement/train/cli/placement-manage.html) 可用于将数据库表迁移到另一个主机，并转换为它们的正确形式，包括创建它们。`connection`选项所描述的数据库必须已经存在，并定义了适当的访问控制。

另一个同步选项是在配置中设置[`placement_database.sync_on_startup`](https://docs.openstack.org/placement/train/configuration/config.html#placement_database.sync_on_startup)为`True`。这将在placement web service 启动时执行任何缺少的数据库迁移。您是选择自动同步还是使用命令行工具取决于环境和部署工具的限制。

> 在Stein发行版中，placement代码是从nova中提取的。如果要升级到使用提取的placement ，则需要迁徙`nova_api`数据库到`placement`数据库。您可以找到示例脚本，在 [placement repository](https://opendev.org/openstack/placement)中的脚本（[mysql-migrate-db.sh](https://opendev.org/openstack/placement/raw/branch/master/tools/mysql-migrate-db.sh) and [postgresql-migrate-db.sh](https://opendev.org/openstack/placement/raw/branch/master/tools/postgresql-migrate-db.sh).）将帮助你 另见[升级备注](https://docs.openstack.org/placement/train/admin/upgrade-notes.html).



#### 创建帐户并更新服务目录

使用keystone创建拥有admin角色的palcement `service`

 placement API 是一个独立的服务，因此应该注册在**placement** service之下写入 service catalog。placement的客户端,如nova-compute节点中的资源跟踪器，将使用 service catalog查找placement endpoint.。

有关创建服务用户和目录条目的示例，看  [Configure User and Endpoints](https://docs.openstack.org/placement/train/install/from-pypi.html#configure-endpoints-pypi) 

Devstack sets up the placement service 在默认HTTP端口(80)上使用`/placement`前缀，而不是使用独立端口。

#### 安装包

本节提供有关从Linux发行包安装placement的说明。



> 其他一些OpenStack服务(尤其是nova)需要Placement，因此Placement应该安装在其他服务之前，但应该安装在 Identity (keystone)之后.



　

#### centos配置placement

##### 创建数据库、服务凭据和API端点。

```bash
[root@openstack-mysql-master ~]# mysql -uroot -popenstack123
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 65
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE DATABASE placement;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'placement123';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'placement123';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> \q
Bye
[root@openstack-mysql-master ~]# mysql -uplacement -pplacement123
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 66
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> \q
Bye

```

配置用户和endpoint

> controller节点

```bash
[root@openstack-controller1 ~]# . admin-openrc
[root@openstack-controller1 ~]# openstack user create --domain default --password-prompt placement
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | f3aff31674004f0db3aa132744192060 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+



[root@openstack-controller1 ~]# openstack role add --project service --user placement admin


# 创建Placement API服务endpoints：
[root@openstack-controller1 ~]# openstack service create --name placement   --description "Placement API" placement
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | 76f5a29b95e34e4c9bb5f08cf905bada |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+
[root@openstack-controller1 ~]# openstack endpoint create --region RegionOne placement public http://openstack-vip.magedu.local:8778
+--------------+----------------------------------------+
| Field        | Value                                  |
+--------------+----------------------------------------+
| enabled      | True                                   |
| id           | 149fed9b3977438c819f66f0d2cd1827       |
| interface    | public                                 |
| region       | RegionOne                              |
| region_id    | RegionOne                              |
| service_id   | 76f5a29b95e34e4c9bb5f08cf905bada       |
| service_name | placement                              |
| service_type | placement                              |
| url          | http://openstack-vip.magedu.local:8778 |
+--------------+----------------------------------------+
[root@openstack-controller1 ~]# openstack endpoint create --region RegionOne placement internal http://openstack-vip.magedu.local:8778
+--------------+----------------------------------------+
| Field        | Value                                  |
+--------------+----------------------------------------+
| enabled      | True                                   |
| id           | 52d72316b44547f1adc5757cff7a8cfe       |
| interface    | internal                               |
| region       | RegionOne                              |
| region_id    | RegionOne                              |
| service_id   | 76f5a29b95e34e4c9bb5f08cf905bada       |
| service_name | placement                              |
| service_type | placement                              |
| url          | http://openstack-vip.magedu.local:8778 |
+--------------+----------------------------------------+
[root@openstack-controller1 ~]# openstack endpoint create --region RegionOne placement admin http://openstack-vip.magedu.local:8778
+--------------+----------------------------------------+
| Field        | Value                                  |
+--------------+----------------------------------------+
| enabled      | True                                   |
| id           | 1783855afe7b495ea3d0f9b520d1f133       |
| interface    | admin                                  |
| region       | RegionOne                              |
| region_id    | RegionOne                              |
| service_id   | 76f5a29b95e34e4c9bb5f08cf905bada       |
| service_name | placement                              |
| service_type | placement                              |
| url          | http://openstack-vip.magedu.local:8778 |
+--------------+----------------------------------------+

```



haproxy暴露placement api

```bash
[root@openstack-haproxy1 ~]# cat /etc/haproxy/haproxy.cfg 
#...



listen placement-api-8778
  bind 172.16.0.248:8778
  mode tcp
  server 172.16.0.101 172.16.0.101:8778  check port 8778 inter 3s fall 2 rise 5


[root@openstack-haproxy1 ~]# systemctl restart haproxy

```

![image-20201219193220370](http://myapp.img.mykernel.cn/image-20201219193220370.png)

##### 安装和配置组件

```bash
# 安装软件包：
yum install openstack-placement-api -y


# /etc/placement/placement.conf
[placement_database]
# ...
connection = mysql+pymysql://placement:placement123@openstack-vip.magedu.local/placement

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
# keystone
auth_url = http://openstack-vip.magedu.local:5000/v3
# memcached
memcached_servers = openstack-vip.magedu.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
# place user for identify user
password = placement

# 填充placement数据库
# su -s /bin/sh -c "placement-manage db sync" placement


# 验证表已经添加
[root@openstack-mysql-master ~]# mysql -uplacement -pplacement123
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 78
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use placement;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [placement]> show tables;
+------------------------------+
| Tables_in_placement          |
+------------------------------+
| alembic_version              |
| allocations                  |
| consumers                    |
| inventories                  |
| placement_aggregates         |
| projects                     |
| resource_classes             |
| resource_provider_aggregates |
| resource_provider_traits     |
| resource_providers           |
| traits                       |
| users                        |
+------------------------------+
12 rows in set (0.001 sec)

```



httpd读取placement

```bash
[root@openstack-controller1 ~]# rpm -qc openstack-placement-api
/etc/httpd/conf.d/00-placement-api.conf


# systemctl restart httpd


# 准备重启脚本
[root@openstack-controller1 ~]# cat placement-api.sh
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-19
#FileName：             placement-api.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************
 systemctl restart httpd
[root@openstack-controller1 ~]# chmod +x placement-api.sh

```

#### 检验

```bash
[root@openstack-controller1 ~]# . admin-openrc
[root@openstack-controller1 ~]# placement-status upgrade check
+----------------------------------+
| Upgrade Check Results            |
+----------------------------------+
| Check: Missing Root Provider IDs |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
| Check: Incomplete Consumers      |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
```





### 4.7.2 [计算服务 nova compute](https://docs.openstack.org/nova/train/install/)



OpenStack Compute与OpenStack keystone交互以进行身份验证，

OpenStack Placement用于资源库存跟踪和选择，

OpenStack image服务用于磁盘和服务器映像，

OpenStack dashboard用于用户和管理界面。 

OpenStack Compute由以下区域及其组件组成：

- nova-api 接受和响应用户调用，它强制执行一些策略，并启动大多数业务流程活动，例如运行实例。
- **nova-scheduler** 从队列获取虚拟机实例请求，并确定它在哪个计算服务器主机上运行。
- `nova-conductor` module 调解 `nova-compute` service and the database
- `nova-novncproxy` daemon 提供通过VNC访问实例的代理
- mq 在守护进程之间传递消息的中心中心
- sql databases 存储云基础设施的大多数构建时和运行时状态



#### 4.7.2.1 控制器节点上安装和配置Compute服务。



#####  先在Mysql节点上完成相应准备

```bash
[root@openstack-mysql-master ~]# mysql -uroot -popenstack123
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 80
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE DATABASE nova_api;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> CREATE DATABASE nova;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> CREATE DATABASE nova_cell0;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost'   IDENTIFIED BY 'nova123';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%'   IDENTIFIED BY 'nova123';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> 
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost'   IDENTIFIED BY 'nova123';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%'   IDENTIFIED BY 'nova123';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> 
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost'   IDENTIFIED BY 'nova123';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%'   IDENTIFIED BY 'nova123';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> \q
Bye
[root@openstack-mysql-master ~]# mysql -unova -pnova123 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 81
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> ^DBye
[root@openstack-mysql-master ~]# 

```

##### 计算服务凭据

```bash
[root@openstack-controller1 ~]# . admin-openrc 
[root@openstack-controller1 ~]# openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | a982c999cd2d4574a4e5f37751ac29b2 |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
[root@openstack-controller1 ~]# openstack role add --project service --user nova admin
[root@openstack-controller1 ~]# openstack service create --name nova   --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 92aaf19991904b6ebf5812d33cc01367 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
[root@openstack-controller1 ~]# openstack endpoint create --region RegionOne compute public http://openstack-vip.magedu.local:8774/v2.1
+--------------+---------------------------------------------+
| Field        | Value                                       |
+--------------+---------------------------------------------+
| enabled      | True                                        |
| id           | e9abeca08fcc42558f768288eefded1a            |
| interface    | public                                      |
| region       | RegionOne                                   |
| region_id    | RegionOne                                   |
| service_id   | 92aaf19991904b6ebf5812d33cc01367            |
| service_name | nova                                        |
| service_type | compute                                     |
| url          | http://openstack-vip.magedu.local:8774/v2.1 |
+--------------+---------------------------------------------+
[root@openstack-controller1 ~]# openstack endpoint create --region RegionOne compute internal http://openstack-vip.magedu.local:8774/v2.1
+--------------+---------------------------------------------+
| Field        | Value                                       |
+--------------+---------------------------------------------+
| enabled      | True                                        |
| id           | b1b185aa7fda422c89cc38ef2a75087d            |
| interface    | internal                                    |
| region       | RegionOne                                   |
| region_id    | RegionOne                                   |
| service_id   | 92aaf19991904b6ebf5812d33cc01367            |
| service_name | nova                                        |
| service_type | compute                                     |
| url          | http://openstack-vip.magedu.local:8774/v2.1 |
+--------------+---------------------------------------------+
[root@openstack-controller1 ~]# openstack endpoint create --region RegionOne compute admin http://openstack-vip.magedu.local:8774/v2.1
+--------------+---------------------------------------------+
| Field        | Value                                       |
+--------------+---------------------------------------------+
| enabled      | True                                        |
| id           | 8e056835f7ac46239d957f881ad4f769            |
| interface    | admin                                       |
| region       | RegionOne                                   |
| region_id    | RegionOne                                   |
| service_id   | 92aaf19991904b6ebf5812d33cc01367            |
| service_name | nova                                        |
| service_type | compute                                     |
| url          | http://openstack-vip.magedu.local:8774/v2.1 |
+--------------+---------------------------------------------+

```

##### 组件

```bash
yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-novncproxy openstack-nova-scheduler
  
# /etc/nova/nova.conf
[DEFAULT]
# ...
#enable only the compute and metadata APIs
enabled_apis = osapi_compute,metadata

transport_url = rabbit://openstack:openstack123@openstack-vip.magedu.local:5672/

# enable support for the Networking service
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
# ...
connection = mysql+pymysql://nova:nova123@openstack-vip.magedu.local/nova_api



[database]
# ...
connection = mysql+pymysql://nova:nova123@openstack-vip.magedu.local/nova

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://openstack-vip.magedu.local:5000/
auth_url = http://openstack-vip.magedu.local:5000/
memcached_servers = openstack-vip.magedu.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
# user for keystone
password = nova


[vnc]
enabled = true
# ...
server_listen = 172.16.0.101
server_proxyclient_address = 172.16.0.101


[glance]
# ...
api_servers = http://openstack-vip.magedu.local:9292


[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp


[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://openstack-vip.magedu.local:5000/v3
username = placement
# identify or keystone user 
password = placement



#填充nova-api数据库：
# su -s /bin/sh -c "nova-manage api_db sync" nova

#注册cell0数据库：
# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
#创建cell1
# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
#填充nova数据库：
# su -s /bin/sh -c "nova-manage db sync" nova
#验证nova cell 0和cell 1是否正确注册：
[root@openstack-controller1 ~]#  su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
+-------+--------------------------------------+----------------------------------------------------------+-----------------------------------------------------------------+----------+
|  名称 |                 UUID                 |                      Transport URL                       |                            数据库连接                           | Disabled |
+-------+--------------------------------------+----------------------------------------------------------+-----------------------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                          none:/                          | mysql+pymysql://nova:****@openstack-vip.magedu.local/nova_cell0 |  False   |
| cell1 | c4163205-dd39-4a1e-81ed-50fca114ba04 | rabbit://openstack:****@openstack-vip.magedu.local:5672/ |    mysql+pymysql://nova:****@openstack-vip.magedu.local/nova    |  False   |
+-------+--------------------------------------+----------------------------------------------------------+-----------------------------------------------------------------+----------+


```



Compute服务并将其配置为在系统启动时启动：

```bash
# systemctl enable \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
# systemctl start \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
    
    
    
生成 compute nova管理脚本
[root@openstack-controller1 ~]# cat nova-restart.sh
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-19
#FileName：             nova-restart.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************
systemctl restart     openstack-nova-api.service     openstack-nova-scheduler.service     openstack-nova-conductor.service     openstack-nova-novncproxy.service
[root@openstack-controller1 ~]# chmod +x nova-restart.sh

```



haproxy

```bash
listen nova-controller-node-api-8774
  bind 172.16.0.248:8774
  mode tcp
  server 172.16.0.101 172.16.0.101:8774  check port 8774 inter 3s fall 2 rise 5
[root@openstack-haproxy1 ~]# systemctl restart haproxy

```

![image-20201219195711087](http://myapp.img.mykernel.cn/image-20201219195711087.png)

#### 4.7.2.2 计算节点上安装和配置compute服务

该服务支持多个管理程序来部署实例或虚拟机(VM)。默认支持全虚拟化的KVM, 如果不支持全虚拟化就使用qemu。



##### 安装配置组件 

```bash
 # yum install openstack-nova-compute
 
 # /etc/nova/nova.conf
[DEFAULT]
# ...
enabled_apis = osapi_compute,metadata

# rabbitmq
transport_url = rabbit://openstack:openstack123@openstack-vip.magedu.local
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver


[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://openstack-vip.magedu.local:5000/
auth_url = http://openstack-vip.magedu.local:5000/
memcached_servers = openstack-vip.magedu.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
# nova user for keystone
password = nova

[vnc]
# ...
enabled = true
server_listen = 0.0.0.0
# node ip
server_proxyclient_address = 172.16.0.107
novncproxy_base_url = http://openstack-vip.magedu.local:6080/vnc_auto.html

[glance]
# ...
api_servers = http://openstack-vip.magedu.local:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://openstack-vip.magedu.local:5000/v3
username = placement
# placement user for keystone
password = placement




# 添加主机名解析
# /etc/hosts
172.16.0.248 openstack-vip.magedu.local
```

##### 最后安装

```bash
# 确定计算节点是否支持虚拟机的硬件加速：
 egrep -c '(vmx|svm)' /proc/cpuinfo
 1

```

> 如果此命令返回值为`one or greater`，您的计算节点支持硬件加速，这通常不需要额外的配置。
>
> 如果此命令返回值为`zero`，您的计算节点不支持硬件加速，您必须配置`libvirt`使用QEMU而不是KVM。
>
> ```bash
> 
> # /etc/nova/nova.conf
> [libvirt]
> # ...
> virt_type = qemu
> ```



##### 启动Compute服务，包括其依赖项

并将其配置为在系统启动时自动启动：

```bash
# systemctl enable libvirtd.service openstack-nova-compute.service
# systemctl start libvirtd.service openstack-nova-compute.service
```

> 如果`nova-compute`服务启动失败，请检查`/var/log/nova/nova-compute.log`。错误信息`AMQP server on controller:5672is unreachable`可能表示控制器节点上的防火墙正在阻止对端口5672的访问。将防火墙配置为在控制器节点上打开端口5672并重新启动`nova-compute`计算节点上的服务。

准备node脚本

```bash
# cat compute-node.sh 
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-21
#FileName：             compute-node.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************
systemctl restart libvirtd.service openstack-nova-compute.service
```

##### 将计算节点添加到单元数据库中

```bash
# . admin-openrc
#获取管理凭据以启用只管理的CLI命令，然后确认数据库中有计算主机：
# openstack compute service list --service nova-compute
+----+--------------+------------------------------+------+---------+-------+----------------------------+
| ID | Binary       | Host                         | Zone | Status  | State | Updated At                 |
+----+--------------+------------------------------+------+---------+-------+----------------------------+
|  5 | nova-compute | openstack-node1.magedu.local | nova | enabled | up    | 2020-12-21T06:57:17.000000 |
+----+--------------+------------------------------+------+---------+-------+----------------------------+
#发现计算主机：
#  su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova

```

> 添加新的计算节点时，必须运行`nova-manage cell_v2 discover_hosts`在控制器节点上注册这些新的计算节点。或者，您可以在`/etc/nova/nova.conf`:
>
> `/etc/nova/nova.conf`:
>
> ```
> [scheduler]
> discover_hosts_in_cells_interval = 300
> ```

#### 4.7.2.3 验证操作

```bash
[root@openstack-controller1 ~]# . admin-openrc

#验证每个进程成功启动和注册的服务组件
[root@openstack-controller1 ~]# openstack compute service list
+----+----------------+------------------------------------+----------+---------+-------+----------------------------+
| ID | Binary         | Host                               | Zone     | Status  | State | Updated At                 |
+----+----------------+------------------------------------+----------+---------+-------+----------------------------+
|  3 | nova-conductor | openstack-controller1.magedu.local | internal | enabled | up    | 2020-12-21T06:59:38.000000 |
|  4 | nova-scheduler | openstack-controller1.magedu.local | internal | enabled | up    | 2020-12-21T06:59:38.000000 |
|  5 | nova-compute   | openstack-node1.magedu.local       | nova     | enabled | up    | 2020-12-21T06:59:37.000000 |
+----+----------------+------------------------------------+----------+---------+-------+----------------------------+

#列出认证服务中的API端点，以验证与认证服务的连接
[root@openstack-controller1 ~]# openstack catalog list
+-----------+-----------+---------------------------------------------------------+
| Name      | Type      | Endpoints                                               |
+-----------+-----------+---------------------------------------------------------+
| keystone  | identity  | RegionOne                                               |
|           |           |   admin: http://openstack-vip.magedu.local:5000/v3/     |
|           |           | RegionOne                                               |
|           |           |   internal: http://openstack-vip.magedu.local:5000/v3/  |
|           |           | RegionOne                                               |
|           |           |   public: http://openstack-vip.magedu.local:5000/v3/    |
|           |           |                                                         |
| glance    | image     | RegionOne                                               |
|           |           |   admin: http://openstack-vip.magedu.local:9292         |
|           |           | RegionOne                                               |
|           |           |   public: http://openstack-vip.magedu.local:9292        |
|           |           | RegionOne                                               |
|           |           |   internal: http://openstack-vip.magedu.local:9292      |
|           |           |                                                         |
| placement | placement | RegionOne                                               |
|           |           |   public: http://openstack-vip.magedu.local:8778        |
|           |           | RegionOne                                               |
|           |           |   admin: http://openstack-vip.magedu.local:8778         |
|           |           | RegionOne                                               |
|           |           |   internal: http://openstack-vip.magedu.local:8778      |
|           |           |                                                         |
| nova      | compute   | RegionOne                                               |
|           |           |   admin: http://openstack-vip.magedu.local:8774/v2.1    |
|           |           | RegionOne                                               |
|           |           |   internal: http://openstack-vip.magedu.local:8774/v2.1 |
|           |           | RegionOne                                               |
|           |           |   public: http://openstack-vip.magedu.local:8774/v2.1   |
|           |           |                                                         |
+-----------+-----------+---------------------------------------------------------+

#列出image服务中的image,以验证与image服务的连接：
[root@openstack-controller1 ~]#  openstack image list
+--------------------------------------+--------------+--------+
| ID                                   | Name         | Status |
+--------------------------------------+--------------+--------+
| a0b5e9a5-e3ae-44b5-952e-d02cf13bcaa1 | cirros-0.5.1 | active |
+--------------------------------------+--------------+--------+



#检查单元和 placement API是否成功，其他必要的先决条件是否到位：
[root@openstack-controller1 ~]# placement-status upgrade check
+----------------------------------+
| Upgrade Check Results            |
+----------------------------------+
| Check: Missing Root Provider IDs |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
| Check: Incomplete Consumers      |
| Result: Success                  |
| Details: None                    |
+----------------------------------+

[root@openstack-controller1 ~]#  nova-status upgrade check
错误:
Traceback (most recent call last):
  File "/usr/lib/python2.7/site-packages/nova/cmd/status.py", line 398, in main
    ret = fn(*fn_args, **fn_kwargs)
  File "/usr/lib/python2.7/site-packages/oslo_upgradecheck/upgradecheck.py", line 102, in check
    result = func(self)
  File "/usr/lib/python2.7/site-packages/nova/cmd/status.py", line 164, in _check_placement
    versions = self._placement_get("/")
  File "/usr/lib/python2.7/site-packages/nova/cmd/status.py", line 154, in _placement_get
    return client.get(path, raise_exc=True).json()
  File "/usr/lib/python2.7/site-packages/keystoneauth1/adapter.py", line 386, in get
    return self.request(url, 'GET', **kwargs)
  File "/usr/lib/python2.7/site-packages/keystoneauth1/adapter.py", line 248, in request
    return self.session.request(url, method, **kwargs)
  File "/usr/lib/python2.7/site-packages/keystoneauth1/session.py", line 943, in request
    raise exceptions.from_response(resp, method, url)
Forbidden: Forbidden (HTTP 403)

#https://ask.openstack.org/en/question/103325/ah01630-client-denied-by-server-configuration-usrbinnova-placement-api/

[root@openstack-controller1 ~]#  nova-status upgrade check
+--------------------------------+
| Upgrade Check Results          |
+--------------------------------+
| Check: Cells v2                |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Placement API           |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Ironic Flavor Migration |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Cinder API              |
| Result: Success                |
| Details: None                  |
+--------------------------------+
```



### 4.7.3 [网络服务](https://docs.openstack.org/neutron/train/install/)

OpenStack Networking (neutron)允许您创建并附加由其他OpenStack服务管理的接口设备到网络。Plug-ins的实践以适应不同的网络设备和软件，Plug-ins提供了灵活性的OpenStack架构和部署。

- neutron-server 接受API请求并将其路由到适当的OpenStack网络插件以供操作。

- OpenStack Networking plug-ins and agents 插入和拔出端口，创建网络或子网，并提供IP地址。这些插件和代理根据特定云中使用的供应商和技术而有所不同。OpenStack网络为Cisco虚拟和物理交换机、NEC OpenFlow产品、Open vSwitch、Linux桥接和VMware NSX产品提供插件和代理。

  常见的代理是L3(第3层)、DHCP(动态主机IP地址)和一个插件代理

- Messaging queue  大多数OpenStack网络安装用来在 neutron-server和 various agents之间路由信息.还充当一个数据库来存储特定插件的网络状态。

OpenStack网络主要与OpenStack Compute交互，为其实例提供网络和连接。

##### 4.7.3.1 安装和配置控制节点

先决条件

```bash
[root@openstack-mysql-master ~]# mysql -uroot -popenstack123
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 80
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE DATABASE neutron;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost'   IDENTIFIED BY 'neutron123';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%'   IDENTIFIED BY 'neutron123';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> \q
Bye


# 验证
[root@openstack-mysql-master ~]# mysql -uneutron -pneutron123
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 85
Server version: 10.3.20-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> \q
Bye

```

 service credentials

```bash
[root@openstack-controller1 ~]# . admin-openrc

#Create the neutron user:
[root@openstack-controller1 ~]#  openstack user create --domain default --password-prompt neutron
User Password: neutron
Repeat User Password: neutron
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 3e5dbb6a967a4b1b9f120a0a65ec35e4 |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

#Add the admin role to the neutron user:
[root@openstack-controller1 ~]# openstack role add --project service --user neutron admin

#Create the neutron service entity:
[root@openstack-controller1 ~]# openstack service create --name neutron   --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 275438b5ed1f4363959daf2d4069b9ba |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+


# Create the Networking service API endpoints:

[root@openstack-controller1 ~]# openstack endpoint create --region RegionOne network public http://openstack-vip.magedu.local:9696
+--------------+----------------------------------------+
| Field        | Value                                  |
+--------------+----------------------------------------+
| enabled      | True                                   |
| id           | b61a43ee372d436fa88c57f52941e1bf       |
| interface    | public                                 |
| region       | RegionOne                              |
| region_id    | RegionOne                              |
| service_id   | 275438b5ed1f4363959daf2d4069b9ba       |
| service_name | neutron                                |
| service_type | network                                |
| url          | http://openstack-vip.magedu.local:9696 |
+--------------+----------------------------------------+
[root@openstack-controller1 ~]# 
[root@openstack-controller1 ~]# openstack endpoint create --region RegionOne network internal http://openstack-vip.magedu.local:9696
+--------------+----------------------------------------+
| Field        | Value                                  |
+--------------+----------------------------------------+
| enabled      | True                                   |
| id           | b3c2eee85a6748edb0c0c1df3c70ea50       |
| interface    | internal                               |
| region       | RegionOne                              |
| region_id    | RegionOne                              |
| service_id   | 275438b5ed1f4363959daf2d4069b9ba       |
| service_name | neutron                                |
| service_type | network                                |
| url          | http://openstack-vip.magedu.local:9696 |
+--------------+----------------------------------------+
[root@openstack-controller1 ~]# openstack endpoint create --region RegionOne network admin http://openstack-vip.magedu.local:9696
+--------------+----------------------------------------+
| Field        | Value                                  |
+--------------+----------------------------------------+
| enabled      | True                                   |
| id           | 417de8173cd84b37af7ffea198fe423b       |
| interface    | admin                                  |
| region       | RegionOne                              |
| region_id    | RegionOne                              |
| service_id   | 275438b5ed1f4363959daf2d4069b9ba       |
| service_name | neutron                                |
| service_type | network                                |
| url          | http://openstack-vip.magedu.local:9696 |
+--------------+----------------------------------------+

```

##### 配置网络选项

部署最简单的架构，只支持将实例附加到提供者(外部)网络。没有自助服务(专用)网络、路由器或浮动IP地址.只有`admin`或者其他特权用户可以管理提供者网络。

- [Networking Option 1: Provider networks](https://docs.openstack.org/neutron/train/install/controller-install-option1-obs.html)

```bash
# yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables
  
  
#包括数据库、身份验证机制、消息队列、拓扑更改通知和插件。
# /etc/neutron/neutron.conf
[DEFAULT]
#  enable the Modular Layer 2 (ML2) plug-in and disable additional plug-ins:
core_plugin = ml2
service_plugins =

transport_url = rabbit://openstack:openstack123@openstack-vip.magedu.local:5672
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[database]
# ...
connection = mysql+pymysql://neutron:neutron123@openstack-vip.magedu.local/neutron


[keystone_authtoken]
# ...
www_authenticate_uri = http://openstack-vip.magedu.local:5000
auth_url = http://openstack-vip.magedu.local:5000
memcached_servers = openstack-vip.magedu.local:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
# neutron user for keystone
password = neutron


[nova]
# ...
auth_url = http://openstack-vip.magedu.local:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
# nova user of keystone
password = nova


[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp



# /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
# enable flat and VLAN networks:
type_drivers = flat,vlan
# disable self-service networks:
tenant_network_types =
# , enable the Linux bridge mechanism:
mechanism_drivers = linuxbridge
# enable the port security extension driver:
extension_drivers = port_security



[ml2_type_flat]
# configure the provider virtual network as a flat network:
flat_networks = provider


[securitygroup]
# enable ipset to increase efficiency of security group rules:
enable_ipset = true




# /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
#map the provider virtual network to the provider physical network interface:
physical_interface_mappings = provider:eth0
[vxlan]
# disable VXLAN overlay networks:
enable_vxlan = false
[securitygroup]
# enable security groups and configure the Linux bridge iptables firewall driver:
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

```

> securitygroup 安全组打开
>
> physical_interface_mappings = provider:eth0 配置当前主机的物理网卡
>
> **必须配置消息队列**

```bash
[root@openstack-controller1 ~]# cat /etc/sysctl.d/openstack.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
[root@openstack-controller1 ~]# modprobe br_netfilter
[root@openstack-controller1 ~]# sysctl -p /etc/sysctl.d/openstack.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```



##### 配置DHCP代理

```bash
# /etc/neutron/dhcp_agent.ini
[DEFAULT]
# ...
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

##### 配置元数据代理

```bash
# /etc/neutron/metadata_agent.ini
[DEFAULT]
# ...
nova_metadata_host = openstack-vip.magedu.local
metadata_proxy_shared_secret = 2GosJ41hQbkcku8
```

##### 配置计算服务以使用网络服务

```bash
# /etc/nova/nova.conf
[neutron]
# ...
auth_url = http://openstack-vip.magedu.local:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
# neutron user of keystone
password = neutron
service_metadata_proxy = true
metadata_proxy_shared_secret = 2GosJ41hQbkcku8
```

> 和元数据代理的secret一样

##### 最后安装

```bash
[root@openstack-controller1 ~]# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
[root@openstack-controller1 ~]#  su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

# 重新启动ComputeAPI服务：
# systemctl restart openstack-nova-api.service
```

启动网络服务，并将其配置为在系统启动时启动

```bash
# systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
# systemctl start neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
```

生成neutron脚本

```bash
[root@openstack-controller1 ~]# cat neutron-restart.sh
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-21
#FileName：             neutron-restart.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************
systemctl restart openstack-nova-api.service
systemctl restart neutron-server.service  neutron-linuxbridge-agent.service neutron-dhcp-agent.service  neutron-metadata-agent.service
[root@openstack-controller1 ~]# chmod +x neutron-restart.sh
```

##### 4.7.3.2 安装和配置计算节点

```bash
# yum install openstack-neutron-linuxbridge ebtables ipset -y





# /etc/neutron/neutron.conf
[DEFAULT]
# rabbitmq
transport_url = rabbit://openstack:openstack123@openstack-vip.magedu.local
auth_strategy = keystone

[keystone_authtoken]
# keystone
www_authenticate_uri = http://openstack-vip.magedu.local:5000
auth_url = http://openstack-vip.magedu.local:5000
memcached_servers = openstack-vip.magedu.local:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
# neutron user of identify
password = neutron

[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp


# /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
# map the provider virtual network to the provider physical network interface:
physical_interface_mappings = provider:eth0
[vxlan]
# disable VXLAN overlay networks
enable_vxlan = false
[securitygroup]
# enable security groups and configure the Linux bridge iptables firewall driver:
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver


```



```bash
#  enable networking bridge support

[root@openstack-controller1 ~]# cat /etc/sysctl.d/openstack.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
[root@openstack-controller1 ~]# modprobe br_netfilter
[root@openstack-controller1 ~]# sysctl -p /etc/sysctl.d/openstack.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```



```bash
# configure access parameters
# /etc/nova/nova.conf
[neutron]
# ...
auth_url = http://openstack-vip.magedu.local:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron

```

```bash
# Restart the Compute service:
# systemctl restart openstack-nova-compute.service


# 启动Linux桥代理并将其配置为在系统启动时启动
# systemctl enable neutron-linuxbridge-agent.service
# systemctl start neutron-linuxbridge-agent.service

```

生成脚本

```bash
[root@openstack-node1 ~]# cat neutron-node-restart.sh 
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-21
#FileName：             neutron-node-restart.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************
systemctl restart openstack-nova-compute.service
systemctl restart neutron-linuxbridge-agent.service
```

##### 4.7.3.3 检验操作

```bash
[root@openstack-controller1 ~]# . admin-openrc 

# 列表加载的扩展以验证neutron-server程序：
[root@openstack-controller1 ~]# openstack extension list --network
+----------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Name                                                                                                                                                           | Alias                          | Description                                                                                                                                              |
+----------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Subnet Pool Prefix Operations                                                                                                                                  | subnetpool-prefix-ops          | Provides support for adjusting the prefix list of subnet pools                                                                                           |
| Default Subnetpools                                                                                                                                            | default-subnetpools            | Provides ability to mark and use a subnetpool as the default.                                                                                            |
| Network IP Availability                                                                                                                                        | network-ip-availability        | Provides IP availability data for each network and subnet.                                                                                               |
| Network Availability Zone                                                                                                                                      | network_availability_zone      | Availability zone support for network.                                                                                                                   |
| Subnet Onboard                                                                                                                                                 | subnet_onboard                 | Provides support for onboarding subnets into subnet pools                                                                                                |
| Network MTU (writable)                                                                                                                                         | net-mtu-writable               | Provides a writable MTU attribute for a network resource.       

# 验证提供式网络
# List agents to verify successful launch of the neutron agents:
[root@openstack-controller1 ~]# openstack network agent list
+--------------------------------------+--------------------+------------------------------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host                               | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------------------------------+-------------------+-------+-------+---------------------------+
| 67495ad5-567b-48ad-bb0b-159d89cbffce | Metadata agent     | openstack-controller1.magedu.local | None              | :-)   | UP    | neutron-metadata-agent    |
| b0617dab-86f3-4c80-acdf-0a4d7dd64f6f | Linux bridge agent | openstack-node1.magedu.local       | None              | :-)   | UP    | neutron-linuxbridge-agent |
| d5257b56-0091-4021-8f7e-d70ecdcae26a | Linux bridge agent | openstack-controller1.magedu.local | None              | :-)   | UP    | neutron-linuxbridge-agent |
+--------------------------------------+--------------------+------------------------------------+-------------------+-------+-------+---------------------------+

```





### 4.7.4 安装dashboard

控制节点安装

```bash
yum install openstack-dashboard -y
/etc/openstack-dashboard/local_settings

OPENSTACK_HOST = "openstack-vip.magedu.local"
ALLOWED_HOSTS = ['one.example.com', 'two.example.com']


SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'openstack-vip.magedu.local:11211',
    }
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}

TIME_ZONE = "Asia/Shanghai"


```

> 允许的主机也可以是[‘*’]来接受所有主机。这可能对开发工作有用，但可能不安全，不应用于生产。看见Https://docs.djangoproject.com/en/dev/ref/settings/#allowed-hosts以获取更多信息。



```bash
# /etc/httpd/conf.d/openstack-dashboard.conf
WSGIApplicationGroup %{GLOBAL}

WSGIScriptAlias /dashboard /usr/share/openstack-dashboard/openstack_dashboard/wsgi/django.wsgi
Alias /dashboard/static /usr/share/openstack-dashboard/static
Alias /static /usr/share/openstack-dashboard/static
```



```bash
# controller
systemctl restart httpd.service 

# mysql单机 
systemctl restart memcached.service
```

haproxy反代

```bash
listen openstack-80
  bind 172.16.0.248:80
  mode http
  log global
  server 172.16.0.101 172.16.0.101:80 check port 9696 inter 3s fall 2 rise 5
  
# systemctl restart haproxy
```



windows添加解析

```bash
172.16.0.248 openstack-vip.magedu.com
```



访问http://openstack-vip.magedu.com/dashboard/auth/login/

![image-20201221175353346](http://myapp.img.mykernel.cn/image-20201221175353346.png)

>  default域，admin, admin

![image-20201221175435148](http://myapp.img.mykernel.cn/image-20201221175435148.png)

登陆后，访问http://openstack-vip.magedu.com/dashboard





#  4.8 运行一个实例

http://blog.mykernel.cn/2020/12/25/openstack%E9%9B%86%E7%BE%A4%E6%93%8D%E4%BD%9C%E4%B9%8B%E8%BF%90%E8%A1%8C%E4%B8%80%E4%B8%AA%E7%A4%BA%E4%BE%8B%E5%AE%9E%E4%BE%8B/