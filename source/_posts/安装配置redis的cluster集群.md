---
title: 安装配置redis的cluster集群
date: 2020-11-05 06:57:15
tags: plan
toc: true
---



# 1. 规划

| IP            | 主机名                                         | 配置      | 系统版本        | redis版本 |
| ------------- | ---------------------------------------------- | --------- | --------------- | --------- |
| 172.20.100.66 | cd-ch-linux48-ubuntu-redis-100-66.magedu.local | 1C-2G-50G | ubuntu 1804     | 4.0.14    |
| 172.20.100.67 | cd-ch-linux48-redis-100-67                     | 1C-2G-50G | centos 7.6 1810 | 4.0.14    |
| 172.20.100.68 | cd-ch-linux48-redis-100-68                     | 1C-2G-50G | centos 7.6 1810 | 4.0.14    |

![image-20201109092205893](http://myapp.img.mykernel.cn/image-20201109092205893.png)



# 2. 三台主机安装redis

http://myapp.img.mykernel.cn/install-redis-3.2.12.tar

修改脚本

```bash
# wget https://download.redis.io/releases/redis-4.0.14.tar.gz
# sed -i 's@3.2.12@4.0.14@g' install-redis.sh 
```



<!--more-->

## 2.1. master配置 [172.20.100.66]

注意，config命令一定不能修改, 修改会造成不能同步。

```bash
requirepass linux48 
bind 0.0.0.0
port 6379      
protected-mode yes 
tcp-backlog 511 
timeout 0              
tcp-keepalive 300     
 daemonize no        
 pidfile /apps/redis/run/redis_6379.pid  
 supervised no 
 loglevel notice 
 logfile "/apps/redis/logs/redis_6379.log"         
 databases 16   
 save 900 1 
 save 300 10 
 save 60 10000 
 stop-writes-on-bgsave-error no 
 rdbcompression yes 
 rdbchecksum yes    
 dbfilename dump_6379.rdb   
 dir /apps/redis/data/ 
 slave-serve-stale-data yes      
 slave-read-only yes           
 repl-diskless-sync no        
 repl-diskless-sync-delay 5 
 repl-ping-slave-period 10 
 repl-timeout 60            
 repl-disable-tcp-nodelay no 
 repl-backlog-size 512mb     
 repl-backlog-ttl 3600 
 slave-priority 100 
 #rename-command CONFIG 01019c6a6f8eb7a643fcab667885a3e8eb 
 rename-command FLUSHDB b85461b1455b4986a4bc2a68de60ce0175  
 rename-command FLUSHALL 8f7e61c47b5da648773c393870d3bfd53c 
 rename-command EVAL 15b2e97a0259800ad7a72c65744c1a2b25 
 maxclients 100000 
 maxmemory 1032847360   
 appendonly yes 
 appendfilename "appendonly_6379.aof" 
 appendfsync everysec 
 no-appendfsync-on-rewrite no 
 auto-aof-rewrite-min-size 64mb 
 auto-aof-rewrite-percentage 100 
 aof-load-truncated yes 
 lua-time-limit 5000 
 cluster-enabled no 
 cluster-config-file /apps/redis/etc/nodes-6379.conf 
 cluster-slave-validity-factor 10 
 cluster-migration-barrier 1 
 cluster-require-full-coverage yes 
 slowlog-log-slower-than 10000     
 slowlog-max-len 128 
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

## 2.2. slave配置 [172.20.100.67] [172.20.100.68]

```bash
requirepass linux48 
bind 0.0.0.0
port 6379      
protected-mode yes 
tcp-backlog 511 
timeout 0              
tcp-keepalive 300     
 daemonize no        
 pidfile /apps/redis/run/redis_6379.pid  
 supervised no 
 loglevel notice 
 logfile "/apps/redis/logs/redis_6379.log"         
 databases 16   
 save 900 1 
 save 300 10 
 save 60 10000 
 stop-writes-on-bgsave-error no 
 rdbcompression yes 
 rdbchecksum yes    
 dbfilename dump_6379.rdb   
 dir /apps/redis/data/ 



 slave-serve-stale-data yes      
 slave-read-only yes           
 repl-diskless-sync no        
 repl-diskless-sync-delay 5 
 repl-ping-slave-period 10 
 repl-timeout 60            
 repl-disable-tcp-nodelay no 
 repl-backlog-size 512mb     
 repl-backlog-ttl 3600 
 slave-priority 100 
# rename-command CONFIG 01019c6a6f8eb7a643fcab667885a3e8eb 
 rename-command FLUSHDB b85461b1455b4986a4bc2a68de60ce0175  
 rename-command FLUSHALL 8f7e61c47b5da648773c393870d3bfd53c 
 rename-command EVAL 15b2e97a0259800ad7a72c65744c1a2b25 
 maxclients 100000 
 maxmemory 1032847360   
 appendonly yes 
 appendfilename "appendonly_6379.aof" 
 appendfsync everysec 
 no-appendfsync-on-rewrite no 
 auto-aof-rewrite-min-size 64mb 
 auto-aof-rewrite-percentage 100 
 aof-load-truncated yes 
 lua-time-limit 5000 
 cluster-enabled no 
 cluster-config-file /apps/redis/etc/nodes-6379.conf 
 cluster-slave-validity-factor 10 
 cluster-migration-barrier 1 
 cluster-require-full-coverage yes 
 slowlog-log-slower-than 10000     
 slowlog-max-len 128 
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

 slaveof 172.20.100.66 6379 
 masterauth linux48         

```



## 2.3. 启动所有redis

```bash
root@cd-ch-linux48-ubuntu-redis-100-66:~# systemctl restart redis
[root@cd-ch-linux48-centos-redis-100-67 ~]# systemctl restart redis
[root@chengdu-chenghua-linux48-centos-redis-100-68 ~]# systemctl restart redis


```



master日志

```bash
root@cd-ch-linux48-ubuntu-redis-100-66:~# tail -f /apps/redis/logs/redis_6379.log 
4433:M 09 Nov 09:34:25.038 * Background saving terminated with success
4433:M 09 Nov 09:34:25.039 * Synchronization with slave 172.20.100.67:6379 succeeded
4433:M 09 Nov 09:34:27.198 * Slave 172.20.100.68:6379 asks for synchronization
4433:M 09 Nov 09:34:27.198 * Full resync requested by slave 172.20.100.68:6379
4433:M 09 Nov 09:34:27.198 * Starting BGSAVE for SYNC with target: disk
4433:M 09 Nov 09:34:27.198 * Background saving started by pid 4437
4437:C 09 Nov 09:34:27.211 * DB saved on disk
4437:C 09 Nov 09:34:27.212 * RDB: 0 MB of memory used by copy-on-write
4433:M 09 Nov 09:34:27.276 * Background saving terminated with success
4433:M 09 Nov 09:34:27.276 * Synchronization with slave 172.20.100.68:6379 succeeded

```



slave日志

```bash
[root@cd-ch-linux48-centos-redis-100-67 ~]# tailf /apps/redis/logs/redis_6379.log 
18833:S 09 Nov 09:34:25.052 * MASTER <-> SLAVE sync: Finished with success
18833:S 09 Nov 09:34:25.053 * Background append only file rewriting started by pid 18836
18833:S 09 Nov 09:34:25.133 * AOF rewrite child asks to stop sending diffs.
18836:C 09 Nov 09:34:25.133 * Parent agreed to stop sending diffs. Finalizing AOF...
18836:C 09 Nov 09:34:25.134 * Concatenating 0.00 MB of AOF diff received from parent.
18836:C 09 Nov 09:34:25.134 * SYNC append only file rewrite performed
18836:C 09 Nov 09:34:25.135 * AOF rewrite: 6 MB of memory used by copy-on-write
18833:S 09 Nov 09:34:25.203 * Background AOF rewrite terminated with success
18833:S 09 Nov 09:34:25.204 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
18833:S 09 Nov 09:34:25.204 * Background AOF rewrite finished successfully
[root@chengdu-chenghua-linux48-centos-redis-100-68 ~]# tailf /apps/redis/logs/redis_6379.log 
15783:S 09 Nov 09:34:27.295 * MASTER <-> SLAVE sync: Finished with success
15783:S 09 Nov 09:34:27.296 * Background append only file rewriting started by pid 15786
15783:S 09 Nov 09:34:27.387 * AOF rewrite child asks to stop sending diffs.
15786:C 09 Nov 09:34:27.388 * Parent agreed to stop sending diffs. Finalizing AOF...
15786:C 09 Nov 09:34:27.388 * Concatenating 0.00 MB of AOF diff received from parent.
15786:C 09 Nov 09:34:27.388 * SYNC append only file rewrite performed
15786:C 09 Nov 09:34:27.389 * AOF rewrite: 6 MB of memory used by copy-on-write
15783:S 09 Nov 09:34:27.403 * Background AOF rewrite terminated with success
15783:S 09 Nov 09:34:27.403 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
15783:S 09 Nov 09:34:27.404 * Background AOF rewrite finished successfully

```

连接Master

```bash
~]# time for i in {1..100}; do  /apps/redis/bin/redis-cli  -a linux48 set k$i v$i; done
real	0m1.868s
user	0m0.222s
sys	0m1.049s

