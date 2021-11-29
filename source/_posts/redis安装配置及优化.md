---
title: redis安装配置及优化
date: 2020-11-03 09:54:45
tags: 
toc: true
---

# 1. redis安装

google: redis inurl: release

https://download.redis.io/releases/ 下载不同版本

https://redis.io/download 安装步骤

<!--more-->

## 1.1 yum安装

| IP            | 主机名                     | 配置      | 系统版本        | redis版本 |
| ------------- | -------------------------- | --------- | --------------- | --------- |
| 172.20.100.67 | cd-ch-linux48-redis-100-67 | 1C-1G-50G | centos 7.6 1810 | 3.2.12    |

```bash
# 安装epel源
[root@cd-ch-linux48-redis-100-67 ~]# yum install epel-release -y
# 安装redis
[root@cd-ch-linux48-redis-100-67 ~]# yum install redis -y





# 配置文件
[root@cd-ch-linux48-redis-100-67 ~]# rpm -ql redis
/etc/logrotate.d/redis # 日志切割配置文件
/var/log/redis/*.log { 
    weekly # 每周切割
    rotate 10 # 保留 10 个备份
    copytruncate #使用copytruncate参数，向上面说的，配置了它以后，操作方式是把log 复制一份 成为log.1，然后清空log的内容，使大小为0，那此时log依然时原来的旧log，对进程（nginx）来说，依然打开的是原来的文件描述符，可以继续往里面写日志，而不用发送信号给nginx
    delaycompress # delaycompress 和 compress 一起使用时，转储的日志文件到下一次转储时才压缩
    compress   # 通过gzip 压缩转储以后的日志
    notifempty # 如果是空文件的话，不转储
    missingok  # 在日志轮循期间，任何错误将被忽略，例如“文件无法找到”之类的错误。
}

/etc/redis-sentinel.conf # 哨兵配置
/etc/redis.conf          # redis配置
/etc/systemd/system/redis-sentinel.service.d
/etc/systemd/system/redis-sentinel.service.d/limit.conf
/etc/systemd/system/redis.service.d
/etc/systemd/system/redis.service.d/limit.conf
/usr/bin/redis-benchmark      # 二进制文件
/usr/bin/redis-check-aof
/usr/bin/redis-check-rdb
/usr/bin/redis-cli
/usr/bin/redis-sentinel
/usr/bin/redis-server
/usr/lib/systemd/system/redis-sentinel.service
/usr/lib/systemd/system/redis.service # unit脚本
/usr/libexec/redis-shutdown # 停止redis的脚本
/usr/share/doc/redis-3.2.12 # 文档
/usr/share/doc/redis-3.2.12/00-RELEASENOTES
/usr/share/doc/redis-3.2.12/BUGS
/usr/share/doc/redis-3.2.12/CONTRIBUTING
/usr/share/doc/redis-3.2.12/MANIFESTO
/usr/share/doc/redis-3.2.12/README.md
/usr/share/licenses/redis-3.2.12
/usr/share/licenses/redis-3.2.12/COPYING
/usr/share/man/man1/redis-benchmark.1.gz # man手册路径
/usr/share/man/man1/redis-check-aof.1.gz
/usr/share/man/man1/redis-check-rdb.1.gz
/usr/share/man/man1/redis-cli.1.gz
/usr/share/man/man1/redis-sentinel.1.gz
/usr/share/man/man1/redis-server.1.gz
/usr/share/man/man5/redis-sentinel.conf.5.gz
/usr/share/man/man5/redis.conf.5.gz
/var/lib/redis # 数据目录
/var/log/redis # 日志
/var/run/redis # 运行存放pid



# 启动
[root@cd-ch-linux48-redis-100-67 ~]# systemctl enable redis
Created symlink from /etc/systemd/system/multi-user.target.wants/redis.service to /usr/lib/systemd/system/redis.service.
[root@cd-ch-linux48-redis-100-67 ~]# systemctl start redis
# 验证
[root@cd-ch-linux48-redis-100-67 ~]# lsof -ni :6379 # 默认监听于6379的127.0.0.1地址
COMMAND    PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
redis-ser 6959 redis    4u  IPv4  36445      0t0  TCP 127.0.0.1:6379 (LISTEN)

# 连接
[root@cd-ch-linux48-redis-100-67 ~]# redis-cli 
127.0.0.1:6379> SELECT 0 # 切换库
OK
127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]> SELECT 16
(error) ERR invalid DB index
127.0.0.1:6379[1]> SELECT 15
OK
127.0.0.1:6379[15]> DBSIZE # 当前库的KEY的个数
(integer) 0
127.0.0.1:6379[15]> SET key1 value1 # 设定KEY
OK
127.0.0.1:6379[15]> DBSIZE
(integer) 1
127.0.0.1:6379[15]> KEYS * # 列出当前库所有KEY名
1) "key1"
127.0.0.1:6379[15]> FLUSHDB # 清空当前库所有KEY
OK
127.0.0.1:6379[15]> KEYS * 
(empty list or set)
127.0.0.1:6379[15]> FLUSHALL # 清空所有库所有KEY
OK
127.0.0.1:6379[15]> BGSAVE   # 保存RDB数据文件
Background saving started
127.0.0.1:6379[15]> exit      
[root@cd-ch-linux48-redis-100-67 ~]# ls /var/lib/redis/
dump.rdb
[root@cd-ch-linux48-redis-100-67 ~]# ll -h /var/lib/redis/
总用量 4.0K
-rw-r--r-- 1 redis redis 77 11月  4 11:16 dump.rdb

```



