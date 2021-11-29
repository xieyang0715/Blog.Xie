---
title: docker日志清理
date: 2021-08-10 21:07:15
tags:
- docker
---



# 日志

```bash
[root@alyy docker]# find /var/lib/docker/containers/ -size +100M -name "*.log"  -exec du -sh {} \; 
11G	/var/lib/docker/containers/7921853e84637d8d06e1079f1bdbe5e1296c8349cf9ac37771059efd6eca7d72/7921853e84637d8d06e1079f1bdbe5e1296c8349cf9ac37771059efd6eca7d72-json.log
```

参考: https://docs.docker.com/storage/storagedriver/

```bash
Advanced: metadata and logs storage used for containers

Note: This step requires a Linux machine, and does not work on Docker Desktop for Mac or Docker Desktop for Windows, as it requires access to the Docker Daemon’s file storage.

While the output of docker ps provides you information about disk space consumed by a container’s writable layer, it does not include information about metadata and log-files stored for each container.

More details can be obtained my exploring the Docker Daemon’s storage location (/var/lib/docker by default).

 sudo du -sh /var/lib/docker/containers/*
Each of these containers only takes up 36k of space on the filesystem.
```

containers下面存储容器的日志和元数据。

容器的可写层存储的数据大小和 前面大小+只读层的镜像大小。使用`docker ps -s`查看



# 配置docker容器

https://docs.docker.com/config/containers/logging/configure/

As a default, Docker uses the [`json-file` logging driver](https://docs.docker.com/config/containers/logging/json-file/), which caches container logs as JSON internally. 

默认使用json-file驱动，是 为了兼容老版本的docker, 但是会导致大量的docker日志输出，导致磁盘使用光了。现在建议使用**local**驱动，会避免这个问题。

更新日志驱动时，必须**重建容器**，才可以使日志选项生效。

当前默认驱动

```bash
[root@alyy ~]# docker info --format '{{.LoggingDriver}}'
json-file
```

获取运行容器的日志驱动, substituting the container name or ID for `<CONTAINER>`

```bash
[root@alyy ~]# docker inspect -f '{{.HostConfig.LogConfig.Type}}'  redis_proxy
json-file
```

现在切换**local**驱动,By default, the `local` driver preserves 100MB of log messages per container and uses automatic compression to reduce the size on disk. The 100MB default value is based on a 20M default size for each file and a default count of 5 for the number of such files (to account for log rotation).

```json
{
  "log-driver": "local",
  "log-opts": {
    "max-size": "10m"
  }
}
```

>  更详细，参考日志配置文件，https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file

You can set the logging driver for a specific container by using the `--log-driver` flag to `docker container create` or `docker run`:

```bash
$ docker run \
      --log-driver local --log-opt max-size=10m \
      alpine echo hello world
```

| Option     | Description                                                  | Example value              |
| :--------- | :----------------------------------------------------------- | :------------------------- |
| `max-size` | The maximum size of the log before it is rolled. A positive integer plus a modifier representing the unit of measure (`k`, `m`, or `g`). Defaults to 20m. | `--log-opt max-size=10m`   |
| `max-file` | The maximum number of log files that can be present. If rolling the logs creates excess files, the oldest file is removed. A positive integer. Defaults to 5. | `--log-opt max-file=3`     |
| `compress` | Toggle compression of rotated log files. Enabled by default. | `--log-opt compress=false` |



# 配置local驱动

`/etc/docker/daemon.json`

```json
{
  "log-driver": "local",
  "log-opts": {
    "max-file": "5",
    "max-size": "20m",
    "compress": "true"
  }
}
```

重启docker

```bash
[root@alyy docker]# systemctl daemon-reload 
[root@alyy docker]# systemctl restart docker
```

检验容器日志驱动

```bash
[root@alyy ~]# docker inspect -f '{{.HostConfig.LogConfig.Type}}'  redis_proxy
json-file
```

重启所有容器， **失败**

```bash
[root@alyy docker]# docker ps  | awk 'NR>1{print $NF}' | xargs -I {} docker restart {}
# 再检验
docker inspect -f '{{.HostConfig.LogConfig.Type}}'  redis_proxy
json-file
```

`docker-compose`

```bash
[root@alyy proxypool]# docker-compose kill
Killing redis_proxy ... done
Killing proxy_pool  ... done
[root@alyy proxypool]# docker-compose rm -f
Going to remove redis_proxy, proxy_pool
Removing redis_proxy ... done
Removing proxy_pool  ... done
[root@alyy proxypool]# docker-compose up -d
Creating redis_proxy ... done
Creating proxy_pool  ... done
[root@alyy ~]# docker inspect -f '{{.HostConfig.LogConfig.Type}}'  redis_proxy
local

```

`docker run命令`

```bash
[root@alyy ~]# docker inspect -f '{{.HostConfig.LogConfig.Type}}'  nginx-pro
local
```



