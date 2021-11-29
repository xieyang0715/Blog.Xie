---
title: "docker空间管理"
date: "2021-08-06 17:25:07"
tags:
- "docker"
---

# 查看空间占用

可以查看是容器占用空间多，还是镜像多

- 容器占用多了，就通知开发少写日志。
- 镜像多了，就优化镜像。

```bash
docker system df -v

                                                                                                                   # 镜像大小
REPOSITORY                                                             TAG           IMAGE ID       CREATED         SIZE   
# 使用共享镜像，   # 独立镜像大小（此大小越小越好）
SHARED SIZE   UNIQUE SIZE   CONTAINERS
registry.cn-hangzhou.aliyuncs.com/ishop-general/mydemo                 latest        225890842aca   6 hours ago     46.76MB   5.595MB       41.17MB       0

Containers space usage:
                                                                                                                # 容器使用的空间
CONTAINER ID   IMAGE                                                   COMMAND                  LOCAL VOLUMES   SIZE      CREATED        STATUS                           NAMES
35030f27e8ed   94a5d5fc965f                                            "/bin/sh -c 'wget ht…"   0               0B        3 days ago     Exited (4) 3 days ago            jolly_cori

Local Volumes space usage:
                                                                             # 卷使用大小
VOLUME NAME                                                        LINKS     SIZE
0d6d82e1fba3a7f462dbb51b0fbb6c7d7d21603e6e17e7a8c26d62e6a12c9a46   1         4B

```



# 清理没有tag镜像不包含缓存

```bash
# 先清理停止的容器
docker container prune -f

# 再清理未使用的镜像
docker rmi $(docker images -f 'dangling=true') 
```

# 清理未使用镜像及缓存

此命令可能会造成，你的app构建会在清理后非常慢，原因是把构建缓存删除了。

```bash
docker image prune -f 
```



# 清理docker

https://github.com/spotify/docker-gc

```bash
docker run --rm --privileged -v /var/run/docker.sock:/var/run/docker.sock -v /etc:/etc:ro spotify/docker-gc
```

