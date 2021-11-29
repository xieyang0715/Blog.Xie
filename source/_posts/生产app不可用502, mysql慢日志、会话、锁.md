---
title: 生产app不可用502, mysql慢日志、会话、锁
date: 2020-12-03 02:31:13
tags: 生产小经验
toc: true
---



今天在打开生产app出现502, 立即连接上k8s-master1, 查看主服务在重启。

kibana查看后台日志, mysql报错

mysql慢日志，cpu

<!--more-->

# 1. kibana查看后台日志

![image-20201203103308785](http://myapp.img.mykernel.cn/image-20201203103308785.png)

# 2. mysql 查看慢日志、监控

![image-20201203114729360](http://myapp.img.mykernel.cn/image-20201203114729360.png)

![image-20201203114833813](http://myapp.img.mykernel.cn/image-20201203114833813.png)

找开发原来是最近加了mysql事件, 大量执行语句，他们去掉



# 3.  mysql 会话

![image-20201203115140307](http://myapp.img.mykernel.cn/image-20201203115140307.png)

开发说可以杀所有会话, 杀掉恢复。

```bash
SHOW PROCESSLIST ;

# 过滤
mysql -e 'SHOW FULL PROCESSLIST ;' | grep  third_task_info
```



再去查看innodb的事务表INNODB_TRX，看下里面是否有正在锁定的事务线程，看看ID是否在show full processlist里面的sleep线程中，如果是，就证明这个sleep的线程事务一直没有commit或者rollback而是卡住了，我们需要手动kill掉。

```mysql
SELECT * FROM information_schema.INNODB_TRX;
# trx_mysql_thread_id: 9930577


SELECT  concat('KILL ',trx_mysql_thread_id,';') FROM information_schema.INNODB_TRX;
```

看到有这条9930577的sql，kill掉，执行kill 9930577;



# 4. mysql死锁

```mysql
MySQL [(none)]> show status like 'innodb_row_lock%';
+-------------------------------+------------+--------------------------------+
| Variable_name                 | Value      | Description                    |
+-------------------------------+------------+--------------------------------+
| Innodb_row_lock_current_waits | 43         |  当前等待row lock的数目          |
| Innodb_row_lock_time          | 2600509530 |  获取row lock花费的总时间，单位毫秒|
| Innodb_row_lock_time_avg      | 47165      |  平均时间                       |
| Innodb_row_lock_time_max      | 51994      |  最大时间                       |
| Innodb_row_lock_waits         | 55136      |  操作等待row lock的数目          |
+-------------------------------+------------+--------------------------------+

# 如果 Innodb_row_lock_waits 和 Innodb_row_lock_time_avg 比较大，则表明锁竞争比较严重。
# information_schema 库中新增了三个关于锁的表，innodb_trx、innodb_locks 和 innodb_lock_waits。
#innodb_trx ## 当前运行的所有事务 
trx_id：事务ID。
trx_state：事务状态，有以下几种状态：RUNNING、LOCK WAIT、ROLLING BACK 和 COMMITTING。
trx_started：事务开始时间。
trx_requested_lock_id：事务当前正在等待锁的标识，可以和 INNODB_LOCKS 表 JOIN 以得到更多详细信息。
trx_wait_started：事务开始等待的时间。
trx_weight：事务的权重。
trx_mysql_thread_id：事务线程 ID，可以和 PROCESSLIST 表 JOIN。
trx_query：事务正在执行的 SQL 语句。
trx_operation_state：事务当前操作状态。
trx_tables_in_use：当前事务执行的 SQL 中使用的表的个数。
trx_tables_locked：当前执行 SQL 的行锁数量。
trx_lock_structs：事务保留的锁数量。
trx_lock_memory_bytes：事务锁住的内存大小，单位为 BYTES。
trx_rows_locked：事务锁住的记录数。包含标记为 DELETED，并且已经保存到磁盘但对事务不可见的行。
trx_rows_modified：事务更改的行数。
trx_concurrency_tickets：事务并发票数。
trx_isolation_level：当前事务的隔离级别。
trx_unique_checks：是否打开唯一性检查的标识。
trx_foreign_key_checks：是否打开外键检查的标识。
trx_last_foreign_key_error：最后一次的外键错误信息。
trx_adaptive_hash_latched：自适应散列索引是否被当前事务锁住的标识。
trx_adaptive_hash_timeout：是否立刻放弃为自适应散列索引搜索 LATCH 的标识。
#innodb_locks ## 当前出现的锁 
lock_id：锁 ID。
lock_trx_id：拥有锁的事务 ID。可以和 INNODB_TRX 表 JOIN 得到事务的详细信息。
lock_mode：锁的模式。有如下锁类型：行级锁包括：S、X、IS、IX，分别代表：共享锁、排它锁、意向共享锁、意向排它锁。表级锁包括：S_GAP、X_GAP、IS_GAP、IX_GAP 和 AUTO_INC，分别代表共享间隙锁、排它间隙锁、意向共享间隙锁、意向排它间隙锁和自动递增锁。
lock_type：锁的类型。RECORD 代表行级锁，TABLE 代表表级锁。
lock_table：被锁定的或者包含锁定记录的表的名称。
lock_index：当 LOCK_TYPE=’RECORD’ 时，表示索引的名称；否则为 NULL。
lock_space：当 LOCK_TYPE=’RECORD’ 时，表示锁定行的表空间 ID；否则为 NULL。
lock_page：当 LOCK_TYPE=’RECORD’ 时，表示锁定行的页号；否则为 NULL。
lock_rec：当 LOCK_TYPE=’RECORD’ 时，表示一堆页面中锁定行的数量，亦即被锁定的记录号；否则为 NULL。
lock_data：当 LOCK_TYPE=’RECORD’ 时，表示锁定行的主键；否则为NULL。

#innodb_lock_waits ## 锁等待的对应关系
requesting_trx_id：请求事务的 ID。
requested_lock_id：事务所等待的锁定的 ID。可以和 INNODB_LOCKS 表 JOIN。
blocking_trx_id：阻塞事务的 ID。
blocking_lock_id：某一事务的锁的 ID，该事务阻塞了另一事务的运行。可以和 INNODB_LOCKS 表 JOIN。


# 锁等待列表
SELECT * FROM innodb_locks WHERE lock_trx_id IN (SELECT blocking_trx_id FROM innodb_lock_waits);
SELECT innodb_locks.* FROM innodb_locks JOIN innodb_lock_waits ON (innodb_locks.lock_trx_id = innodb_lock_waits.blocking_trx_id);
# 正在等待锁的事务列表
SELECT trx_id, trx_requested_lock_id, trx_mysql_thread_id, trx_query FROM innodb_trx WHERE trx_state = 'LOCK WAIT';
SHOW ENGINE INNODB STATUS ;

# KILL PID

# KILL ALL PID
mysqladmin -uroot -p processlist|awk -F "|" '{print $2}'|xargs -n 1 mysqladmin -uroot -p kill
或
select concat('KILL ',id,';') from information_schema.processlist where user='root' into outfile '/tmp/a.txt';
source /tmp/a.txt;
```

查看锁

```mysql
SELECT    
        p2.`HOST` 被阻塞方host,
        p2.`USER` 被阻塞方用户,
        r.trx_id 被阻塞方事务id,    
        r.trx_mysql_thread_id 被阻塞方线程号,    
        TIMESTAMPDIFF(    
            SECOND,    
            r.trx_wait_started,    
            CURRENT_TIMESTAMP    
        ) 等待时间,    
        r.trx_query 被阻塞的查询,    
        l.lock_table 阻塞方锁住的表,  
        m.`lock_mode` 被阻塞方的锁模式,
        m.`lock_type`  "被阻塞方的锁类型(表锁还是行锁)",
        m.`lock_index` 被阻塞方锁住的索引,
        m.`lock_space` 被阻塞方锁对象的space_id,
        m.lock_page 被阻塞方事务锁定页的数量,
        m.lock_rec 被阻塞方事务锁定行的数量,
        m.lock_data  被阻塞方事务锁定记录的主键值,  
        p.`HOST` 阻塞方主机,
        p.`USER` 阻塞方用户,
        b.trx_id 阻塞方事务id,    
        b.trx_mysql_thread_id 阻塞方线程号,
        b.trx_query 阻塞方查询,
        l.`lock_mode` 阻塞方的锁模式,
        l.`lock_type` "阻塞方的锁类型(表锁还是行锁)",
        l.`lock_index` 阻塞方锁住的索引,
        l.`lock_space` 阻塞方锁对象的space_id,
        l.lock_page 阻塞方事务锁定页的数量,
        l.lock_rec 阻塞方事务锁定行的数量,
        l.lock_data 阻塞方事务锁定记录的主键值,         
      IF (p.COMMAND = 'Sleep', CONCAT(p.TIME,' 秒'), 0) 阻塞方事务空闲的时间           
    FROM    
        information_schema.INNODB_LOCK_WAITS w    
    INNER JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id    
    INNER JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id    
    INNER JOIN information_schema.INNODB_LOCKS l ON w.blocking_lock_id = l.lock_id  AND l.`lock_trx_id`=b.`trx_id`
      INNER JOIN information_schema.INNODB_LOCKS m ON m.`lock_id`=w.`requested_lock_id` AND m.`lock_trx_id`=r.`trx_id`
    INNER JOIN information_schema. PROCESSLIST p ON p.ID = b.trx_mysql_thread_id   
INNER JOIN information_schema. PROCESSLIST p2 ON p2.ID = r.trx_mysql_thread_id
    ORDER BY    
        等待时间 DESC;
```





# Lock wait timeout exceeded; try restarting transaction] with root cause

Lock wait timeout exceeded; try restarting transaction] with root cause

```bash
delete from third_task_info where id <= 78458017
> 1205 - Lock wait timeout exceeded; try restarting transaction
> 时间: 50.522s
```

您好,没有必要修改事务超时,您要检查下为何有事务一直没有提交.这个值默认是50秒,足以将sql执行完成了.

```bash
# 获取当前关于这个表的事务
mysql -e 'SHOW FULL PROCESSLIST ;' | grep  third_task_info

# 终止事务
mysql -e 'KILL 349539844;'

# 不要执行delete. 原因如下
# mysql -e 'delete from third_task_info where id <= 78458017'
```

执行过程中，如果有其他用户操作这个表，用户操作阻塞在mysql写节点，cpu和连接数打满可能影响业务

![image-20210115111334895](http://myapp.img.mykernel.cn/image-20210115111334895.png)

所有会话全部过期，需要kill所有会话, 之后节点cpu、ram下降才恢复





