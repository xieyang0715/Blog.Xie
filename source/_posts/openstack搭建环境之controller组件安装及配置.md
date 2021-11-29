---
title: openstack搭建环境之controller组件安装及配置
date: 2020-12-25 03:48:12
tags:
toc: true
---



controller主要配置keystone, glance, placement, nova, neutrone组件



# 前提

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
```



# 1. keystone

Keystone中主要涉及到如下几个概念： User 、 Tenant 、 Role 、 Token:
User：使用 openstack 的用户 。
Tenant：租户 可以理解为一个人、项目或者组织拥有的资源的合集。在一个租户中可以拥有
很多个用户，这些用户可以根据权限的划分使用租户中的资源。
Role：角色，用于分配操作的权限。角色可以被指定给用户 使得该用户获得角色对应的操
作权限。
Token：指的是一串比特值或者字符串，用来作为访问资源的记号。 Token 中含有可访问资源
的范围和有效时间， token 是用户的一种凭证，需要使用 正确的用户名 和 密码向 Keystone 服
务申请才能得到 token 如果用户每次都采用用户名 密码访问 OpenStack API ，容易泄露用
户信息，带来安全隐患， 所以 OpenStack 要求用户访问其 API 前，必须先获取 token ，然后
用 token 作为用户凭据访问 OpenStack API 。

![keystone](http://myapp.img.mykernel.cn/keystone.png)



1. 用户向keystone请求credentials, keystone返回token
2. 用户拿着token就可以为所欲为了。
3. 用户创建虚拟机：
   - 用户请求nova, nova检验token
   - nova请求glance镜像
   - nova请求neutron网络访问
   - nova返回给用户成功响应

```bash
# controller要解析vip为域名
cat > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.0.248 openstack-vip.magedu.local
EOF


# 安装组件
yum install openstack-keystone httpd mod_wsgi -y
# mod_wsgi httpd通过此模块代码给后端的openstack API.


# 生成配置，注意真正配置时不能有中文，否则会出问题。python2解析中文有问题
cat > /etc/keystone/keystone.conf <<EOF
[database] # keystone得访问数据库，在放上这个选项前需要手工复制mysql -ukeystone -pkeystone123 -hopenstack-vip.magedu.local -D keystone是正常的，才可以配置。
connection = mysql+pymysql://keystone:keystone123@openstack-vip.magedu.local/keystone

[token]  #配置Fernet令牌提供者
provider = fernet    # 表示keystone自身提供令牌
#expiration = 3600   #token有效期 3600s. openstack token issue可以看到expire是下一个小时

EOF


# 在database指定的库中生成keystone需要的表。
#配置前查看keystone库中空表 #mysql -ukeystone -pkeystone123 -h openstack-vip.magedu.local -D keystone -e 'show tables;'
su -s /bin/sh -c "keystone-manage db_sync" keystone
#配置后查看keystone库中存在表。

# 初始化fernet密钥，生成在/etc/keystone目录中，用于加密生成token
# ls /etc/keystone 没有目录
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
# ls /etc/keystone检验生成以下目录
#drwx------ 2 keystone keystone     24 Dec 26 08:08 credential-keys
#drwx------ 2 keystone keystone     24 Dec 26 08:08 fernet-keys

# admin ,admin
# 生成openstack的管理员账户，并把bootstrap-*-url写入keystone.endpoint
#5000端口是keystone提供认证的端口
keystone-manage bootstrap --bootstrap-password admin \
--bootstrap-admin-url http://openstack-vip.magedu.local:5000/v3/ \
--bootstrap-internal-url http://openstack-vip.magedu.local:5000/v3/ \
--bootstrap-public-url http://openstack-vip.magedu.local:5000/v3/ \
--bootstrap-region-id RegionOne

  # admin:管理网络
  #   192.168.0.0/16 , 如果不单独在网络，当用户量大时, openstack将无法管理（管理虚拟机的扩容或删除）
  # internal：内部网络
  #   10.20.0.0/16， 内部网络，进行数据传输，如虚拟机访问存储和数据库、zookeeper等中间件，这个网络是不能被外网访问的，只能用于企业内部访问 
  # public：共有网络
  #   172.16.0.0/16，可以给用户访问的(如公有云)
   
  # region：地区、区域，本次指定区域名。
  # domain：机房级别
  # project：项目

