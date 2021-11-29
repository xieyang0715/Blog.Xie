---
title: docker-compose启动mysql每表备份
date: 2021-03-22 02:18:59
tags:
- docker-compose
- mysql
---



# docker-compose

```bash
[root@alyy mysqlbackup]# cat docker-compose.yaml 
version: "3.3"
services:
  cron:
    image: mysql-backup:v1.20.1
    container_name: cron-mysqlbackup
    restart: always
    # 主机网络
    network_mode:  host
    build:
      context: .
      dockerfile: Dockerfile-backup
      args: 
      # 生成crontab, 在dockerfile中
      - minute=${minute} # 分钟
      - hour=$hour # 小时
      - day=${day}
      - month=${month}
      - week=${week}
      - backupscriptfilename=${backupscriptfilename}
    environment:
    # 脚本中读取认证信息
    - DB_USER=${DB_USER}
    - DB_HOST=${DB_HOST}
    - DB_PORT=${DB_PORT}
    - DB_PASS=${DB_PASS}
    # 在脚本中, 读取这个文件
    - BACKUP_DBS_FNAME=${BACKUP_DBS_FNAME}
    volumes:
    - ${BACKUP_DIR}:/tables_backup/

```

## 环境变量

```bash
[root@alyy mysqlbackup]# cat .env 
# 备份用户名
DB_USER=root
# 备份主机名 SLAVE主机最好. master主机备份时，业务有影响
DB_HOST=127.0.0.1
# 备份端口
DB_PORT=3306
# 备份密码
DB_PASS=123456


# 每天凌晨3:00备份，遵循crontab格式
minute=00
hour=03
day=*
month=*
week=*
# 备份脚本名
backupscriptfilename=backup.sh
# 文件每行定义一个db名，db中的所有表将会被备份
BACKUP_DBS_FNAME=dbnames
# 备份目录
BACKUP_DIR=/data/wordpress_data/
```



# dockerfile

```bash
[root@alyy mysqlbackup]# cat Dockerfile-backup 
FROM alpine

RUN  sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories
RUN apk update && apk add --no-cache  mysql-client 


ARG minute
ARG hour
ARG day
ARG month
ARG week
ARG backupscriptfilename
COPY  . /
RUN echo "$minute $hour $day $month $week /bin/sh -c 'export; /$backupscriptfilename baktables'" | /usr/bin/crontab -

ENTRYPOINT /usr/sbin/crond -f -l 8

```

## 准备文件

```bash
[root@alyy mysqlbackup]# cat dbnames 
# 每行一个db名字
test
wordpress
```



# 备份脚本

```bash
[root@alyy mysqlbackup]# cat backup.sh 
#!/bin/sh
# Author: 个人博客 pxrux => www.mykernel.cn 
# Author: 个人博客 slcnux => liangcheng.mykernel.cn
# Describe: 备份结构
#BACKUP_PATH
# # 2020-11-11
# # # A库
# #   # ------a
# #   # ------b
# #   # ------c
# # # B库
# #   # ------a
# #   # ------b
# #   # ------c

backup_single_table() {
  #备份各个表
  for name in $(cat $CURRENT_DB_BACKUP_PATH/sql_name.txt)
  do
    path_=$1
    echo "备份"$name"开始..." 
    # 避免锁表, 避免加锁，使用InnoDB支持的快照
    # 将备份文件以天放
    mysqldump -h $HOST -u $USERNAME -P$PORT -p$PASSWORD  --single-transaction  --skip-add-locks --skip-lock-tables  -R  $database_name $name|gzip -c  > $CURRENT_DB_BACKUP_PATH/$database_name-$name-$DATE.gz

    # 从备份表中清理当前备份的表,并保留最近一次修改前的文件名
    sed -i.bak "/^${name}$/d" $CURRENT_DB_BACKUP_PATH/sql_name.txt

    echo "备份"$name"完成..."
  done
}

backup_single_db() {
  database_name=$1
  CURRENT_DB_BACKUP_PATH=${BACKUP_PATH}/${TODAY}/${database_name}
  install -dv $CURRENT_DB_BACKUP_PATH
  #列出所有的表名称
  mysql -h $HOST -u $USERNAME -P$PORT -p$PASSWORD -e "SELECT TABLES.TABLE_NAME FROM information_schema.TABLES WHERE TABLES.TABLE_SCHEMA = '$database_name'"  > $CURRENT_DB_BACKUP_PATH/sql_name.txt # A库 a表

  # 备份A库各个表
  backup_single_table $CURRENT_DB_BACKUP_PATH/sql_name.txt 
}

baktables() {
    for database_name in $(sed '/#/d' ${BACKUP_DB_NAME_PATH}) # A B C 库
    do
      backup_single_db  $database_name
    done

    # 备份mysql配置文件到七牛存储
    #/alidata/scripts/qiniu/qshell  qupload 10 /alidata/scripts/qiniu/monthly-mysql-backup.conf

    # 清理
    find $BACKUP_PATH  -maxdepth 1 ! -path $BACKUP_PATH -type d -mtime +3 -exec rm -fr {} \;
}


########################################################### start ################################################################
set -eu
echo "程序开始...."

HOST=${DB_HOST} # mysql slave
PORT=${DB_PORT}
USERNAME=${DB_USER}
PASSWORD=${DB_PASS}


# docker-compose volumes挂载在此目录
BACKUP_PATH=/tables_backup/
BACKUP_DB_NAME_PATH=/${BACKUP_DBS_FNAME}

DATE=$(date "+%Y%m%d%H%M%S")
TODAY=`date +%F`
starttime=`date +'%Y-%m-%d %H:%M:%S'`
start_seconds=$(date --date="$starttime" +%s);
case $1 in
baktables)
  baktables
  ;;
*)
  exit
  ;;
esac
#执行程序
endtime=`date +'%Y-%m-%d %H:%M:%S'`
end_seconds=$(date --date="$endtime" +%s);
echo "本次运行时间： "$((end_seconds-start_seconds))"s"
########################################################### end ################################################################
```

# 启动mysql备份

在修改backup.sh 或.env时，均通过以下命令进行加载配置

```bash
docker-compose up --build -d
```

查看日志

```bash
docker-compose logs -f
```

# 备份考虑

```bash
多线程(几个线程备份)异地备份，毕竟数据重要

银行级别 还要 异地灾备，定期每个月 去异地机房 瞄几眼

有些大厂 备份服务器都是分布式文件系统的， 防止单点故障

3个备份节点，1个服务器挂了，还能用另外两台备份
```

