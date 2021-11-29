---
title: kubernetes节点发生 diskpressure
date: 2021-07-06 15:37:09
tags:
- 生产小经验
- kubernetes
---





# 前言

生产出现node报警， disk has pressure



<!--more-->
# 节点磁盘使用

```bash
[root@izbp16exe6p6dolijnp7csz ~]# df -TH
Filesystem                   Type      Size  Used Avail Use% Mounted on
devtmpfs                     devtmpfs   17G     0   17G   0% /dev
tmpfs                        tmpfs      17G     0   17G   0% /dev/shm
tmpfs                        tmpfs      17G  5.1M   17G   1% /run
tmpfs                        tmpfs      17G     0   17G   0% /sys/fs/cgroup
/dev/vda1                    ext4      106G   81G   21G  80% /
tmpfs                        tmpfs      17G   13k   17G   1% /var/lib/kubelet/pods/83782f28-5dc1-468b-b4c4-37f4d12f97ab/volumes/kubernetes.io~secret/default-token-qgh8d
tmpfs                        tmpfs      17G   13k   17G   1% /var/lib/kubelet/pods/48eca7e3-211a-4646-8de0-7476f18f1e27/volumes/kubernetes.io~secret/terway-token-fqqmv
tmpfs                        tmpfs      17G   13k   17G   1% /var/lib/kubelet/pods/e8e79edf-744d-4571-9c3b-4f2879efdfbc/volumes/kubernetes.io~secret/kube-proxy-token-9zhc6
overlay                      overlay   106G   81G   21G  80% /var/lib/docker/overlay2/2d1b827c28bb1d6dc44c42d3a7dbfe9388cc8af503ffda5b8faf16ef55b17da4/merged
overlay                      overlay   106G   81G   21G  80% /var/lib/docker/overlay2/c6f161a384493a25c67a9e042c9b67077a1935e9c892435397d75d77eb4d1de0/merged
```

确定哪个目录使用大

```bash
[root@izbp16exe6p6dolijnp7csz ~]# du -sh /*
0       /bin
188M    /boot
0       /dev
38M     /etc
28K     /home
14M     /kubelet.log
0       /lib
0       /lib64
16K     /lost+found
4.0K    /media
124K    /mnt
193M    /opt
du: cannot access ‘/proc/6131/task/9403/fd/60’: No such file or directory
du: cannot access ‘/proc/1540483/task/1540483/fd/3’: No such file or directory
du: cannot access ‘/proc/1540483/task/1540483/fdinfo/3’: No such file or directory
du: cannot access ‘/proc/1540483/fd/3’: No such file or directory
du: cannot access ‘/proc/1540483/fdinfo/3’: No such file or directory
du: cannot access ‘/proc/2389880/task/2389880/fdinfo/470’: No such file or directory
du: cannot access ‘/proc/2389880/task/2389885/fd/470’: No such file or directory
du: cannot access ‘/proc/2389880/task/2389887/fdinfo/470’: No such file or directory
du: cannot access ‘/proc/2389880/task/2389918/fd/470’: No such file or directory
du: cannot access ‘/proc/2389880/task/2395704/fd/470’: No such file or directory
du: cannot access ‘/proc/2389880/task/1540094/fd/470’: No such file or directory
0       /proc
43M     /root
4.9M    /run
0       /sbin
4.0K    /srv
0       /sys
32K     /tmp
2.3G    /usr
107G    /var

```

一看是var目录，继续跟进

```bash
4.0K    /var/adm
200M    /var/cache
4.0K    /var/crash
24K     /var/db
8.0K    /var/empty
4.0K    /var/games
4.0K    /var/gopher
12K     /var/kerberos
103G    /var/lib
4.0K    /var/local
0       /var/lock
4.2G    /var/log
0       /var/mail
4.0K    /var/nis
4.0K    /var/opt
4.0K    /var/preserve
0       /var/run
108K    /var/spool
16K     /var/tmp
4.0K    /var/yp

```

可以明确是docker数据占用过多

# docker使用情况

https://docs.docker.com/engine/reference/commandline/system_df/

```bash
[root@izbp16exe6p6dolijnp7csz ~]# docker system df -v
Images space usage:        SIZE                SHARED SIZE         UNIQUE SiZE 
Containers space usage:    SIZE字段
Local Volumes space usage: VOLUME NAME         LINKS               SIZE
```

> - `SHARED SIZE` 是图像与另一个图像共享的空间量（即它们的公共数据）
> - `UNIQUE SIZE` 是仅由给定图像使用的空间量
> - `SIZE`是图像的虚拟尺寸，它是`SHARED SIZE`和`UNIQUE SIZE`

通过分析是image space usage过多

容器才1个



# 定时清理镜像

```bash
docker rmi $(docker images -f 'dangling=true')
docker image prune -f
```



