---
title: zabbix安装及配置
date: 2021-02-20 01:21:50
tags:
---





# 前言

跟随官方的zabbix的yum源去安装zabbix时，后面的zabbix版本会发生变化，但是源码编译安装不会，但是要快速安装出来，需要一定时间，为了节约时间，可以直接yum源安装。

源码编译安装，并无什么特别难的地方。 因为结果都是二进制程序接选项启动。只是启动时需要加一些编译参数。

以下以rpm包安装为例



为了避免官方的yum源会不断更新，就需要在本地环境镜像官方源yum源，以供日后每次安装拥有相同版本的程序包及依赖的包。就算没有镜像，其实aliyun镜像源也有官方的镜像。



<!--more-->
# zabbix 安装

https://www.zabbix.com/download?zabbix=4.0&os_distribution=red_hat_enterprise_linux&os_version=7&db=mysql&ws=apache

## 安装zabbix repo

```bash
 rpm --replacepkgs -ivh https://repo.zabbix.com/zabbix/4.0/rhel/7/x86_64/zabbix-release-4.0-2.el7.noarch.rpm


# 验证
[root@k8s-master1 ~]# cat /etc/yum.repos.d/zabbix.repo 
[zabbix]
name=Zabbix Official Repository - $basearch
baseurl=http://repo.zabbix.com/zabbix/4.0/rhel/7/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591

[zabbix-debuginfo]
name=Zabbix Official Repository debuginfo - $basearch
baseurl=http://repo.zabbix.com/zabbix/4.0/rhel/7/$basearch/debuginfo/
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591
gpgcheck=1

[zabbix-non-supported]
name=Zabbix Official Repository non-supported - $basearch
baseurl=http://repo.zabbix.com/non-supported/rhel/7/$basearch/
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX
gpgcheck=1
```

也可以配置aliyun镜像`https://mirrors.aliyun.com/zabbix/?spm=a2c6h.13651104.0.0.3617233ciO7wyX`

aliyun的镜像地址是`https://mirrors.aliyun.com/zabbix/zabbix/4.0/rhel/7/x86_64/`

```bash
[root@k8s-master2 ~]# cat /etc/yum.repos.d/zabbix.repo
[zabbix]
name=Zabbix Official Repository - $basearch
baseurl=https://mirrors.aliyun.com/zabbix/zabbix/4.0/rhel/7/$basearch
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/zabbix/RPM-GPG-KEY-ZABBIX-A14FE591

[zabbix-non-supported]
name=Zabbix Official Repository non-supported - $basearch
baseurl=https://mirrors.aliyun.com/zabbix/non-supported/rhel/7/$basearch/
enabled=1
gpgkey=https://mirrors.aliyun.com/zabbix/RPM-GPG-KEY-ZABBIX
gpgcheck=1
```





## Install Zabbix server, frontend, agent

获取之前zabbxi server的版本

```bash
[root@k8s-node1 mysql]# rpm -qa | grep zabbix | awk -F. -v OFS='.' '{print $1,$2,$3}' | sed 's@-[^-]\+$@@g' | xargs
zabbix-web-pgsql-4.0.24 zabbix-get-4.0.24 zabbix-web-4.0.24 zabbix-agent-4.0.24 zabbix-release-4.0 zabbix-server-mysql-4.0.24 zabbix-web-mysql-4.0.24 zabbix-sender-4.0.25
```

安装