```

连接slave

```bash
[root@cd-ch-linux48-centos-redis-100-67 ~]# /apps/redis/bin/redis-cli 
127.0.0.1:6379> AUTH linux48
OK
127.0.0.1:6379> keys *
1) "k1"
127.0.0.1:6379> 
[root@chengdu-chenghua-linux48-centos-redis-100-68 ~]# /apps/redis/bin/redis-cli 
127.0.0.1:6379> auth linux48
OK
127.0.0.1:6379> keys *
1) "k1"
127.0.0.1:6379> get k1
"v1"
127.0.0.1:6379> exit

```



# 3. 三台主机安装sentinel

```bash
bind 0.0.0.0  # 绑定的地址，为了通用直接监听在所有接口上。生产使用应该在内网地址上。
port 26379    
dir /apps/redis/logs # 存放pid,log文件的位置
sentinel monitor mymaster 172.20.100.66 6379 2 # 监控master位置，和哨兵数量/2向上取整
sentinel auth-pass mymaster linux48            # 主节点的密码
sentinel down-after-milliseconds mymaster 15000 # 主观下线时间，毫秒
sentinel parallel-syncs mymaster 1              # 故障转移时，非提升为master的slave节点会向新的master节点进行全量同步数据，这里限制可以同时有几个slave可以同时向新master同步数据。
sentinel failover-timeout mymaster 180000       # slave超时，超出时间认为slave宕机。
daemonize yes                                   # yes以守护进程运行。no为非守护进程，为前台进程启动。
pidfile "redis-sentinel.pid"                    # pid文件名
logfile "sentinel_26379.log"                    # 日志文件名
```



## 3.1 master配置[172.20.100.66]

```bash
root@cd-ch-linux48-ubuntu-redis-100-66:~# cp /usr/local/src/redis-4.0.14/sentinel.conf /apps/redis/etc/

