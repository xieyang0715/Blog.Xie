---
title: 安装及简单配置envoy
date: 2021-03-16 02:58:42
tags:
- istio
---

| 操作系统    | 环境   | 主机        |
| ----------- | ------ | ----------- |
| Ubuntu 1804 | docker | 192.168.0.8 |

# 环境准备

```bash
root@ubuntu-template:~/linux_scripts/docker-ce# bash install-docker.sh 
```



# 准备envoy程序

## 源码构建

```bash
root@ubuntu-template:~# git clone https://github.com/envoyproxy/envoy.git
Cloning into 'envoy'...
remote: Enumerating objects: 132, done.
remote: Counting objects: 100% (132/132), done.
remote: Compressing objects: 100% (105/105), done.
remote: Total 207460 (delta 43), reused 47 (delta 25), pack-reused 207328
Receiving objects: 100% (207460/207460), 82.68 MiB | 9.37 MiB/s, done.
Resolving deltas: 100% (157361/157361), done.
Checking out files: 100% (8125/8125), done.
```

```bash
root@ubuntu-template:~/envoy# export https_proxy=http://192.168.0.33:808
root@ubuntu-template:~/envoy# ./ci/run_envoy_docker.sh  './ci/do_ci.sh bazel.release' # 构建
root@ubuntu-template:~/envoy# docker build -f ci/Dockerfile-envoy-alpine -t envoy-alpine:latest # 制作镜像
```

## 官方预制envoy镜像

https://hub.docker.com/r/envoyproxy/envoy-alpine/tags?page=1&ordering=last_updated

![image-20210316110652303](http://myapp.img.mykernel.cn/image-20210316110652303.png)

```bash
root@ubuntu-template:~# docker pull envoyproxy/envoy-alpine:v1.17-latest
```



# 配置envoy

https://www.envoyproxy.io/docs/envoy/latest/configuration/configuration

https://www.envoyproxy.io/docs/envoy/latest/api/api

![image-20210316114631402](http://myapp.img.mykernel.cn/image-20210316114631402.png)

## 源码构建的envoy

```bash
# 源码目录
mkdir -p generated/configs
bazel build //configs:example_configs # 基于configs/*.py脚本和configs/*.json模板生成
# 默认启用lightstep跟踪机制和全局限速服务
```



## 官方预制docker镜像

```bash
# 默认配置文件 /etc/envoy/envoy.yaml

# 准备配置方式
1. 二次制作镜像，COPY/ADD新配置
2. 宿主机共享存储卷方式提供配置, -v /path/to/hostpath:/etc/envoy
```

### 二次制作镜像

```bash
root@ubuntu-template:~# cat envoy.echo/envoy.yaml 
static_resources: # 顶级字段
  listeners: # 侦听器列表
  - name: listener_0
    address: # 地址
      socket_address:
        address: 0.0.0.0
        port_value: 15001
    filter_chains: # 多个chains
    - filters:     # 每个chains上有多个filter
      - name: envoy.echo # 第一个filter, envoy.echo 主要用于演示网络过滤器API的功能

```

```bash
root@ubuntu-template:~/envoy.echo# cat Dockerfile
FROM envoyproxy/envoy-alpine:v1.17-latest
ADD envoy.yaml /etc/envoy/
```

```bash
root@ubuntu-template:~/envoy.echo# docker build -t envoy-echo:v0.1 ./
Sending build context to Docker daemon  3.072kB
Step 1/2 : FROM envoyproxy/envoy-alpine:v1.17-latest
 ---> 4fab18b785c6
Step 2/2 : ADD envoy.yaml /etc/envoy/
 ---> 2428050e92ca
Successfully built 2428050e92ca
Successfully tagged envoy-echo:v0.1
```

```bash
root@ubuntu-template:~/envoy.echo# docker run --rm --name echo  envoy-echo:v0.1
```

验证envoy

```bash
doroot@ubuntu-template:~# docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS          PORTS       NAMES
55d6e5f4f193   envoy-echo:v0.1   "/docker-entrypoint.…"   29 seconds ago   Up 25 seconds   10000/tcp   echo
root@ubuntu-template:~# docker exec -it echo /bin/sh
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02  
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:11 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:906 (906.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ # netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 0.0.0.0:15001           0.0.0.0:*               LISTEN      
/ # root@ubuntu-template:~# 
root@ubuntu-template:~# 
root@ubuntu-template:~# 
root@ubuntu-template:~# nc 172.17.0.2 15001
echo
echo
Hi Envoy
Hi Envoy
^C
root@ubuntu-template:~# 
```

### 共享宿主机目录

```bash
root@ubuntu-template:~/envoy.echo# docker run --rm -v /root/envoy.echo/:/etc/envoy --name echo1 envoyproxy/envoy-alpine:v1.17-latest
```

验证

```bash
root@ubuntu-template:~# docker ps
CONTAINER ID   IMAGE                                  COMMAND                  CREATED          STATUS          PORTS       NAMES
8304365573ec   envoyproxy/envoy-alpine:v1.17-latest   "/docker-entrypoint.…"   18 seconds ago   Up 14 seconds   10000/tcp   echo1
root@ubuntu-template:~# docker exec -it echo1 /bin/sh
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02  
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:10 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:836 (836.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ # netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 0.0.0.0:15001           0.0.0.0:*               LISTEN      
/ # root@ubuntu-template:~# 
root@ubuntu-template:~# 
root@ubuntu-template:~# nc 172.17.0.2 15001
Hi Envoy
Hi Envoy
how are you?
how are you?
^C
```



# 工具

## 配置文件检测

config_load_check_tool

## 路由表检测

router_check_tool

## 路由模式验证

schema_validator_tool