```bash
# yum install zabbix-web-pgsql-4.0.24 zabbix-get-4.0.24 zabbix-web-4.0.24 zabbix-agent-4.0.24 zabbix-release-4.0 zabbix-server-mysql-4.0.24 zabbix-web-mysql-4.0.24 zabbix-sender-4.0.25

 Package                                                                         Arch                                                               Version                                                                         Repository                                                                        Size
===========================================================================================================================================================================================================================================================================================================================
Installing:
 zabbix-get                                                                      x86_64                                                             4.0.24-1.el7                                                                    zabbix                                                                           297 k
 zabbix-release                                                                  noarch                                                             4.0-2.el7                                                                       zabbix                                                                            13 k
 zabbix-sender                                                                   x86_64                                                             4.0.25-1.el7                                                                    zabbix                                                                           329 k
 zabbix-server-mysql                                                             x86_64                                                             4.0.24-1.el7                                                                    zabbix                                                                           2.2 M
 zabbix-web                                                                      noarch                                                             4.0.24-1.el7                                                                    zabbix                                                                           2.9 M
 zabbix-web-mysql                                                                noarch                                                             4.0.24-1.el7                                                                    zabbix                                                                            11 k
 zabbix-web-pgsql                                                                noarch                                                             4.0.24-1.el7                                                                    zabbix                                                                            11 k
Installing for dependencies:
 OpenIPMI                                                                        x86_64                                                             2.0.27-1.el7                                                                    base                                                                             243 k
 OpenIPMI-libs                                                                   x86_64                                                             2.0.27-1.el7                                                                    base                                                                             523 k
 OpenIPMI-modalias                                                               x86_64                                                             2.0.27-1.el7                                                                    base                                                                              16 k
 apr                                                                             x86_64                                                             1.4.8-7.el7                                                                     base                                                                             104 k
 apr-util                                                                        x86_64                                                             1.5.2-6.el7                                                                     base                                                                              92 k
 fping                                                                           x86_64                                                             3.10-4.el7                                                                      epel                                                                              46 k
 gnutls                                                                          x86_64                                                             3.3.29-9.el7_6                                                                  base                                                                             680 k
 httpd                                                                           x86_64                                                             2.4.6-97.el7.centos                                                             updates                                                                          2.7 M
 httpd-tools                                                                     x86_64                                                             2.4.6-97.el7.centos                                                             updates                                                                           93 k
 iksemel                                                                         x86_64                                                             1.4-2.el7.centos                                                                zabbix-non-supported                                                              49 k
 libXpm                                                                          x86_64                                                             3.5.12-1.el7                                                                    base                                                                              55 k
 libtool-ltdl                                                                    x86_64                                                             2.4.2-22.el7_3                                                                  base                                                                              49 k
 libzip                                                                          x86_64                                                             0.10.1-8.el7                                                                    base                                                                              48 k
 mailcap                                                                         noarch                                                             2.1.41-2.el7                                                                    base                                                                              31 k
 net-snmp-libs                                                                   x86_64                                                             1:5.7.2-49.el7_9.1                                                              updates                                                                          751 k
 nettle                                                                          x86_64                                                             2.7.1-8.el7                                                                     base                                                                             327 k
 php                                                                             x86_64                                                             5.4.16-48.el7                                                                   base                                                                             1.4 M
 php-bcmath                                                                      x86_64                                                             5.4.16-48.el7                                                                   base                                                                              58 k
 php-cli                                                                         x86_64                                                             5.4.16-48.el7                                                                   base                                                                             2.7 M
 php-common                                                                      x86_64                                                             5.4.16-48.el7                                                                   base                                                                             565 k
 php-gd                                                                          x86_64                                                             5.4.16-48.el7                                                                   base                                                                             128 k
 php-ldap                                                                        x86_64                                                             5.4.16-48.el7                                                                   base                                                                              53 k
 php-mbstring                                                                    x86_64                                                             5.4.16-48.el7                                                                   base                                                                             506 k
 php-mysql                                                                       x86_64                                                             5.4.16-48.el7                                                                   base                                                                             102 k
 php-pdo                                                                         x86_64                                                             5.4.16-48.el7                                                                   base                                                                              99 k
 php-pgsql                                                                       x86_64                                                             5.4.16-48.el7                                                                   base                                                                              87 k
 php-xml                                                                         x86_64                                                             5.4.16-48.el7                                                                   base                                                                             126 k
 postgresql-libs                                                                 x86_64                                                             9.2.24-4.el7_8                                                                  base                                                                             234 k
 t1lib                                                                           x86_64                                                             5.1.2-14.el7                                                                    base                                                                             166 k
 trousers                                                                        x86_64                                                             0.3.14-2.el7                                                                    base                                                                             289 k
 unixODBC                                                                        x86_64                                                             2.3.1-14.el7                                                                    base                                                                             413 k

Transaction Summary
===========================================================================================================================================================================================================================================================================================================================
Install  7 Packages (+31 Dependent packages)
```

## 准备数据库

将之前的数据库备份，直接离线直接托过来

```bash
[root@k8s-master2 ~]# mkdir /mnt/mysql
[root@k8s-node1 mysql]# scp mariadb.tar.bz2 172.16.0.223:/mnt/mysql/

# pwd
/mnt/mysql
[root@k8s-master2 mysql]# tar xvf mariadb.tar.bz2 
```

