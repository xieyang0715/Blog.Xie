---
title: mysql主从复制实现
date: 2021-03-23 08:30:55
tags:
- mysql
---

# 前言

mysql软件采用双授权，CE和EE。基于体积小、速度快，开放源码，中小网站开发都选择mysql作为网站数据库。

> 有参考: 
>
> https://github.com/qiurunze123/miaosha/blob/master/docs/mysql-master-slave.md

<!--more-->
# mysql主从复制

mysql主从是实现mysql高可用、数据备份、读写分离架构的一种最常见的解决方案，在绝大多数公司都有使用，要实现mysql主从复制，master(bin-log, server_id), 因为整个复制过程就是slave获取master的二进制日志。然后slave端顺序的执行日志中记录的各种操作，二进制日志中几乎记录了除select以外的所有针对数据库的sql操作语句。

## 过程

### 新建slave同步master

slave端io线程连接master, 请求指定日志文件的指定位置之后的日志

slave io线程接收日志写入relay-log, 并将读取到的bin-log和位置记录在master-info文件中，以便下一次读取的时候能够清楚的告诉master“我需要从哪个bin-log哪个位置开始，往后的日志请发给我”。

slave sql线程检查到relay-log新增内容后，会马上将relay-log中的内容解析为在master端真实执行的命令，并顺序执行，保证slave的mysql完成相同的增加或删除，最终实现和master数据一致。

## 减小主从同步延时方案

### slave SSD数据盘

从库快速同步主数据写入relay-log

### master避免大量写入

可以使用非关系型数据库的尽量使用非关系型数据库

### master和slave专用网络

### 主从一致要求高场景，不查询从库，数据可能未及时同步

### 读场景大的，多个从库，将读请求分散给多个slave

## 数据库延迟复制方案

### 数据库误操作后，快速恢复

误删除表，slave的延迟操作比当前操作时间长的slave, slave的数据还没有变化，可以使用这个slave进行数据恢复，然后把一小时内的bin-log补写到master即可。（需要slave有二进制日志）

### 延迟测试

做好的数据库读写分离，把从库作为读库，即测试主从数据延迟五分钟或指定的时候，业务会发生什么样的问题，即可以模拟数据库延迟指定时间的业务错误。

### 老数据查询

经常需要查看某天前的一个表或者字段的数值，你可能需要把备份恢复之后才能进行查看，但是如果有延迟从库，比如延迟一周，那么久可以解决这一的类似的业务需求场景。

### 设置延迟复制

```bash
#SLAVE上的MASTER TO MASTER_DELAY参数来实现：
CHANGE MASTER TO MASTER_DELAY = N;

#N为多少秒，该语句设置从库延时N秒后再与主数据库进行同步复制
```

具体MySQL延迟复制操作，登录到SLAVE之上进行操作：

```bash
mysql> stop slave;
mysql> CHANGE MASTER TO MASTER_DELAY = 600; 
mysql> start  slave;
mysql> SHOW SLAVE STATUS \G; 
```

## 生产mysql经验 

### 线上授权用户不能delete

生产应用MySQL用户，不允许执行delete，可以只授权给update、select、insert，可以把delete更改为update，将要删除的数据加一个字段标记为已删除，因此线上的业务可以不需要delete

### 在有条件的情况下做一个延迟一小时等制定时间的从库

### 所有DML操作之前必须备份

insert、update、delete、merge

### 规范开发在update数据库必须有两个脚本

数据库修改脚本，修改数据库的脚本

数据库回滚脚本，当数据更新失败的时候用于回滚

## DML操作备份脚本

```bash
dml_backup.sh  stable_name  #输入表明即可备份表
```

# 主从复制实现

## 安装mysql

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-23
#FileName：             install.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************
if which apt &> /dev/null; then
    apt install libaio-dev -y
else
    yum install libaio-devel -y
fi