root@cd-ch-linux48-ubuntu-redis-100-66:~# vim /apps/redis/etc/sentinel.conf
bind 0.0.0.0
port 26379
dir "/apps/redis/logs"
sentinel monitor mymaster 172.20.100.66 6379 2
sentinel auth-pass mymaster "linux48"
sentinel down-after-milliseconds mymaster 15000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
daemonize yes
pidfile "redis-sentinel.pid"
logfile "sentinel_26379.log"

root@cd-ch-linux48-ubuntu-redis-100-66:~# scp /apps/redis/etc/sentinel.conf 172.20.100.67:/apps/redis/etc/sentinel.conf
root@cd-ch-linux48-ubuntu-redis-100-66:~# scp /apps/redis/etc/sentinel.conf 172.20.100.68:/apps/redis/etc/sentinel.conf
root@cd-ch-linux48-ubuntu-redis-100-66:~# /apps/redis/bin/redis-sentinel /apps/redis/etc/sentinel.conf 
root@cd-ch-linux48-ubuntu-redis-100-66:~# tailf /apps/redis/logs/sentinel_26379.log 
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

15841:X 09 Nov 11:18:21.814 # Sentinel ID is 6947aedff9e630b7d4dbdb7a048bf2f3c43b8b8a
15841:X 09 Nov 11:18:21.814 # +monitor master mymaster 172.20.100.66 6379 quorum 2
15841:X 09 Nov 11:18:21.824 * +slave slave 172.20.100.67:6379 172.20.100.67 6379 @ mymaster 172.20.100.66 6379
15841:X 09 Nov 11:18:21.826 * +slave slave 172.20.100.68:6379 172.20.100.68 6379 @ mymaster 172.20.100.66 6379
root@cd-ch-linux48-ubuntu-redis-100-66:~#  lsof -ni:26379
COMMAND     PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
redis-sen 15841 root    4u  IPv4  44221      0t0  TCP *:26379 (LISTEN)