## 1.2 apt安装

| IP            | 主机名                     | 配置      | 系统版本    | redis版本 |
| ------------- | -------------------------- | --------- | ----------- | --------- |
| 172.20.100.66 | cd-ch-linux48-redis-100-66 | 1C-1G-50G | ubuntu 1804 | 4.0.9     |

```bash
# 查看可以安装的版本
root@cd-ch-linux48-redis-100-66:~# apt-cache madison redis
     redis | 5:4.0.9-1ubuntu0.2 | http://mirrors.aliyun.com/ubuntu bionic-updates/universe amd64 Packages
     redis | 5:4.0.9-1ubuntu0.2 | http://mirrors.aliyun.com/ubuntu bionic-updates/universe i386 Packages
     redis | 5:4.0.9-1ubuntu0.2 | http://security.ubuntu.com/ubuntu bionic-security/universe amd64 Packages
     redis | 5:4.0.9-1ubuntu0.2 | http://security.ubuntu.com/ubuntu bionic-security/universe i386 Packages
     redis |  5:4.0.9-1 | http://mirrors.aliyun.com/ubuntu bionic/universe amd64 Packages
     redis |  5:4.0.9-1 | http://mirrors.aliyun.com/ubuntu bionic/universe i386 Packages

# 安装5:5:4.0.9-1ubuntu0.2
root@cd-ch-linux48-redis-100-66:~# apt install redis=5:4.0.9-1ubuntu0.2

# 配置文件
root@cd-ch-linux48-redis-100-66:~# dpkg -L redis-server
/.
/etc
/etc/default
/etc/default/redis-server
/etc/init.d
/etc/init.d/redis-server
/etc/logrotate.d
/etc/logrotate.d/redis-server
/etc/redis
/etc/redis/redis.conf # 配置文件
/lib
/lib/systemd
/lib/systemd/system
/lib/systemd/system/redis-server.service # unit文件
/lib/systemd/system/redis-server@.service
/usr
/usr/bin
/usr/share
/usr/share/doc
/usr/share/doc/redis-server
/usr/share/doc/redis-server/MANIFESTO.gz
/usr/share/doc/redis-server/README.md.gz
/usr/share/doc/redis-server/copyright
/usr/share/man
/usr/share/man/man1
/usr/share/man/man1/redis-server.1.gz
/usr/bin/redis-server
/usr/share/doc/redis-server/00-RELEASENOTES.gz
/usr/share/doc/redis-server/NEWS.Debian.gz
/usr/share/doc/redis-server/changelog.Debian.gz


# 启动
root@cd-ch-linux48-redis-100-66:~# systemctl enable redis-server.service 
Synchronizing state of redis-server.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable redis-server
Created symlink /etc/systemd/system/redis.service → /lib/systemd/system/redis-server.service.
root@cd-ch-linux48-redis-100-66:~# systemctl start redis-server.service 

# 连接
root@cd-ch-linux48-redis-100-66:~# redis-cli 
127.0.0.1:6379> SET key1 value1
OK
127.0.0.1:6379> DBSIZE
(integer) 1
127.0.0.1:6379> KEYS *
1) "key1"
127.0.0.1:6379> FLUSHDB
OK
127.0.0.1:6379> KEYS *
(empty list or set)
127.0.0.1:6379> BGSAVE
Background saving started
127.0.0.1:6379> EXIT
root@cd-ch-linux48-redis-100-66:~# ls /var/lib/redis/
dump.rdb

```



