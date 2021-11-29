---
title: RDB安装及配置优化
date: 2021-02-20 05:20:48
tags:
- mysql
---



# 安装数据库

```bash
[root@k8s-master2 ~]# apt  install mariadb-server -y
```

其他安装略...

# 优化数据库

优化数据库建议 `https://developer.aliyun.com/article/671163` 

```bash
mysql_secure_installation
# wget https://github.com/major/MySQLTuner-perl/tarball/master
# tar xzf master
# cd major-MySQLTuner-perl-7dabf27
# ./mysqltuner.pl


Storage Engine Statistics 
启用的存储引擎
InnoDB 启用，但是没有使用
[root@k8s-master2 major-MySQLTuner-perl-bf8d1e6]# ./mysqltuner.pl | grep '!!'
[!!] Successfully authenticated with no password - SECURITY RISK! # 匿名用户 OK
[!!] Your MySQL version 5.5.68-MariaDB is EOL software!  Upgrade soon!
[!!] InnoDB is enabled but isn't being used
[!!] User ''@'k8s-master2' is an anonymous account. Remove with DROP USER ''@'k8s-master2';
[!!] User ''@'localhost' is an anonymous account. Remove with DROP USER ''@'localhost';
[!!] User ''root'@'127.0.0.1'' has no password set.
[!!] User ''root'@'::1'' has no password set.
[!!] User ''root'@'k8s-master2'' has no password set.
[!!] User ''root'@'localhost'' has no password set.
[!!] name resolution is active : a reverse name resolution is made for each new connection and can reduce performance # 名称解析 OK
[!!] Thread cache is disabled # 线程缓存 
[!!] Key buffer used: 18.2% (24M used / 134M cache) # key buffer
[!!] Read Key buffer hit rate: 80.0% (5 cached / 1 reads) # read key buffer 
[!!] No tables are Innodb 
[!!] InnoDB File per table is not activated # 每表单文件 OK
[!!] Ratio InnoDB log file size / InnoDB Buffer pool size (7.8125 %): 5.0M * 2/128.0M should be equal to 25% # innodb log file大小

-------- Recommendations ---------------------------------------------------------------------------
General recommendations:
    Control warning line(s) into /var/log/mariadb/mariadb.log file
    Add skip-innodb to MySQL configuration to disable InnoDB
    MySQL was started within the last 24 hours - recommendations may be inaccurate
    Reduce your overall MySQL memory footprint for system stability
    Dedicate this server to your database for highest performance.
    Enable the slow query log to troubleshoot bad queries
    Set thread_cache_size to 4 as a starting value
    For MySQL 5.6.2 and lower, Max combined innodb_log_file_size should have a ceiling of (4096MB / log files in group) - 1MB.
    Before changing innodb_log_file_size and/or innodb_log_files_in_group read this: https://bit.ly/2TcGgtU
Variables to adjust:
  *** MySQL's maximum memory usage is dangerously high ***
  *** Add RAM before increasing MySQL buffer variables ***
    thread_cache_size (start at 4)
    innodb_log_file_size should be (=16M) if possible, so InnoDB total log files size equals to 25% of buffer pool size.

```

> 参考 https://linux.cn/article-5730-1.html
>
> 参考 https://www.wencst.com/archives/1781

## 非匿名用户

```bash
[root@k8s-master2 ~]# openssl rand -hex 14 | md5sum | cut -c -8
6ff50115

# mysql_secure_installation
```

```diff
+[root@k8s-master2 ~]# mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

+Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

+Set root password? [Y/n] y
+New password: 
+Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

+Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

+Disallow root login remotely? [Y/n] y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

+Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

+Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!

```

```bash
# 验证密码
[root@k8s-master2 ~]# mysql -p6ff50115
```

## 每表单文件及名称解析

 /etc/my.cnf

```bash
innodb_file_per_table=1 # 每张表一个单独的.idb文件，提升磁盘IO
skip-name-resolve       # 避免主机名反解连接慢
```



## 最大连接数 

 /etc/my.cnf

```bash
max_connections = 10000
```



## 调整内存

如果不调整，大量数据写入时，mysql服务器直接崩溃。

在我把生产的数据导入至mysql预发环境时，mysql直接崩溃，调整此参数后正常

 /etc/my.cnf

