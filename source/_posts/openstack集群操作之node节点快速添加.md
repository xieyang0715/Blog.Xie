---
title: openstack集群操作之node节点快速添加
date: 2020-12-28 02:27:23
tags:
toc: true
---



当后期添加新物理服务器作为计算 节点 如果按照 上面的过程 安装 配置 的 话 会 非常的慢， 但
是 可以 通过复制 配置文件的方式快速添加。

- [x] 安装组件
- [x] 复制配置
- [x] 生成脚本
- [x] 启动服务



# 1. 基于模板克隆node2

```bash
cd /etc/sysconfig/network-scripts/
# ifcfg-eth0
IPADDR=172.16.0.108
# ifcfg-eth1
IPADDR=10.0.0.108

# /etc/hostname
openstack-node2.magedu.local
```



# 2. 安装配置node2



## 2.1 nova compute

```bash
# 需要源
yum install centos-release-openstack-train -y
yum install https://rdoproject.org/repos/rdo-release.rpm -y
yum install openstack-nova-compute -y



# node1 107
# scp  /etc/hosts /etc/nova/nova.conf compute-node.sh 172.16.0.108:
\cp nova.conf /etc/nova/nova.conf 
\cp hosts /etc/hosts


systemctl enable libvirtd.service openstack-nova-compute.service --now 



chmod +x compute-node.sh


tail -f /var/log/nova/nova-compute.log   #通过查看日志，判断 nova compute 是否启动成功
```

验证，controller上

```bash
[root@openstack-controller2 ~]# source admin-openrc 
[root@openstack-controller2 ~]# openstack compute service list
+----+----------------+------------------------------------+----------+---------+-------+----------------------------+
| ID | Binary         | Host                               | Zone     | Status  | State | Updated At                 |
+----+----------------+------------------------------------+----------+---------+-------+----------------------------+
|  1 | nova-conductor | openstack-controller1.magedu.local | internal | enabled | down  | 2020-12-28T02:45:07.000000 |
|  4 | nova-scheduler | openstack-controller1.magedu.local | internal | enabled | down  | 2020-12-28T02:45:03.000000 |
|  5 | nova-compute   | openstack-node1.magedu.local       | nova     | enabled | up    | 2020-12-28T02:48:28.000000 |
|  6 | nova-conductor | openstack-controller2.magedu.local | internal | enabled | up    | 2020-12-28T02:48:28.000000 |
|  8 | nova-scheduler | openstack-controller2.magedu.local | internal | enabled | down  | 2020-12-28T02:29:15.000000 |
| 12 | nova-compute   | openstack-node2.magedu.local       | nova     | enabled | up    | 2020-12-28T02:48:28.000000 | # 表示计算节点已经添加
+----+----------------+------------------------------------+----------+---------+-------+----------------------------+
```



## 2.2 neutron compute

```bash
yum install openstack-neutron-linuxbridge ebtables ipset -y

# node1 107
#scp /etc/sysctl.d/openstack.conf /etc/neutron/neutron.conf /etc/neutron/plugins/ml2/linuxbridge_agent.ini neutron-node-restart.sh compute-node.sh 172.16.0.108:
\cp neutron.conf /etc/neutron/neutron.conf
\cp linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini
\cp openstack.conf /etc/sysctl.d/openstack.conf
systemctl enable neutron-linuxbridge-agent.service --now

sysctl -p /etc/sysctl.d/openstack.conf

chmod +x neutron-node-restart.sh

tail  /var/log/neutron/*.log /var/log/nova/*.log
```

验证，controller上