## 1.3 编译安装

```bash
# 展开编译，准备配置
tar xvf redis-3.2.12.tar.gz -C /usr/local/src/
cd /usr/local/src/redis-3.2.12/
make PREFIX=/apps/redis install
install -dv /apps/redis/{bin,etc,logs,data,run}
cp redis.conf /apps/redis/etc/

# 准备Unit
cat >  /lib/systemd/system/redis.service <<'EOF'
[Unit]
Description=Redis persistent key-value database
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/apps/redis/bin/redis-server /apps/redis/etc/redis.conf --supervised systemd
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
Type=notify
User=redis
Group=redis
RuntimeDirectory=redis
RuntimeDirectoryMode=0755
# ubuntu必须设定
LimitNOFILE=65536
# 避免redis宕机
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
EOF

# add user
groupadd -g 2020 redis
useradd -g redis -M -s /sbin/nologin -u 2020 redis
# 配置ownership
chown -R redis.redis /apps/redis/

# 重载unit
systemctl daemon-reload
systemctl enable redis
systemctl start redis
systemctl status redis
#journalctl -xe
#journalctl -xe -u redis
lsof -ni:6379


# 程序
ln -sv /apps/redis/bin/* /usr/local/bin
```

```bash
# ubuntu优化
		touch /etc/rc.local
		cat > /etc/rc.local  <<'EOF'
#!/bin/bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
exit 0
EOF
		systemctl start rc.local
		systemctl status rc.local
		chmod +x /etc/rc.local

# centos优化
		cat > /etc/rc.local  <<'EOF'
#!/bin/bash
touch /var/lock/subsys/local
echo never > /sys/kernel/mm/transparent_hugepage/enabled
EOF
		chmod +x /etc/rc.local

```





# 2. redis配置和优化

编译配置文件 redis.conf

redis配置3.2.12， 获取可用配置`sed 's@#.*@@g' redis.conf | grep -v '^$\|^[[:space:]]*$'`

> 一句话：密码，快照：错误还写？大小：压缩。aof: 大小：重写，主从：同步过程，从读。集群：master有几个slave才正常。

