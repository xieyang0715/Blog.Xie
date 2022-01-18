---
title: Docker常用命令
date: 2022-01-18 11:30:46
categories: Docker
toc: true
---

构建镜像：

```
docker build -t my_webapi05 .
```

运行镜像(-d 指定容器后台运行,--name 容器别名，-p 指定端口 主机端口：容器端口)

```
docker run --name my_webapi_test -p 8088:80 -d my_webapi_06
```

<!--more-->

删除容器 xx

```
docker rm xx
```

删除所有停止的容器：

```
docker image prune -a
```

列出所有运行的容器

```
docker ps
```

进入容器

```
docker exec -it 23d1b53d6560 /bin/bash
```

退出容器

```
Ctrl+D / exit
```

查看容器 IP 等信息

```
docker inspect containerid/containername
```

启动容器 5fe7880a01ee

```
docker start 5fe7880a01ee
```

停止容器 5fe7880a01ee

```
docker stop 5fe7880a01ee
```

重命名容器：

```
docker rename f85a06be671c cdzq.test
```

备份容器

```
docker container commit e70eb5997db2 jenkins-hexo-back:v1.0
```

当 Docker 重启时,容器自动启动

```
--restart=always
```

如果创建时未指定 --restart=always ,可通过 update 命令设置

```
docker update --restart=always 容器ID
```

Docker 时区同步 linux 主机

```
docker cp /usr/share/zoneinfo/Asia/Shanghai 【容器ID】:/etc/localtime
docker restart 【容器ID】
```

docker 复制：

```
docker cp  /usr/src/public-data/blog/public/.  blog:/var/www/html/
```

**[Docker 文档](https://docs.docker.com/)**  
**[Docker 菜鸟教程](https://www.runoob.com/docker/docker-run-command.html)**  
**[加速地址 https://ung2thfc.mirror.aliyuncs.com](https://ung2thfc.mirror.aliyuncs.com)**
