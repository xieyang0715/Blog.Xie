---
title: 一次本地docker数据丢失的惨痛
date: 2021-10-29 15:26:48
tags:
- docker
- 个人日记
---



# 前言
<!--more-->

现在docker特别火，为了更熟悉的使用，平常工作中，我都尽量使用docker完成工作任务。

之前在本地制作的docker，可以真正的完成日常工作，目前我负责几个事情，每个用户完成一个特别的事情让环境隔离开，最近因为windows更新，把目录修改了一下，导致所有的docker镜像不可用，数据丢失，非常可恨。

> 好在一些重要的文档早已托管git, 目前只是重做环境

以下描述基于docker的工作环境怎么搭建

<!--more-->
# 部署工作环境

要使用命令行时，我首先采用通过dockerfile制作一个基础镜像，这个镜像需要有以下功能

- 正常的包
- 正常的时间
- 正常的语言

```dockerfile
FROM ubuntu:18.04
LABEL AUTHOR=2192383945@qq.com
# 基础环境
RUN sed -i.bak -e 's@archive.ubuntu.com@mirrors.aliyun.com@g'  -e 's@security.ubuntu.com@mirrors.aliyun.com@g' /etc/apt/sources.list && \
        apt update && \
        apt install --no-install-recommends iproute2 ntpdate tcpdump telnet traceroute nfs-kernel-server nfs-common lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute gcc make openssh-server lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute iotop unzip zip lsof make curl iputils-ping net-tools vim jq tzdata git bash-completion python3-pip jq -y && \
        pip3 install yq  -i https://pypi.tuna.tsinghua.edu.cn/simple 
        

# 添加ssh服务 docker -p xx:22
RUN sed -i 's@#PermitRootLogin prohibit-password@PermitRootLogin yes@g' /etc/ssh/sshd_config && \
        service ssh restart

# 清理
ENV LANG=zh_CN.utf8
ENV TZ=Asia/Shanghai 
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get install -y locales  language-pack-zh-hans && \
    && localedef -i zh_CN -c -f UTF-8 -A /usr/share/locale/locale.alias zh_CN.UTF-8 && \
    ln -fs /usr/share/zoneinfo/${TZ} /etc/localtime  && echo ${TZ} > /etc/timezone && dpkg-reconfigure --frontend noninteractive tzdata && apt autoremove && apt clean && rm -rf /tmp/* /var/tmp/* /var/lib/apt/lists/*

# 密码
RUN echo 'root:123python' | chpasswd

# 前台命令
ENTRYPOINT  /usr/sbin/sshd -D
```

> 1. --no-install-recommends 只安装主依赖的包
> 2. `language-pack-zh-hans` 汉语包 + `dockerhub ubuntu`的汉语配置 + `ENV LANG=zh_CN.utf8`
> 3. `tzdata` + `ENV TZ=Asia/Shanghai `
> 4. windows中这个ENV, 一定要写成KEY=VALUE格式，每行一格，经过测试这样才正常。
> 5. <strong style="color: red">如果想使用其他系统镜像，在附录中</strong>

现在开始构建

```bash
docker build -t registry.cn-shanghai.aliyuncs.com/grasp_base_images/kubernetes-work:20211029 ./
```

> 使用下面的centos:
>
> ```bash
> docker build -t  registry.cn-shanghai.aliyuncs.com/grasp_base_images/centos:7.5-openssh ./
> ```

测试环境是否正常：时间和语言

```bash
 docker run -it --rm --entrypoint=/bin/bash registry.cn-shanghai.aliyuncs.com/grasp_base_images/kubernetes-work:20211029 
```

> ```bash
> root@8fce70817e64:/# printenv
> # 环境变量中有我们关心的环境变量
> LANG=zh_CN.utf8
> TZ=Asia/Shanghai
> DEBIAN_FRONTEND=noninteractive
> TERM=xterm
> SHLVL=1
> PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
> _=/usr/bin/printenv
> root@8fce70817e64:/#
> root@8fce70817e64:/# date # 时间正常
> 2021年 10月 29日 星期五 15:45:19 CST
> root@8fce70817e64:/# help hash
> hash: hash [-lr] [-p 路径名] [-dt] [名称 ...] # 显示了中文
>     记住或显示程序位置。
> 
>     确定并记住每一个给定 NAME 名称的命令的完整路径。
>     如果不提供参数，则显示已经记住的命令的信息。
> 
>     选项：
>       -d                忘记每一个已经记住的 NAME 的位置
>       -l                以可作为输入重用的格式显示
>       -p pathname       使用 pathname 路径作为 NAME 命令的全路径
>       -r                忘记所有记住的位置
>       -t                打印记住的每一个 NAME 名称的位置，如果指定了多个
>                 NAME 名称，则每个位置前面会加上相应的 NAME 名称
> 
>     参数：
>       NAME              每个 NAME 名称会在 $PATH 路径变量中被搜索，并且添加到记住的命令
>     列表中。
> 
>     退出状态：
>     返回成功，除非 NAME 命令没有找到或者使用了无效的选项。
> ```

接下来就是避免丢失数据了，将/home目录挂载到windows硬盘，然后操作容器只基于普通用户即可。

这样自己的普通用户家目录下的所有数据将在windows盘上