```bash
# 每个连接分配
sort_buffer_size = 256M       # 排序缓存
read_buffer_size = 256M        # 查询的缓存
join_buffer_size = 256M        # join的缓存


###################### innodb相关 ######################
innodb_buffer_pool_size =  # 缓存数据和索引 60%-70% 
#free -m | awk '/Mem/{print $NF*0.6}' | awk -F. '{printf "%dM\n",$1}'


innodb_buffer_pool_instances = 4 # 缓冲池实例个数，推荐设置4个或8个
innodb_log_buffer_size = 16M     # 由于日志最长每秒钟刷新一次，所以一般不用超过16M
innodb_flush_log_at_trx_commit = 1 
# 0 每s同步刷写磁盘。
# 1 每1条sql同步刷写磁盘。 正式环境建议为1
# 2 先写缓存，后续在写磁盘，性能最好，但是断开会丢更多的数据。


###################### Myisam ######################
# 生成索引缓存
key_buffer_size = 256M        # Myisam索引缓存 

# 打开查询缓存
query_cache_type = ON
query_cache_limit = 1M
query_cache_size = 32M  

# 打开文件数目, 和系统限制一样即可
open_files_limit = 52706963


# 内存表和临时表
tmp_table_size= 64M #默认大小是 32M。GROUP BY 多不多的问题
max_heap_table_size= 64M

# 网络传输中一次消息传输量的最大值。系统默认值 为1MB，最大值是1GB，必须设置1024的倍数。
max_allowed_packet = 32M


```

> 如果运行在docker中需要将mysql的资源限制取消



mysql大量使用内存，可以使用60%, 但是系统在使用swap规则是60%，可能就出现使用swap的情况，需要把系统的参数修改使用swap在80%

```bash
echo vm.swappiness = 80 >> /etc/sysctl.conf 
sysctl -p

# 验证
root@uat-1:/mnt/docker-jar# sysctl vm.swappiness 
vm.swappiness = 80
```



## wait_timeout

空闲连接超时, 如果不调整，会有大量连接为sleep在mysql中，会消耗mysql内存来保持这个连接.

参数默认值：28800秒（8小时)

```bash
root@uat-1:/mnt/docker-jar# mysqladmin processlist -u root -p -h127.0.0.1 | grep  Sleep
mysqladmin: [Warning] Using a password on the command line interface can be insecure.
| 33   | root | 222.211.249.91:11778 | zhibo | Sleep   | 914  |        |                                                         
```

 /etc/my.cnf

```bash
wait_timeout = 300
```

其他时间参考：https://sre.ink/mysql-timeout-2/

## 慢日志和错误日志

 Enable the slow query log to troubleshoot bad queries

启动错误日志查看错误

 /etc/my.cnf

```bash
# 慢查询日志文件
slow_query_log=ON
log_output=FILE
long_query_time=3
slow_query_log_file=/data/mysql/slow.log
log_queries_not_using_indexes=ON
# 错误日志
log_error=/data/mysql/error.log
```



## 时间点还原

通过全库恢复+binlog
 /etc/my.cnf

```bash
#开启mysql的binlog日志功能
log-bin = /data/mysql-bin     
binlog_format = mixed   						#binlog日志格式，mysql默认采用statement，建议使用mixed
expire_logs_days = 7                           	#binlog过期清理时间
max_binlog_size = 100m                    		#binlog每个日志文件大小
binlog_cache_size = 4m                        	#binlog缓存大小
max_binlog_cache_size= 512m             		#最大binlog缓存大
```

## 数据库的位置

 /etc/my.cnf

```bash
[mysqld]
datadir=/var/lib/mysql 					# 应该是挂载的盘
socket=/var/lib/mysql/mysql.sock 		# sock文件
```

## 合并配置

> 一句话概述：mysql优化 数据目录、连接数、日志(错误、慢、二进制、通用)、缓存（重要数据、查询相关）。

