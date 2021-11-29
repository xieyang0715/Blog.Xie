---
title: harbor自动清理和gc回收和kubelet日志清理
date: 2020-11-04 02:32:04
tags: 
- 生产小经验
- harbor
toc: true
---

在生产使用过程中，有大量推送到harbor会打满磁盘, 通过harbor配置策略完成自动清理标签

就算定时清理了，但是发现df -TH的/有大量占用, 实际并没有占用这么多空间，需要清理gc



日志清理

```bash
find /mnt/docker/lib/ -name "*.log" -size +1G
: > /mnt/docker/lib/overlay2/99693f7f2acc1973610b3bee263b9c99564ee475462d6f6778eb8e335dec2cf7/merged/usr/local/nginx/logs/access.log

# 在启动docker时，就应该把日志放在一个非常大的文件系统
[root@izpxkqmt2zlej4z ~]# cat /etc/docker/daemon.json
{
  "data-root": "/mnt/docker/lib", # docker数据目录
  "exec-opts": ["native.cgroupdriver=cgroupfs"],
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn", "http://hub-mirror.c.163.com"], 
  "insecure-registries": ["127.0.0.1/8"],
  "max-concurrent-downloads": 10,
  "log-driver": "json-file",
  "log-level": "warn",
  "log-opts": {
    "max-size": "15m",
    "max-file": "3"
    },
  "storage-driver": "overlay2"
}
```

<!--more-->

# harbor策略

![image-20201104103347492](http://myapp.img.mykernel.cn/image-20201104103347492.png)

![image-20201104103402467](http://myapp.img.mykernel.cn/image-20201104103402467.png)

相当于每天的凌晨执行定时任务

![image-20201104103415388](http://myapp.img.mykernel.cn/image-20201104103415388.png)

![image-20201104103448338](http://myapp.img.mykernel.cn/image-20201104103448338.png)

# harbor gc回收

```bash
cd /data/harbor/
# 停止harbor
docker-compose stop
# 测试看看会清理哪些镜像
docker run -it --name gc --rm --volumes-from registry vmware/registry:2.6.2-photon garbage-collect --dry-run /etc/registry/config.yml
# 去掉--dry-run选项，清理镜像
docker run -it --name gc --rm --volumes-from registry vmware/registry:2.6.2-photon garbage-collect  /etc/registry/config.yml
# 启动harbor
docker-compose start

```

![image-20201104103613385](http://myapp.img.mykernel.cn/image-20201104103613385.png)