[ -f mysql-5.6.49-linux-glibc2.12-x86_64.tar.gz ] || wget http://myapp.img.qiniu.mykernel.cn/mysql-5.6.49-linux-glibc2.12-x86_64.tar.gz 
tar xvf mysql-5.6.49-linux-glibc2.12-x86_64.tar.gz -C /usr/local/
ln -sv /usr/local/mysql-5.6.49-linux-glibc2.12-x86_64/ /usr/local/mysql
groupadd -r mysql
useradd -r -s /bin/nologin -g mysql -m -d /home/mysql mysql
chown -R mysql:mysql /usr/local/mysql
install -dv -o mysql -g mysql /data/mysql
install -m 644 -o mysql -g mysql my.cnf /home/mysql/.my.cnf


# 初始化
su -s /bin/bash mysql -c '/usr/local/mysql/scripts/mysql_install_db --user=mysql --basedir=/usr/local/mysql/'

cp mysql.service /etc/systemd/system/mysql.service
systemctl daemon-reload
systemctl enable mysql
systemctl status mysql
systemctl start mysql
systemctl status mysql

# 
lsof -ni:3306


# prog
echo '
export PATH=/usr/local/mysql/bin/:$PATH' > /etc/profile.d/mysql.sh
source /etc/profile.d/mysql.sh

# 链接
ln -sv /data/mysql/mysql.sock /tmp


echo '请另启终端访问mysql
PLEASE REMEMBER TO SET A PASSWORD FOR THE MySQL root USER !
To do so, start the server, then issue the following commands:

  /usr/local/mysql//bin/mysqladmin -u root password 'new-password'
    /usr/local/mysql//bin/mysqladmin -u root -h 172.16.100.77 password 'new-password'

	Alternatively you can run:

	  /usr/local/mysql//bin/mysql_secure_installation
'
```



## [配置及优化](http://blog.mykernel.cn/2021/02/20/RDB%E5%AE%89%E8%A3%85%E5%8F%8A%E9%85%8D%E7%BD%AE%E4%BC%98%E5%8C%96/#%E5%90%88%E5%B9%B6%E9%85%8D%E7%BD%AE)



## master

主要确保有id，二进制日志

```bash
[mysqld]
server_id = 10
log-bin   = /data/mysql/mysql-bin     # 主从复制基础，增量备份
```



## slave

```bash
[mysqld]
relay-log = /data/mysql #  SQL线程写的日志
server_id = 20
log-bin   =  /data/mysql/mysql-bin    # 基于此二进制日志完成恢复
```



## master创建同步账号

```bash
MySQL [(none)]> GRANT REPLICATION SLAVE,REPLICATION CLIENT ON *.* TO 'magedu'@'192.168.0.%' IDENTIFIED BY 'mageedu'; # 授权magedu用户从192.168.0.%主机使用mageedu密码同步 
Query OK, 0 rows affected (0.00 sec)
```

## master导出数据，复制到slave

```bash
# InnoDB
[root@elk ~]# mysqldump -uroot -p --single-transaction  --skip-add-locks --skip-lock-tables --all-databases  --flush-logs --master-data=2  > all-$(date +%F).sql
Enter password: 


[root@elk ~]# scp all-2021-03-23.sql 192.168.0.92:
all-2021-03-23.sql                                                                                                                                                                                                                                                                       100% 1959KB  11.2MB/s   00:00    
```

##  slave导入数据，记录哪个bin-log什么位置

```bash
root@master04:~# mysql < all-2021-03-23.sql 
```

```bash
root@master04:~# head -n 30 all-2021-03-23.sql  | grep CHANGE
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000023', MASTER_LOG_POS=120;

```



## slave同步，设置slave只读

```bash
mysql> CHANGE MASTER TO    MASTER_HOST='192.168.0.171',MASTER_USER='magedu',MASTER_PASSWORD='mageedu',MASTER_LOG_FILE='mysql-bin.000023',MASTER_LOG_POS=120;
Query OK, 0 rows affected, 2 warnings (0.09 sec)

mysql> show global variables like "%read_only%";
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| innodb_read_only | OFF   |
| read_only        | OFF   |
| tx_read_only     | OFF   |
+------------------+-------+
3 rows in set (0.00 sec)

mysql> set global read_only=1; #开启只读模式,只读模式仅对普通用户生效，对root不生效
Query OK, 0 rows affected (0.00 sec)