```bash
requirepass linux48 # 配置密码, master和slave就当一样, 这样用户连接slave密码不修改情况可以使用 < MODIFY>
bind 0.0.0.0 # 监听地址,一般内网地址, 不需要公网使用 < MODIFY>
port 6379      # 端口
#改端口的场景:  
#测试环境：  做集群3个机器不够, 一个服务器上开多个redis, 监听不同的端口。（没有意义，没有高可用)
#开发、测试、生产：连接集群和单机的代码不一样，主要开发人员连接测试环境使用集群模式连接redis.

protected-mode yes # 此选项, 没有意义。 只在没有bind, requirepass时, 只监听127.0.0.1。
#redis-cli连接远程地址会报错, 报错中有4个解决方式。

tcp-backlog 511 # tcp等待队列长度，不修改

timeout 0              # java连接redis超时, 默认0，永不超时,一般不修改。
#即使修改，redis可以达到成千上万的连接，不担心连接池撑爆，所以超时可以3-5分钟。

tcp-keepalive 300     ## 如果配置timeout，就让java和redis建立长连接，避免3次握手和断开的开销。 
#mysql连接数上升。
# 1) 容器自动扩展, 连接数会上升。
# 2）连接不释放，1.杀空连接； 2.开发要关连接。
 
daemonize no        
#- no: 前台进程，systemd, docker管理的进程必须运行在前台。 MODIFY     
#- yes: 后台进程, 打开后pid文件默认在/var/run/redis_6379.pid.(多个守护进程使用_port区分。)
pidfile /apps/redis/run/redis_6379.pid  # lsof -ni:6379的PID和cat这个文件一致 < MODIFY>    
supervised no 
#1. no 默认
#2. upstart
#3. systemd
#一般这个选项直接通过启动参数传递给redis-server, 直接传递的优先级比配置文件更高
#/apps/redis/bin/redis-server /apps/redis/etc/redis.conf --supervised systemd#

loglevel notice # debug(排错) - verbose - notice(建议) - warning "一般不修改"
logfile "/apps/redis/logs/redis_6379.log"         # 日志文件位置 < MODIFY> 


databases 16   # 默认16个库, 设置redis DB数量。SELECT用来切换库。"一般不修改" 
# 为什么使用db? 逻辑隔离，同一个db使用相同的key和value会覆盖。不同的db使用相同的key不存在覆盖。
# 怎么用？一个业务或一个应用，一个db



#### 快照配置
save 900 1 # 900s 至少1个key变化, 就触发做rdb快照。 
#先生成一个dump.rdb.temp, 快照完成后将temp替换成dump.rdb。
#手工生成快照 redis-cli> BGSAVE

save 300 10 # 300s 至少10个key变化 
save 60 10000 # 60s 至少10000个key变化 
# 默认规则即可。如果实验，可以配置save 1 1, 能快点看到rdb.

stop-writes-on-bgsave-error no # 默认yes时， redis保存内存数据到磁盘的rdb文件
#如果保存不成功（磁盘满了）,redis就会写入禁止。就会影响业务 < MODIFY>


rdbcompression yes # 压缩
rdbchecksum yes    # 检验

dbfilename dump_6379.rdb   # rdb文件保存的名称, 基于端口, 方便定位是哪个端口生成的数据。 < MODIFY> 
dir /apps/redis/data/ # rdb快照和aof文件保存的目录位置 < MODIFY> 

################################# REPLICATION #####
# 跨主机备份
#主从同步，注意，当同步开始时，会将slave的内存数据清空重新从master拉一份数据。
#slaveof <masterip> <masterport>     # 当前redis作为哪个redis的slave
# masterauth <master-password>       # master角色的redis的密码。

slave-serve-stale-data yes      # slave和master失去联系时
#yes表示继续响应client(一般配置这个选项). 
#no表示不响应。 < MODIFY> 

# user
# haproxy (server master; server slave backup, 当主挂了，slave备用会启用)
# master           slave(backup)


slave-read-only yes           # yes为slave只读。
#(建议只读, 如果有写，可能开发使用了slave地址，写了slave,数据会不一致) < MODIFY> 

repl-diskless-sync no        # 主从同步的选项
#默认(diskless)no, 有盘复制，磁盘IO性能强时使用
#(内存数据复制前会写磁盘，将写的文件传输给slave。
#需要磁盘IO高，好处是多个slave,一个写的文件可以复用)。        

#配置为(diskless)yes时, 当内网性能非常好，磁盘性能差时使用
#(内存数据直接基于socket复制，需要网络好。)。 < MODIFY> 

repl-diskless-sync-delay 5 # 有盘工作时，延迟多久才往slave发数据？

repl-ping-slave-period 10 # slave给master发送ping的周期

repl-timeout 60            # 复制超时，必须大于ping检测时长。否则会一直超时。


repl-disable-tcp-nodelay no # 无盘模式(基于socket复制模式)下
#yes会合并报文，延迟发送, 节省带宽。
#no不会合并报文，没有延迟，但消耗更多的带宽。 < MODIFY> 

repl-backlog-size 512mb     # 复制缓冲大小。
#只有slave连接到master才分配内存, 此内存和redis内存严格分离。
#根据经验redis配置的maxmemory和物理服务器内存各占一点。
#例如：redis配置8G, 物理服务器16G内存, 否则会出很多问题
#(内存不够了，kill redis, 问题就来了)。
#
#当slave和master同步时, 新写的数据会放在缓冲区中, 同步完成后将缓冲区数据在发送给slave。
#所以缓冲区大小根据实际生产在同步完成之前可能占用的空间决定这个缓冲队列大小。 < MODIFY> 

repl-backlog-ttl 3600 # 多长时间没有slave连接master, 就清空backlog缓冲区。

slave-priority 100 # master挂了，会根据slave这个配置的最低优先级来提升为master.
#其他优先级的slave会指向新的slave, 并清空自己的内存数据，重新从新的master同步数据。 < MODIFY> 



# 高危命令生成别名  openssl rand  -hex 17 < MODIFY> 
rename-command CONFIG 01019c6a6f8eb7a643fcab667885a3e8eb # 动态修改配置，
#示例：命令使用: 172.20.100.66:6379> 01019c6a6f8eb7a643fcab667885a3e8eb GET *
rename-command FLUSHDB b85461b1455b4986a4bc2a68de60ce0175  # 清空当前db
rename-command FLUSHALL 8f7e61c47b5da648773c393870d3bfd53c # 清空所有db
rename-command EVAL 15b2e97a0259800ad7a72c65744c1a2b25 # 执行脚本



################ 客户端配置
maxclients 100000 # 最大连接数 < MODIFY> 
# maxmemory <bytes> # 物理主机的一半  free -m | awk '/Mem/{print $2/2*1024*1024}'。 
#4G: 4 * 1024(M) * 1024(K) * 1024(bytes)
# 0 无限制，可以使用当前主机内存所有。
# 物理内存/2. 不包括复制时的缓冲区
# overcommit需要配置为1, 可以申请所有内存。实际申请由maxmemory决定 
maxmemory 509607936   #486M < MODIFY> 
# redis使用完内存需要手动删除键，所以zabbix监控redis当前使用的内存 INFO中的内存。
#当达到 maxmemory * 80%为阈值(407686348 = 388M)时就报警
#~]# redis-cli -h 172.20.100.66 -p 6379  -a linux48 INFO | grep -Po '(?<=used_memory:)\d+'
#822824      #used_memory:822824
# zabbix还需要监控 connected_clients:37




############################## APPEND ONLY MODE ###############################
appendonly yes # 默认不会打开, 此处打开, 相当于记录二进制日志。
#如果aof和rdb同时存在。
#redis中途宕机(rdb的save策略生效前900s只变化2个key,机器断电会丢部分数据)需要aof恢复。
#在断电启动机器时，redis会从aof中读取加载至内存。 < MODIFY> 

appendfilename "appendonly_6379.aof" # aof文件名，也在dir配置的目录下 < MODIFY> 
#
#
# 生产环境不要随便打开文件，先ls, 看看文件大小。
#root@cd-ch-linux48-ubuntu-redis-100-66:~# ll /apps/redis/data/
#-rw-r--r-- 1 redis redis    0 Nov  5 10:09 appendonly_6379.aof
#-rw-r--r-- 1 redis redis   77 Nov  5 10:10 dump_6379.rdb
# 在file看看文件格式
#root@cd-ch-linux48-ubuntu-redis-100-66:~# file /apps/redis/data/appendonly_6379.aof 
#/apps/redis/data/appendonly_6379.aof: ASCII text, with CRLF line terminators
# 确定是ASCII文本，在使用head, tail命令查看。
#一般不使用cat,vim查看，因为如果一个文件几十个G一下内存打满会把机器搞崩的。
#
#root@cd-ch-linux48-ubuntu-redis-100-66:~# head /apps/redis/data/appendonly_6379.aof
#*2
#$6
#SELECT
#$1
#0
#*3
#$3
#SET
#$2
#k1



# appendfsync always  # 每次都同步至磁盘。1s 1000ms, 1000次操作。每次都写, 磁盘IO压力特别高
appendfsync everysec # 每s同步至磁盘。丢1s数据，建议使用此选项 
# appendfsync no # os决定同步至磁盘



### 降低aof文件体积让redis恢复更快###
no-appendfsync-on-rewrite no # 默认no, 在aof重写期间，新的aof记录不暂缓写入磁盘。
#yes，会暂缓写入，后由linux同步策略(默认30s)，同步至磁盘。
# 如果配置yes, 可能丢失30s数据，但是对磁盘压力小。 
#yes性能较好，会避免出现阻塞，因为比较推荐。 
#如果配置了主从redis建议开启，因为有跨主机备份。单机还是别开了 < MODIFY> 
# aof重写, 将上面SET key1 value1, 下面 DELETE key1 value1这种相当于没有写的语句合并起来。让aof文件更小。

auto-aof-rewrite-min-size 64mb # 触发aof rewrite最小文件大小。
auto-aof-rewrite-percentage 100 # aof增长超过此处指定比率开始重写。 0不重写。推荐打开。 64mb -> 128mb

##################

aof-load-truncated yes # 就算aof文件异常(末尾异常, 断电, kill导致)，也从aof加载

#aof-use-rdb-preamble no # redis4.0新增, aof和rdb混合使用。默认no，因为生产还没有大量使用。此选项可以忽略

lua-time-limit 5000 # lua脚本执行超时，很少执行lua脚本，忽略





################################ REDIS CLUSTER  ######
cluster-enabled no #默认no,不打开集群模式 < MODIFY> 
cluster-config-file /apps/redis/etc/nodes-6379.conf # 集群配置文件名称，自动生成 < MODIFY> 
cluster-slave-validity-factor 10 # 故障转移时，超时10s不能做master. < MODIFY> 
# master - slave1 同步数据在12s前
# master - slave2 同步数据在8s前。可以做master


cluster-migration-barrier 1 # 一个master至少拥有多少个正常的slave < MODIFY> 
# 1表示至少一个正常的slave, 从挂了会找一个新的slave
cluster-require-full-coverage yes # 一个16384个槽位分摊到各个master, 
#如果一个master挂了，没有slave有合适的槽位, 就出现槽位不全。
# yes 槽位不全，就不提供服务 < MODIFY> 
# no 槽位不全，也提供服务
# 多个master, 数据放哪个主机？
#数据对所有槽位取模，结果肯定在16384以内, 值是哪个，就在哪个主机上(一个主机是槽位的一个范围)。

slowlog-log-slower-than 10000     # 默认10ms,慢日志, 是保存在内存中，不影响性能。
#为负数会禁用慢日志，为0会记录每个命令操作 < MODIFY> 
#
#微秒 1s=10^3ms=10^6microseconds 1000000 is equivalent to one second
slowlog-max-len 128 # 最多多少条慢日志, 达到最大日志数，下一条日志会从初始日志处开始覆盖写入。
##172.20.100.66:6379> SLOWLOG LEN # 统计慢查询日志个数 , 可以临时配置slowlog-log-slower-than 0，测试长度。
##(integer) 0
##172.20.100.66:6379> SLOWLOG GET # 查慢日志给开发
##(empty list or set)
##172.20.100.66:6379> SLOWLOG RESET #清空慢日志
##OK

########## 忽略
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes
```







