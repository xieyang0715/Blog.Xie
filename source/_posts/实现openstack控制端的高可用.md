---
title: 实现openstack控制端的高可用
date: 2020-12-28 00:48:09
tags: plan
toc: true
---



nova/neutron controller 仅提供虚拟机管理，他们宕机完全不影响虚拟机运行。在宕机后需要快速恢复controller，可以起一个相同controller实例，平时关机，在其中一个controller宕机时，才开机。



# 1. 基于模板克隆controller2

```bash
cd /etc/sysconfig/network-scripts/
# ifcfg-eth0
IPADDR=172.16.0.102
# ifcfg-eth1
IPADDR=10.0.0.102

# /etc/hostname
openstack-controller2.magedu.local
```





# 2. 安装配置controller

由于和controller1一样，所以快速配置的方式

- [x] 不准备数据库
- [x] 安装组件
- [x] 复制配置
- [x] 准备脚本
- [x] 启动服务
- [x] 验证controller



## 2.1 keystone

```bash

# openstack版本为Train
yum -y  install centos-release-openstack-train
# openstack源
yum -y install https://rdoproject.org/repos/rdo-release.rpm
# controller需要连接mysql, memcached, openstack
yum install -y python-openstackclient openstack-selinux python-memcached python2-PyMySQL 
# 在mysql安装成功，授权后需要验证mysql连接
# telnet验证mysql/memcached/openstack可以通。VIP及域名
yum install -y mariadb telnet 

# 101
# scp -rp /etc/keystone/keystone.conf /etc/httpd/conf/httpd.conf  /etc/keystone/credential-keys/ /etc/keystone/fernet-keys/ 172.16.0.102:
\cp -a credential-keys/ fernet-keys/  keystone.conf  /etc/keystone/
chown -R keystone.keystone /etc/keystone/
\cp httpd.conf /etc/httpd/conf/httpd.conf

# 102
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
systemctl enable httpd.service --now

# curl localhost:5000

cat >  admin-openrc <<EOF
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
EOF

tail /var/log/keystone/*.log
```

验证，haproxy keystone指向102

在102 controller上

```bash
# 测试admin用户
source admin-openrc 
openstack token issue
```

成功后，haproxy keystone指向101，102

## 2.2 glance

```bash
# 安装组件
yum install openstack-glance -y


# 101
#scp /etc/glance/glance-api.conf 172.16.0.102:
\cp glance-api.conf /etc/glance/glance-api.conf

install -dv -o glance -g glance /var/lib/glance/images/
showmount -e 172.16.0.248
mount -t nfs  172.16.0.248:/data/glance /var/lib/glance/images/
cat >> /etc/fstab <<EOF
172.16.0.248:/data/glance /var/lib/glance/images/  nfs defaults        0 0
EOF

systemctl enable openstack-glance-api.service
systemctl restart openstack-glance-api.service

#9292
cat > glance-restart.sh<<EOF
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
EOF
chmod +x glance-restart.sh

tail /var/log/glance/*.log 
```

验证，haproxy glance指向102

在102 controller上

```bash
. admin-openrc
# 下载镜像
wget http://download.cirros-cloud.net/0.5.0/cirros-0.5.0-x86_64-disk.img

# 上传镜像
glance image-create --name "cirros-0.5.0"   --file cirros-0.5.0-x86_64-disk.img   --disk-format qcow2 --container-format bare   --visibility=public

# nfs服务中查看镜像是否存在
ll /var/lib/glance/images/ -h

# 获取镜像列表
glance image-list
openstack image list
```

成功后，haproxy glance指向101，102

## 2.3 placement

```bash
yum install openstack-placement-api -y



# 101
#scp /etc/placement/placement.conf 172.16.0.102:
#scp /etc/httpd/conf.d/00-placement-api.conf 172.16.0.102:

\cp placement.conf /etc/placement/placement.conf
\cp 00-placement-api.conf /etc/httpd/conf.d/00-placement-api.conf 



systemctl restart httpd
cat > placement-api.sh <<EOF
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
EOF
chmod +x placement-api.sh

tail /var/log/httpd/*.log

#8778
#curl localhost:8778


```



验证，haproxy placement指向102

在102 controller上

```bash
. admin-openrc
placement-status upgrade check
```

成功后，haproxy placement指向101，102



## 2.4 nova controller

```bash
yum install -y openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler


# 101
#scp  /etc/nova/nova.conf 172.16.0.102: 
\cp nova.conf  /etc/nova/nova.conf


systemctl enable \
openstack-nova-api.service \
openstack-nova-scheduler.service \
openstack-nova-conductor.service \
openstack-nova-novncproxy.service --now 


cat > nova-restart.sh <<EOF
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
EOF
chmod +x nova-restart.sh
tail /var/log/nova/nova-*.log
```





验证，haproxy nova指向102 8774 8775 6080

在102 controller上