cp /etc/httpd/conf/httpd.conf{,.bak}

cat >> /etc/httpd/conf/httpd.conf <<'EOF'
ServerName 172.16.0.248:80  # 当前主机的ip
EOF

# mod_wsgi模块, 里面会listen 5000, 并加载各种wsgi模块，反代用户请求至openstack的python API. 
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

httpd -t

systemctl enable httpd.service --now
lsof -ni:80
lsof -ni:5000

# 验证keystone是否正常，只要请求端口返回json数据
curl openstack-vip.magedu.local:5000


export OS_USERNAME=admin # admin用户
export OS_PASSWORD=admin # 在bootstrap时指定的密码
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://openstack-vip.magedu.local:5000/v3 # keystone连接地址，此处是haproxy输出的vip.
export OS_IDENTITY_API_VERSION=3


# 获取openstack用户列表
openstack user list 




# 创建域
# 查看域（机房）openstack domain list
openstack domain create --description "An Example Domain" example
# 再次查看, 检验已经添加成功


# 查看project(业务、服务) openstack project list
openstack project create --domain default   --description "Service Project" service 
# 再次查看, 检验已经添加成功

openstack project create --domain default   --description "Demo Project" myproject

# 创建属于默认机房的myuser用户，为了方便，使用同用户名的密码
openstack user create --domain default   --password-prompt myuser

# 创建角色myrole, role用于后期授权，role权限在/etc/keystone/policy.json定义，现在此文件为空。
openstack role create myrole
# 查看openstack role list

# 授权myuser对myproject业务有myrole权限
openstack role add --project myproject --user myuser myrole

# 销毁2个变量
unset OS_AUTH_URL OS_PASSWORD

# 通过keystone的地址认证admin用户
openstack --os-auth-url http://openstack-vip.magedu.local:5000/v3   --os-project-domain-name Default --os-user-domain-name Default   --os-project-name admin --os-username admin token issue

# 通过keystone的地址认证myuser用户
openstack --os-auth-url http://openstack-vip.linux.local:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name myproject --os-username myuser token issue

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


cat > demo-openrc <<EOF
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=myuser
export OS_AUTH_URL=http://openstack-vip.magedu.local:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF


# 测试admin用户
source admin-openrc 
openstack token issue

# 测试myuser用户
source demo-openrc 
openstack token issue
```



# 2. glance

Glance是 OpenStack 镜像服务组件， glance 服务默认监听在 9292 端口，其接收 REST API 请求，然后通过其他模块（ glance registry 及 image store ）来完成诸如镜像的获取、上传、删除等操作



组件

- glace api             监听**9292**，接收镜像的删除、上传和读取，通过image store存储接口操作，image store可以存储至 Amazon 的 S3 、 openstack 本身的 swift 、还有 ceph 、 glusterFS 、 sheepdog 等分布式存储。
-  glance Registry 监听的端口是 **9191**， 负责与 mysql 数据交互，用于存储或获取镜像的元数据（metadata ），提供镜像元数据相关的 REST 接口，通过 glance registry 可以向数据库中写入或获取镜像的各种数据



glance库

-  glance 表
- image表 保存了镜像格式、大小等信息
-  imane property 表保存了镜像的定制化信息



 glance 不需要配置消息队列，但是需要配置数据库和 keystone 。

glance可以单独安装在一台服务器上，也可以和controler安装在同一台服务器上



```bash
source admin-openrc 

# default域中创建glance用户
openstack user create --domain default --password-prompt glance

# glance用户对service业务有admin权限，注册glance的API，需要对service项目有admin权限
openstack role add --project service --user glance admin

# 添加image service
openstack service create --name glance  --description "OpenStack Image" image

# 给image service配置端点。(k8s通过label选择pod) 
openstack endpoint create --region RegionOne image public http://openstack-vip.magedu.local:9292
openstack endpoint create --region RegionOne image internal http://openstack-vip.magedu.local:9292
openstack endpoint create --region RegionOne image admin http://openstack-vip.magedu.local:9292

# 验证endpoint已经添加成功
#openstack endpoint list



# 安装组件
yum install openstack-glance -y