## 2.1 配置密码 *

```bash
requirepass linux48
```

```bash
# 重启
root@cd-ch-linux48-ubuntu-redis-100-66:~# systemctl restart redis

# 连接
root@cd-ch-linux48-ubuntu-redis-100-66:~# redis-cli -h 127.0.0.1 -p 6379 # 方式1
127.0.0.1:6379> AUTH linux48 # 认证
OK
127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]> exit
root@cd-ch-linux48-ubuntu-redis-100-66:~# redis-cli -h 127.0.0.1 -p 6379 -a linux48 # 方式2
127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]> exit
root@cd-ch-linux48-ubuntu-redis-100-66:~# redis-cli -h 127.0.0.1 -p 6379 -a linux49 # 错误密码
127.0.0.1:6379> SELECT 1
(error) NOAUTH Authentication required.
127.0.0.1:6379> exit

```



## 2.2 监听地址和端口 *

```bash
bind 127.0.0.1 # 监听地址
port 6379      # 端口

protected-mode yes # 此选项, 没有意义。 只在没有bind, requirepass时, 只监听127.0.0.1。redis-cli连接远程地址会报错。
```



### 示例：配置监听在本地内网地址上 

```bash
bind 172.20.100.66 # 配置地址
port 6379 # 修改端口
requirepass linux48
```