> 参考 [mysql优化配置](http://blog.mykernel.cn/2021/02/20/RDB%E5%AE%89%E8%A3%85%E5%8F%8A%E9%85%8D%E7%BD%AE%E4%BC%98%E5%8C%96/)

```diff
[root@k8s-master2 fonts]# cat /etc/my.cnf.d/my.cnf 
[mysqld]
max_connections = 10000
+innodb_buffer_pool_size = 2535M
innodb_buffer_pool_instances = 4
innodb_log_buffer_size = 16M     
innodb_flush_log_at_trx_commit = 1
innodb_file_per_table=1
skip-name-resolve
wait_timeout = 300
slow_query_log=ON
log_output=FILE
long_query_time=3
slow_query_log_file=/mnt/mysql/slow.log
log_queries_not_using_indexes=ON
log_error=/mnt/mysql/error.log
log-bin = /mnt/mysql/mysql-bin     
binlog_format = mixed   
expire_logs_days = 7   
max_binlog_size = 100m
binlog_cache_size = 4m
max_binlog_cache_size= 512m 
# 由于zabbix很多表使用MyISAM
sort_buffer_size = 256M       
read_buffer_size = 256M        
join_buffer_size = 256M        
key_buffer_size = 256M  
#并且有大量的查询操作
query_cache_type = ON
query_cache_limit = 1M
query_cache_size = 32M  
open_files_limit = 52706963
tmp_table_size= 64M
max_heap_table_size= 64M
```

## 准备zabbix配置

直接从之前的server拷贝配置

```bash
[root@k8s-node1 mysql]# scp -rp /etc/zabbix/* 172.16.0.223:/etc/zabbix
```

```bash
# 报警脚本
[root@k8s-node1 mysql]# scp -rp /usr/lib/zabbix/alertscripts/* 172.16.0.223:/usr/lib/zabbix/alertscripts
```

## 启动zabbix

 /etc/httpd/conf/httpd.conf 

```bash
Listen 8082
```

/etc/httpd/conf.d/zabbix.conf 

```bash
php_value date.timezone Asia/Shanghai
```

准备字体, windows中找到simkai.ttf文件`C:\Windows\Fonts\simkai.ttf`

```bash
[root@k8s-master2 fonts]# pwd
/usr/share/zabbix/assets/fonts
[root@k8s-master2 fonts]# ls
graphfont.ttf
[root@k8s-master2 fonts]# rz -E
rz waiting to receive.
[root@k8s-master2 fonts]# ls
graphfont.ttf  simkai.ttf
[root@k8s-master2 fonts]# mv simkai.ttf graphfont.ttf 
mv: overwrite ‘graphfont.ttf’? y
```

准备zabbix server

```bash
[root@k8s-master2 fonts]# cat /etc/zabbix/zabbix_server.conf
ListenPort=10051
LogFile=/var/log/zabbix/zabbix_server.log
LogFileSize=0
DebugLevel=3
PidFile=/tmp/zabbix_server.pid
PidFile=/var/run/zabbix/zabbix_server.pid
SocketDir=/tmp
DBHost=127.0.0.1
DBName=zabbix
DBUser=zabbix
DBPassword=zabbixweizhixiu
DBPort=3306
StartPollers=10
StartIPMIPollers=0
StartPreprocessors=10
StartPollersUnreachable=3
StartTrappers=10
StartPingers=1
StartDiscoverers=1
StartHTTPPollers=1
SNMPTrapperFile=/var/log/snmptrap/snmptrap.log
Timeout=4
AlertScriptsPath=/usr/lib/zabbix/alertscripts
ExternalScripts=/usr/lib/zabbix/externalscripts
Fping6Location=/usr/sbin/fping6
LogSlowQueries=3000
AllowRoot=1
#HistoryStorageURL=http://172.16.0.224:9200
#HistoryStorageTypes=str,log,text


#调整zabbix_server进程
StartDBSyncers=6
#调整内存大小
VMwareCacheSize=64M
CacheSize=32M
HistoryCacheSize=256M
TrendCacheSize=64M
HistoryIndexCacheSize = 32M
ValueCacheSize=64M

```

配置zabbix web

```bash
[root@k8s-master2 fonts]# cat /etc/zabbix/web/zabbix.conf.php 
<?php
// Zabbix GUI configuration file.
global $DB;

$DB['TYPE']     = 'MYSQL';
$DB['SERVER']   = '127.0.0.1';
$DB['PORT']     = '3306';
$DB['DATABASE'] = 'zabbix';
$DB['USER']     = 'zabbix';
$DB['PASSWORD'] = 'xxxxxxx';

// Schema name. Used for IBM DB2 and PostgreSQL.
$DB['SCHEMA'] = '';

$ZBX_SERVER      = 'localhost';
$ZBX_SERVER_PORT = '10051';
$ZBX_SERVER_NAME = '';

$IMAGE_FORMAT_DEFAULT = IMAGE_FORMAT_PNG;
```

> 如果2个zabbix, 则使用相同的数据库

启动

```bash
[root@k8s-master2 major-MySQLTuner-perl-bf8d1e6]# systemctl start zabbix-agent zabbix-server httpd
[root@k8s-master2 major-MySQLTuner-perl-bf8d1e6]# systemctl enable zabbix-agent zabbix-server httpd
```

## 访问zabbix

http://{your ip}:8082/zabbix



# 配置zabbix agent

```bash
[root@k8s-master2 fonts]# cat /etc/zabbix/zabbix_agentd.conf
PidFile=/var/run/zabbix/zabbix_agentd.pid
LogFile=/var/log/zabbix/zabbix_agentd.log
LogFileSize=10
DebugLevel=3
SourceIP=172.16.0.223 # 本地ip
EnableRemoteCommands=1
LogRemoteCommands=1
Server=172.16.0.222,172.16.0.225,172.16.0.223 # 多个zabbix server, 被动
ListenPort=10050
StartAgents=10
ServerActive=172.16.0.222
Hostname=Zabbix Server
RefreshActiveChecks=120
BufferSize=200
Timeout=30
AllowRoot=1
Include=/etc/zabbix/zabbix_agentd.d/*.conf
UnsafeUserParameters=0
UserParameter=haproxy,/etc/zabbix/scripts/haproxy.sh
UserParameter=mongo_slow,/etc/zabbix/scripts/mongo_slow.py
UserParameter=mysql_slow[*],/etc/zabbix/scripts/mysql_slow.py
```

> server写多个，可能报警就会多次发送
>
> 