```bash
[root@openstack-controller2 ~]# nova service-list
+--------------------------------------+----------------+------------------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
| Id                                   | Binary         | Host                               | Zone     | Status  | State | Updated_at                 | Disabled Reason | Forced down |
+--------------------------------------+----------------+------------------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
| 938cb60e-e8f3-4344-845d-bd8db75ce072 | nova-conductor | openstack-controller1.magedu.local | internal | enabled | up    | 2020-12-28T01:47:42.000000 | -               | False       |
| 846645c6-ea41-4c76-873d-1da852c629ca | nova-scheduler | openstack-controller1.magedu.local | internal | enabled | up    | 2020-12-28T01:47:45.000000 | -               | False       |
| 06d40d83-2768-4646-bafb-0839b7cfe93f | nova-compute   | openstack-node1.magedu.local       | nova     | enabled | up    | 2020-12-28T01:47:48.000000 | -               | False       |
| 4e17adca-c9f5-4ea2-b035-7e68d40f3b31 | nova-conductor | openstack-controller2.magedu.local | internal | enabled | up    | 2020-12-28T01:47:48.000000 | -               | False       |
| 8c626674-ed0a-48df-8f57-b7376421e0a7 | nova-scheduler | openstack-controller2.magedu.local | internal | enabled | up    | 2020-12-28T01:47:44.000000 | -               | False       |
+--------------------------------------+----------------+------------------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
# 出现2个controller2
```

成功后，haproxy placement指向101，102



## 2.5 neutron controller

```bash

yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y

#101
#scp /etc/neutron/neutron.conf  /etc/neutron/plugins/ml2/ml2_conf.ini  /etc/neutron/plugins/ml2/linuxbridge_agent.ini  /etc/sysctl.d/openstack.conf  /etc/neutron/dhcp_agent.ini  /etc/neutron/metadata_agent.ini   /etc/nova/nova.conf 172.16.0.102: 
# 配置neutron
\cp  neutron.conf /etc/neutron/neutron.conf
#配置二层插件，不能有中文
\cp  ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini
#定义桥接网络与指定的物理网络做关联
\cp linuxbridge_agent.ini  /etc/neutron/plugins/ml2/linuxbridge_agent.ini
\cp openstack.conf /etc/sysctl.d/openstack.conf
#配置DHCP，让虚拟机通过DHCP获取IP地址
\cp dhcp_agent.ini /etc/neutron/dhcp_agent.ini
#配置桥接与自服务网络的通用配置
\cp metadata_agent.ini /etc/neutron/metadata_agent.ini
#配置 nova 配置文件
\cp  nova.conf /etc/nova/nova.conf

#网络服务初始化脚本, 需要/etc/neutron/plugin.ini指向ML2插件配置文件的符号链接
ln -sv /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini


# 修改了nova配置，所以重启
systemctl restart openstack-nova-api.service

# 启动neutron
systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service --now 

# 加载内核参数
sysctl -p /etc/sysctl.d/openstack.conf

 cat > neutron-restart.sh <<EOF
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
systemctl restart neutron-server.service  neutron-linuxbridge-agent.service neutron-dhcp-agent.service  neutron-metadata-agent.service
EOF
chmod +x neutron-restart.sh


#neutron日志中不能出现任何报错，否则服务无法启动
tail   /var/log/neutron/*.log   
```



验证，haproxy neutron指向102

在102 controller上

```bash
[root@openstack-controller2 ~]# neutron agent-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+--------------------+------------------------------------+-------------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host                               | availability_zone | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+------------------------------------+-------------------+-------+----------------+---------------------------+
| 46938d8e-0c87-4e6d-a03c-a2c83566f988 | Linux bridge agent | openstack-controller1.magedu.local |                   | :-)   | True           | neutron-linuxbridge-agent |
| 6fb6df01-3f24-4956-9433-8f05b09af4d3 | Metadata agent     | openstack-controller1.magedu.local |                   | :-)   | True           | neutron-metadata-agent    |
| 86281eab-e064-4892-b66d-db97f5467a96 | Metadata agent     | openstack-controller2.magedu.local |                   | :-)   | True           | neutron-metadata-agent    |
| a88b7976-0ad5-4a55-94e0-ce53dd7ea0f7 | Linux bridge agent | openstack-node1.magedu.local       |                   | :-)   | True           | neutron-linuxbridge-agent |
| bd4ffda5-355e-4de2-94da-dcddcb9c8843 | Linux bridge agent | openstack-controller2.magedu.local |                   | :-)   | True           | neutron-linuxbridge-agent |
| ec77cd7d-0a2d-45c9-9d9c-bbedbb13fb12 | DHCP agent         | openstack-controller1.magedu.local | nova              | :-)   | True           | neutron-dhcp-agent        |
| fdbd12b5-630b-44e7-ad21-b824aa0b8d8d | DHCP agent         | openstack-controller2.magedu.local | nova              | :-)   | True           | neutron-dhcp-agent        |
+--------------------------------------+--------------------+------------------------------------+-------------------+-------+----------------+---------------------------+
# 出现3个controller2
```

成功后，haproxy neutron指向101，102





## 2.6 dashboard

```bash
yum install openstack-dashboard -y
#  scp /etc/httpd/conf.d/openstack-dashboard.conf /etc/openstack-dashboard/local_settings 172.16.0.102:

\cp local_settings /etc/openstack-dashboard/local_settings 
\cp openstack-dashboard.conf /etc/httpd/conf.d/openstack-dashboard.conf 


systemctl restart httpd
```



验证，haproxy dashboard指向102, nova/neutron全部指向102

在dashboard上创建虚拟主机

 ![image-20201228101918537](http://myapp.img.mykernel.cn/image-20201228101918537.png)

![image-20201228101956753](http://myapp.img.mykernel.cn/image-20201228101956753.png)

![image-20201228102026817](http://myapp.img.mykernel.cn/image-20201228102026817.png)

成功后，haproxy nova/neutron/dashboard指向101，102