mysql> show global variables like "%read_only%"; #验证是否生效
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| innodb_read_only | OFF   |
| read_only        | ON    |
| tx_read_only     | OFF   |
+------------------+-------+
3 rows in set (0.00 sec)

mysql> 
```

## 验证Slave的IO线程与SQL线程是OK的

```bash
mysql> start slave;
Query OK, 0 rows affected (0.01 sec)

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.0.171
                  Master_User: magedu
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000023
          Read_Master_Log_Pos: 120
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 283
        Relay_Master_Log_File: mysql-bin.000023
             Slave_IO_Running: Yes #这两个线程状态必须为YES才表示同步是成功的
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 120
              Relay_Log_Space: 456
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0 #表示从Master收到的bin-log到当前的同步位置差多少秒，不是严格意义的主从差多少秒，因为Master的bin-log更新以后可能还没有同步到Slave
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 136ebafb-edc0-11ea-b684-00cfe04c7dc1
             Master_Info_File: /data/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
1 row in set (0.00 sec)

ERROR: 
No query specified

```



## 关于Master_Info_File

```bash
root@master04:~# cat /data/mysql/master.info
23
mysql-bin.000023 #Master bin-log的文件名
120              #Master的Position位置 
192.168.0.171 #Master地址
magedu        # 同步用户
mageedu       # 同步pass
3306          # 同步端口
60
0





0
1800.000

0
136ebafb-edc0-11ea-b684-00cfe04c7dc1
86400


0

```

### master信息

```bash
MySQL [(none)]> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000023 |      120 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

```

## 从只读

为了保证主从同步可以一直进行，在slave库上要保证具有super权限的root等用户只能在本地登录，不会发生数据变化，其他远程连接的应用用户只按需分配为select,insert,update,delete等权限。

保证没有super权限，则只需要将salve设定“read_only=1”模式，即可保证主从同步，又可以实现从库只读。



## 在Master 新插入数据，验证是否回同步到Slave

```bash
MySQL [(none)]> use test;
Database changed

MySQL [test]> CREATE TABLE tb1 (id int);
Query OK, 0 rows affected (0.01 sec)

```

### slave验证

```bash
mysql> use test
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| tb1            |
+----------------+
1 row in set (0.00 sec)

mysql> 

```

## 主从同步监控

### mysql exporter

```bash
root@master04:~# tar xvf mysqld_exporter-0.12.1.linux-amd64.tar.gz -C /usr/local/
mysqld_exporter-0.12.1.linux-amd64/
mysqld_exporter-0.12.1.linux-amd64/NOTICE
mysqld_exporter-0.12.1.linux-amd64/mysqld_exporter
mysqld_exporter-0.12.1.linux-amd64/LICENSE
root@master04:~# ln -sv /usr/local/mysqld_exporter-0.12.1.linux-amd64/ /usr/local/mysqld_exporter
'/usr/local/mysqld_exporter' -> '/usr/local/mysqld_exporter-0.12.1.linux-amd64/'
root@master04:~# cat ~/.my.cnf 
[client]
user=exporter
password=exporter
host=localhost

root@master04:~# /usr/local/mysqld_exporter/mysqld_exporter
INFO[0000] Starting mysqld_exporter (version=0.12.1, branch=HEAD, revision=48667bf7c3b438b5e93b259f3d17b70a7c9aff96)  source="mysqld_exporter.go:257"
INFO[0000] Build context (go=go1.12.7, user=root@0b3e56a7bc0a, date=20190729-12:35:58)  source="mysqld_exporter.go:258"
INFO[0000] Enabled scrapers:                             source="mysqld_exporter.go:269"
INFO[0000]  --collect.global_status                      source="mysqld_exporter.go:273"
INFO[0000]  --collect.global_variables                   source="mysqld_exporter.go:273"
INFO[0000]  --collect.slave_status                       source="mysqld_exporter.go:273"
INFO[0000]  --collect.info_schema.innodb_cmp             source="mysqld_exporter.go:273"
INFO[0000]  --collect.info_schema.innodb_cmpmem          source="mysqld_exporter.go:273"
INFO[0000]  --collect.info_schema.query_response_time    source="mysqld_exporter.go:273"
INFO[0000] Listening on :9104                            source="mysqld_exporter.go:283"

