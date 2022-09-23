---
title: 实战，一小时搭建mysql主从同步
date: 2022-09-23 11:24:46
categories: MySQL
toc: true

---

主从复制原理主要有三个线程不断在工作：

 - 主（master）数据库启动bin二进制日志，这样会有一个Dump线程，这个线程是把主（master）数据库的写入操作都会记录到这个bin的二进制文件中。
 - 然后从（slave）数据库会启动一个I/O线程，这个线程主要是把主（master）数据库的bin二进制文件读取到本地，并写入到中继日志（Relay log）文件中。
 - 最后从（slave）数据库其他SQL线程，把中继日志（Relay log）文件中的事件再执行一遍，更新从（slave）数据库的数据，保持主从数据一致。


配置：
当前实战部署在一台服务器多个容器中
 - CentOS Linux release 7.9.2009 (Core)
 - Mysql8+
 - Docker

<!--more-->
## 搭建master ##

 1. 添加mysql主数据库容器
    docker run -itd -p 3307:3306 --name mysql-master -v /data/mysql/master/mysql-files:/var/lib/mysql-files -v /data/mysql/master/log:/var/log/mysql -v /data/mysql/master/data:/var/lib/mysql -v /data/mysql/master/conf:/etc/mysql -e MYSQL_ROOT_PASSWORD=root mysql

 2. 修改主库配置mysql服务的id，配置主从需要同步的库
 vi /data/mysql/slave1/conf/my.cnf

```
[client]
default-character-set=utf8

[mysql]
default-character-set=utf8

[mysqld]
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
skip-character-set-client-handshake
skip-name-resolve
# 注意：skip-name-resolve 一定要加，不然连接 mysql 会超级慢

# 以下是添加 master 主从复制部分配置
#每个mysql服务的id不能相同
server_id=1
#开启二进制
log-bin=mysql-bin
#表示只读
read-only=1
#表示需要同步的库
binlog-do-db=test1
binlog-do-db=test2
#表示不需要同步的库
replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
```

 3. 重启master
 3. 创建数据库（存在则跳过）
 4. 为master 授权用户来同步他的数据
```
 create user backup@'%' identified by '123456';
grant REPLICATION SLAVE on *.* to backup@'%'; 
show master status; 
```

## 搭建slave ##

 1. 添加从数据库
 ```
docker run -itd -p 3336:3306 --name mysql-slave1 -v /data/mysql/slave1/mysql-files:/var/lib/mysql-files -v /data/mysql/slave1/log:/var/log/mysql -v /data/mysql/slave1/data:/var/lib/mysql -v /data/mysql/slave1/conf:/etc/mysql -e MYSQL_ROOT_PASSWORD=root mysql
vi /data/mysql/slave1/conf/my.cnf
 ```
 2. 修改从库配置mysql服务的id，配置主从需要同步的库
 vi /data/mysql/slave1/conf/my.cnf
```
[client]
default-character-set=utf8

[mysql]
default-character-set=utf8

[mysqld]
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
skip-character-set-client-handshake
skip-name-resolve
# 注意：skip-name-resolve 一定要加，不然连接 mysql 会超级慢

# 以下是添加 master 主从复制部分配置
#每个mysql服务的id不能相同
server_id=2
#开启二进制
log-bin=mysql-bin
#表示只读
read-only=1
#表示需要同步的库
binlog-do-db=test1
binlog-do-db=test2
#表示不需要同步的库
replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
```
 3. 重启slave
 4. 查看slave的状态
 ```
show slave status\G;
```
 5. 导出主库数据到从库
 6. 配置 slaver 同步 master 数据，设置主库连接
 ```
change master to master_host='xxx.xxx.xxx.xxx',master_user='backup',master_password='123456',master_log_file='mysql-bin.000002',master_log_pos=1573,master_port=3307;
```
注：

 - master_host为主库的地
 - master_user为用户名
 - master_password为密码
 - master_log_file为上一步mysql-master,show master status查出来的File
 - master_log_pos为mysql-master,show master status查出来的Position
 - master_port为主库端口

 7. 开启主从复制
 ```
start slave;
```
 8. 查看从库状态
  ```
show slave status\G;
```
Slave_IO_Running,Slave_SQL_Running两个为Yes就成功了
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
 9. master测试新增数据
 

