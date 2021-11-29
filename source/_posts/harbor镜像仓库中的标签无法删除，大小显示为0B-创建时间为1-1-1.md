---
title: 'harbor镜像仓库中的标签无法删除，大小显示为0B, 创建时间为1/1/1'
date: 2021-02-24 00:53:37
tags:
- 生产小经验
- harbor
---





![image-20210224085427723](http://myapp.img.mykernel.cn/image-20210224085427723.png)

https://github.com/goharbor/harbor/issues/10396



需要升级harbor为2.0

https://github.com/goharbor/harbor

![image-20210224092741569](http://myapp.img.mykernel.cn/image-20210224092741569.png)

参考：https://codingnote.cc/p/157243/

参考：https://goharbor.io/docs/2.1.0/administration/upgrade/

参考：https://goharbor.io/docs/2.0.0/administration/upgrade/

只能从v1.9直接升级到v2.0.0

https://github.com/goharbor/harbor/releases/tag/v2.0.0

```bash
# 停止harbor
cd /data/harbor/
docker-compose down

# 备份harbor
cd ..
mv  harbor harbor-v1.9-2020-02-24.bak
cp -r /data/database/ database-v1.9
cp -a harbor.bak/harbor.yml /usr/local/src/harbor.yml 


# 准备配置
 docker pull goharbor/prepare:v2.0.0
docker run -it --rm -v /:/hostfs goharbor/prepare:v2.0.0 migrate -i /usr/local/src/harbor.yml 
# tar xvf /root/harbor-offline-installer-v2.0.0.tgz -C /data/
cp /usr/local/src/harbor.yml  /data/harbor
cd /data/harbor


```



回滚

https://goharbor.io/docs/2.1.0/administration/upgrade/roll-back-upgrade/





harbor仓库垃圾清理

![image-20210224105055854](http://myapp.img.mykernel.cn/image-20210224105055854.png)