#glance有两个配置文件，还有一个glance-registry.conf 
# 配置文件一定不能有中文
cat >/etc/glance/glance-api.conf <<EOF

[database] # glance连接数据库写入镜像的元数据, 写入此行验证 mysql -uglance -pglance123 -h openstack-vip.magedu.local -D glance 正常
connection = mysql+pymysql://glance:glance123@openstack-vip.magedu.local/glance

[glance_store]
stores = file,http #指定存储类型为文件存储；还有http类型，基于api调用方式，把镜像放到其他存储上
default_store = file
filesystem_store_datadir = /var/lib/glance/images/ #指定镜像存放的本地目录,glance用户和组
# 配置存储库
#install -dv -o glance -g glance /var/lib/glance/images/
#showmount -e 172.16.0.248
#mount -t nfs  172.16.0.248:/data/glance /var/lib/glance/images/
# /etc/fstab
#172.16.0.248:/data/glance /var/lib/glance/images/  nfs defaults        0 0
# 确保glance的id在nfs主机的/data/glance目录有权限。chown 161.161 /data/glance/ 否则上传不了镜像

[keystone_authtoken]
# 配置以下行需要telnet openstack-vip.magedu.local 5000, telnet openstack-vip.magedu.local 11211检验正常
www_authenticate_uri  = http://openstack-vip.magedu.local:5000
auth_url = http://openstack-vip.magedu.local:5000
memcached_servers = openstack-vip.magedu.local:11211
auth_type = password #认证方式采用密码
project_domain_name = Default
user_domain_name = Default
project_name = service #glance用户针对service项目拥有admin权限
username = glance      
password = glance


[paste_deploy]
flavor = keystone #glance指定提供认证的服务器为keystone

EOF

install -dv -o glance -g glance /var/lib/glance/images/
showmount -e 172.16.0.248
mount -t nfs  172.16.0.248:/data/glance /var/lib/glance/images/
cat >> /etc/fstab <<EOF
172.16.0.248:/data/glance /var/lib/glance/images/  nfs defaults        0 0
EOF

# 验证表空 mysql -uglance -pglance123 -h openstack-vip.magedu.local -D glance -e 'show tables;'
# 给glance库添加表 
su -s /bin/sh -c "glance-manage db_sync" glance
# 验证表存在

systemctl restart openstack-glance-api.service
systemctl enable openstack-glance-api.service 
#ss -tnl验证9292
# telnet openstack-vip.magedu.local 9292 端口已经通

# 提供脚本, 方便以后重启
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