```bash
# 重启
root@cd-ch-linux48-ubuntu-redis-100-66:~# systemctl restart redis
# 
root@cd-ch-linux48-ubuntu-redis-100-66:~# lsof -ni :6379
COMMAND    PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
redis-ser 4553 redis    4u  IPv4  33420      0t0  TCP 172.20.100.66:6379 (LISTEN)

# 远程连接
[root@cd-ch-linux48-centos-redis-100-67 ~]# redis-cli -h 172.20.100.66 -p 6379 -a linux48
172.20.100.66:6379> INFO
# Server
redis_version:3.2.12
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:d29145f9b576ac53
redis_mode:standalone
os:Linux 4.15.0-29-generic x86_64 # 66是ubuntu内核版本高
arch_bits:64
multiplexing_api:epoll
gcc_version:7.5.0
process_id:4553
run_id:48a1656aaeb7300e100cfffe185e88218bb9d5a8
tcp_port:6379
uptime_in_seconds:209
uptime_in_days:0
hz:10
lru_clock:10642076
executable:/apps/redis/bin/redis-server
config_file:/apps/redis/etc/redis.conf

```



### 示例：去掉bind和requirepass

```bash
#bind 172.20.100.66 
port 6379
#requirepass linux48
```

```bash
#
systemctl restart redis
lsof -ni :6379
```