```bash
[root@openstack-controller2 ~]# openstack network agent list
+--------------------------------------+--------------------+------------------------------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host                               | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------------------------------+-------------------+-------+-------+---------------------------+
| 09c768f6-645b-4096-b09b-ea2b661ff615 | Linux bridge agent | openstack-node2.magedu.local       | None              | :-)   | UP    | neutron-linuxbridge-agent | # 表示成功
| 46938d8e-0c87-4e6d-a03c-a2c83566f988 | Linux bridge agent | openstack-controller1.magedu.local | None              | XXX   | UP    | neutron-linuxbridge-agent |
| 6fb6df01-3f24-4956-9433-8f05b09af4d3 | Metadata agent     | openstack-controller1.magedu.local | None              | XXX   | UP    | neutron-metadata-agent    |
| 86281eab-e064-4892-b66d-db97f5467a96 | Metadata agent     | openstack-controller2.magedu.local | None              | :-)   | UP    | neutron-metadata-agent    |
| a88b7976-0ad5-4a55-94e0-ce53dd7ea0f7 | Linux bridge agent | openstack-node1.magedu.local       | None              | :-)   | UP    | neutron-linuxbridge-agent |
| bd4ffda5-355e-4de2-94da-dcddcb9c8843 | Linux bridge agent | openstack-controller2.magedu.local | None              | :-)   | UP    | neutron-linuxbridge-agent |
| ec77cd7d-0a2d-45c9-9d9c-bbedbb13fb12 | DHCP agent         | openstack-controller1.magedu.local | nova              | XXX   | UP    | neutron-dhcp-agent        |
| fdbd12b5-630b-44e7-ad21-b824aa0b8d8d | DHCP agent         | openstack-controller2.magedu.local | nova              | :-)   | UP    | neutron-dhcp-agent        |
+--------------------------------------+--------------------+------------------------------------+-------------------+-------+-------+---------------------------+

```



## 2.3 dashboard新建虚拟机

![image-20201228135402277](http://myapp.img.mykernel.cn/image-20201228135402277.png)

![image-20201228135448403](http://myapp.img.mykernel.cn/image-20201228135448403.png)![image-20201228135506968](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20201228135506968.png)



查看运行位置

![image-20201228135908469](http://myapp.img.mykernel.cn/image-20201228135908469.png)

# 3. 脚本



- [x] 将修改的配置文件提前准备好
- [x] 将执行结果修改一下`history | cut -c 8-`
- [x] 优化脚本, echo一些人性化的输出
- [x] 添加内核参数优化，资源限制优化



```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-28
#FileName：             install-compute-node.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************

# nova
yum install centos-release-openstack-train -y
yum install https://rdoproject.org/repos/rdo-release.rpm -y
yum install openstack-nova-compute -y

IPADDR=$(ifconfig eth0 | grep -Po "(?<=inet )[\d.]+")
sed -i "s@\(server_proxyclient_address =\).*@\1 ${IPADDR}@g" nova.conf
\cp nova.conf /etc/nova/nova.conf 
\cp hosts /etc/hosts
egrep -c '(vmx|svm)' /proc/cpuinfo
systemctl enable libvirtd.service openstack-nova-compute.service --now 
chmod +x compute-node.sh
tail  /var/log/nova/nova-compute.log

# neutron
yum install openstack-neutron-linuxbridge ebtables ipset -y
\cp neutron.conf /etc/neutron/neutron.conf
\cp linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini
\cp openstack.conf /etc/sysctl.d/openstack.conf
systemctl enable neutron-linuxbridge-agent.service --now
sysctl -p /etc/sysctl.d/openstack.conf
chmod +x neutron-node-restart.sh
tail  /var/log/neutron/*.log /var/log/nova/*.log


#
cp sysctl.conf /etc/sysctl.conf
cp limits.conf /etc/security/limits.conf
# 
reboot
```

> 注意copy的文件全部来自于其他node节点, 并打包成一个压缩文件install-compute-node.tar.xz, 新节点添加直接解压执行





一键安装node3 109

```bash
# tar xvf install-compute-node.tar.xz 
# bash -x install-compute-node.sh 
```



验证

```bash
openstack compute service list
#| 14 | nova-compute   | openstack-node3.magedu.local       | nova     | enabled | up    | 2020-12-28T07:28:35.000000 |


openstack network agent list
#| 9ba30797-45a3-4ee3-bb97-55fd0e914067 | Linux bridge agent | openstack-node3.magedu.local       | None              | :-)   | UP    | neutron-linuxbridge-agent |
```



新建虚拟机验证可以调度并运行成功

![image-20201228153148109](http://myapp.img.mykernel.cn/image-20201228153148109.png)





 ```bash
# 验证机器运行
# virsh list
 Id    Name                           State
----------------------------------------------------
 1     instance-0000000d              running
 2     instance-0000000e              running

[root@openstack-node3 ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
brq96a81a85-85		8000.000c29ee9ebf	no		eth0         # 桥已经添加了eth0
												tap4c8d5185-92 # 一个虚拟机对应一对儿tap
												tap4d34c407-c8

# xshell连接验证到虚拟机网络已经通了

 ```



vnc通了

![image-20201228160824827](http://myapp.img.mykernel.cn/image-20201228160824827.png)



反复删除node3重新添加

```bash
nova service-list
nova service-delete


openstack network agent list
openstack network agent delete
```