#cat >  /etc/rc.local <<'EOF' # 开机自启
#!/bin/bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
/apps/redis/bin/redis-sentinel /apps/redis/etc/sentinel.conf 
exit 0
EOF
# chmod +x /etc/rc.local


root@cd-ch-linux48-ubuntu-redis-100-66:~# scp /etc/rc.local 172.20.100.67:/etc/rc.local
root@cd-ch-linux48-ubuntu-redis-100-66:~# scp /etc/rc.local 172.20.100.68:/etc/rc.local

```



## 3.2 slave配置

```bash
[root@cd-ch-linux48-centos-redis-100-67 ~]# /etc/rc.local
# 66输出的sentinel日志
#5147:X 10 Nov 09:26:22.634 * +sentinel sentinel 362d828d8941a2316542e09f1dc75638eaabafca 172.20.100.67 26379 @ mymaster 172.20.100.66 6379

[root@cd-ch-linux48-centos-redis-100-67 ~]# lsof -ni:26379
COMMAND     PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
redis-sen 15734 root    4u  IPv4  44037      0t0  TCP *:26379 (LISTEN)
redis-sen 15734 root   11u  IPv4  44049      0t0  TCP 172.20.100.67:44643->172.20.100.66:26379 (ESTABLISHED)
redis-sen 15734 root   12u  IPv4  44050      0t0  TCP 172.20.100.67:26379->172.20.100.66:43437 (ESTABLISHED)
[root@cd-ch-linux48-centos-redis-100-67 ~]# tailf /apps/redis/logs/sentinel_26379.log 
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

15734:X 09 Nov 11:20:26.004 # Sentinel ID is 9e868def7ce158e9618cfc59fe5db2fffd562365
15734:X 09 Nov 11:20:26.004 # +monitor master mymaster 172.20.100.66 6379 quorum 2
15734:X 09 Nov 11:20:26.039 * +slave slave 172.20.100.67:6379 172.20.100.67 6379 @ mymaster 172.20.100.66 6379
15734:X 09 Nov 11:20:26.039 * +slave slave 172.20.100.68:6379 172.20.100.68 6379 @ mymaster 172.20.100.66 6379
15734:X 09 Nov 11:20:26.617 * +sentinel sentinel 6947aedff9e630b7d4dbdb7a048bf2f3c43b8b8a 172.20.100.66 26379 @ mymaster 172.20.100.66 6379