远程连接

```bash
[root@cd-ch-linux48-centos-redis-100-67 ~]# redis-cli -h 172.20.100.66 -p 6379 
172.20.100.66:6379> SELECT 1
(error) DENIED Redis is running in protected mode because protected mode is enabled, no bind address was specified, no authentication password is requested to clients. In this mode connections are only accepted from the loopback interface. If you want to connect from external computers to Redis you may adopt one of the following solutions: 1) Just disable protected mode sending the command 'CONFIG SET protected-mode no' from the loopback interface by connecting to Redis from the same host the server is running, however MAKE SURE Redis is not publicly accessible from internet if you do so. Use CONFIG REWRITE to make this change permanent. 2) Alternatively you can just disable the protected mode by editing the Redis configuration file, and setting the protected mode option to 'no', and then restarting the server. 3) If you started the server manually just for testing, restart it with the '--protected-mode no' option. 4) Setup a bind address or an authentication password. NOTE: You only need to do one of the above things in order for the server to start accepting connections from the outside.
172.20.100.66:6379> 
```

解决方式：

1. 交互式CONFIG SET protected-mode no
2. 编辑配置关闭保护
3. 启动服务时 --protected-mode no
4. 配置bind, requirepass



# 3. rdb和aof对比

rdb：时间点快照，根据save规则来进行快照，或后台执行BGSAVE。

aof：二进制日志，同步进磁盘由no-appendfsync-on-rewrite定义怎么同步，appendfsync everysec多久同步。



|      | 优点                                     | 缺点                                                         | 命令                                           | 恢复    |
| ---- | ---------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------- | ------- |
| rdb  | 只保留最新快照，执行速度快               | 1. 丢失上次快照至本次未快照的数据 2. 不能实时保存数据 3. 数据量大时, 较慢 | bgsave(非阻塞)/save（阻塞)命令自定义时间点备份 | 比aof快 |
| aof  | 按操作顺序将操作实时添加至日志，安全性高 | 1. 操作重复也全部记录 2. 比rdb大                             |                                                |         |





# 4. 消息队列

## 4.1 消息队列类型

1. 生产者(producer)和消费(consumer)者模式
2. 发布者(publisher)和订阅(subscriber)者模式

redis均支持这两种模式

## 4.2 生产者(producer)和消费(consumer)者模式

外部用户请求上层应用，上层将执行结果发到channel中, 下层应用监听channel, 并从中取数据完成后续操作。

多个消费者可以同时监听一个队列，但是一个消息只能被某一个消费者消费。

应用： rabbitmq, kafka, rocketmq, activemq

```bash
# 发布 #从管道的左侧写入
LPUSH channel_name message
# 查看队列消息
LRANGE channel_name 0 -1

# 消费者消费消息 #从管道的右侧消费
RPOP channel1
LRANGE channel1 0 -1 # 消费后队列中无消息
```



## 4.3 发布者(publisher)和订阅(subscriber)者模式

发布者将消息发送到指定channel, 凡是发布前监听了此channel的订阅者均会收到相同的一份消息。

类似对这个channel内的所有订阅者发起广播

```bash
# 订阅#订阅者订阅指定的频道
SUBSCRIBE channel1
# 订阅多个频道
SUBSCRIBE channel1 channel2
# 订阅所有频道
PSUBSCRIBE *
# 订阅匹配的频道
PSUBSCRIBE chann*

# 发布 #发布者发布消息
PUBLISH channel1 test2


```



# 5. redis常用命令

```bash
# 动态修改配置
CONFIG {GET * |SET KEY VALUE} 
# 慢日志
SLOWLOG {GET|RESET|LEN}
# 清空当前db
FLUSHDB
# 清空所有db
FLUSHALL
# 当前db所有key
KEYS *
# 当前db有多少key
DBSIZE
# 切换db
SELECT
```

