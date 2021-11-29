---
title: openstack搭建环境之compute组件安装及配置
date: 2020-12-25 09:34:17
tags:
toc: true
---





# 1. nova compute

在计算节点服务器部署

```bash
# 需要源
yum install centos-release-openstack-train -y
yum install https://rdoproject.org/repos/rdo-release.rpm -y
yum install openstack-nova-compute -y


# 配置文件不能有中文
cat > /etc/nova/nova.conf <<EOF
[DEFAULT]
#支持的api类型；开启之后，就可以通过dashboard或命令行访问
enabled_apis = osapi_compute,metadata 

# 需要telnet验证
transport_url = rabbit://openstack:openstack123@openstack-vip.magedu.local
use_neutron = true #通过neutron获取IP地址
firewall_driver = nova.virt.firewall.NoopFirewallDriver
#通过此驱动程序与neutron进行交互，其实就是一个python文件；/usr/lib/python2.7/site-packages/nova/virt/firewall.py 


[api]
auth_strategy = keystone #nova认证方式采用keystone认证
[api_database]
[barbican]
[cache]
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[database]
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
# 需要telnet验证
api_servers = http://openstack-vip.magedu.local:9292  #nova compute从指定的glance的api获取镜像
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]  #配置keystone的认证信息
# 需要telnet验证
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
[libvirt]
[metrics]
[mks]
[neutron]
# 需要telnet验证
auth_url = http://openstack-vip.magedu.local:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[pci]
[placement] #node节点需要向placement汇到当前node节点的可用资源
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
# 需要telnet验证
auth_url = http://openstack-vip.magedu.local:5000/v3
username = placement
# placement user for keystone
password = placement
[powervm]
[privsep]
[profiler]
[quota]
[rdp]
[remote_debug]
[scheduler]
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc] #此处如果配置不正确，则连接不上虚拟机的控制台
enabled = true
server_listen = 0.0.0.0  #指定vnc的监听地址
# node ip
server_proxyclient_address = 172.16.0.107 #server的客户端地址为本机地址；此地址是管理网的地址
novncproxy_base_url = http://openstack-vip.magedu.local:6080/vnc_auto.html 
#当访问controller vnc的6080时，controller会把请求转发到虚拟机所在node节点的server_proxyclient_address 所设置的地址，所以需要 server_proxyclient_address 指定好本机的监听地址

[workarounds]
[wsgi]
[xenserver]
[xvp]
[zvm]
EOF


cat > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.0.248 openstack-vip.magedu.local
EOF

#验证node节点是否支持硬件辅助虚拟化；vmx是intel虚拟化，svm是amd虚拟化技术；
#一般每盒cpu都支持vmx；如果cpu不支持虚拟化，则需要编辑nova的主配置文件：
#vim /etc/nova/nova.conf
#[libvirt]
#virt_type = qemu  #指定qemu，模拟虚拟化，但是性能非常差

egrep -c '(vmx|svm)' /proc/cpuinfo


systemctl enable libvirtd.service openstack-nova-compute.service --now 

cat > compute-node.sh <<EOF
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
EOF


chmod +x compute-node.sh


tail -f /var/log/nova/nova-compute.log   #通过查看日志，判断 nova compute 是否启动成功
```



# 2. neutron compute

```bash
yum install openstack-neutron-linuxbridge ebtables ipset -y

# 不能有中文
cat > /etc/neutron/neutron.conf <<EOF
[DEFAULT] #neutron的server端与agent端通讯也是通过rabbitmq进行通讯的
# 需要telnet验证5672
transport_url = rabbit://openstack:openstack123@openstack-vip.magedu.local
auth_strategy = keystone #通过keystone做认证

[cors]
[database]
[keystone_authtoken]  #指定keystone认证信息
# 需要telnet验证
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
lock_path = /var/lib/neutron/tmp  #配置锁路径
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[privsep]
[ssl]
EOF

#配置提供者网络
#官方拷贝完整的 linuxbridge_agent.ini 文件，
#https://docs.openstack.org/newton/config-reference/networking/samples/linuxbridge_agent.ini.html

#不能有中文
cat >  /etc/neutron/plugins/ml2/linuxbridge_agent.ini <<EOF
[DEFAULT]
[linux_bridge]
#直接告诉node节点external网络绑定在当前node节点哪个物理网卡上即可，不需要node节点配置网络名称，node节点只需要接收controller节点指令即可；
#controller节点上配置的external网络名称是针对整个openstack环境生效的，所以指定external网络绑定在当前node节点的eth0物理网卡上(也可能是bind0或br0)
physical_interface_mappings = external:eth0 # external是控制节点的falt_network的值
[vxlan]
 #桥接网络不划分子网；自服务才开启
enable_vxlan = false
[securitygroup]
enable_security_group = true #开启安全组
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver #指定安全组驱动文件
EOF


cat > /etc/sysctl.d/openstack.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1 #允许虚拟机的数据通过物理机桥接接口, 再通过SNAT出去
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv6.conf.all.disable_ipv6 = 1     
EOF



systemctl enable neutron-linuxbridge-agent.service --now
sysctl -p /etc/sysctl.d/openstack.conf #neutron服务启动后，br_netfilter模块就会加载，这样就可以让内核参数生效

cat > neutron-node-restart.sh <<EOF
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
EOF


chmod +x neutron-node-restart.sh


tail -f /var/log/neutron/*.log /var/log/nova/*.log #查看node端日志，不能有任何报错
```

