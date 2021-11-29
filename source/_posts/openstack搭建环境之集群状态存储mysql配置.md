---
title: openstack搭建环境之集群状态存储mysql配置
date: 2020-12-25 03:14:58
tags:
toc: true
---



可以单独安装至其他服务器，openstack 的各组件都要使用数据库保存数据， 除 了 nova 使用
API 与其他组件进行调用之外：

MySQL 官方 下载地址： https://dev.mysql.com/downloads/

<!--more-->

# 1. 基于模板克隆mysql

```bash
cd /etc/sysconfig/network-scripts/
# ifcfg-eth0
IPADDR=172.16.0.103
# ifcfg-eth1
IPADDR=10.0.0.103

# /etc/hostname
openstack-mysql-master.magedu.local
```



# 2. 安装配置mysql

```bash
# 只有安装openstack的源，对应的mysql才是契合度最高的mysql
yum install centos-release-openstack-train -y
yum install https://rdoproject.org/repos/rdo-release.rpm -y

# 安装mysql
yum install mariadb mariadb-server -y

# 优化mysql
cat > /etc/my.cnf.d/openstack.cnf <<'EOF'
[mysqld]
# 绑定在mysql主机所有接口的地址
bind-address = 0.0.0.0

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
# 使用utf8字符集
collation-server = utf8_general_ci
character-set-server = utf8
EOF

#启动
systemctl enable mariadb.service --now

#配置mysql的root密码openstack123
mysqladmin password openstack123

# 登陆验证
mysql -uroot -popenstack123  -e 'SELECT 1'
```



# 3. 准备所有组件需要的库

```bash

# 添加keystone组件的用户和准备其数据库
mysql -u root -popenstack123 <<EOF
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'keystone123';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone123';
EOF
mysql -ukeystone -pkeystone123  -e 'SELECT 1'

# 添加glance组件的用户和准备其数据库
mysql -u root -popenstack123 <<EOF
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'glance123';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance123';
EOF
mysql -uglance -pglance123  -e 'SELECT 1'


# 添加placement组件的用户和准备其数据库
mysql -u root -popenstack123 <<EOF
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'placement123';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'placement123';
EOF
mysql -uplacement -pplacement123  -e 'SELECT 1'



# 添加nova组件的用户和准备其数据库nova, nova_api, nova_cell0
mysql -u root -popenstack123 <<EOF
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost'   IDENTIFIED BY 'nova123';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%'   IDENTIFIED BY 'nova123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost'   IDENTIFIED BY 'nova123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%'   IDENTIFIED BY 'nova123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost'   IDENTIFIED BY 'nova123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%'   IDENTIFIED BY 'nova123';
EOF
mysql -unova -pnova123  -e 'SELECT 1'



# 添加neutron组件的用户和准备其数据库
mysql -u root -popenstack123 <<EOF
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'neutron123';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'neutron123';
EOF
mysql -uneutron -pneutron123  -e 'SELECT 1'



# 验证vip可以连通 
mysql -uneutron -pneutron123 -h172.16.0.248

```

