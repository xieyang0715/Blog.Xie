---
title: gitlab备份
date: 2021-03-22 02:57:34
tags:
- gitlab
---



# 备份

```bash
[root@wzx sonarqube]# cat /weizhixiu/backup-gitlab.sh 
#!/bin/sh
set -xeu

DATE=$(date +%F)
BACKUP_DIR=/weizhixiu/gitlab-backup/
CURRENT_DIR=${BACKUP_DIR}${DATE}

function backup_all {
	install -dv ${CURRENT_DIR}
	gitlab-backup create  && cd /var/opt/gitlab/backups/ && cp $(ls -t /var/opt/gitlab/backups/ | head -n1) ${CURRENT_DIR}
	gitlab-ctl backup-etc && cd /etc/gitlab/config_backup && cp $(ls -t | head -n1) /secret/gitlab/backups/ && cp $(ls -t | head -n1) /secret/gitlab/backups/ &&  cd  /secret/gitlab/backups/ && cp $(ls -t | head -n1)  ${CURRENT_DIR}
}


clearfile() {
	 cd ${BACKUP_DIR}
	 COUNT=`ls | wc -l` 
	 [ $COUNT -gt 10 ] && rm -rf `ls -tr | head -n1`
	 # 清理默认备份目录
	 find  /var/opt/gitlab/backups/ -mtime +3 -delete
}

backup_all
clearfile
```

# 恢复

参考：

https://cloud.tencent.com/developer/article/1622317

```bash
ls /var/opt/gitlab/backups/1530156812_2018_06_28_10.8.4_gitlab_backup.tar
sudo gitlab-ctl stop unicorn
sudo gitlab-ctl stop puma
sudo gitlab-ctl stop sidekiq
# Verify
sudo gitlab-ctl status

[root@gitlab-new ~]# chmod 777 /var/opt/gitlab/backups/1530156812_2018_06_28_10.8.4_gitlab_backup.tar
#修改权限，如果是从本服务器恢复可以不修改
[root@gitlab ~]# gitlab-rake gitlab:backup:restore BACKUP=1530156812_2018_06_28_10.8.4	
#从1530156812_2018_06_28_10.8.4编号备份中恢复
```