[root@chengdu-chenghua-linux48-centos-redis-100-68 ~]# /etc/rc.local 
# 66输出的sentinel日志
#5147:X 10 Nov 09:27:06.744 * +sentinel sentinel 0a80cabc8ffef3b27a332155833f2216ef2dde77 172.20.100.68 26379 @ mymaster 172.20.100.66 6379
[root@chengdu-chenghua-linux48-centos-redis-100-68 ~]# lsof -ni:26379
COMMAND     PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
redis-sen 15727 root    4u  IPv4  44117      0t0  TCP *:26379 (LISTEN)
redis-sen 15727 root   11u  IPv4  44129      0t0  TCP 172.20.100.68:52919->172.20.100.66:26379 (ESTABLISHED)
redis-sen 15727 root   12u  IPv4  44130      0t0  TCP 172.20.100.68:51314->172.20.100.67:26379 (ESTABLISHED)
redis-sen 15727 root   13u  IPv4  44131      0t0  TCP 172.20.100.68:26379->172.20.100.66:59461 (ESTABLISHED)
redis-sen 15727 root   14u  IPv4  44133      0t0  TCP 172.20.100.68:26379->172.20.100.67:12015 (ESTABLISHED)
[root@chengdu-chenghua-linux48-centos-redis-100-68 ~]# tailf /apps/redis/logs/sentinel_26379.log 
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

15727:X 09 Nov 11:20:51.347 # Sentinel ID is c59bb653348debf9e96afd97fdf1edd25ff90a3b
15727:X 09 Nov 11:20:51.347 # +monitor master mymaster 172.20.100.66 6379 quorum 2
15727:X 09 Nov 11:20:51.362 * +slave slave 172.20.100.67:6379 172.20.100.67 6379 @ mymaster 172.20.100.66 6379
15727:X 09 Nov 11:20:51.364 * +slave slave 172.20.100.68:6379 172.20.100.68 6379 @ mymaster 172.20.100.66 6379
15727:X 09 Nov 11:20:51.626 * +sentinel sentinel 6947aedff9e630b7d4dbdb7a048bf2f3c43b8b8a 172.20.100.66 26379 @ mymaster 172.20.100.66 6379
15727:X 09 Nov 11:20:52.574 * +sentinel sentinel 9e868def7ce158e9618cfc59fe5db2fffd562365 172.20.100.67 26379 @ mymaster 172.20.100.66 6379

```



## 3.3 验证哨兵

```bash
root@cd-ch-linux48-ubuntu-redis-100-66:~# /apps/redis/bin/redis-cli -h 172.20.100.66 -p 26379 info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.20.100.66:6379,slaves=2,sentinels=3 # 主节点，slave个数，3个哨兵
root@cd-ch-linux48-ubuntu-redis-100-66:~# /apps/redis/bin/redis-cli -h 172.20.100.67 -p 26379 info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.20.100.66:6379,slaves=2,sentinels=3
root@cd-ch-linux48-ubuntu-redis-100-66:~# /apps/redis/bin/redis-cli -h 172.20.100.68 -p 26379 info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.20.100.66:6379,slaves=2,sentinels=3

```





# 4. 验证主宕机

```bash
root@cd-ch-linux48-ubuntu-redis-100-66:~# systemctl stop redis



