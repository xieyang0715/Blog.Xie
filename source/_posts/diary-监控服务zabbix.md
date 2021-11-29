---
title: 监控服务zabbix
date: 2021-04-21 13:51:47
tags:
- zabbix
- 个人日记
---



# zabbix规划及部署
<!--more-->

部署环境：

服务系统： `ubuntu 18.04/centos 7.5.1804`

| 主机类型        | IP        | PORT                     |
| --------------- | --------- | ------------------------ |
| zabbix server   | 127.0.0.1 | agent 10050 server 10051 |
| zabbix 主动代理 | 127.0.0.1 | proxy 10050              |
| zabbix 被动代理 | 127.0.0.1 | proxy 10050              |
| mysql master    | 127.0.0.1 | mysql 3306 agent 10050   |
| mysql slave     | 127.0.0.1 | mysql 3306 agent 10050   |
| web server1     | 127.0.0.1 | nginx 80 agent 10050     |
| web server2     | 127.0.0.1 | nginx 80 agent 10050     |

- [x] zabbix server 总服务端
- [x] zabbix proxy 替代server收集的数据的agent，主动、被动
- [x] mysql zabbix将数据放在关系型数据库中，**数据库和zabbix在一起，监控主机少，没有问题。**如果监控主机非常多时，放在一起，会查询慢，因为数据库相当消耗IO资源。