# 一定要查看日志是否有报错，一旦有报错，整个集群将不可用。
tail /var/log/glance/*.log 
#This option is deprecated against new config option
#``default_backend`` which acts similar to ``default_store`` config
#option.

##This option is scheduled for removal in the U development
#cycle.
#).  Its value may be silently ignored in the future.
##2020-12-26 08:36:13.731 2397 INFO glance.common.wsgi [-] Starting 1 workers
#2020-12-26 08:36:13.735 2397 INFO glance.common.wsgi [-] Started child 2409
#2020-12-26 08:36:13.752 2409 INFO eventlet.wsgi.server [-] (2409) wsgi starting up on http://0.0.0.0:9292


```



检验glance服务

```bash

. admin-openrc
# 下载镜像
wget http://download.cirros-cloud.net/0.5.1/cirros-0.5.1-x86_64-disk.img

# 上传镜像
glance image-create --name "cirros-0.5.1"   --file cirros-0.5.1-x86_64-disk.img   --disk-format qcow2 --container-format bare   --visibility=public

# nfs服务中查看镜像是否存在
ll /var/lib/glance/images/ -h

# 获取镜像列表
glance image-list
openstack image list
```



准备nfs服务器，在haproxy服务器上

```bash
os_family=$(grep -Po '(?<=^ID=)\S+' /etc/os-release)
if [ "$os_family" == "ubuntu" ]; then
	apt install nfs-kernel-server -y
else
	yum install nfs-utils -y
fi
mkdir -pv /data/glance
echo "/data/glance *(rw,sync,no_root_squash)" > /etc/exports
exportfs -arv
if [ "$os_family" == "ubuntu" ]; then
	systemctl restart  nfs-kernel-server
else
	systemctl enable nfs --now
fi
rpcinfo -p localhost
showmount -e
```



# 3. placement

收集各个node节点的可用资源，把node节点的资源统计写入到mysql, Placement服务会被nova scheduler服务进行调用 

监听8778

```bash
echo "Start Placement Configuration"
source admin-openrc 

echo add placement user, password placement 
openstack user create --domain default --password-prompt placement

echo  授权placement用户对service服务有admin权限
openstack role add --project service --user placement admin

echo 创建Placement API服务
openstack service create --name placement   --description "Placement API" placement

echo add image endpoint 
openstack endpoint create --region RegionOne placement public http://openstack-vip.magedu.local:8778
openstack endpoint create --region RegionOne placement internal http://openstack-vip.magedu.local:8778
openstack endpoint create --region RegionOne placement admin http://openstack-vip.magedu.local:8778

echo intsall component
yum install openstack-placement-api -y


# 不能有中文
cat > /etc/placement/placement.conf <<EOF
[DEFAULT]
[api]
auth_strategy = keystone
[cors]
[keystone_authtoken]
# 需要telnet验证5000, curl有json返回
auth_url = http://openstack-vip.magedu.local:5000/v3
# 需要telnet验证
memcached_servers = openstack-vip.magedu.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service #admin权限
username = placement
# place user for identify user
password = placement
[oslo_policy]
[placement]
[placement_database]
# 需要mysql连接 mysql -u -p -h -D验证
connection = mysql+pymysql://placement:placement123@openstack-vip.magedu.local/placement
[profiler]
EOF

# 查看表空
su -s /bin/sh -c "placement-manage db sync" placement
# 查看表已经创建
#mysql -uplacement -pplacement123 -h openstack-vip.magedu.local -D placement -e 'show tables;'

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


echo check placement
. admin-openrc
placement-status upgrade check
echo "Placement Configuration End"


tail /var/log/httpd/*.log
# ==> /var/log/httpd/keystone_access.log <==
# 172.16.0.105 - - [26/Dec/2020:08:52:53 +0800] "POST /v3/auth/tokens HTTP/1.1" 201 2124 "-" "openstacksdk/0.36.4 keystoneauth1/3.17.3 python-requests/2.21.0 CPython/2.7.5"
# 172.16.0.105 - - [26/Dec/2020:08:52:55 +0800] "GET /v3/services/placement HTTP/1.1" 404 90 "-" "python-keystoneclient"
# 172.16.0.105 - - [26/Dec/2020:08:52:57 +0800] "GET /v3/services?name=placement HTTP/1.1" 200 376 "-" "python-keystoneclient"
# 172.16.0.105 - - [26/Dec/2020:08:52:58 +0800] "POST /v3/endpoints HTTP/1.1" 201 354 "-" "python-keystoneclient"
# 172.16.0.105 - - [26/Dec/2020:08:53:04 +0800] "GET /v3 HTTP/1.1" 200 266 "-" "openstacksdk/0.36.4 keystoneauth1/3.17.3 python-requests/2.21.0 CPython/2.7.5"
# 172.16.0.105 - - [26/Dec/2020:08:53:04 +0800] "POST /v3/auth/tokens HTTP/1.1" 201 2291 "-" "openstacksdk/0.36.4 keystoneauth1/3.17.3 python-requests/2.21.0 CPython/2.7.5"
# 172.16.0.105 - - [26/Dec/2020:08:53:05 +0800] "POST /v3/auth/tokens HTTP/1.1" 201 2291 "-" "openstacksdk/0.36.4 keystoneauth1/3.17.3 python-requests/2.21.0 CPython/2.7.5"
# 172.16.0.105 - - [26/Dec/2020:08:53:06 +0800] "GET /v3/services/placement HTTP/1.1" 404 90 "-" "python-keystoneclient"
# 172.16.0.105 - - [26/Dec/2020:08:53:07 +0800] "GET /v3/services?name=placement HTTP/1.1" 200 376 "-" "python-keystoneclient"
```





# 4. nova

nova是 ope nstack 最早的 组件 之一 nova 分为控制节点和计算节点，计算节点通过 nova computer 进行虚拟机创建，通过 libvirt 调用 kvm 创建虚拟机， nova 之间通信通过 rabbitMQ队列进行通信， 其 组件和功能如下：
API 负责接收 和 响应外部请求 。
Scheduler 负责 调度虚拟机所在的物理机。
Conductor ：计算节点访问数据库的中间件。
Consoleauth：用于控制台的授权认证。
Novncproxy VNC 代理 ，用于 显示 虚拟机 操作 终端。

```bash

echo "Start Nova Configuration"
. admin-openrc 

echo add nova user, password nova 
openstack user create --domain default --password-prompt nova
echo nova用户对service服务有admin权限
openstack role add --project service --user nova admin
echo 创建名为nova的compute服务
openstack service create --name nova   --description "OpenStack Compute" compute
echo 创建端点
openstack endpoint create --region RegionOne compute public http://openstack-vip.magedu.local:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://openstack-vip.magedu.local:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://openstack-vip.magedu.local:8774/v2.1

echo install component
yum install -y openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler


# 不能有中文
cat > /etc/nova/nova.conf << EOF
[DEFAULT]
#支持的api类型；开启之后，就可以通过dashboard或命令行访问
enabled_apis = osapi_compute,metadata
# 需要telnet验证 rabbitmq
transport_url = rabbit://openstack:openstack123@openstack-vip.magedu.local:5672/
use_neutron = true #使用neutron获取网络信息
firewall_driver = nova.virt.firewall.NoopFirewallDriver 
#通过此驱动程序与neutron进行交互，其实就是一个python文件；/usr/lib/python2.7/site-packages/nova/virt/firewall.py 
[api]
auth_strategy = keystone #nova认证方式采用keystone认证
[api_database]
# 需要mysql检验
connection = mysql+pymysql://nova:nova123@openstack-vip.magedu.local/nova_api
[barbican]
[cache]
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[database]
# 需要mysql检验
connection = mysql+pymysql://nova:nova123@openstack-vip.magedu.local/nova
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]  #nova需要连接glance获取镜像 
# 需要telnet检验
api_servers = http://openstack-vip.magedu.local:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
# 需要telnet检验
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
[libvirt]
[metrics]
[mks]
[neutron]
[notifications]
[osapi_v21]
[oslo_concurrency] #指定锁路径
lock_path = /var/lib/nova/tmp #锁的作用是创建虚拟机时，在执行某个操作的时候，需要等此步骤执行完后才能执行下一个步骤，不能并行执行，保证操作是一步一步的执行
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]  #通过调用placement服务，获取当前可用的主机列表
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
# 需要telnet检验
auth_url = http://openstack-vip.magedu.local:5000/v3 
username = placement
# identify or keystone user 
password = placement

[powervm]
[privsep]
[profiler]
[quota]
[rdp]
[remote_debug]
[scheduler]
# 周期发现compute主机, 并添加至cell中
discover_hosts_in_cells_interval = 300
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]  #只有vnc配置正确，后期才可以通过浏览器管理虚拟机，否则报1006或者1005状态码，即vnc配置错误
enabled = true
# vnc监听地址
server_listen = 0.0.0.0
server_proxyclient_address = 172.16.0.101
[workarounds]
[wsgi]
[xenserver]
[xvp]
[zvm]
EOF

#初始化nova_api数据库
# 查看空表
su -s /bin/sh -c "nova-manage api_db sync" nova
# 表已经生成

#注册cell0数据库；nova服务内部把资源划分到不同的cell中，把计算节点划分到不同的cell中；openstack内部基于cell把计算节点进行逻辑上的分组
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova

#创建cell1单元格；
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

#初始化nova数据库；可以通过 /var/log/nova/nova-manage.log 日志判断是否初始化成功
su -s /bin/sh -c "nova-manage db sync" nova

wait 
#验证cell0和cell1组件是否注册成功
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova

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
echo "Nova Configuration End"



#确保nova在启动过程中没有任何报错；
#日志中可能会出现关于mysql连接错误，此报错是因为nova是通过haproxy代理与mysql进行连接的，一但haproxy到了超时时间，nova没有与mysql进行交互，则会自动断开连接(重启nova服务后，日志中就不会出现该报错)，想要让该报错少一些的话，可以修改haproxy配置文件中的配置：timeout client、timeout server、timeout http-keep-alive
tail /var/log/nova/nova-*.log  

```





# 5. neutron

网络：在 显示 的网络环境 中我们使用 交换机 将 多个计算机 连接 起来 从而 形成了网络， 而 在
neutron 的环境里， 网络 的功能也是将 多个 不同的云主机连接起来。

子网是现实 的网络 环境 下 可以 将一个网络划分成多个 逻辑 上的 子网络 从而 实现网络 隔离
在 neutron 里面子网也是属于网络。

端口计算机 连接交换机通过 网线 连 而网线插在交换机的不 同端口 在 neutron 里面端口
属于子网， 即 每个云主机的子网 都会 对应到一个端口。

路由器用于 连接不通的网络或者子网。

![ML2](http://myapp.img.mykernel.cn/ML2.png)

openstack的neutron网络服务通过多个插件实现：
**ML2插件**是实现虚拟机桥网络(基于mac地址通讯)，包含linux bridge、openvswitch插件；
**DHCP-Agent**，自动分配虚拟机地址；
**L3-Agent**，3层代理，用于**自服务网络**；
**LBAAS-Agent**，实现**负载均衡**等服务；
**其他Agent**；

网络类型：
provider(桥接）： 虚拟机桥接 到物理机， 并且虚拟机 必须和物理机在同一个网络段。 

自服务网络：可以自己创建网络， 最终 会通过虚拟路由器 连接 外网。SNAT

端口：

9696



```bash
echo "Neutron Configuration Start"
. admin-openrc
echo add neutron user, password neutron 
openstack user create --domain default --password-prompt neutron

# 授权neutron用户对service服务有admin权限
openstack role add --project service --user neutron admin

openstack service create --name neutron   --description "OpenStack Networking" network
openstack endpoint create --region RegionOne network public http://openstack-vip.magedu.local:9696
openstack endpoint create --region RegionOne network internal http://openstack-vip.magedu.local:9696
openstack endpoint create --region RegionOne network admin http://openstack-vip.magedu.local:9696

# 安装组件
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y


# 不能有中文
cat > /etc/neutron/neutron.conf <<EOF
[DEFAULT]
core_plugin = ml2 #启用二层网络插件
service_plugins = #禁用三层网络插件
transport_url = rabbit://openstack:openstack123@openstack-vip.magedu.local #配置rabbitmq连接

auth_strategy = keystone # 使用keystone认证
notify_nova_on_port_status_changes = true #当网络接口发生变化时，通知给计算节点
notify_nova_on_port_data_changes = true   #当端口数据发生变化，通知计算节点


[cors]
[database]
# 需要telnet验证
connection = mysql+pymysql://neutron:neutron123@openstack-vip.magedu.local/neutron

[keystone_authtoken] #配置keystone认证信息
# 需要telnet验证
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

[oslo_concurrency]  #配置锁路径
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[privsep]
[ssl]
[nova]  #neutron需要给nova返回数据
# 需要telnet验证
auth_url = http://openstack-vip.magedu.local:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
# nova user of keystone
password = nova
EOF


#配置二层插件，不能有中文
cat > /etc/neutron/plugins/ml2/ml2_conf.ini<<EOF
[DEFAULT]
[ml2]
#配置驱动类型；单一扁平网络(桥接)和vlan；让二层网络支持桥接，支持基于vlan做子网划分
type_drivers = flat,vlan

tenant_network_types = #租户网络类型；禁止用户创建自己的网络(基于vxlan)，自服务网络才会开启

mechanism_drivers = linuxbridge #启用linux桥接；二层网络中使用的是桥接
#启用端口安全扩展驱动程序，基于iptables实现访问控制；但配置了扩展安全组会导致一些端口限制，造成一些服务无法启动
extension_drivers = port_security

[ml2_type_flat]
#设置桥接网络名称；通常用内网或外网命名，external表示用于访问外网
flat_networks = external

[securitygroup]
#启用ipset以提高安全组规则的效率；ipset是iptables的集合，可以实现iptables更高的访问效率；这个安全组类似于阿里云的安全策略，在安全组中设置访问策略，哪个端口允许，哪个端口禁止
enable_ipset = true
EOF


#安装完neutron服务后，linuxbridge_agent.ini 此文件内容是不全的，到官网拷贝完整的配置文件：
#https://docs.openstack.org/newton/configreference/networking/samples/linuxbridge_agent.ini

#定义桥接网络与指定的物理网络做关联
cat > /etc/neutron/plugins/ml2/linuxbridge_agent.ini <<EOF
[DEFAULT]
[linux_bridge]
#指定flat_networks名称，与eth0物理网卡做关联，后期给虚拟机分配external网络，就可以通过eth0上外网；物理网卡有可能是bind0、br0等
physical_interface_mappings = external:eth0
[vxlan]
enable_vxlan = false  #不需要用户创建自定义网络(3层网络)，所以关闭vxlan
[securitygroup]
enable_security_group = true #开启安全组
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
#指定安全组驱动程序，调用这个python文件管理安全组(iptables规则)；此python文件在 /usr/lib/python2.7/site-packages/neutron/agent/linux/iptables_firewall.py文件中IptablesFirewallDriver类
EOF


modprobe br_netfilter # 以下2个参数需要此模块，此模块在neutron服务启动时会自动加载，所以也可以启动neutron之后在sysctl -p 读内核参数。
cat > /etc/sysctl.d/openstack.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1  #让操作系统内核支持网桥过滤器；允许虚拟机(容器)的网络经过物理机，如果不设定此参数，则虚拟机(容器)的数据无法通过宿主机网络出去
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl -p /etc/sysctl.d/openstack.conf

 #配置DHCP，让虚拟机通过DHCP获取IP地址
cat > /etc/neutron/dhcp_agent.ini <<EOF
[DEFAULT]
#指定接口驱动为 linuxbridge
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq #指定 DHCP 驱动
enable_isolated_metadata = true #开启 iso 元数据
EOF

#配置桥接与自服务网络的通用配置
cat > /etc/neutron/metadata_agent.ini <<EOF
[DEFAULT]
# ...
nova_metadata_host = openstack-vip.magedu.local #指定controller地址，并且 9696 端口必须打开才可
metadata_proxy_shared_secret = 2GosJ41hQbkcku8 #配置共享秘钥，共享秘钥会给 nova 使用，nova 需要访问网络数据，如虚拟机的 IP 地址等，nova 访问 neutron 时，需要使用到 neutron 用户名、密码以及此共享秘钥，才可以拿到 neutron 的所有元数据；共享秘钥中不能有特殊字符@
EOF

#配置 nova 配置文件
cat > /etc/nova/nova.conf <<EOF
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:openstack123@openstack-vip.magedu.local:5672/
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api]
auth_strategy = keystone
[api_database]
connection = mysql+pymysql://nova:nova123@openstack-vip.magedu.local/nova_api
[barbican]
[cache]
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[database]
connection = mysql+pymysql://nova:nova123@openstack-vip.magedu.local/nova
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://openstack-vip.magedu.local:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
www_authenticate_uri = http://openstack-vip.magedu.local:5000/
auth_url = http://openstack-vip.magedu.local:5000/
memcached_servers = openstack-vip.magedu.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova
[libvirt]
[metrics]
[mks]
[neutron] #nova会携带用户的token到neutron中申请IP地址
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
metadata_proxy_shared_secret = 2GosJ41hQbkcku8 #nova拿着token和这个secret才可以向neutron请求metadata.
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
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://openstack-vip.magedu.local:5000/v3
username = placement
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
[vnc]
enabled = true
server_listen = 172.16.0.101
server_proxyclient_address = 172.16.0.101
[workarounds]
[wsgi]
[xenserver]
[xvp]
[zvm]
EOF

#网络服务初始化脚本, 需要/etc/neutron/plugin.ini指向ML2插件配置文件的符号链接
ln -sv /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini

#初始化neutron数据库 
# 查看neutron库
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
# 生成表


wait 

# 修改了nova配置，所以重启
systemctl restart openstack-nova-api.service


# 启动neutron
systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service --now 




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

echo "Neutron Configuration Stop"
```



# 6. 配置过程日志报错

获取日志报错rabbitmq报错，3次重装3天。 经过重温杰哥视频，发现**neutron/nova的DEFAULT段忘记配置rabbitmq**.

所以在rabbitmq报错，首先确认对应服务的配置文件结合官方文档查看是否配置正常。连接是否可用。

确保配置文件中不能有中文