root@cd-ch-linux48-ubuntu-redis-100-66:~# tail -f /apps/redis/logs/sentinel_26379.log 
5147:X 10 Nov 09:37:27.177 # +sdown master mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:27.243 # +odown master mymaster 172.20.100.66 6379 #quorum 2/2
5147:X 10 Nov 09:37:27.245 # +new-epoch 2
5147:X 10 Nov 09:37:27.247 # +try-failover master mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:27.254 # +vote-for-leader 1a1988dc27f23d697625a73b2eee4f921320e959 2
5147:X 10 Nov 09:37:27.263 # 0a80cabc8ffef3b27a332155833f2216ef2dde77 voted for 1a1988dc27f23d697625a73b2eee4f921320e959 2
5147:X 10 Nov 09:37:27.276 # 362d828d8941a2316542e09f1dc75638eaabafca voted for 1a1988dc27f23d697625a73b2eee4f921320e959 2
5147:X 10 Nov 09:37:27.314 # +elected-leader master mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:27.315 # +failover-state-select-slave master mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:27.377 # +selected-slave slave 172.20.100.68:6379 172.20.100.68 6379 @ mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:27.380 * +failover-state-send-slaveof-noone slave 172.20.100.68:6379 172.20.100.68 6379 @ mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:27.458 * +failover-state-wait-promotion slave 172.20.100.68:6379 172.20.100.68 6379 @ mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:27.881 # +promoted-slave slave 172.20.100.68:6379 172.20.100.68 6379 @ mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:27.882 # +failover-state-reconf-slaves master mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:27.942 * +slave-reconf-sent slave 172.20.100.67:6379 172.20.100.67 6379 @ mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:27.949 * +slave-reconf-inprog slave 172.20.100.67:6379 172.20.100.67 6379 @ mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:28.454 # -odown master mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:28.963 * +slave-reconf-done slave 172.20.100.67:6379 172.20.100.67 6379 @ mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:29.017 # +failover-end master mymaster 172.20.100.66 6379
5147:X 10 Nov 09:37:29.017 # +switch-master mymaster 172.20.100.66 6379 172.20.100.68 6379
5147:X 10 Nov 09:37:29.020 * +slave slave 172.20.100.67:6379 172.20.100.67 6379 @ mymaster 172.20.100.68 6379
5147:X 10 Nov 09:37:29.024 * +slave slave 172.20.100.66:6379 172.20.100.66 6379 @ mymaster 172.20.100.68 6379
# 注意，当切换后，66和67成为备用节点。66是中断的，所以 + 一个66是主观宕机
5147:X 10 Nov 09:37:44.050 # +sdown slave 172.20.100.66:6379 172.20.100.66 6379 @ mymaster 172.20.100.68 6379

```

<img src="http://myapp.img.mykernel.cn/image-20201110095328886.png" alt="image-20201110095328886"  />

# 5. 监控redis

redis端口，进程        6379

sentinel进程，端口  26379

# 6. 恢复redis

假设运维发现了redis宕机，恢复66

## 6.1 确定master

```bash
[root@chengdu-chenghua-linux48-centos-redis-100-68 ~]# /apps/redis/bin/redis-cli   -p 26379 info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.20.100.68:6379,slaves=2,sentinels=3

```

## 6.2宕机的redis指向新的master

66会重新进行全量同步

```bash
# 仅需要在/apps/redis/etc/redis.conf最后添加2行
slaveof 172.20.100.68 6379
masterauth "linux48"
# 启动redis
systemctl start redis
```

```bash
[root@chengdu-chenghua-linux48-centos-redis-100-68 ~]# tailf /apps/redis/logs/redis_6379.log 

10513:M 10 Nov 09:40:31.674 * Full resync requested by slave 172.20.100.66:6379
10513:M 10 Nov 09:40:31.674 * Starting BGSAVE for SYNC with target: disk
10513:M 10 Nov 09:40:31.675 * Background saving started by pid 10526
10526:C 10 Nov 09:40:31.677 * DB saved on disk
10526:C 10 Nov 09:40:31.677 * RDB: 0 MB of memory used by copy-on-write
10513:M 10 Nov 09:40:31.737 * Background saving terminated with success
10513:M 10 Nov 09:40:31.741 * Synchronization with slave 172.20.100.66:6379 succeeded
```



## 6.3 新架构

![image-20201110095144223](http://myapp.img.mykernel.cn/image-20201110095144223.png)