![image-20210422161542083](http://myapp.img.mykernel.cn/image-20210422161542083.png)

安装前准备亿图，安装后，每安装一个组件，就将图颜色改变，这个活，一天做不完，第二天来了看了颜色可以知道进度。

工作日报也可以看出，每天做了哪些。

## 系统环境

### ubuntu 18.04

```bash
FROM ubuntu:18.04
LABEL AUTHOR=2192383945@qq.com

# 基础环境
RUN sed -i.bak -e 's@archive.ubuntu.com@mirrors.aliyun.com@g'  -e 's@security.ubuntu.com@mirrors.aliyun.com@g' /etc/apt/sources.list && \
	apt update && \
	apt install iproute2 ntpdate tcpdump telnet traceroute nfs-kernel-server nfs-common lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute gcc make openssh-server lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute iotop unzip zip lsof make curl iputils-ping net-tools vim -y && \
	apt-get install language-pack-zh* -y && echo 'LANG="zh_CN.UTF-8"' > /etc/default/locale && dpkg-reconfigure --frontend=noninteractive locales && update-locale LANG=zh_CN.UTF-8

# 添加ssh服务 docker -p xx:22
RUN sed -i 's@#PermitRootLogin prohibit-password@PermitRootLogin yes@g' /etc/ssh/sshd_config && \
	service ssh restart

# 密码
RUN echo 'root:MBZh/TAmW9CynXO0' | chpasswd

# 前台命令
ENTRYPOINT  /usr/sbin/sshd -D
```

```bash
# docker build -t ubuntu-baseimg-ssh-ssh:18.04 ./

# 启动一个示例
docker run -d --name zbx-server 		-p 30001:22 -p 31001:10050	 -p 32001:10051  ubuntu-baseimg-ssh:18.04
docker run -d --name zbx-proxy-active 	-p 30002:22 -p 31002:10050   				 ubuntu-baseimg-ssh:18.04
docker run -d --name zbx-proxy-passive 	-p 30003:22 -p 31003:10050   				 ubuntu-baseimg-ssh:18.04
docker run -d --name mysql-master 		-p 30004:22 -p 31004:10050   -p 32004:3306 	 ubuntu-baseimg-ssh:18.04
docker run -d --name mysql-slave 		-p 30005:22 -p 31005:10050   -p 32005:3306   ubuntu-baseimg-ssh:18.04
docker run -d --name nginx1 			-p 30006:22 -p 31006:10050   -p 32006:80 	 ubuntu-baseimg-ssh:18.04
docker run -d --name nginx2 			-p 30007:22 -p 31007:10050   -p 32007:80 	 ubuntu-baseimg-ssh:18.04

# 删除
docker kill  zbx-server zbx-proxy-active zbx-proxy-passive mysql-master mysql-slave nginx1 nginx2 	
docker rm  zbx-server zbx-proxy-active zbx-proxy-passive mysql-master mysql-slave nginx1 nginx2 		
```

```bash
 for i in {30002..30007}; do sed "s@30001@$i@g" "127.0.0.1 30001.xsh" > 127.0.0.1\ $i.xsh ; done
```



### centos 7

https://hub.docker.com/_/centos

```bash
FROM centos:7.5.1804
LABEL AUTHOR=2192383945@qq.com

ENV LANG=zh_CN.utf8

# 基础环境
RUN yum install -y vim wget tree  lrzsz gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop ntpdate lsof bzip2

# 中文
RUN yum -y install kde-l10n-Chinese  glibc-common && localedef -c -f UTF-8 -i zh_CN zh_CN.utf8
	
# ssh服务
RUN yum install -y openssh-server openssh-clients
RUN sed -i 's@#PermitRootLogin prohibit-password@PermitRootLogin yes@g' /etc/ssh/sshd_config && ssh-keygen -A

# 密码
RUN echo 'MBZh/TAmW9CynXO0' | passwd --stdin root

# 前台命令
ENTRYPOINT  /usr/sbin/sshd -D
```

```bash
# docker build -t centos-baseimage-ssh:7.5.1804 ./


# 启动示例
docker run -d --name c7-zbx-server 			-p 40001:22 -p 41001:10050	 -p 42001:10051  centos-baseimage-ssh:7.5.1804
docker run -d --name c7-zbx-proxy-active 	-p 40002:22 -p 41002:10050   				 centos-baseimage-ssh:7.5.1804
docker run -d --name c7-zbx-proxy-passive 	-p 40003:22 -p 41003:10050   				 centos-baseimage-ssh:7.5.1804
docker run -d --name c7-mysql-master 		-p 40004:22 -p 41004:10050   -p 42004:3306 	 centos-baseimage-ssh:7.5.1804
docker run -d --name c7-mysql-slave 		-p 40005:22 -p 41005:10050   -p 42005:3306   centos-baseimage-ssh:7.5.1804
docker run -d --name c7-nginx1 				-p 40006:22 -p 41006:10050   -p 42006:80 	 centos-baseimage-ssh:7.5.1804
docker run -d --name c7-nginx2 				-p 40007:22 -p 41007:10050   -p 42007:80 	 centos-baseimage-ssh:7.5.1804

docker kill  c7-zbx-server c7-zbx-proxy-active c7-zbx-proxy-passive c7-mysql-master c7-mysql-slave c7-nginx1 c7-nginx2 	
docker rm  c7-zbx-server c7-zbx-proxy-active c7-zbx-proxy-passive c7-mysql-master c7-mysql-slave c7-nginx1 c7-nginx2 		
```

shell中添加一个40001的主机, 然后通过进入session目录，快速生成其他主机

```bash
$ for i in {40002..40007}; do sed "s@40001@$i@g" "127.0.0.1 40001.xsh" > 127.0.0.1\ $i.xsh ; done
```

## 准备zabbix依赖包

### ubuntu 18.04

```bash
apt install iproute2 ntpdate tcpdump telnet traceroute nfs-kernel-server nfs-common lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute gcc openssh-server lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute iotop unzip zip curl vim net-tools -y
```

### centos 7

```bash
yum install vim iotop bc gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl openssl-devel zip unzip zlib-devel   net-tools lrzsz tree ntpdate telnet lsof tcpdump wget libevent libevent-devel curl -y
```

## apt/yum安装zabbix

https://www.zabbix.com/documentation/4.0/zh/manual 手册

https://www.zabbix.com/download 安装手册

https://www.zabbix.com/download_sources 编译安装

公司选择LTS版本, 所以当前以4.0为例。

![image-20210422162028819](http://myapp.img.mykernel.cn/image-20210422162028819.png)

3.0和4.0版本差别不大

unsupported version: 就是之前的版本，2.2.x, 3.0~3.0.3有sql注入漏洞，要解决就**升级zabbix版本**。

### apt安装zabbix

安装包安装zabbix: https://www.zabbix.com/download

选择版本4.0LTS, 操作系统选择ubuntu,1804,mysql, apache。**5.0LTS支持nginx后端**

生成安装文档

| 主机类型      | 安装的包                                |
| ------------- | --------------------------------------- |
| zabbix server | zabbix-server-mysql zabbix-frontend-php |
| web主机       | zabbix-agent                            |

#### 创建数据库

```bash
# 生成apt源  https://repo.zabbix.com/zabbix/4.0/ubuntu/pool/main/z/zabbix-release/
# 不更新源，ubuntu官方源，是3.0.12 是非常旧的，所以使用官方自己的源，才可以使用4.0 
#root@d925f295b10a:~# apt-cache madison zabbix-server-mysql
#zabbix-server-mysql | 1:3.0.12+dfsg-1 | http://mirrors.aliyun.com/ubuntu bionic/universe amd64 Packages
#

# 下载包
wget https://repo.zabbix.com/zabbix/4.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_4.0-3+bionic_all.deb
# 查看文件列表
root@d925f295b10a:~# dpkg -c zabbix-release_4.0-3+bionic_all.deb 
drwxr-xr-x root/root         0 2019-07-30 16:34 ./
drwxr-xr-x root/root         0 2019-07-30 16:34 ./etc/
drwxr-xr-x root/root         0 2019-07-30 16:34 ./etc/apt/
drwxr-xr-x root/root         0 2019-07-30 16:34 ./etc/apt/sources.list.d/
-rw-r--r-- root/root       118 2019-07-30 16:34 ./etc/apt/sources.list.d/zabbix.list # 生成zabbix.list 镜像源文件
drwxr-xr-x root/root         0 2019-07-30 16:34 ./etc/apt/trusted.gpg.d/
-rwxr-xr-x root/root      2083 2019-07-30 16:34 ./etc/apt/trusted.gpg.d/zabbix-official-repo.gpg # key
drwxr-xr-x root/root         0 2019-07-30 16:34 ./usr/
drwxr-xr-x root/root         0 2019-07-30 16:34 ./usr/share/
drwxr-xr-x root/root         0 2019-07-30 16:34 ./usr/share/doc/ # 文档
drwxr-xr-x root/root         0 2019-07-30 16:34 ./usr/share/doc/zabbix-release/
-rw-r--r-- root/root       267 2019-07-30 16:17 ./usr/share/doc/zabbix-release/README.Debian
-rw-r--r-- root/root      1665 2019-07-30 16:34 ./usr/share/doc/zabbix-release/changelog.Debian
-rw-r--r-- root/root       561 2019-07-30 16:17 ./usr/share/doc/zabbix-release/copyright

# 安装
dpkg -i zabbix-release_4.0-3+bionic_all.deb

# 查看
root@d925f295b10a:~# cat /etc/apt/sources.list.d/zabbix.list 
deb-src http://repo.zabbix.com/zabbix/4.0/ubuntu bionic main
deb http://repo.zabbix.com/zabbix/4.0/ubuntu bionic main         # zabbix官方安装地址 http://repo.zabbix.com/zabbix/
##deb指向这4个目录
#conf/                                              23-Nov-2020 15:13                   -
#db/                                                29-Jun-2020 09:06                   -
#dists/                                             29-Jun-2020 09:06                   -
#pool/                                              29-Jun-2020 09:06                   -

##安装包
#pool/main/z/zabbix/
#zabbix-agent_4.0.0-2+bionic_amd64.deb              29-Jun-2020 09:06              171144
#zabbix-agent_4.0.0-2+bionic_i386.deb               29-Jun-2020 09:06              179476
#zabbix-agent_4.0.0-2+trusty_amd64.deb              29-Jun-2020 09:06              168308
#zabbix-agent_4.0.0-2+trusty_i386.deb               29-Jun-2020 09:06              166230
#zabbix-agent_4.0.0-2+xenial_amd64.deb              29-Jun-2020 09:06              168670
#zabbix-agent_4.0.0-2+xenial_i386.deb               29-Jun-2020 09:06              175052

# 更新缓存
apt update

```

开始安装

```bash
apt install zabbix-server-mysql zabbix-frontend-php zabbix-agent

#zabbix-frontend-php 前端页面
```

#### 导入数据库初始化

```bash
# 安装数据库于 40004
apt -y install mysql-server mysql-client


#/etc/mysql/mysql.conf.d/mysqld.cnf 
bind-address            = 0.0.0.0

#重启
/etc/init.d/mysql restart
      
```

|      |                |        |            |         |
| ---- | -------------- | ------ | ---------- | ------- |
|      | zabbix_server1 | zabbix | 172.17.0.% | linux48 |
|      | zabbix_server2 |        |            |         |
|      | zabbix_server3 |        |            |         |



```bash
# mysql -uroot -p
create database zabbix_server1 character set utf8 collate utf8_bin;
create user zabbix@'172.17.0.%' identified by 'linux48';
grant all privileges on zabbix_server1.* to zabbix@'172.17.0.%';
quit;
```

检验zabbix能否连接mysql, 意味着zabbix可以将数据写入mysql

```bash

```



#### zabbix-server配置

#### php时区

#### 启动



### yum安装zabbix

安装包安装zabbix: https://www.zabbix.com/download

选择版本4.0LTS, 操作系统选择centos, 7,mysql, apache

生成安装文档

| 主机类型      | 安装的包                             |
| ------------- | ------------------------------------ |
| zabbix server | zabbix-server-mysql zabbix-web-mysql |
| web主机       | zabbix-agent                         |

#### 创建数据库

```bash
# 添加源
rpm -Uvh https://repo.zabbix.com/zabbix/4.0/rhel/7/x86_64/zabbix-release-4.0-2.el7.noarch.rpm

# 查看生成文件
#[root@12246dc07380 ~]# rpm -ql zabbix-release
#/etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX #key
#/etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591
#/etc/yum.repos.d/zabbix.repo # 仓库 
#/usr/share/doc/zabbix-release-4.0 # 文档
#/usr/share/doc/zabbix-release-4.0/GPL

# 仓库 
# [root@12246dc07380 ~]# cat /etc/yum.repos.d/zabbix.repo
# [zabbix]
# name=Zabbix Official Repository - $basearch
# baseurl=http://repo.zabbix.com/zabbix/4.0/rhel/7/$basearch/ # $basearch == x86_64
# enabled=1 # 启用仓库 
# gpgcheck=1 # gpgkey检验
# gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591 # gpgkey位置

# [zabbix-debuginfo]
# name=Zabbix Official Repository debuginfo - $basearch
# baseurl=http://repo.zabbix.com/zabbix/4.0/rhel/7/$basearch/debuginfo/
# enabled=0
# gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591
# gpgcheck=1

# [zabbix-non-supported]
# name=Zabbix Official Repository non-supported - $basearch
# baseurl=http://repo.zabbix.com/non-supported/rhel/7/$basearch/
# enabled=1
# gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX
# gpgcheck=1

# baseurl位置 http://repo.zabbix.com/zabbix/4.0/rhel/7/x86_64/
#../
#debuginfo/                                         29-Mar-2021 09:29                   -
#repodata/                                          29-Mar-2021 09:29                   - # baseurl指向这个位置
#zabbix-agent-4.0.0-2.el7.x86_64.rpm                01-Oct-2018 09:40              388552 # rpm包

# 清理缓存
yum clean all

```

开始安装

```bash
yum install zabbix-server-mysql zabbix-web-mysql zabbix-agent

#zabbix-web-mysql 前端页面
```



#### 导入数据库初始化

#### zabbix-server配置

#### php时区

#### 启动

### 替换apache为nginx?

**5.0LTS支持nginx后端**

nginx location php转发至php-fpm

替换意义不大，apache并发小，监控系统并发不大，不给客户使用，只有运维、开发、测试、领导访问(10个人在线不错了）。

所以不需要替换