```diff
# this is only for the mysqld standalone daemon
[mysqld]

#
# * Basic Settings
#
# 用户
user		= mysql
pid-file	= /data/mysql/mysqld.pid
socket		= /data/mysql/mysqld.sock
port		= 3306
basedir		= /usr/local/mysql/
#[Modify]
+datadir		= /data/mysql
#[Modify]
# 备份服务器不应该设置为可以清除的目录，slave服务器依赖临时文件来记录复制
+#tmpdir		= /tmp
#这意味着您不能在同一数据目录上运行两个 mysqld服务器，并且如果使用myisamchk则必须小心 。尽管如此，尝试使用该选项作为测试可能会很有帮助
skip-external-locking


# Instead of skip-networking the default is now to listen only on
# localhost which is more compatible and is not less secure.
+bind-address		= 0.0.0.0

#
# * Fine Tuning
#
key_buffer_size		= 16M
max_allowed_packet	= 16M
thread_stack		= 192K
# cpu数量 * 8
thread_cache_size       = 8
# This replaces the startup script and checks MyISAM tables if needed
# the first time they are touched
# FORCE表坏了可以自动恢复
myisam_recover_options  = BACKUP,FORCE
#[Modify]
+max_connections        = 10000
#table_cache            = 64
table_open_cache       = 64
thread_concurrency     = 10


#
# * Query Cache Configuration
#
#[Modify]
+query_cache_limit	= 1M
+query_cache_size        = 16M


#
# * Logging and Replication
#
# Both location gets rotated by the cronjob.
# Be aware that this log type is a performance killer.
# As of 5.1 you can enable the log at runtime!
#[Modify]
general_log_file        = /data/mysql/genral.log
general_log             = 1
#
# Error log - should be very few entries.
#
#[Modify]
+log_error = /data/mysql/error.log
#
# Enable the slow query log to see queries with especially long duration
#[Modify]
+slow_query_log_file	= /data/mysql/slow.log
+long_query_time = 1
#log-queries-not-using-indexes
#
# The following can be used as easy to replay backup logs or for replication.
# note: if you are setting up a replication slave, see README.Debian about
#gunzip -c /usr/share/doc/mariadb-server-10.1/README.Debian.gz | less
# slave不应该使用tmpdir
#       other settings you may need to change.
#[Modify]
+server-id		= 2
#[Modify] slave not need.
+#log_bin			= /var/log/mysql/mysql-bin.log
#二进制日志是拯救你的最后一根稻草，除非磁盘空间不够，不应该删除
+#expire_logs_days	= 10
+max_binlog_size   = 100M

# 写二进制日志时，包含或忽略
#binlog_do_db		= include_database_name
#binlog_ignore_db	= exclude_database_name
# slave复制时忽略
#replicate_do_db
#replicate_do_table
#replicate_ignore_db
#replicate_ignore_table
# 通配
#replicate_wild_ignore_table = mysql.%
#replicate_wild_do_table

#
# * InnoDB
#
# InnoDB is enabled by default with a 10MB datafile in /var/lib/mysql/.
# Read the manual for more InnoDB related options. There are many!

# 打开文件数目, 和系统限制一样即可
+open_files_limit = 52706963

slow_query_log=ON                       
log_output=FILE

# 每张表一个单独的.idb文件，提升磁盘IO
+innodb_file_per_table=1 
# 避免主机名反解连接慢
+skip-name-resolve       

# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
#[Modify]
+innodb_buffer_pool_size = 128M


sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES

# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.                                                
+join_buffer_size = 128M
+sort_buffer_size = 2M
+read_rnd_buffer_size = 2M 


+wait_timeout = 300


#
# * Security Features
#
# Read the manual, too, if you want chroot!
#[Modify]
#chroot = /data/mysql/
#
# For generating SSL certificates you can use for example the GUI tool "tinyca".
#
# ssl-ca=/etc/mysql/cacert.pem
# ssl-cert=/etc/mysql/server-cert.pem
# ssl-key=/etc/mysql/server-key.pem
#
# Accept only connections using the latest and most secure TLS protocol version.
# ..when MariaDB is compiled with OpenSSL:
# ssl-cipher=TLSv1.2
# ..when MariaDB is compiled with YaSSL (default in Debian):
# ssl=on

#
# * Character sets
#
# MySQL/MariaDB default is Latin1, but in Debian we rather default to the full
# utf8 4-byte character set. See also client.cnf
#
character-set-server  = utf8mb4
collation-server      = utf8mb4_general_ci
```

