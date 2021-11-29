---
title: mysqldump备份
date: 2020-10-20 03:23:41
tags: 
- 生产小经验
- mysql
---



mysql备份会锁表, 导致连接超时, 连接打满

```bash
mysqldump --single-transaction  --skip-add-locks --skip-lock-tables
```

备份所有库前100行，给预发使用

```bash
/usr/bin/mysqldump -h readonly-youninyouqu.rwlb.rds.aliyuncs.com --databases zhibo -uzhibo -pxxxxxx --single-transaction  --skip-add-locks --skip-lock-tables  --where "1=1 limit 100" > 100-zhibo.sql

mysql -uroot -pxxxxxx -h47.108.20.243 zhibo < 100-zhibo.sql 
```