# CTRL +C
root@master04:~# nohup /usr/local/mysqld_exporter/mysqld_exporter & # 后台

root@master04:~# lsof -ni:9104
COMMAND    PID USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME
mysqld_ex 8025 root    3u  IPv6 2220906      0t0  TCP *:9104 (LISTEN)
```

### prometheus添加endpoint

```bash
    - job_name: 'mysql' 
      static_configs:
      - targets: ['192.168.0.171:9104','192.168.0.92:9104']


# 更新yaml
root@master01:/data/weizhixiu/yaml/prometheus# kubectl apply -f prometheus-configmap.yaml

```

#### 验证pod中已经有数据

![image-20210323180843132](http://myapp.img.mykernel.cn/image-20210323180843132.png)

#### 热重载

```bash
curl --location --request POST 'http://prometheus.youwoyouqu.io/-/reload'
```

### 查看prometheus

![image-20210323181604049](http://myapp.img.mykernel.cn/image-20210323181604049.png)

### 配置prometheus rules

```bash
# io线程不运行
	- alert: MysqlSlaveIoThreadNotRunning
        expr: mysql_slave_status_master_server_id > 0 and ON (instance) mysql_slave_status_slave_io_running == 0 
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: MySQL Slave IO thread not running (instance {{ $labels.instance }})
          description: "MySQL Slave IO thread not running on {{ $labels.instance }}\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"
# sql线程不运行
      - alert: MysqlSlaveSqlThreadNotRunning
        expr: mysql_slave_status_master_server_id > 0 and ON (instance) mysql_slave_status_slave_sql_running == 0 
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: MySQL Slave SQL thread not running (instance {{ $labels.instance }})
          description: "MySQL Slave SQL thread not running on {{ $labels.instance }}\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

# 复制延迟大于30s
      - alert: MysqlSlaveReplicationLag
        expr: mysql_slave_status_master_server_id > 0 and ON (instance) (mysql_slave_status_seconds_behind_master - mysql_slave_status_sql_delay) > 30 
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: MySQL Slave replication lag (instance {{ $labels.instance }})
          description: "MySQL replication lag on {{ $labels.instance }}\n  VALUE : {{ $value }}\n  LABELS: {{ $labels }}"

```

### 测试停止slave io线程

```bash
mysql> stop slave io_thread;
Query OK, 0 rows affected (0.01 sec)
```

prometheus报警

![image-20210323183028438](http://myapp.img.mykernel.cn/image-20210323183028438.png)

![image-20210323183113887](http://myapp.img.mykernel.cn/image-20210323183113887.png)

### 恢复slave io线程

```bash
mysql> start slave io_thread;
Query OK, 0 rows affected (0.00 sec)

```

![image-20210323183257166](http://myapp.img.mykernel.cn/image-20210323183257166.png)



# 提升新主服务器

## 停止主写入

让所有客户端（不是复制连接）退出，如果使用虚拟IP，简单关闭虚拟IP，然后断开客户端连接以关闭开放的事务。

可选利用, 停止写入。或read_only选项把主设定为只读

```bash
FLUSH TABLES WITH READ LOCK
```

**haproxy下线主**

## 选择从成为新主

从服务把read log执行完

验证新主与旧主数据是否一致



### 决定哪个从有最新数据

每个从检查

```bash
SHOW SLAVE STATUS

# 选择Master_Log_File/Read_Master_Log_Pos的坐标最新的那个服务器。
```

mysqldump在从服务器二进制日志或中继日志检查最后执行的事件

### 中继日志执行完



## 新主上，停止复制

```bash
STOP SLAVE
```

## 新主执行

```bash
CHANGE MASTER TO MASTER_HOST=''

RESTE SLAVE # 断开与老主服务器连接，并且丢掉master.info连接信息。如果master.info定义在my.cnf，此方式无法工作
```

```bash
SHOW MASTER STATUS # 了解新主二进制日志坐标。
```

## 比较每个从服务器的Master_Log_File/Read_Master_Log_Pos的坐标



## 其他从指向指向新主

haproxy上线新主

## 关闭旧主