```bash
docker run -d --restart always -p 30092:22 --name k8s-work -v G:/dockerfile/baseimage/ubuntu1804/DATA:/home/ -v G:/dockerfile/baseimage/ubuntu1804/ROOTDATA:/root  --cap-add SYS_ADMIN --device /dev/fuse registry.cn-shanghai.aliyuncs.com/grasp_base_images/kubernetes-work:20211029 


docker run -d  -p 30093:22 --name centos-env  -v G:/dockerfile/baseimage/centos7/ROOTDATA:/root  --cap-add SYS_ADMIN --device /dev/fuse  registry.cn-shanghai.aliyuncs.com/grasp_base_images/centos:7.5-openssh
```

> 1. always 开机就启动工具，非常方便
> 2. 30092暴露给xshell
> 3. 名称
> 4. 挂载持久数据

启动时，win10右下角会提示是否共享此目录，同意。

之后就进行系统初始化一些用户, 为了方便，统一定义用户名`项目名-环境名`, 密码统一`123python`, 这样xshell直接复制就可以使用。

```bash
# 目前我们有2个项目 yun, ishop
# 目前每个项目有2个环境， 测试和生产
for i in yun ishop; do 
	for j in test prod ; do 
		useradd -m  $i-$j -s /bin/bash; 
		echo "$i-$j:123python" | chpasswd; 
	done 
done
```

> 1. -m `ubuntu的useradd`不会自动创建家目录。
> 2. `-s 新用户登陆之后`统一使用bash, 才会有更多的bash特性。

验证windows有数据, ok



# xshell连接

如下图，我们只需要使用相同主机和端口，不同用户、相同密码，就实现了隔离性的操作不同业务

![image-20211029155405789](http://myapp.img.mykernel.cn/image-20211029155405789.png)

# 附录

## ubuntu

```dockerfile
FROM ubuntu:18.04
LABEL AUTHOR=2192383945@qq.com
# 基础环境
RUN sed -i.bak -e 's@archive.ubuntu.com@mirrors.aliyun.com@g'  -e 's@security.ubuntu.com@mirrors.aliyun.com@g' /etc/apt/sources.list && \
        apt update && \
        apt install iproute2 ntpdate tcpdump telnet traceroute nfs-kernel-server nfs-common lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute gcc make openssh-server lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute iotop unzip zip lsof make curl iputils-ping net-tools vim jq tzdata -y

# 添加ssh服务 docker -p xx:22
RUN sed -i 's@#PermitRootLogin prohibit-password@PermitRootLogin yes@g' /etc/ssh/sshd_config && \
        service ssh restart

# 清理
ENV LANG=zh_CN.utf8
ENV TZ=Asia/Shanghai 
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get install -y locales  language-pack-zh-hans && rm -rf /var/lib/apt/lists/* \
    && localedef -i zh_CN -c -f UTF-8 -A /usr/share/locale/locale.alias zh_CN.UTF-8 && \
    ln -fs /usr/share/zoneinfo/${TZ} /etc/localtime  && echo ${TZ} > /etc/timezone && dpkg-reconfigure --frontend noninteractive tzdata
EXPOSE 22 
# 密码
RUN echo 'root:123python' | chpasswd

# 前台命令
ENTRYPOINT  /usr/sbin/sshd -D
```

## centos

```dockerfile
FROM centos:7.5.1804
LABEL AUTHOR=1062670898@qq.com

ENV LANG=zh_CN.UTF-8 TZ=Asia/Shanghai

# 基础环境
RUN yum install -y vim wget tree  lrzsz gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop ntpdate lsof bzip2 tzdata

# 中文
RUN yum -y install kde-l10n-Chinese  glibc-common && localedef -c -f UTF-8 -i zh_CN zh_CN.utf8
	
# ssh服务
RUN yum install -y openssh-server openssh-clients
RUN sed -i 's@#PermitRootLogin prohibit-password@PermitRootLogin yes@g' /etc/ssh/sshd_config && ssh-keygen -A

# 密码
RUN echo '123python' | passwd --stdin root

EXPOSE 22 

# 前台命令
ENTRYPOINT  /usr/sbin/sshd -D
```

## alpine

```dockerfile
root@harbor01:~/weizhixiu# cat dockerfile/baseimage-dockerfile/Dockerfile 
FROM alpine

LABEL AUTHOR=songliangcheng QQ=1062670898

#更改源
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
#安装常用软件
RUN apk update && apk add bash tzdata iotop gcc gcc libgcc libc-dev libcurl libc-utils pcre-dev zlib-dev libnfs make pcre pcre2 zip unzip net-tools pstree wget libevent libevent-dev iproute2 tcpdump vim
# 这里是安装glibc,参考https://github.com/sgerrand/alpine-pkg-glibc 官方文档
RUN apk --no-cache add ca-certificates && \
wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub 

COPY glibc-2.33-r0.apk  glibc-bin-2.33-r0.apk   glibc-i18n-2.33-r0.apk  ./
RUN apk add glibc-2.33-r0.apk  glibc-bin-2.33-r0.apk  glibc-i18n-2.33-r0.apk && rm -f glibc-2.33-r0.apk  glibc-bin-2.33-r0.apk  glibc-i18n-2.33-r0.apk
RUN echo 'en_AG en_AU en_BW en_CA en_DK en_GB en_HK en_IE en_IN en_NG en_NZ en_PH en_SG en_US en_ZA en_ZM en_ZW zh_CN zh_HK zh_SG zh_TW zu_ZA' | xargs -n1 | xargs -I {} /usr/glibc-compat/bin/localedef -i {} -f UTF-8 {}.UTF-8

# 语言 时区
ENV LANG=zh_CN.UTF-8 TZ=Asia/Shanghai

CMD ["/bin/bash"]
```

