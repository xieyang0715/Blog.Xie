---
title: simon老师分享
date: 2020-09-27 08:55:28
tags:  马哥企业教练
toc: true
---


------------------------------
# 1. CentOS 7 内核优化   

```bash
cat > /etc/sysctl.d/k8s.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp.keepaliv.probes = 3
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp.max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp.max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.top_timestamps = 0
net.core.somaxconn = 16384
EOF
```

然后执行sysctl --system

<!--more-->



各参数注释



1. net.ipv4.ip_forward, 支持转发



2. docker, 必须打开的选项

net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1



3. 当系统有容器运行时，需要设置为1

fs.may_detach_mounts = 1



4. overcommit_memory

0， 表示内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。
1， 表示内核允许分配所有的物理内存，而不管当前的内存状态如何。
2， 表示内核允许分配超过所有物理内存和交换空间总和的内存

vm.overcommit_memory=1



5. OOM就是out of memory的缩写，遇到内存耗尽、无法分配的状况。kernel面对OOM的时候，咱们也不能慌乱，要根据OOM参数来进行相应的处理。

值为0：内存不足时，启动 OOM killer。
值为1：内存不足时，有可能会触发 kernel panic（系统重启），也有可能启动 OOM killer。
值为2：内存不足时，表示强制触发 kernel panic，内核崩溃GG（系统重启）。

vm.panic_on_oom=0

6. 表示同一用户同时可以添加的watch数目（watch一般是针对目录，决定了同时同一用户可以监控的目录数量）

fs.inotify.max_user_watches=89100

7. 所有进程最大的文件数

fs.file-max=52706963

8. 单个进程可分配的最大文件数

fs.nr_open=52706963

9. KeepAlive的空闲时长，或者说每次正常发送心跳的周期，默认值为7200s（2小时）

net.ipv4.tcp_keepalive_time = 600

10. 在tcp_keepalive_time之后，没有接收到对方确认，继续发送保活探测包次数，默认值为9（次）

net.ipv4.tcp.keepaliv.probes = 3

11. KeepAlive探测包的发送间隔，默认值为75s

net.ipv4.tcp_keepalive_intvl = 15

12. Nginx 之类的中间代理一定要关注这个值，因为它对你的系统起到一个保护的作用，一旦端口全部被占用，服务就异常了。 tcp_max_tw_buckets 能帮你降低这种情况的发生概率，争取补救时间。

net.ipv4.tcp.max_tw_buckets = 36000



13. tw_reuse 只对客户端起作用，开启后客户端在1s内回收

14. 这个值表示系统所能处理不属于任何进程的socket数量，当我们需要快速建立大量连接时，就需要关注下这个值了。 

net.ipv4.tcp_max_orphans

15. 出现大量fin-wait-1

 首先，fin发送之后，有可能会丢弃，那么发送多少次这样的fin包呢？fin包的重传，也会采用退避方式，在2.6.358内核中采用的是指数退避，2s，4s，最后的重试次数是由**tcp_orphan_retries来限制的。**

16. tcp_syncookies是一个开关，是否打开SYN Cookie功能，该功能可以防止部分SYN攻击。tcp_synack_retries和tcp_syn_retries定义SYN的重试次数。

17. tcp_max_syn_backlog 进入SYN包的最大请求队列.默认1024.对重负载服务器,增加该值显然有好处.

18. 我们的线上web服务器在访问量很大时，就会出现网络连接丢包的问题，通过dmesg命令查看日志，发现如下信息：

kernel: ip_conntrack: table full, dropping packet.
kernel: printk: 1 messages suppressed.
kernel: ip_conntrack: table full, dropping packet.
kernel: printk: 2 messages suppressed.
kernel: ip_conntrack: table full, dropping packet.

centos5 netfilter 参数配置文件：

/proc/sys/net/ipv4/netfilter/ip_conntrack_max或者/proc/sys/net/ipv4/ip_conntrack_max

centos6 netfilter 参数配置文件：

/proc/sys/net/netfilter/nf_conntrack_max

19. tcp_max_syn_backlog是指定所能接受SYN同步包的最大客户端数量，即半连接上限；

20. 在使用 iptables 做 nat 时，发现内网机器 ping 某个域名 ping 的通，而使用 curl 测试不通, 原来是 net.ipv4.tcp_timestamps 设置了为 1 ，即启用时间戳

21. net.core.somaxconn 是Linux中的一个kernel参数，表示socket监听（listen）的backlog上限。什么是backlog呢？backlog就是socket的监听队列，当一个请求（request）尚未被处理或建立时，他会进入backlog。而socket server可以一次性处理backlog中的所有请求，处理后的请求不再位于监听队列中。当server处理请求较慢，以至于监听队列被填满后，新来的请求会被拒绝。 



------------------------------
# 2. K8S坑 kubeasz 2.2.1

------------------------------
	1.  关闭宿主机的NetworkManager, 不然/etc/resolv.conf search会影响k8s
	2. 内核优化使用上面的内核参数




------------------------------
# 3. 给大家分享一个升级内核的方法
------------------------------
默认就是centos7的3.1内核，可以升级到4.x内核。
首先第一步打开rpm包链接，下载内核，链接放在这里：http://elrepo.org/linux/kernel/el7/x86_64/RPMS/


下载下来之后，使用yum localinstall 进行安装。最后再设置一下grub即可，这是一个流程，我把他编写进了一个脚本

```bash
#!/bin/bash
# upgrade kernel by Simon

yum localinstall -y kernel-lt*
if [ $? -eq 0 ];then
	grub2-set-default 0 && grub2-mkconfig -o /etc/grub2.cfg
	grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
fi
echo "please reboot your system quick!!!"
```

------------------------------
# 4. 系统初始化

centos初始化

```bash
yum install -y vim wget tree  lrzsz gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop
```

ubuntu初始化

```bash
# apt purge ufw lxd lxd-client lxcfs lxc-common 
# apt install iproute2 ntpdate tcpdump telnet traceroute nfs-kernel-server nfs-common lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute gcc openssh-server lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute iotop unzip zip
```





#  5. CentOS镜像下载

------------------------------
Centos系统ISO镜像各个版本下载链接：https://mirrors.cnnic.cn/centos/

#  6. nginx配置

##  6.1 locatoin /a -> /

`api.youninyouqu.com/service-user/user/login -> http://localhost:30081/user/login`

```nginx
server {
        listen 443 ssl http2;
        server_name api.youninyouqu.com;

		location /service-user {
                proxy_pass http://localhost:30081/;

                proxy_set_header   Host             $host;
                proxy_set_header   X-Real-IP        $remote_addr;
                proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
}

```

## 6.2 location /a -> /a

 `api.youninyouqu.com/service-user/user/login -> http://localhost:30081/service-user/user/login`

```
server {
        listen 443 ssl http2;
        server_name api.youninyouqu.com;

		location /service-user {
                proxy_pass http://localhost:30081;

                proxy_set_header   Host             $host;
                proxy_set_header   X-Real-IP        $remote_addr;
                proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
}
```



# 7. nginx 日志切割

```bash
这是nginx的，道理都是一样的，你改改就能用
LOGS_PATH=/usr/local/nginx/logs/history
CUR_LOGS_PATH=/usr/local/nginx/logs
YESTERDAY=$(date -d "yesterday" +%Y-%m-%d)
mv ${CUR_LOGS_PATH}/access.log ${LOGS_PATH}/access_${YESTERDAY}.log
mv ${CUR_LOGS_PATH}/error.log ${LOGS_PATH}/error_${YESTERDAY}.log
## 像nginx主进程发送USR1信号，USR1信号是重新打开日志文件
kill -USR1 $(cat /usr/local/nginx/logs/nginx.pid)

```

或者nginx配置

```bash
log_format access_json escape=json '{"@timestamp":"$time_iso8601",'
 '"host":"$server_addr",'
 '"clientip":"$remote_addr",'
 '"size":$body_bytes_sent,'
 '"responsetime":$request_time,'
 '"upstreamtime":"$upstream_response_time",'
 '"upstreamhost":"$upstream_addr",'
 '"http_host":"$host",'
 '"url":"$uri",'
 '"domain":"$host",'
 '"tcp_xff":"$http_x_forwarded_for",'
 '"xff":"$proxy_protocol_addr",'
 '"referer":"$http_referer",'
 '"user-agent":"$http_user_agent",'
 '"method":"$request",'
 '"request-body":"$request_body",'
 '"deviceno":"$http_deviceno",'
 '"model":"$http_model",'
 '"buildversion":"$http_buildversion",'
 '"status":"$status"}';
map $time_iso8601 $logdate {
      '~^(?<ymd>\d{4}-\d{2}-\d{2})' $ymd;
      default 'date-not-found';
}
    access_log  /mnt/data1/logs/access-frontend-$logdate.log  access_json;

```



# 8. windows文本向linux中传送

```bash
Simon企业教练 17:08:08
初学者喜欢在windows里面写好脚本，比如使用notepad++等编辑器，然后传到Linux上执行。不过大部分情况下会出现报错，比如

Simon企业教练 17:08:57
执行脚本报错，脚本很简单，也没有常规性的错误，报“: bad interpreter: No such file or directory”错。一看这错，看看他是不是在windows下编写的脚本，然后在上传到linux服务器的……果然。
原因：在DOS/Windows里，文本文件的换行符为rn，而在unix系统里则为n，所以DOS/Windows里编辑过的文本文件到了unix里，每一行都多了个^M。

Simon企业教练 17:09:26
解决：
1）重新在linux下编写脚本；
2）vim   :% s/r//g    :% s/^M//g （^M输入用Ctrl+v, Ctrl+m）
附：sh -x 脚本文件名 ，可以单步执行并回显结果，有助于排查复杂脚本问题。

Simon企业教练 17:10:03
喜欢在windows下写脚本的注意了，不止是脚本，包括文本也同样，可能得不到你想要的结果但是还不好找到这种原因

```



# 9. 提高make编译速度

```bash

make -j 4
make 使用更多的cpu 内核
```



# 10.服务器可以ping通IP，ping不通域名

```bash
检查DNS是否有配置
样例：
cat /etc/resolv.conf

nameserver 114.114.114.114
nameserver 8.8.8.8
```



# 11. 给大家介绍个编辑软件，

最近看到的，愿意尝试的同学可以看看。我目前还没使用，哈哈。

https://www.wolai.com/



# 12.  nexus

我指的就是你nexus加个代理仓库。就是你上面nexus图片里面的，只是再加个源。比如加个阿里云的代理仓库



可以了吗？如果不行，那我怀疑是你的deploy路径有问题。你需要检查pom.xml文件有没有问题，你这个整个流程是没有问题的。
正常情况下，maven会根据pom.xml的配置进行解析，解析的路径就是你私服的路径。那个pom.xml文件里面有个grouprd，grouprd的路径就是对应的私服的路径，路径是一层一层的。
这点检查检查

对了，阿里云的仓库也换成这个 http://maven.aliyun.com/nexus/content/groups/public/



1. 代理仓库

   ![image-20200927102552891](http://myapp.img.mykernel.cn/image-20200927102552891.png)

2. setings.xml配置

   ```xml
   [root@wzx common]# cat /usr/local/apache-maven/conf/settings.xml
   <?xml version="1.0" encoding="UTF-8"?>
   
   <settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
   
   
     <mirrors>
   		     <mirror>    
         <id>nexus-aliyun</id>  
         <name>nexus-aliyun</name>
         <!--<url>http://maven.aliyun.com/nexus/content/groups/public</url>  -->
   			<url>http://nexus.youwoyouqu.io/nexus/content/groups/public/</url>
         <mirrorOf>central</mirrorOf>    
       </mirror>
     </mirrors>
   
     <profiles>
       <profile>
   
          <id>nexus</id>
           <repositories>
               <!-- 公司私服配置 -->
               <repository>
                   <id>public-wzx</id>
                   <url>http://nexus.youwoyouqu.io/nexus/content/groups/public/</url>
                   <releases>
                       <enabled>true</enabled>
                       <updatePolicy>always</updatePolicy>
                       <checksumPolicy>warn</checksumPolicy>
                   </releases>
                   <snapshots>
                       <enabled>true</enabled>
                       <updatePolicy>always</updatePolicy>
                       <checksumPolicy>warn</checksumPolicy>
                   </snapshots>
               </repository>
           </repositories>
   
       </profile>
   
     </profiles>
   
     <activeProfiles>
   <activeProfile>nexus</activeProfile>
     </activeProfiles>
   </settings>
   
   ```

3. ```xml
   [root@wzx common]# cat pom.xml 
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
   
       <repositories>
           <repository>
               <id>wzx-nexus</id>
               <name>wzx-nexus</name>
               <url>http://nexus.youwoyouqu.io/nexus/content/groups/public/</url>
           </repository>
       </repositories>
       <distributionManagement>
           <repository>
               <id>releases-wzx</id>
               <name>Internal Releases</name>
               <url>http://nexus.youwoyouqu.io/nexus/content/repositories/releases/</url>
           </repository>
           <snapshotRepository>
               <id>snapshots-wzx</id>
               <name>Internal Snapshots</name>
               <url>http://nexus.youwoyouqu.io/nexus/content/repositories/snapshots/</url>
           </snapshotRepository>
       </distributionManagement>
   </project>
   ```



```bash
test
```



![image-20200927100829619](http://myapp.img.mykernel.cn/image-20200927100829619.png)

#  13. CPU信息查看

```bash
cat /proc/cpuinfo
```



# 14. top命令技巧

```
shift +m 按内存大小排序
shift + p 按CPU大小排序
```
# 15. vimrc shell头部自动生成

编辑 $HOME/.vimrc

```
set ignorecase    
set cursorline    
set autoindent   
autocmd BufNewFile *.sh exec ":call SetTitle()"
func SetTitle()
        if expand("%:e") == 'sh'
        call setline(1,"#!/bin/bash")
        call setline(2,"#")
        call setline(3,"#********************************************************************")
        call setline(4,"#Author:                songliangcheng")
        call setline(5,"#QQ:                    1062670898")
        call setline(6,"#Date:                  ".strftime("%Y-%m-%d"))
        call setline(7,"#FileName：             ".expand("%"))
        call setline(8,"#URL:                   http://www.magedu.com")
        call setline(9,"#Description：          script")
        call setline(10,"#Copyright (C):        ".strftime("%Y")." All rights reserved")
        call setline(11,"#********************************************************************")
        call setline(12,"")
        endif
endfunc
autocmd BufNewFile * normal G
```

> ignorecase, 忽略大小写
>
> cursorline  \#设置光标所在行的标识线
>
> autoindent  #自动缩进



# 16. zbxtable报表zabbix

zbxTable 是一个开源的 Zabbix 报表系统。其主要功能是导出监控指标的详情趋势，设备告警信息，设备巡检报告，图形数据显示查看，问题清单等到excel表格。方便运维人员分析问题汇总与汇报。详情如下：


![image-20200927101106555](http://myapp.img.mykernel.cn/image-20200927101106555.png)

![image-20200927101642228](http://myapp.img.mykernel.cn/image-20200927101642228.png)

# 17. centos容器非中文

![image-20200927113853329](http://myapp.img.mykernel.cn/image-20200927113853329.png)

```bash
yum -y install kde-l10n-Chinese  glibc-common
localedef -c -f UTF-8 -i zh_CN zh_CN.utf8
export LANG=zh_CN.utf8
```

![image-20200927133353735](http://myapp.img.mykernel.cn/image-20200927133353735.png)



# 18.查看内存信息

```bash
cat /proc/meminfo
```



# 19. 熟悉linux目录

```bash
/bin 存放二进制可执行文件(ls,cat,mkdir等)，常用命令一般都在这里
/etc 配置文件
/home 用户家目录
/root 超级用户（系统管理员）的主目录
/sbin 存放二进制可执行文件，超级权限用户才能访问
/dev 设备文件
/mnt 临时文件系统的安装点
/tmp 存放各种临时文件
/boot 存放用于系统引导时使用的各种文件
/lib 存放跟文件系统中的程序运行所需要的共享库及内核模块
/var 用于存放运行时需要改变数据的文件
```



# 20. 某天工作发现网络不稳定，使用mii-tool命令查看网卡后结果是这样的

```bash
mii-tool eth0
SIOCGMIIPHY on 'eth0' failed: Resource temporarily unavailable
```

这种大概率就是网口坏掉了，换一个。当然驱动方面问题也有可能（这种情况概率很小）





# 21. cp命令

cp 命令，主要用来复制文件和目录，同时借助某些选项，还可以实现复制整个目录，以及比对两文件的新旧而予以升级等功能。

cp 命令的基本格式如下：

```bash
[root@localhost ~]# cp [选项] 源文件 目标文件

选项：
-a：相当于 -d、-p、-r 选项的集合，这几个选项我们一一介绍；
-d：如果源文件为软链接（对硬链接无效），则复制出的目标文件也为软链接；
-i：询问，如果目标文件已经存在，则会询问是否覆盖；
-l：把目标文件建立为源文件的硬链接文件，而不是复制源文件；
-s：把目标文件建立为源文件的软链接文件，而不是复制源文件；
-p：复制后目标文件保留源文件的属性（包括所有者、所属组、权限和时间）；
-r：递归复制，用于复制目录；
-u：若目标文件比源文件有差异，则使用该选项可以更新目标文件，此选项可用于对文件的升级和备用。
```



## 21.1 环境准备

```bash
docker run --rm -it --name cp centos:7.7.1908 bash
```



## 21.2复制软链接

```bash
cp -d /etc/system-release ./
```

![image-20200929144056727](http://myapp.img.mykernel.cn/image-20200929144056727.png)

## 21.3覆盖时询问, 当复制目标为已经存在的文件时，centos有提示

```bash
[root@3c3515924fe3 ~]# alias
alias cp='cp -i'      # centos默认cp命令为cp -i的别名
alias egrep='egrep --color=auto'
alias fgrep='fgrep --color=auto'
alias grep='grep --color=auto'
alias l.='ls -d .* --color=auto'
alias ll='ls -l --color=auto'
alias ls='ls --color=auto'
alias mv='mv -i'
alias rm='rm -i'
[root@3c3515924fe3 ~]# 
```

![image-20200929144315693](http://myapp.img.mykernel.cn/image-20200929144315693.png)





## 21.4创建硬链接

```bash
cp -l issue issue.hard
```

![image-20200929144518700](http://myapp.img.mykernel.cn/image-20200929144518700.png)

## 21.5创建软链接文件

```bash
cp -s issue issue.link
```

![image-20200929145247083](http://myapp.img.mykernel.cn/image-20200929145247083.png)



## 21.6复制后目标文件保留源文件的属性

```bash
cp -p issue issue.owner
```



![image-20200929145441425](http://myapp.img.mykernel.cn/image-20200929145441425.png)

## 21.7 复制目录 

```bash
cp -r /var/log/ ./log
```

![image-20200929145532541](http://myapp.img.mykernel.cn/image-20200929145532541.png)

## 21.8 若目标文件比源文件有差异，则使用该选项可以更新目标文件，此选项可用于对文件的升级和备用。

```bash
cp -u source dst ，如果dst文件比source还要最新，那么source文件是不会覆盖dst文件的。
```



# 22. 误删/dev/

工作环境机器执行rm -rf 误删除/dev目录下的文件导致ssh不能成功连接，使用ssh -v查看到详细的报错信息如下所示：

```bash
[root@node1 ~]# ssh -v root@node2
OpenSSH_7.4p1, OpenSSL 1.0.2k-fips 26 Jan 2017
debug1: Reading configuration data /etc/ssh/ssh_config
debug1: /etc/ssh/ssh_config line 58: Applying options for *
debug1: Connecting to node2 192.168.50.132] port 22.
debug1: Connection established.
debug1: permanently_set_uid: 0/0
debug1: identity file /root/.ssh/id_rsa type 1
debug1: key_load_public: No such file or directory
debug1: identity file /root/.ssh/id_rsa-cert type -1
debug1: key_load_public: No such file or directory
```

把/dev目录下的内容删除了，这是因为urandom这些文件不存在了。所以我们需要恢复生成一下才可以。

mknod -m 0666 /dev/urandom c 1 9
有了这一个，立马就能连接ssh了。顺便这里把random也恢复一下

mknod -m 0666 /dev/random c 1 8



## 22.1 准备环境

```bash
# 启动centos基础镜像
docker run --net host --rm -it --name ssh   centos:7.7.1908 bash

# 安装openssh
yum -y install openssh-server

# 编辑 /etc/ssh/sshd_config配置监听在22322
Port 22322

# 启动
[root@e6db13724073 /]# /usr/sbin/sshd-keygen -A
/usr/sbin/sshd-keygen: line 10: /etc/rc.d/init.d/functions: No such file or directory
Generating SSH2 RSA host key: /usr/sbin/sshd-keygen: line 63: success: command not found
Generating SSH2 ECDSA host key: /usr/sbin/sshd-keygen: line 105: success: command not found
Generating SSH2 ED25519 host key: /usr/sbin/sshd-keygen: line 126: success: command not found
--------------------------------------------------------------------
在root用户目录下创建.ssh目录，并服务需要登录的公钥信息到authorized_keys文件中。然后启动服务，查看监听端口是否正常。
--------------------------------------------------------------------
[root@e6db13724073 /]# ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
[root@e6db13724073 /]# cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
[root@e6db13724073 /]# chmod 0600 ~/.ssh/authorized_keys
[root@e6db13724073 /]# /usr/sbin/sshd -D &
[root@songliangcheng /]# passwd root
Changing password for user root.
New password: 
BAD PASSWORD: The password is shorter than 8 characters
Retype new password: 
passwd: all authentication tokens updated successfully.
[root@songliangcheng /]# 

```



## 22.2 登陆ssh

![image-20200929153132756](http://myapp.img.mykernel.cn/image-20200929153132756.png)

## 22.3 删除/dev在登陆

![image-20200929153201794](http://myapp.img.mykernel.cn/image-20200929153201794.png)

![image-20200929153215827](http://myapp.img.mykernel.cn/image-20200929153215827.png)

## 22.4 恢复/dev的urandom

```bash
[root@songliangcheng /]# mknod -m 0666 /dev/urandom c 1 9
[root@songliangcheng /]# mknod -m 0666 /dev/random c 1 8
```

![image-20200929153256347](http://myapp.img.mykernel.cn/image-20200929153256347.png)

![image-20200929153327703](http://myapp.img.mykernel.cn/image-20200929153327703.png)

# 23. tr命令进行大小写转换

```bash
echo "HELLO WORLD" | tr 'A-Z' 'a-z'
hello world
```



# 24. ssh不能登陆

启动sshd服务后，是正常的，但是客户端并不能连接上sshd服务器端。通过查看日志，显示如下：

Could not load host key: /etc/ssh/ssh_host_rsa_key
从提示信息看是sshd守护进程不能加载主机密钥文件，因为找不到这些密钥文件(配置文件/etc/ssh/sshd_config中已定义密钥文件名与路径)。 一般openssh服务正常安装后，主机会自动生成相应的主机密钥文件，但这里因未知原因并没有完成这一步动作，导致无法远程ssh连接。
所以解决方法就是重新生成主机密钥文件

[root@aefe8007a17d ~]# ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key
[root@aefe8007a17d ~]# ssh-keygen -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key
[root@aefe8007a17d ~]# ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key
再次登录发现正常了，远程主机也可以连接。

# 25. nfs卸载问题， umount时目标忙解决办法

nfs挂载出现问题需要卸载之前挂载的NFS文件系统。但是直接 umount /mnt/test 命令也会卡住，卸载失败；所以要尝试使用 -f 参数 umount -f /mnt/test 命令强制卸载，如果这个还遇到'device is busy' 的错误提示，可以多试几次；当然更好的方法是再加上 -l 参数 umount -f -l /mnt/test 进行lazy umount，这个一般都可以了





［root@localhost /］# umount /mnt
umount： /mnt： device is busy

\1. umount -l /mnt

\2. umount -f /mnt

\3. 先查出/mnt被占用的进程
fuser -cu /mnt
kill 掉 pid，再次 umount /mnt



# 26. 进程启动时间及进程内存使用量

ps aux |grep -v USER | sort -nk +4 | tail # 显示消耗内存最多的10个运行中的进程，以内存使用量排序.cpu +3
\# USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND
%CPU # 进程的cpu占用率
%MEM # 进程的内存占用率
VSZ # 进程虚拟大小,单位K(即总占用内存大小,包括真实内存和虚拟内存)
RSS # 进程使用的驻留集大小即实际物理内存大小
START # 进程启动时间和日期
占用的虚拟内存大小 = VSZ - RSS

ps -eo pid,lstart,etime,args # 查看进程启动时间

# 27. 访问misc/cd下面的本地光盘文件
```bash
yum install -y autofs
systemctl enable --now autofs
```

然后yum直接配置file:///misc/cd/，就可以挂载本地光盘了

![image-20201013220114974](http://myapp.img.mykernel.cn/image-20201013220114974.png)

# 28. shell字符串操作

shell变量截取和拼接
\1. 使用# 号截取，删除左边字符，保留右边字符。
echo ${var#*//}*

其中 var 是变量名，# 号是运算符，*// 表示从左边开始删除第一个 // 号及左边的所有字符

\2. 使用 ## 号截取，删除左边字符，保留右边字符。
echo ${var##*/}*
\##*/ 表示从左边开始删除最后（最右边）一个 / 号及左边的所有字符

\3. 字符串拼接
var1="aaa"
var2="bbb"
var3=${var1}${var2}
echo $var3
aaabbb



示例：取基名

```bash
root@songliangcheng:~# echo $path
/etc/netplan/01-netcfg.yaml
root@songliangcheng:~# echo ${path##*/}
01-netcfg.yaml

```

示例：取目录名

```bash
root@songliangcheng:~# echo $path
/etc/netplan/01-netcfg.yaml
root@songliangcheng:~# echo ${path%/*}
/etc/netplan
```

示例：取域名

```bash
root@songliangcheng:~# echo ${domain}
liangcheng.mykernel.cn/abc/123?path=abc
root@songliangcheng:~# echo ${domain%%/*}
liangcheng.mykernel.cn
```

示例：拼接字符串

```bash
root@songliangcheng:~# level=www
root@songliangcheng:~# domain=magedu.com
root@songliangcheng:~# echo ${level}.${domain}
www.magedu.com
```


# 29.linux环境变量介绍

/etc/profile
这是全局的配置，不管哪个用户登录，都会读取
~/.bash_profile 或~/.bash_login 或~/.profile

针对特定用户通过修改用户目录下的~/.bashrc来新增或者修改环境变量

针对所有用户配置环境变量的时候，修改/etc/profile文件

# 30. 避免发生分区识别混乱

使用UUID挂载文件系统
查看UUID命令 blkid
例: /etc/fstab
UUID=94e4e384-0ace-437f-bc96-057dd64f42ee /data ext4 defaults 0 0

# 31. du和df的小技巧


\1. du 和df的结果为什么不一样？
du，du能看到的文件只是一些当前存在的，没有被删除的。他计算的大小就是当前他认为存在的所有文件大小的累加和。
df, 记录的是通过文件系统获取到的文件的大小，他比du强的地方就是能够看到已经删除
的文件，而且统计在内。

\2. du 显示单位技巧
-h 表示使用K，M，G的人性化形式显示

\3. df详细案例
a：显示全部的档案系统和各分割区的磁盘使用情形
i：显示i -nodes的使用量
k：大小用k来表示 (默认值)
t：显示某一个档案系统的所有分割区磁盘使用量
x：显示不是某一个档案系统的所有分割区磁盘使用量
T：显示每个分割区所属的档案系统名称

# 32. 使用 parted 建立大小超过2T的分区

　　1，parted /dev/sdb
　　可以输入p打印磁盘信息，查看分区的情况，找到起始和结束位置。

　　2，mklabel gpt
　　设置分区类型为gpt

　　3，mkpart primary 0% 100%
　　primary指分区类型为主分区，0是分区开始位置，100%是分区结束位置。相同的命令为：mkpart primary 0-1 或者是：mkpart primary 0 XXXXXX结束的空间

　　4，print
　　打印当前分区,查看分区设置是否正确
　　5，quit
　　完成后用quit命令退出。
　　6，mkfs.ext4 /dev/sdb1

# 33. dd 命令的一些技巧

\1. 向磁盘上写一个大文件, 来看写性能
[root@roclinux ~]# dd if=/dev/zero bs=1024 count=1000000 of=/root/1Gb.file

\2. 从磁盘上读取一个大文件, 来看读性能
[root@roclinux ~]# dd if=/root/1Gb.file bs=64k | dd of=/dev/null


\3. 测试内存的操作速度
［root@localhost /］# dd if=/dev/zero of=./a.log bs=10M count=10
记录了10+0 的读入
记录了10+0 的写出
100457600字节(105 MB)已复制，0.524121 秒，222 MB/秒

4.利用 /dev/urandom 进行格式化(清除机密数据，防止被恢复)
[root@roclinux ~]# dd if=/dev/urandom of=/dev/sda



# 34. gzip 压缩级别

gzip test # 默认压缩级别3
数字1～9 越大压缩级别越高，文件越小，同时越消耗CPU，时间越长。

gzip -9 test # 极限压缩



#       35. rpm必会技能

[root@localhost ~]# rpm -ivh your-package # 直接安装
[root@localhost ~]# rpm --force -ivh your-package.rpm # 忽略报错，强制安装

[root@localhost ~]# rpm -ql tree # 查询tree 的所有文件
[root@localhost ~]# rpm -qa|grep tree # 查询tree 安装包信息
[root@localhost ~]# rpm -e tree # 卸载
[root@localhost ~]# rpm -qf /usr/bin/tree# 反向查询，根据文件查询所属安装包

# 36. PHP执行权限导致"FastCGI sent in stderr: "Primary script unknown"


在近期一次上线web应用部署过程中，相同的配置在测试环境中并没有出现任何问题。部署在生产环境3台服务器上时，一直出现“file not found”，查看nginx服务器的error日志出现了如下错误：

020/07/05 16:32:05 [error] 28033#0: *3 FastCGI sent in stderr: "Primary script unknown" while reading response header from upstream, client 192.0.61.175, server: baogebiji.com, request: "POST /www/getClientInfoList.php HTTP/1.1", upstream: "fastcgi://unix:/dev/shm/php-cgi.sock:", host: "! 192.0.61.176", referrer: " http://192.0.61.176/www"
生产环境的部署脚本及配置文件均一摸一样，出现这样的问题感觉到很恐慌。
一、测试php是否正常可以进行
在网站根迷路下，部署test.php文件，访问网站，发现是没有问题的。
二、检查网站运行权限
1、检查nginx.conf文件，nginx运行的用户为app200。
2，检查网站目录，均为app200。
3，检查php运行权限。
查看php的配置文件看看权限

$ cat /usr/local/php/etc/php-fpm.conf
;;;;;;;;;;;;;;;;;;;;;
; FPM Configuration ;
;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;
; Global Options ;
;;;;;;;;;;;;;;;;;;

[global]
pid = run/php-fpm.pid
error_log = log/php-fpm.log
log_level = warning
emergency_restart_threshold = 30
emergency_restart_interval = 60s
process_control_timeout = 5s
daemonize = yes
;;;;;;;;;;;;;;;;;;;;

; Pool Definitions ;
;;;;;;;;;;;;;;;;;;;;

[www]
listen = /dev/shm/php-cgi.sock
listen.backlog = -1
listen.allowed_clients = 127.0.0.1
listen.owner = www
listen.group = www
listen.mode = 0666
user = www
group = www
发现user和group均使用www用户，修改以上www用户为app200后，重启systemctl restart php-fpm。故障解除。
总结：
出现该问题的原因事后回忆，因默认安装使用的均为www用户。在应用提出希望使用app200用户可以拥有更新网站的权限后，将nginx及网站权限调整为app200，php-fpm运行权限为www，网站二级目录没有访问权限导致。

# 37.crontab 使用须知


1. 每五分钟执行一次
*/5 * * * * shell.sh

2. 每一分钟执行一次
* * * * * shell.sh

3. 每个周日 0 点执行
0 0 * * 0 shell.sh

4. 凌晨2至4点，每小时执行一次
0 2-4 * * * shell.sh

5. 凌晨2至4点，18和20点，每小时执行一次
0 2-4,18,20 * * * shell.sh

6. 每周末的凌晨一点钟执行一次
0 1 * * Sun /usr/sbin/raid-check

# 38.sed技巧(一)

1.将test.txt文件的aaa替换成bbb
sed 's/aaa/bbb/g' test.txt

2.将test.txt文件的以aaa开头替换成bbb
sed 's/^aaa/bbb/g' test.txt

3.所有以 2 位数结尾的行后面都被加上.5
sed 's/[0-9][0-9]$/&.5/' ceshi.txt

4.打印从第 5 行开始第一个以 northeast 开头的行之间的所有行。
sed -n '5,/northeast/p' ceshi.txt



# 39. 写一个脚本当内存用量达到80%的时候触发邮件报警

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-10-29
#FileName：             sendmail.sh
#URL:                   http://www.magedu.com
#Description：          A test script from N49025+长沙+赵龙川
#Copyright (C):        2020 All rights reserved
#********************************************************************
SENDTO="2192383945@qq.com"

notify() {
	local title="$(hostname) Warnging: Memory over 80%"
	local mailbody="$(date +'%Y年%m月%d日%T'): $(hostname) Warnging: Memory over 80%"
	echo "$mailbody" | mail -s "$title" $SENDTO
}

main() {
		while true; do
			sleep .5;
			mem_used=$(free | awk '/Mem/{print int($3/$2*100)}')
			[ $mem_used -ge 80 ] && notify && break
		done
}

mail
```

# 40.sed技巧(二)

今日小技巧：

1.追加功能(命令a) 字符串 Hello， World！被加在以 north 开头的各行之后
sed '/^north/a Hello world!' ceshi.txt

2.打印完第 5 行之后， q 让 sed 程序退出。
sed '5q' ceshi.txt

3.删除所有行的首数字
sed 's/^[0-9][0-9]*//g' ceshi.txt

\4. 把字符串里的空格换成换行
sed -i 's/ /\n/g' 文件名

# 41.redis 队列延迟 问题 , 该如何解决?

阻塞读在队列没有数据的时候，
会立即进入休眠状态，
一旦数据到来，
则立刻醒过来。消息的延迟几乎为零。



# 42. 灰度发布, ip分流

上游服务器就是提供服务的服务器，比如web。下游是面向用户的



在下游服务器配置

https://www.cnblogs.com/pyng/p/10435688.html

```bash
upstream test-01.com {
  server 192.168.1.100:8080;
}

upstream test-02.com {
  server 192.168.1.200:8080;
}

server {

  listen 80;
  server_name www.test.com;

  location /  {
      if ( $http_x_forwarded_for ~* ^(212)\.(.*)\.(.*)\.(.*)$){
          proxy_pass http://test-01.com;
          break;
      }
      proxy_pass http://test-02.com;
  }

}


if指令: 判断表达式的值是否为真(true)， 如果为真则执行后面大括号中的内容。

以下是一些条件表达式的常用比较方法：

变量的完整比较可以使用=或!=操作符
部分匹配可以使用或*的正则表达式来表示
~表示区分大小写
~*表示不区分大小写(nginx与Nginx是一样的)
!与!*是取反操作，也就是不匹配的意思
检查文件是否存在使用-f或!-f操作符
检查目录是否存在使用-d或!-d操作符
检查文件、目录或符号连接是否存在使用-e或!-e操作符
检查文件是否可执行使用-x或!-x操作符
正则表达式的部分匹配可以使用括号，匹配的部分在后面可以用$1~$9变量代替
```





# 43. 显示所有用户的cron

```bash
for i in /var/spool/cron/*; do 
	echo ${i}; 
	sed 's/^/\t/' $i; 
	echo; 
done 
```

# 44. ss命令必会。显示查看网络状态信息，包括TCP、UDP连接，端口

今日小技巧：

-a 显示所有网络连接
-l 显示LISTEN状态的连接(连接打开)
-m 显示内存信息(用于tcp_diag)
-o 显示Tcp 定时器x
-p 显示进程信息
-s 连接统计

-d 只显示 DCCP信息 (等同于 -A dccp)
-u 只显示udp信息 (等同于 -A udp)
-w 只显示 RAW信息 (等同于 -A raw)
-t 只显示tcp信息 (等同于 -A tcp)
-x 只显示Unix通讯信息 (等同于 -A unix)

-4 只显示 IPV4信息
-6 只显示 IPV6信息
--help 显示帮助信息
--version 显示版本信息

[root@test ~]# ss -t -a #查看所有的tcp连接

[root@test ~]# ss -u -a #查看所有的udp连接

[root@test ~]# ss -pl #显示LISTEN状态的进程信息

# 45. linux 踢人命令

今日小技巧：

首先使用who命令查看在线用户，然后踢人。
强制踢人命令格式：pkill -kill -t tty
解释：

pkill -kill -t 踢人命令

tty　所踢用户的TTY或者pts/x（x代表数字）
如上踢出liu用户的命令为： pkill -kill -t pts/1
只有root用户才能踢人。
如果同时有二个人用root用户登录，任何其中一个可以踢掉另一个。
任何用户都可以踢掉自己

# 46. 比top更强大的实时监控工具-htop

今日小技巧：

安装：yum install ncurses-devel htop -y
\1. 搜索进程
鼠标点击Search 或者按下F3 或者输入"/"， 输入进程名进行搜索，例如搜索ssh

\2. 按下F4，进入过滤器，相当于关键字搜索，不区分大小写，例如过滤dev

\3. 显示树形结构 输入"t"或按下F5，显示树形结构

\4. 按下F6 就可以选择依照什么来排序，最常排序的内容就是cpu 和memory
\5. 操作进程 F7、F8分别对应nice-和nice+，F9对应kill给进程发信号
\6. 显示某个用户的进程，在左侧选择用户 输入"u"，在左侧选择用户

# 47. 查看tcp连接值

今日小技巧：

netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
CLOSED：无连接是活动的或正在进行
LISTEN：服务器在等待进入呼叫
SYN_RECV：一个连接请求已经到达，等待确认
SYN_SENT：应用已经开始，打开一个连接
ESTABLISHED：正常数据传输状态
FIN_WAIT1：应用说它已经完成
FIN_WAIT2：另一边已同意释放
ITMED_WAIT：等待所有分组死掉
CLOSING：两边同时尝试关闭
TIME_WAIT：另一边已初始化一个释放
LAST_ACK：等待所有分组死掉



# 48. 手动释放linux内存cache

echo 3 > /proc/sys/vm/drop_caches



# 49.uname 必会参数

[root@localhost ~]# uname -a #显示系统信息
Linux localhost.localdomain 2.6.18-238.12.1.el5 #1 SMP Tue May 31 13:23:01 EDT 2011 i686 i686 i386 GNU/Linux

[root@localhost ~]# uname -m #显示系统类型，一般情况下，i386,i686是32位系统，X86，X86_64是64位系统
i686

[root@localhost ~]# uname -n #查看主机名
localhost.localdomain

[root@localhost ~]# uname
Linux





# 50.for 循环1-100

今日小技巧：


样例一：
\#!/bin/bash
for((i=1;i<=100;i++));
do
echo $i
done

样例二：
\#!/bin/bash
for i in $(seq 1 100)
do
echo $i
done

样例三：
\#!/bin/bash
awk 'BEGIN{for(i=1; i<=100; i++) print i}'

样例四：
\#!/bin/bash
for i in {1..100}
do
echo $i
done



# 51. 开启ip转发

今日小技巧：

方法一：(重启会失效)
echo 1 > /proc/sys/net/ipv4/ip_forward

方法二：(永久生效)
vim /etc/sysctl.conf
net.ipv4.ip_forward = 0 //该行的0改为1即可

sysctl -p



# 51.  systemctl和service命令对照表

今日小技巧：


任务
使某服务自动启动 chkconfig --level 3 httpd on
systemctl enable httpd.service

使某服务不自动启动 chkconfig --level 3 httpd off
systemctl disable httpd.service

检查服务状态 service httpd status
systemctl status httpd.service

显示所有已启动的服务 chkconfig --list
systemctl list-units --type=service

启动某服务 service httpd start
systemctl start httpd.service

停止某服务 service httpd stop
systemctl stop httpd.service

重启某服务 service httpd restart
systemctl restart httpd.service

某服务重新加载配置文件 service httpd reload
systemctl reload httpd.service



# 52. systemctl 常用命令

今日小技巧：

\# systemctl #输出已激活单元
\# systemctl list-units #输出已激活单元
\# systemctl --failed #输出运行失败的单元
\# systemctl list-unit-files #查看所有已安装服务
\# systemctl start nginx #启动nginx
\# systemctl stop nginx #停止nginx
\# systemctl restart nginx #重启nginx
\# systemctl reload nginx #重新加载nginx配置
\# systemctl status nginx #输出nginx运行状态
\# systemctl is-enabled nginx #检查nginx是否配置为自动启动
\# systemctl enable nginx #开机自动启动nginx
\# systemctl disable nginx #取消开机自动启动nginx
\# systemctl help nginx #显示nginx的手册页
\# systemctl daemon-reload #重新载入 systemd，扫描新的或有变动的单元
\# systemctl reboot #重启
\# systemctl poweroff #退出系统并停止电源
\# systemctl suspend #待机

查看是否安装了某个服务，systemctl list-unit-files | grep xxx  大部分情况下比rpm -qa | grep xxx好用一些

systemctl cat sshd   可以直接查看/usr/lib/systemd/system/sshd.service文件内容

systemctl enable --now sshd  ，设置开机自启动的同时顺便启动服务，相当于  systemctl enable sshd; systemctl start sshd      。与之对应的操作  systemctl disable --now sshd



# 53. 临时和永久关闭Selinux

今日小技巧：

临时关闭：

[root@localhost ~]# getenforce
Enforcing

[root@localhost ~]# setenforce 0
[root@localhost ~]# getenforce
Permissive



永久关闭：
[root@localhost ~]# vim /etc/sysconfig/selinux
SELINUX=enforcing 改为 SELINUX=disabled
重启服务reboot



# 54. awk 一些内置函数使用技巧

今日小技巧：

\1. 使用awk内置函数获取随机数
awk 'BEGIN{srand();fr=int(100*rand());print fr;}'

\2. 逐行读取外部文件(getline使用方法）
awk 'BEGIN{while(getline < "/etc/passwd"){print $0;};close("/etc/passwd");}'

\3. 调用外部应用程序(system使用方法）
awk 'BEGIN{b=system("ls -al");print b;}'

\4. 字符串分割（split使用）
awk 'BEGIN{info="this is a test";split(info,tA," ");print length(tA);for(k in tA){print k,tA[k];}}'



# 55. dns软件的选型

今日小技巧:

DNS在企业内网环境中的应用比较广泛,那么市面上一些常用的dns软件该如何选型呢？
dnsmasq：适合小型的应用环境,解析记录直接保存在文件中类似于hosts文件一样.使用较为简单方便
bind: 应用比较广泛,目前来说互联网上大部分的dns服务器都使用的是bind来构建的,默认也是配置文件方式管理解析记录,配置比较复杂,管理解析记录不太友好
powerdns：成立于1990年,也是一个老牌dns服务器软件了.默认支持mysql来存储解析记录,并佩带了web管理页面.安装配置及使用很方便





# 56. 域名解析流程

今日小技巧:
这里假设本地某主机去请求www.magedu.com域名时的流程
1,本地主机查询本地dns缓存及本地hosts文件中是否有www.magedu.com域名的记录,如果有直接使用,如果没有则会向本地定义的dns服务器去请求(/etc/resolv.conf)
2,dns服务器收到主机请求则查询dns服务器本地是否有www.magedu.com域名的解析记录,如果有直接返回给客户端,如果没有则dns服务器直接向.根 服务器请求查询
3,根(.)服务器收到dns服务器的查询请求发现是查询.com域的信息,然后根服务器则返回.com域的服务器ip给到dns服务器
4,dns服务器收到.com的服务器IP,则再次向.com的服务器请求magedu.com的域名服务器ip
5,dns服务器收到.com返回magedu.com域名服务器IP则直接再次请求magedu.com域名服务器,查询www的解析记录
6,dns服务器查询到www.magedu.com的解析记录后则直接返回给客户端并缓存此记录
7,客户端主机则拿到www.magedu.com的ip就直接访问到目标主机了,并缓存了此解析记录



# 57. nslookup使用详解

```bash
今日小技巧:
# 直接查询某域名的IP,默认使用系统的dns服务器来查询
nslookup www.baidu.com
# 指定dns服务器查询某域名,指定使用114.114.114.114dns服务器来查询域名
nslookup www.baidu.com 114.114.114.114
# 指定查询某域名的记录类型,默认查询的是A记录
nslookup -qt=MX mail.163.com
可查询的类型有如下
A 地址记录
AAAA 地址记录
AFSDB Andrew文件系统数据库服务器记录
ATMA ATM地址记录
CNAME 别名记录
HINFO 硬件配置记录，包括CPU、操作系统信息
ISDN 域名对应的ISDN号码
MB 存放指定邮箱的服务器
MG 邮件组记录
MINFO 邮件组和邮箱的信息记录
MR 改名的邮箱记录
MX 邮件服务器记录
NS 名字服务器记录
PTR 反向记录
RP 负责人记录
RT 路由穿透记录
SRV TCP服务器信息记录
TXT 域名对应的文本信息
X25 域名对应的X.25地址记录

```

# 58. 网络丢包

故障现象：
有台Linux Centos7.4 的物理机，2张万兆网卡各一个口做的bond0（mode=1主备模式）。
但目前发现存在丢包问题：

1. ping自己网关/同网段内机器都不丢包；
2. ping其他网段的网关不会丢包，但是ping其他网段的机器会丢包；（用了2个测试网段，2台测试机器ip）

系统下ipconfig命令也没发现都错包、丢包；（搜集光衰也正常；关闭一个网口，测试也丢包；也拔插过网线、重启过机器）；
网络侧交换机排查，也没发现有错包问题；
厂家分析系列日志，也没发现有硬件问题；（网卡驱动也没问题；）

请问各位大佬，系统下还有什么方法可以排查下丢包原因的吗？感谢。@Simon企业教练 
具体丢包截图：

![image-20201126153436015](http://myapp.img.mykernel.cn/image-20201126153436015.png)

![image-20201126153446440](http://myapp.img.mykernel.cn/image-20201126153446440.png)

yum install mtr一下，然后就测你这个ip就行，mtr 10.128.11.24

![image-20201126153515211](http://myapp.img.mykernel.cn/image-20201126153515211.png)

过几分钟

![image-20201126153534654](http://myapp.img.mykernel.cn/image-20201126153534654.png)





考虑ip冲突 

```bash
# 如果ipmac改变就退出
watch -eg -d -n1 'arp'

# -e 命令非0, 退出
# -g 发生改变，退出
# -d 变化显示在屏幕上
# -n 1 间隔1s执行一次命令
```







# 59. openssl 自签ssl证书,用于nginx配置https使用

今日小技巧:：

openssl genrsa -out www.xxx.com.key 1024
openssl req -new -x509 -keywww.xxx.com.key -out www.xxx.com.crt
\# ssl自签证书生成好了,直接放到某目录里配置nginx即可







# 60. httpd编译安装常用选项说明

今日小技巧

\# 指定配置文件的目录
--sysconfdir=/etc/httpd
\# 如果是编译安装的apr且指定了特殊目录则需要使用此选项指定apr安装的目录
--with-apr=/usr/local/apr
\# 如果是编译安装的apr-util且指定了特殊目录则需要使用此选项指定apr-util安装的目录
--with-apr-util=/usr/local/apr-util
\# 将http的功能编译为模块
--enable-so
\# 启用ssl
--enable-ssl
\# 启用cgi
--enable-cgi
\# 启用重写
--enable-rewrite
\# 添加zlib
--with-zlib
\# 添加pcre
--with-pcre







# 61.mysql命令行常见查询

```mysql
# 查看支持的存储引擎
mysql>SHOW ENGINES;
# 查看一张表的状态信息
mysql>SHOW TABLE STATUS LIKE 'user'\G

# 查看支持的所有字符集
mysql>SHOW CHARACTER SET;

# 查看所有字符集的排序规则
mysql>SHOW COLLATION;

# 查看全局变量
mysql>SHOW GLOBAL VARIABLES;

# 查看全局变量并过滤显示,支持通配符
mysql>SHOW GLOBAL VARIABLES　LIKE '关键字';

# 设定变量值
mysql>SET GLOBAL 变量名 = '值';

```


# 62. 查询某数据库内的所有表的大小

今日小技巧

需要将table_schema='DB_NAME'中的DB_NAME替换为你实际的数据库名称
SELECT table_name,ENGINE,index_length/1024/1024 "index(MB)",data_length/1024/1024 "data(MB)" FROM information_schema.tables WHERE table_schema='DB_NAME' ORDER BY "index(MB)";





# 63. mysql命令行使用详解

今日小技巧

客户端命令
mysql --help # 查看mysql命令的帮助信息
--host, -h # 指定要连接的远程mysql数据库的IP或域名
--user, -u # 指定连接mysql服务器的用户名
--password,-p # 指定连接mysql服务器的用户的密码
--port,-P # 指定需要连接的mysql服务器的端口
--database,-D # 指定连接mysql服务器之后的默认数据库
--execute,-e # 在非交互式模式下执行mysql指令,如 mysql -e "show databases;"
--version,-V # 显示mysql的版本信息
--html,-H # 执行语句的返回结果以html格式返回
--xml,-X # 执行语句的返回结果以xml格式返回

mysqladmin --help # 显示帮助信息
ping # 检测mysql服务器是否在线
processlist # 查看正在执行的线程列表
status # 查看mysql服务器的状态
--sleep # 睡眠多长时间再执行,循环执行
--count # 执行多少次
extended-status # 显示状态变量及值
variables # 显示服务器变量
flush-privileges# 刷新授权表
flush-threads # 刷新线程池
flush-status # 重置状态变量的值
flush-logs # 二进制日志、中继日志的滚动
flush-hosts # 清除主机的内部信息.
kill # 杀死mysql线程
refresh # 相当于同时执行flush-hosts和flush-logs
shutdown # 关闭mysql服务器
version # 显示mysql服务器版本及状态信息
start-slave # 启动从服务器的线程
stop-slave # 停止从服务器的线程



# 64. 配置vsftpd虚拟用户,使用本地db库作为认证来源

今日小技巧

1、安装相关软件包
yum -y install db4-utils

2、创建虚拟用户映射的本地用户
useradd -d /ftp -s /sbin/nologin vuser
chmod o=rwx /ftp

3、修改vsftpd的主配置文件,
\#vim /etc/vsftpd/vsftpd.conf 内容如下
anonymous_enable=YES
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
listen=YES

pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES
guest_enable=yes #启用用户映射功能
guest_username=vuser #虚拟用户映射的系统用户
user_config_dir=/etc/vsftpd/vuser_conf #虚拟用户的权限目录

4、修改PAM认证
\#vim /etc/pam.d/vsftpd #注释或删除文件里的内容,内容如下
auth required pam_![img](file:///C:\Users\Administrator\AppData\Roaming\Tencent\QQTempSys\%W@GJ$ACOF(TYDYECOKVDYB.png)userdb.so db=/etc/vsftpd/ftpuser
account required pam_![img](file:///C:\Users\Administrator\AppData\Roaming\Tencent\QQTempSys\%W@GJ$ACOF(TYDYECOKVDYB.png)userdb.so db=/etc/vsftpd/ftpuser

5、建立虚拟用户的用户名密码文件
vim /etc/vsftpd/user.txt #内容如下
test #用户名
1111 #test的密码
tech #用户名
2222 #tech的密码
6,生成虚拟用户的db库文件
\#db_load -T -t hash -f /etc/vsftpd/user.txt /etc/vsftpd/ftpuser.db
\#chmod 600 /etc/vsftpd/ftpuser.db
7、给虚拟用户配置访问权限
给test用户配置权限
\#vim /etc/vsftpd/vuser_conf/test 内容如下
local_root=/ftp #设置ftp用户的根目录
anon_upload_enable=yes #可以上传文件
anon_mkdir_write_enable=yes #可以创建目录及写入权限
anon_other_write_enable=yes #用户有其他的权限(如对文件改名覆盖及删除

给tech用户设置权限
\#vim /etc/vsftpd/vuser_conf/tech 内容如下
local_root=/ftp #设置根目录
anon_upload_enable=yes #可以上传文件
anon_mkdir_write_enable=yes #可以创建目录及写入权限
anon_other_write_enable=yes #用户有其他的权限(如对文件改名覆盖及删除
8,重启vsftpd服务
\#service vsftpd restart







# 65. iptables规则的备份与恢复

今日小技巧

\# 备份现有规则到某文件中,一般在操作iptables之前都应该先备份下
iptables-save > /root/iptables-`date +%F`
ls /root

\# 通过备份的规则文件恢复规则
iptables-restore < /root/iptables-2019-10-09
iptables -L -n



# 66. iptables规则避免将管理员电脑IP误拒绝

昨日小技巧

需要将该规则添加到策略首位置。-I 表示则策略首部插入规则，-A 表示在策略尾部追加规则。
iptables -I INPUT -s <your IP> -j ACCEPT



# 67. 关于时间服务(ntpd、chrony)的应用场景

今日小技巧

一般来说在中大型的网络环境中时间服务器基本上来说是必须的,在内网中建立一台时间服务器,其他的服务器都来此服务器同步时间
像虚拟机如果关机或者是被暂停了之后再开机的时候时间还是原来的时间,这个时候就需要从新同步时间,如果时间不对可能对业务环境有影响
比如说像一些分布式系统、集群等相互协调工作的应用都需要时间同步服务来确保每个服务器的时间是一致的.
ntpd、chrony这两款时间管理服务应该怎样选型呢？centos7.4之前的系统默认是提供的ntpd来作为时间管理服务的,之后就是chrony,其实两个都差不多,看个人喜好





# 68.  Linux将命令行执行的命令记录到日志文件中便于审计使用

今日小技巧：


配置方法：
1,编辑/etc/bashrc文件
vim /etc/bashrc
\# 在此文件的最后一行加入如下内容
export PROMPT_COMMAND='{ msg=$(history 1 | { read x y; echo $y; });logger "[hostname- $(hostname)]": "[euid=$(whoami)]":$(who am i):[`pwd`]:"$msg"; }'
\# 保存退出

2,重新加载下bashrc
source /etc/bashrc

3,查看配置结果
\# 在执行如下指令之前,可以随意执行几个命令,以便显示效果
tail -f /var/log/messages



# 69. cisco交换机破解密码或恢复出厂设置

今日小技巧:
cisco交换机破解密码或恢复出厂设置、注:恢复出厂设置只需要操作到第6步或第7步
1、拔下交换机电源线
2、用手按在交换的MODE按键上，插上电源线
3、看到控制台后，松开MODE键
4、在里执行“flash-init”命令
switch: flash-init
5、查看flash文件,在switch：dir flash：，把“config.text”文件改为“config.old”
switch：rename flash:config.text flash:config.old
6、重启交换机，
switch：boot
7、交换机出现是否配置的对话，选择”no“命令
8、进入特权模式查看flash文件。把文件“config.old”改为“config.text”
switch>enable
switch#rename flash:config.old flash:config.text
9、把“config.text”考入系统的”running-config“
switch#copy flash:config.text system:running-config
10、重新设置密码，并保存
copy run start





# 70. mysql审计平台

https://dbawsp.com/1940.html





# 71.  ks文件参考模板

自建机房会使用

不过很多还是采用的云服务器

在企业里面如果是几十台机器，采用这个方法还行，勉强能使用。对于机器数量庞大，实际上在使用的时候光有pxe往往不够，不是很方便。

```bash
今日小技巧：
ks文件参考模板
install
text
#安装时的镜像源,就是yum仓库的地址
url --url=http://1.1.1.1/CentOS/6/os/x86_64
lang en_US
keyboard us
#network --onboot yes --bootproto dhcp --noipv6
rootpw redhat
firewall --service=ssh --port=9618
authconfig --enableshadow --passalgo=sha512
selinux --disabled
timezone Asia/Shanghai
bootloader --location=mbr
#clearpart --none
#zerombr yes
clearpart --all --initlabel
part swap --fstype="swap" --size=8192
part /boot --fstype="ext4" --size=300
part / --fstype="ext4" --grow --size=1
reboot

%packages
@base
vim
lftp
lrzsz
telnet
tree
nscd
gcc
gcc-c++
glibc-devel
libtool
make
crontabs


%post
cd /tmp
# 下载一个初始化系统的脚本并执行,里面可以写一些对系统需要做的基础修改等命令或脚本
wget http://1.1.1.1/ks/install_post.sh
bash install_post.sh && rm -rf install_post.sh

# clean install info
echo "" > /root/anaconda-ks.cfg
%end

```

# 72.dhcp的安装配置

```bash
今日小技巧：
dhcp的安装配置
# 安装dhcp相关软件包
yum install dhcp -y
# 编辑配置文件
vim /etc/dhcp/dhcpd.conf
# 内容如下
subnet 192.168.1.0 netmask 255.255.255.0 {
option routers 192.168.1.253;
option subnet-mask 255.255.255.0;
# filename "pxelinux.0";
# next-server 192.168.0.20;
option time-offset -18000; # Eastern Standard Time
option domain-name-servers 119.29.29.29;
range dynamic-bootp 192.168.1.50 192.168.1.130;
default-lease-time 21600;
max-lease-time 43200;
}
#end

# 启动dhcp并设置开机启动
systemctl start dhcpd
systemctl enable dhcpd

# 查看端口是否监听
netstat -tnlpu
```



# 73. 学习shell

https://www.cnblogs.com/yizhangheka/p/12038261.html#%E5%86%85%E5%AD%98%E7%94%A8%E9%87%8F%E6%8A%A5%E8%AD%A6



# 74. ansible远程执行shell脚本

```bash
今日小技巧：
ansible远程执行shell脚本
# 首先创建一个shell脚本
vim /tmp/test.sh
#!/bin/bash
echo `date` > /tmp/ansible_test.txt
# 然后把该脚本分发到各个机器上
ansible webserver -m copy -a "src=/tmp/test.sh dest=/tmp/test.sh mode=0755"
最后是批量执行该shell脚本
ansible webserver -m shell -a "/tmp/test.sh"
shell模块，还支持远程执行命令并且带管道
ansible webserver -m shell -a "cat /etc/passwd|wc -l"

```





# 75. MySQL命令大全

https://segmentfault.com/a/1190000019792483





# 76. 使用ipmitool管理服务器电源及设置引导设备

今日小技巧:
使用ipmitool管理服务器电源及设置引导设备
IPMItool是一个用于管理和配置,
支持智能平台管理接口（IPMI）1.5版和2.0版规范的
设备的实用程序。 IPMI是一个开放的标准、监控、记录、回收、库存
和硬件实现独立于主CPU，BIOS，以及操作系统的控制权,
服务处理器（或底板管理控制器，BMC）的背后是平台管理的大脑,
其主要目的是处理自主传感器监控和事件记录功能
\# 安装
yum install ipmitool -y

ipmitool 常用方法
-H 指定远端设备的IP或host
-I 指定接口,这里的接口是IPMI协议的接口有如下几种
Interfaces:
open Linux OpenIPMI Interface [default]
imb Intel IMB Interface
lan IPMI v1.5 LAN Interface
lanplus IPMI v2.0 RMCP+ LAN Interface

-U 指定访问远端设备的用户名(ipmi接口的用户名)
-P 指定访问远端设备的用户名的密码

其他参数,见 --help

电源管理
\# 查看电源状态
ipmitool -I lanplus -H xxx.xxx.xxx.xxx -U xxx -P xxx power status
\# 开机
ipmitool -I lanplus -H xxx.xxx.xxx.xxx -U xxx -P xxx power on
\# 强制重启
ipmitool -I lanplus -H xxx.xxx.xxx.xxx -U xxx -P xxx power reset
\# 强制关机,相当于一直按着关机键直到关机为止那种
ipmitool -I lanplus -H xxx.xxx.xxx.xxx -U xxx -P xxx power off
\# 软关机,想当于只按一下关机键那种
ipmitool -I lanplus -H xxx.xxx.xxx.xxx -U xxx -P xxx power soft


BOOT引导管理,如下是支持的设备名称
none : Do not change boot device order
pxe : Force PXE boot
disk : Force boot from default Hard-drive
safe : Force boot from default Hard-drive, request Safe Mode
diag : Force boot from Diagnostic Partition
cdrom : Force boot from CD/DVD
bios : Force boot into BIOS Setup
floppy: Force boot from Floppy/primary removable media

\#修改远端设备下次引导的设备名称
ipmitool -I lanplus -H xxx.xxx.xxx.xxx -U xxx -P xxx chassis bootdev <device>







# 77.推荐两个linux上的网络实时监控工具iptraf-ng、iftop

今日小技巧:
推荐两个linux上的网络实时监控工具iptraf-ng、iftop
安装
yum install epel*
yum install iptraf-ng -y
yum install iftop -y

使用方法
\# 查看某网络接口的网络实时带宽中哪个IP的带宽占用比较大
iftop -i eth0

\# iptraf-ng 它可以显示每个连接以及主机之间传输的数据量
iptraf-ng
\# 执行后根据提示选择监控项



# 78. qemu-img命令使用详解

```bash
今日小技巧
qemu-img命令使用详解
qemu-img是 QEMU/KVM 的磁盘管理工具

qemu-img command [command options]

qemu-img
# fmt 是指磁盘镜像文件格式,qemu-img支持二十多种镜像格式,常用的有qcow,qcow2,qed,raw,vdi,vmdk等
create [-f fmt] [-o options] filename size
-f fmt 指定镜像文件的格式
filename=镜像文件的名字
size=大小K,M,G,T

# 查看镜像文件的信息
info [-f fmt] filename
info temp.img

# 检查镜像文件的一致性,查找镜像文件中的错误信息,目前尽对qcow2,qed,vdi格式的文件检查
check [-f fmt] filename

# 修改镜像文件的大小
resize filename [+|-]size

# 改变镜像文件的后端镜像文件,只有qcow2,和qed格式支持rebase
rebase [-f fmt] [-t cache] [-p] [-u] -b backing_file [-F backing_fmt] filename

# 更改后端的镜像文件
commit [-f fmt] filename

# 镜像文件格式转换
convert [-c][-f fmt][-O output_fmt] [-o options] filename [filename1...] output_filename

# 快照管理
snapshot [-l|-a snapshot|-c snapshot|-d snapshot] filename
-l # 列出镜像文件中的所有快照
-a snapshot # 让镜像文件使用某一个快照
-c snapshot # 创建一个快照
-d snapshot # 删除某一个快照

```



# 79. 私有Registry--Harbor安装

```bash
今日小技巧
私有Registry--Harbor安装
1,安装docker-compose
rpm -ivh https://dl.fedoraproject.org/pub/epel/7/x86_64/Packages/e/epel-release-7-11.noarch.rpm
yum install docker docker-compose -y

systemctl start docker
systemctl enable docker

2,安装harbor
# 申请ssl证书,在阿里云或腾讯云上申请免费的ssl证书并上传到服务器
# 放置/data/harbor/ssl_cert目录下
mkdir /data/harbor/ssl_cert -pv
ls /data/harbor/ssl_cert/
reg.xxxxx.com.crt reg.xxxxx.com.key

# 在线安装
下载在线安装包
https://github.com/goharbor/harbor/releases
在上述链接中下载对应的版本,这里使用的是1.5.2版本的,可自行下载最新版的
wget https://storage.googleapis.com/harbor-releases/harbor-online-installer-v1.5.2.tgz
tar xvf harbor-online-installer-v1.5.2.tgz
cd harbor
vim harbor.cfg
hostname = reg.xxxxx.com
# 这里建议使用https协议,免费的ssl证书在阿里云上很容易就申请到了,因为不用https协议docker那边需要修改配置如果是一两台docker修改倒也无所谓,多的时候就很麻烦了
ui_url_protocol = https
customize_crt = off
ssl_cert = /data/harbor/ssl_cert/reg.xxxxx.com.crt
ssl_cert_key = /data/harbor/ssl_cert/reg.xxxxx.com.key
# 其他的参数根据需要修改
# end

# 需要注意的是docker-compose必须要安装,及本机上不能监听80,443端口
## 修改所有存储数据目录为/data/harbor,默认harbor所属的组件的数据均存储在/data目录下,很不方便,如果本机部署有其他服务的数据也存储在/data目录的话 管理会很不方便
docker-compose.yml prepare docker-compose.chartmuseum.yml
# 分别打开上述文件搜索data关键字,在每一个/data替换为/data/harbor

# 执行安装脚本
./install.sh
# 直到出现以下信息则表示安装成功
 ----Harbor has been installed and started successfully.----

Now you should be able to visit the admin portal at https://reg.xxxxx.com.
For more details, please visit https://github.com/vmware/harbor .

# 测试
浏览器打开https://reg.xxxxx.com
默认用户密码为 admin/Harbor12345

# 命令行登录 registry
docker login reg.xxxxx.com

```



# 80. 无法上网

今日小技巧
无法上网一般是iptables的问题,是启动docker后iptables -F 的结果.
解决方法,重启机器即可解决

# 81. 基于已有的容器制作镜像

```bash
今日小技巧
# 基于已有的容器制作镜像
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
-a, --author 作者信息
-m, --message 提交信息,备注
-c, --change 改变dockerfile 中的指令
-p, --pause 制作镜像时暂停某容器

docker commit -a "Andy_f" -p -m "test image" -c 'CMD ["/bin/httpd","-f","-h","/data/html"]' 源容器名 目标仓库:标签

制作一个nginx的镜像
1,下载一个CentOS的镜像,基于CentOS
docker image pull centos:centos6
2,启动centos6容器
docker run --name t1 -it --rm centos:centos6 /bin/bash
3,定制容器,根据自己需要安装或做相应的设置
rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
yum install nginx -y
vi /etc/nginx/nginx.conf
daemon off;

4,基于当前容器的状态生成新的镜像
docker commit -a "Andy_f" -p -m 'test image' -c 'CMD ["/usr/sbin/nginx","-c","/etc/nginx/nginx.conf"]' t1 test:nginx1

5,测试制作的镜像
docker run --name t2 --rm test:nginx1

```





# 82. go的入门资料

```bash
https://github.com/unknwon/the-way-to-go_ZH_CN/blob/master/eBook/directory.md     
```





# 83. alpine 安装glibc

```bash
今日小技巧
alpine基础镜像制作jdk镜像时的报错
#./java
/bin/sh: ./java: not found
如上所示,基于alpine制作的jdk镜像.已确保制作方式及方法没问题.
而启动容器后执行java报如上错误.

原因是由于
java 是基于glibc开发编译的
alpine 是基于MUSL libc开发编译的
若想知如上二者的区别请自行查阅相关资料(如baidu.com),解决方法如下
1,不使用alpine作为基础镜像来制作,而基于centos基础镜像来制作
2,如果非要使用alpine来制作jdk等其他glibc开发编译的程序,如下
dockerfile中最前面的两个RUN指令如下

# 如下的RUN中是更换软件源为mirrors.ustc.edu.cn,需要注意v3.9是alpine的版本,需要根据自己的版本替换下
RUN echo http://mirrors.ustc.edu.cn/alpine/v3.9/main > /etc/apk/repositories && \
echo http://mirrors.ustc.edu.cn/alpine/v3.9/community >> /etc/apk/repositories && \
# 这里是安装glibc,参考https://github.com/sgerrand/alpine-pkg-glibc 官方文档
RUN apk --no-cache add ca-certificates && \
wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub && \
wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/2.29-r0/glibc-2.29-r0.apk && \
apk add glibc-2.29-r0.apk
```



# 84. xtrabackup实现不落地异地备份

```bash
使用xtrabackup实现不落地异地备份http://www.linuxe.cn/post-592.html 
来学习学习
```

# 85. 在线ping

```bash
介绍个不错的在线ping网站，http://www.webkaka.com/ping.aspx

```



# 86. docker使用存储卷

```bash
今日小技巧：
docker使用存储卷
docker run --name nginx -it -v /data nginx:1.16.0
实例中的/data目录自动挂载到宿主机的目录下，路径可以通过docker inspect nginx查看

```



# 87. 多个容器的卷使用同一个主机目录

```bash
今日小技巧：
多个容器的卷使用同一个主机目录
docker run -it --name n1 -v /docker/volumes/v1:/data nginx:1.16.0
docker run -it --name c2 -v /docker/volumes/v1:/data nginx:1.16.0

```



# 88. 虚拟画板

https://excalidraw.com/



# 89. Aria2的跨平台GUI

https://github.com/persepolisdm/persepolis



# 90. LVS NAT工作模式

今日小技巧：

实现原理：NAT模型其实就是一个多路的DNAT，客户端对VIP进行请求，Director通过事先指定好的调度算法计算出应该转发到那台RS上，并修改请求报文的目标地址为RIP，通过DIP送往RS。当RS响应客户端报文给CIP，在经过Director时，Director又会修改源地址为VIP并将响应报文发送给客户端，这段过程对于用户来说是透明的。

NAT特性：
1）RS和Director必须要在同一个IP网段中。
2）RS的网关必须指向DIP
3）可以实现端口映射
4）请求报文和响应报文都会经过Director
5）RS可以是任意OS
6）DIP和RIP只能是内网IP



# 91. nginx和kill信号

USR1相当于reopen。也就是重新打开文件。而HUP才是对应reload

而且这个日志切割，一定要用mv命令，不能用cp。



# 92. grafana监控kubernetesa

https://mp.weixin.qq.com/s/1hPGCA3gwT7QJ62ro26ZJg

kubegraf 插件监控kuberentes

新建图表所有节点，添加标签, 变量引用

sum(key{mode="idle"}[1m]) by instance





# 93. nginx特性和功能

```bash
今日小技巧
nginx特性和功能
特性
模块化设计、较好的扩展性
高可靠性：一个master启动一或多个worker,每个worker响应多个请求
低内存消耗：10000个keepalive连接在Nginx中仅消耗2.5MB内存（官方数据）
支持热部署：不停机更新配置文件、更新日志文件、更新服务器程序版本
功能
静态web资源服务器，能够缓存打开的文件描述符
支持http/imap/pop3/smtp的反向代理；支持缓存、负载均衡
支持fastcgi(fpm)
模块化，非DSO机制，支持过滤器zip压缩，SSI以及图像大小调整
支持SSL

```



# 94. 交换机故障

https://mp.weixin.qq.com/s?__biz=MzIxMTEyOTM2Ng==&mid=2247492229&idx=1&sn=d9ce593d9d951ededc026f33d9ab6d8b&chksm=9758a7fca02f2eea9a0616883c459e310ebb5c576d9dad730605bb2888729d93c445caeb57a5&mpshare=1&scene=23&srcid=0112DRCKZll1lBD8t3kzRepw&sharer_sharetime=1610417414955&sharer_shareid=526a33875b341a963104be96ad05b723#rd



# 95. mysql读写分离

https://www.dbdocker.com/archives/630/

```bash
简介
Atlas是由 Qihoo 360公司Web平台部基础架构团队开发维护的一个基于MySQL协议的数据中间层项目。它在MySQL官方推出的MySQL-Proxy 0.8.2版本的基础上，修改了大量bug，添加了很多功能特性。而且安装方便。配置的注释写的蛮详细的，都是中文。

主要功能
读写分离
从库负载均衡
IP过滤
自动分表
DBA可平滑上下线DB
自动摘除宕机的DB
下载地址：https://github.com/Qihoo360/Atlas

1.rpm安装
rpm -ivh Atlas-2.2.1.el6.x86_64.rpm
2.配置
cd /usr/local/mysql-proxy/conf
mv test.cnf test.cnf.bak

配置文件解读：
[mysql-proxy]   #模块名字
admin-username = user   #用来控制atlas的用户（不是mysql用户）
admin-password = pwd
proxy-backend-addresses = 10.0.0.160:3306    #后端提供写操作的节点
proxy-read-only-backend-addresses = 10.0.200:3306,10.0.0.120:3306  #提供读的节点
pwds = repl:3yb5jEku5h4=,mha:O2jBXONX098=   #这2个用户是后端数据库真实存在的

daemon = true  #后台运行

keepalive = true    #高可用检查节点的状态
event-threads = 8  #线程的个数（连接池）
log-level = message #日志记录的等级
log-path = /usr/local/mysql-proxy/log  #日志路径
sql-log=ON  #经过atlas路由的sql语句
proxy-address = 0.0.0.0:33060  #代理的地址和端口号
admin-address = 0.0.0.0:2345  #管理地址
charset=utf8    #字符编码
/usr/local/mysql-proxy/bin/mysql-proxyd test start #启动atlas

验证：netstat -lntup | grep proxy

tcp 0 0 0.0.0.0:2345 0.0.0.0:* LISTEN 7719/mysql-proxy
tcp 0 0 10.0.0.140:33060 0.0.0.0:* LISTEN 7719/mysql-proxy

测试：mysql -umha -p... -h 10.0.0.160(配合mha-vip使用) -P 端口33060

读的操作是2台slave服务器
写操作是1台master服务器
环境要求：
需要添加一个应用用户
grant update,insert on *.* to 'app'@'10.0.0.%' identified by '123.com'; #授权并创建用户 查看每个节点是否同步

在atlas中添加用户
 /usr/local/mysql-proxy/bin/encrypt llll  #使用此命令生成加密字符（不同用户间用,隔开）
tF5TeinkMj8=

重启mysql-proxy服务
/usr/local/mysql-proxy/bin/mysql-proxyd test start
验证：
mysql -uapp -plll -h10.0.0.160 -P33060
[hermit auto="1" loop="1" unexpand="0" fullheight="0"]netease_songs#:109998[/hermit]

```

# 96. 磁盘性能监控

```bash
我也来分享个
文档：磁盘性能监控.md
链接：http://note.youdao.com/noteshare?id=e6b279cc798e21b97fe63f6ae4cea6fa&sub=7967988AFFD844B7B5A1CECD59E40441

```



# 97. Nginx进程详解

```bash
今日小技巧：
Nginx进程详解

主进程主要完成如下工作：
读取并验正配置信息；
创建、绑定及关闭套接字；
启动、终止及维护worker进程的个数；
无须中止服务而重新配置工作特性；
控制非中断式程序升级，启用新的二进制程序并在需要时回滚至老版本；
重新打开日志文件，实现日志滚动；
编译嵌入式perl脚本；
worker进程主要完成的任务包括：
接收、传入并处理来自客户端的连接；
提供反向代理及过滤功能；
nginx任何能完成的其它任务；

cache loader进程主要完成的任务包括：
检查缓存存储中的缓存对象；
使用缓存元数据建立内存数据库；

cache manager进程的主要任务：
缓存的失效及过期检验；

```

# 98. 编译安装nginx1.16

```bash
~]# dnf install -y lrzsz psmisc lsof wget ntpdate gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl openssl-devel systemd-devel net-tools iotop bc zip unzip zlib-devel nfs-utils automake libxml2 libxml2-devel libxslt libxslt-devel perl perl-ExtUtils-Embed gd-devel GeoIP GeoIP-devel
~]# wget -O /usr/local/nginx-1.16.1.tar.gz http://nginx.org/download/nginx-1.16.1.tar.gz
~]# tar xf /usr/local/nginx-1.16.1.tar.gz -C /usr/local
~]# cd /usr/local/nginx-1.16.1
nginx-1.16.1]# ./configure --prefix=/usr/local/nginx --with-http_ssl_module \
--with-http_v2_module \
--with-http_realip_module \
--with-http_addition_module \
--with-http_image_filter_module \
--with-http_geoip_module \
--with-http_gunzip_module \
--with-http_stub_status_module \
--with-http_gzip_static_module \
--with-pcre \
--with-stream \
--with-stream_ssl_module \
--with-stream_realip_module
nginx-1.16.1]# make && make install
nginx-1.16.1]# useradd nginx -s /sbin/nologin -u 2000
nginx-1.16.1]# chown nginx.nginx -R /usr/local/nginx
nginx-1.16.1]# cd /usr/local/nginx
# 验证版本
nginx]# sbin/nginx -V

```

# Nginx的反向代理

```bash
今日小技巧：
Nginx的反向代理
Nginx可以通过proxy模块实现反向代理功能，在作为web反向代理服务器时，Nginx复制接收客户端请求，并能够根据URL、客户端参数或者其它的处理逻辑将用户请求调度至上游服务器上（upstream server)。
Nginx在实现反向代理功能时最重要的指令为proxy_pass，它能够将location中定义的某URI代理至指定的上游服务器（组）上。如下面的示例中，location的URI将被替换为上游服务器上的newURI。
### 例如：
[root@mail conf.d]# vim nginx-vhost.conf
server {
listen 80;
server_name www.a.com;
add_header X-Via $server_addr;

location / {
root /www/html/a;
index index.html index.htm;
}
location = /node2 {
proxy_pass http://192.168.9.11/;
}
}
##上游服务器必须要配置相应服务及页面
[root@mail conf.d]# service nginx reload
[root@mail conf.d]# curl www.a.com
<img src="http://www.b.net/images/1.jpg">

```

# HAProxy简介

```bash
今日小技巧：
HAProxy简介

（1）HAProxy 是一款提供高可用性、负载均衡以及基于TCP（第四层）和HTTP（第七层）应用的代理软件，支持虚拟主机，它是免费、快速并且可靠的一种解决方案。 HAProxy特别适用于那些负载特大的web站点，这些站点通常又需要会话保持或七层处理。HAProxy运行在时下的硬件上，完全可以支持数以万计的 并发连接。并且它的运行模式使得它可以很简单安全的整合进您当前的架构中， 同时可以保护你的web服务器不被暴露到网络上。

（2）HAProxy 实现了一种事件驱动、单一进程模型，此模型支持非常大的并发连接数。多进程或多线程模型受内存限制 、系统调度器限制以及无处不在的锁限制，很少能处理数千并发连接。事件驱动模型因为在有更好的资源和时间管理的用户端(User-Space) 实现所有这些任务，所以没有这些问题。此模型的弊端是，在多核系统上，这些程序通常扩展性较差。这就是为什么他们必须进行优化以 使每个CPU时间片(Cycle)做更多的工作。

（3）HAProxy 支持连接拒绝 : 因为维护一个连接的打开的开销是很低的，有时我们很需要限制攻击蠕虫（attack bots），也就是说限制它们的连接打开从而限制它们的危害。 这个已经为一个陷于小型DDoS攻击的网站开发了而且已经拯救

了很多站点，这个优点也是其它负载均衡器没有的。

（4）HAProxy 支持全透明代理（已具备硬件防火墙的典型特点）: 可以用客户端IP地址或者任何其他地址来连接后端服务器. 这个特性仅在Linux 2.4/2.6内核打了cttproxy补丁后才可以使用. 这个特性也使得为某特殊服务器处理部分流量同时又不修改服务器的地址成为可能。

```



# calico网络模式

```bash
在同一子网中使用BGP，跨子网时使用VXLAN或IPIP。


 看了一下Calico的设计说明，VXLAN确实不能与BGP结合使用，VXLAN的跨子网模式是与AWS的VPC一同使用的，在同一子网中，直接使用VPC网络的功能，而跨子网时才启用VXLAN。	
 
 
 若要与BGP协同，需要IPIP+BGP。

```



# HAProxy 配置文件

```bash
今日小技巧：HAProxy 基础配置文件详解
HAProxy 配置文件根据功能和用途，主要有 5 个部分组成，但有些部分并不是必须的， 可以根据需要选择相应的部分进行配置。
1、global 部分
用来设定全局配置参数，属于进程级的配置，通常和操作系统配置有关。

2、defaults 部分
默认参数的配置部分。在此部分设置的参数值，默认会自动被引用到下面的 frontend、

backend 和 listen 部分中，因此，如果某些参数属于公用的配置，只需在 defaults 部分添加一次即可。而如果在 frontend、backend 和 listen 部分中也配置了与 defaults 部分一样的参数，那么defaults 部分参数对应的值自动被覆盖。

3、frontend 部分
此部分用于设置接收用户请求的前端虚拟节点。frontend 是在 HAProxy1.3 版本之后才引入的一个组件，同时引入的还有 backend 组件。通过引入这些组件，在很大程度上简化了 HAProxy 配置文件的复杂性。frontend 可以根据 ACL 规则直接指定要使用的后端

4、backend 部分
此部分用于设置集群后端服务集群的配置，也就是用来添加一组真实服务器，以处理前端用户的请求。添加的真实服务器类似于 LVS 中的real server 节点。

5、listen 部分
此部分是 frontend 部分和 backend 部分的结合体。在 HAProxy1.3 版本之前，

HAProxy 的所有配置选项都在这个部分中设置。为了保持兼容性，HAProxy 新的版本仍然保留了 listen 组件的配置方式。目前在 HAProxy 中，两种配置方式任选其一即可。

```



# MySQL的备份

```bash
，分享给大家https://www.cnblogs.com/f-ck-need-u/p/9018716.html
```



# haproxy 解决集群 session 共享问题

```bash
今日小技巧

用户 IP 识别
haroxy 将用户 IP 经过 hash 计算后 指定到固定的真实服务器上（类似于 nginx 的 IP hash 指令）

配置指令： balance source

backend htmpool
mode http
option redispatch
option abortonclose
balance source
cookie SERVERID
option httpchk GET /index.jsp
server 237server 192.168.81.237:8080 cookie server1 weight 6 check inter 2000 rise 2 fall 3
server iivey234 192.168.81.234:8080 cookie server2 weight 3 check inter 2000 rise 2 fall 3

```

# k8s数据的高可用持久化存储项目

k8s，最近看到一个longhorn的项目，数据的高可用持久化存储项目，可以看看



# varnish软件包中的关键程序

```bash
今日小技巧：
varnish软件包中的关键程序
varnishd：

varnishd是主程序，启动后生成两个进程，Manager Process(父进程)和Cacher Process(子进程)，前者在启动时读入配置文件(稍后会讲到)并fork()出后者,同时具有CLI接口接收管理员的指令以用于管理后者，后者是真正处理缓存业务的进程，其下又可以生成若干线程，这些线程各司其职，完成缓存业务的处理并记录日志。这里的Manager进程和Cacher进程类似于Nginx的Master和Worker

varnishadm：
varnishadm可以用来建立一个CLI来连接varnishd，使用-n名称或者-T和-S参数。如果提供了-n参数，密钥文件和address:port会在共享内存中被查找。如果没有提供的话，那么varnishadm将会查找一个实例而不需要指定一个名字。如果给出一个命令，命令和参数会通过CLI连接发送出去，并且结果会返回给stdout。如果没有命令参数提供，varnishadm会通过cli socket和stdin/stdout来传递命令和输出。

-n ident 通过这个名字连接varnishd
-S secretfile 指定认证的密钥文件。这个需要和提供给varnishd的-S参数一致。只有它可以读取该文件的内容，并验证这个CLI连接。
-t timeout 操作的超时时间。
-T 连接指定地址和端口的管理接口

```



# Ubuntu中安装LAMP

https://mp.weixin.qq.com/s/EHHjPSRCzQBhrm4UthBA1Q



# debian无人值守安装

https://mp.weixin.qq.com/s?__biz=MzU2MjU1OTE0MA==&mid=2247490966&idx=1&sn=f593dd5f07502464ac3638663f6c4fb6&chksm=fc66fc5dcb11754bda7a65a8cf31c220dc18b3411c44b8c4297cc3ed3f2234d8e8dc7ab81c5c&mpshare=1&scene=23&srcid=0123mhAhQkFAhqU9AwrMaUry&sharer_sharetime=1611411730226&sharer_shareid=526a33875b341a963104be96ad05b723#rd



# CPU被挖矿了，却找不到哪个进程！

https://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666550500&idx=1&sn=9e6cc70e53291b16f7feb5de25882b2b&chksm=80dc904fb7ab19591ccec1bf0bf985f076286545c03a680775a659aaa7e05057c5b8d8e45e11&mpshare=1&scene=23&srcid=0124OSJW32r89rZe9zJf5YKK&sharer_sharetime=1611488968087&sharer_shareid=526a33875b341a963104be96ad05b723#rd





# 高可用集群基本概念

```bash
今日小技巧：
高可用集群基本概念：

什么是高可用集群：
所谓高可用集群，就是在出现故障时，可以把业务自动转移到其他主机上并让服务正常运行的集群构架
高可用集群的构架层次：

1. 后端主机层： 这一层主要是正在运行在物理主机上的服务。
2. Message layer： 信息传递层，主要传递心跳信息
2. Cluster Resources Manager(CRM): 集群资源管理器层，这一层是心跳信息传递层管理器。用于管理心跳信息的传递和收集
3. Local Resources Manager(LRM): 本地资源管理器层， 用于对于收集到的心跳信息进行资源决策调整。是否转移服务等等
4. Resource Agent(RA): 资源代理层，这一层主要是具体启动或停止具体资源的脚本。遵循{start|stop|restart|status}服务脚本使用格式

```



# top命令的使用技巧

```bash
https://linux.cn/article-11491-1.html   

```



# 安装haproxy

```bash
今日小技巧：

[root@localhost ~]# tar zxvf haproxy-1.8.13.tar.gz
[root@localhost ~]# cd haproxy-1.8.13/
>>>>>>> 44569c333ad1e5c1b45765b2b839484291efb9ed
[root@localhost haproxy-1.8.13]# make TARGET=linux31
####这里需要使用uname -r查看系统版本centos6.X需要使用TARGET=linux26 centos7.x使用linux31
[root@localhost haproxy-1.8.13]# uname -r #查询系统内核版本
3.10.0-862.el7.x86_64

[root@localhost haproxy-1.8.13]# make install PREFIX=/usr/local/haproxy
[root@localhost haproxy-1.8.13]# mkdir /usr/local/haproxy/conf
[root@localhost haproxy-1.8.13]# cp examples/option-http_proxy.cfg /usr/local/haproxy/conf/haproxy.cfg
```

# keepalived配置文件详解

```bash
今日小技巧：
keepalived配置文件详解
! Configuration File for keepalived

global_defs {
notification_email {
root@xxxx.cn
xxxxx@qq.com
}
notification_email_from keepalived@localhost
smtp_server 127.0.0.1
smtp_connect_timeout 30
router_id LVS_DEVEL
}

vrrp_instance HA_1 {
state BACKUP #master和slave都配置为BACKUP
interface eth0 #指定HA检测的网络接口
virtual_router_id 80 #虚拟路由标识，主备相同
priority 100 #定义优先级，slave设置90
advert_int 1 #设定master和slave之间同步检查的时间间隔
nopreempt #不抢占模式。只在优先级高的机器上设置即可
authentication {
auth_type PASS
auth_pass 1111
}

virtual_ipaddress { #设置虚拟IP，可以设置多个，每行一个
192.168.1.208/24 dev eth0 #MySQL对外服务的IP，即VIP
}
}

virtual_server 192.168.1.208 3306 {
delay_loop 2 #每隔2秒查询real server状态
lb_algo wrr #lvs 算法
lb_kinf DR #LVS模式（Direct Route）
persistence_timeout 50
protocol TCP

real_server 192.168.1.210 3306 { #监听本机的IP
weight 1
notify_down /usr/local/keepalived/bin/mysql.sh
TCP_CHECK {
connect_timeout 10 #10秒无响应超时
bingto 192.168.1.208
nb_get_retry 3
delay_before_retry 3
connect_port 3306
}
}

}

```



# nginx代理tomcat不能获取真实ip地址解决方法

```bash
https://cloud.tencent.com/developer/article/1401248?from=article.detail.1441501
```



# keepalived基于状态码的检查

```bash
今日小技巧：
virtual_server ip 80 {
    real_server 192.168.2.188 80 {
        weight 1
        HTTP_GET {
                url {
                path /index.html
                status_code 200 #http://192.168.2.188/index.html的返回状态码
            }
        connect_timeout 3
        nb_get_retry 3 # keepalived 2.0 使用retry指令代替
        delay_before_retry 3
        }
    }
    ....
}
```



# 小飞猪运维平台开源地址

```bash
https://hub.fastgit.org/small-flying-pigs/devops-api
```



# 优秀博客，很实战

```bash
http://www.linuxdream.cn/wordpress/index.php/2020/06/14/n45019week9/
```

#  容器的压强测试工具

```bash
https://github.com/estesp/bucketbench
```



# 一些简单的面试题

https://blog.csdn.net/weixin_45413603/article/details/103304255



# pod创建流程

```bash
https://mp.weixin.qq.com/s?__biz=MzI3MTI2NzkxMA==&mid=2247494281&idx=1&sn=0c8b5e3bf155b271bfc055579c0116e8&chksm=eac6cba0ddb142b6d3bd897c3ad6e85597495e4689c8cb6c8421ec90d766ac64e24961fc7a6f&mpshare=1&scene=23&srcid=0202gbw0x0NW2kn3k0pzISAv&sharer_sharetime=1612228040502&sharer_shareid=526a33875b341a963104be96ad05b723#rd
```



# 安装Memcached服务器

```bash
今日小技巧
安装Memcached服务器
1、安装libevent
Libevent是一款跨平台的事件处理接口的封装，可以兼容多个操作系统的事件访问。 Memcached的安装依赖于Libevent，因此需要先完成Libevent的安装。
yum install gcc gcc-c++ make -y #yum安装gcc编译环境包
解压软件包
tar xvf libevent-2.1.8-stable.tar.gz
tar xvf memcached-1.5.6.tar.gz
cd libevent-2.1.8-stable/
./configure --prefix=/usr/localbevent
make && make install
到此libevent安装完毕

2、安装Memcached
安装配置时需指定libevent的安装路径
cd ../memcached-1.5.6/
./configure \
--prefix=/usr/local/memcached \
--with-libevent=/usr/localbevent/ #指定libevent安装路径
make && make install

优化memcached服务
创建软连接，方便使用memcached服务命令
ln -s /usr/local/memcached/bin/* /usr/local/bin/

启动服务
启动 memcached（-d：守护进程、-m：指定缓存大小为32M 、-p：指定默认端口11211 、 -u：指定 登陆用户为 root）
memcached -d -m 32m -p 11211 -u root
netstat -antp | grep memcached #查看启动监听端口

```



# 查询mysql中有哪些事务开启但没有进行提交

```bash
https://dbawsp.com/2085.html

```



#  Jvm参数调优

```bash
今日小技巧：

Tomcat 的启动参数位于tomcat的安装目录\bin目录下，如果你是Linux操作系统就是catalina.sh文件，如果你是Windows操作系统那么 你需要改动的就是catalina.bat文件
JAVA_OPTS="$JAVA_OPTS -server -Xms4096m -Xmx4096m -Xmn1024m -Xss256K -XX:+DisableExplicitGC -XX:MaxTenuringThreshold=15 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:+UseCMSCompactAtFullCollection -XX:LargePageSizeInBytes=128m -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/home/gclogs/gc.log -Djava.awt.headless=true"

解释：
-server：更高的性能
-Xms4096m：初始堆内存4g
-Xmx4096m：最大堆内存4g
-Xmn1024m：年轻代1g
-Xss256K：每个线程占用的空间
-XX:+DisableExplicitGC：禁止显示调用gc
-XX:MaxTenuringThreshold=15：在年轻代存活次数
-XX:+UseParNewGC：对年轻代采用多线程并行回收
-XX:+UseConcMarkSweepGC：年老代采用CMS回收
-XX:+CMSParallelRemarkEnabled：在使用UseParNewGC 的情况下, 尽量减少 mark 的时间
-XX:+UseCMSCompactAtFullCollection：在使用concurrent gc 的情况下, 防止 memoryfragmention, 对live object 进行整理, 使 memory 碎片减少
-XX:LargePageSizeInBytes=128m：指定 Java heap的分页页面大小
-XX:+UseFastAccessorMethods：get,set 方法转成本地代码
-XX:+UseCMSInitiatingOccupancyOnly：指示只有在 oldgeneration 在使用了初始化的比例后concurrent collector 启动收集
-XX:CMSInitiatingOccupancyFraction=70：年老代到达70%进行gc
-Djava.awt.headless=true ：Headless模式是系统的一种配置模式。在该模式下，系统缺少了显示设备、键盘或鼠标。
-XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/home/gclogs/gc.log：打印日志信息

```



# 如何使java应用在被oom之前进行预防呢？

```bash
如何使java应用在被oom之前进行预防呢？
比如Used堆内存利用率达到了80%就产生告警，这个时候就要引起重视，如果young gc变得比以前频繁很多，也要注意
最后还有full gc，full gc一次，线上会变成僵死那么一刹那。
假如出现这种情况，一般处理方法如下：
1. 下掉该节点，进行重启释放线程和堆内存空间
2. 在OOM前，定位到占用堆内存最多的某个线程中的某块代码，给开发，说明原因，让他去改代码，改逻辑
3. 扩容，流量均摊

```

# MySQL 慢查询的相关参数解释：

```bash
今日小技巧：
MySQL 慢查询的相关参数解释：

slow_query_log ：是否开启慢查询日志，1表示开启，0表示关闭。

log-slow-queries ：旧版（5.6以下版本）MySQL数据库慢查询日志存储路径。可以不设置该参数，系统则会默认给一个缺省的文件host_name-slow.log

slow-query-log-file：新版（5.6及以上版本）MySQL数据库慢查询日志存储路径。可以不设置该参数，系统则会默认给一个缺省的文件host_name-slow.log

long_query_time ：慢查询阈值，当查询时间多于设定的阈值时，记录日志。

log_queries_not_using_indexes：未使用索引的查询也被记录到慢查询日志中（可选项）。

log_output：日志存储方式。log_output='FILE'表示将日志存入文件，默认值是'FILE'。log_output='TABLE'表示将日志存入数据库，这样日志信息就会被写入到mysql.slow_log表中。MySQL数据库支持同时两种日志存储方式，配置的时候以逗号隔开即可，如：log_output='FILE,TABLE'。日志记录到系统的专用日志表中，要比记录到文件耗费更多的系统资源，因此对于需要启用慢查询日志，又需要能够获得更高的系统性能，那么建议优先记录到文件。

```

# CPU压测

```bash
小技巧
CPU压测：
yum install  stress
压测命令如下：
stress --cpu 4 --io 4 --vm 2 --vm-bytes 128M --timeout 60s &
最后使用top 查看
也可以采用如下方法：
1.测试CPU负荷
$stress –c 4
增加4个cpu进程，处理sqrt()函数函数，以提高系统CPU负荷

2.内存测试
$stress –i 4 –vm 10 –vm-bytes 1G –vm-hang 100 –timeout 100s
新增4个io进程，10个内存分配进程，每次分配大小1G，分配后不释放，测试100S

```

# 检测域名到期脚本

```bash
检测域名到期的shell脚本：查询指定域名的过期时间，并在到期前一周，每天发一封提醒邮件
#!/bin/bash

mail_u=admin@admin.com
#当前日期时间戳，用于和域名的到期时间做比较
t1=`date +%s`

#检测whois命令是否存在，不存在则安装jwhois包
is_install_whois()
{
    which whois >/dev/null 2>/dev/null
    if [ $? -ne 0 ]
    then
    yum install -y epel-release
        yum install -y jwhois
    fi
}

notify()
{
    #e_d=`whois $1|grep 'Expiry Date'|awk '{print $4}'|cut -d 'T' -f 1`
    e_d=`whois $1|grep 'Expiration'|tail -1 |awk '{print $5}' |awk -F 'T' '{print $1}'`
    #如果e_d的值为空，则过滤关键词'Expiration Time'
    if [ -z "$e_d" ]
    then
        e_d=`whois $1|grep 'Expiration Time'|awk '{print $3}'`
    fi
    #将域名过期的日期转化为时间戳
    e_t=`date -d "$e_d" +%s`
    #计算一周一共有多少秒
    n=`echo "86400*7"|bc`
    e_t1=$[$e_t-$n]
    e_t2=$[$e_t+$n]
    if [ $t1 -ge $e_t1 ] && [ $t1 -lt $e_t ]
    then
        python mail.py  $mail_u "Domain $1 will  to be expired." "Domain $1 expire date is $e_d."
    fi
    if [ $t1 -ge $e_t ] && [ $t1 -lt $e_t2 ]
    then
        python mail.py $mail_u "Domain $1 has been expired" "Domain $1 expire date is $e_d." 
    fi
}

#检测上次运行的whois查询进程是否存在
#若存在，需要杀死进程，以免影响本次脚本执行
if pgrep whois &>/dev/null
then
    killall -9 whois
fi

is_install_whois

for d in aaa.com bbb.com  aaa.cn
do
    notify $d
done
```



# ssh代理转发

```bash
https://www.zsythink.net/archives/2422
```



# k8s证书续期

https://sre.ink/kubernetes-invlaid-certs-out-of-date/

kubeadm安装后证书默认1年
升级kubernetes版本时会刷新证书到期时间
在kubernetes1.15版本后可以用命令`kubeadm alpha certs renew all`随时续期一年
1.14及其下版本需要手动创建证书，[例如](https://github.com/yuyicai/update-kube-cert)
证书升级完成后，需要重建kubeconfig文件：

```bash
mv HOME/.kubeHOME/.kube_bak 
mkdir -p HOME/.kube
sudo cp -i /etc/kubernetes/admin.confHOME/.kube/config
sudo chown (id -u):(id -g) $HOME/.kube/config
```



# 使用mysqldump备份恢复

```bash
今日小技巧：
mysql备份之一：
使用mysqldump备份恢复
[root@node1 ~]# mysql -e 'SHOW MASTER STATUS' #查看当前二进制文件的状态, 并记录下position的数字
+------------------+----------+--------------+------------------+
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000003 | 106 | | |
+------------------+----------+--------------+------------------+

[root@node1 ~]# mysqldump --all-databases --lock-all-tables > backup.sql #备份数据库到backup.sql文件中

mysql> CREATE DATABASE TEST1; #创建一个数据库
Query OK, 1 row affected (0.00 sec)

mysql> SHOW MASTER STATUS; #记下现在的position
+------------------+----------+--------------+------------------+
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000003 | 191 | | |
+------------------+----------+--------------+------------------+
1 row in set (0.00 sec)

[root@node1 ~]# cp arb/mysql/mysql-bin.000003 /root #备份二进制文件
[root@node1 ~]# service mysqld stop #停止MySQL
[root@node1 ~]# rm -rf arb/mysql/* #删除所有的数据文件
[root@node1 ~]# service mysqld start #启动MySQL, 如果是编译安装的应该不能启动(需重新初始化), 如果rpm安装则会重新初始化数据库


mysql> SHOW DATABASES; #查看数据库, 数据丢失!
+--------------------+
| Database |
+--------------------+
| information_schema |
| mysql |
| test |
+--------------------+
3 rows in set (0.00 sec)

mysql> SET sql_log_bin=OFF; #暂时先将二进制日志关闭
Query OK, 0 rows affected (0.00 sec)


mysql> source backup.sql #恢复数据，所需时间根据数据库时间大小而定

mysql> SET sql_log_bin=ON; 开启二进制日志

mysql> SHOW DATABASES; #数据库恢复, 但是缺少TEST1
+--------------------+
| Database |
+--------------------+
| information_schema |
| employees |
| mysql |
| test |
+--------------------+
4 rows in set (0.00 sec)

[root@node1 ~]# mysqlbinlog --start-position=106 --stop-position=191 mysql-bin.000003 | mysql employees #通过二进制日志增量恢复数据

mysql> SHOW DATABASES; #现在TEST1出现了！
+--------------------+
| Database |
+--------------------+
| information_schema |
| TEST1 |
| employees |
| mysql |
| test |
+--------------------+
5 rows in set (0.00 sec)

#完成
```



# 使用xtrabackup备份恢复

```bash
1、下载安装xtrabackup
[root@node1 ~]# wget https://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.3.4/binary/redhat/6/x86_64/percona-xtrabackup-2.3.4-1.el6.x86_64.rpm
[root@node1 ~]# yum localinstall percona-xtrabackup-2.3.4-1.el6.x86_64.rpm #需要EPEL源
2、备份过程
[root@node1 ~]# mkdir /extrabackup #创建备份目录
[root@node1 ~]# innobackupex --user=root /extrabackup/ #备份数据
###################提示complete表示成功*********************

[root@node1 ~]# ls /extrabackup/ #看到备份目录
2016-04-27_07-30-48
3、准备一个完全备份
[root@node1 ~]# innobackupex --apply-log /extrabackup/2016-04-27_07-30-48/ #指定备份文件的目录

#一般情况下下面三行结尾代表成功*****************
InnoDB: Starting shutdown...
InnoDB: Shutdown completed; log sequence number 369661462
160427 07:40:11 completed OK!

[root@node1 ~]# cd /extrabackup/2016-04-27_07-30-48/
[root@node1 2016-04-27_07-30-48]# ls -hl #查看备份文件
total 31M
-rw-r----- 1 root root 386 Apr 27 07:30 backup-my.cnf
drwx------ 2 root root 4.0K Apr 27 07:30 employees
-rw-r----- 1 root root 18M Apr 27 07:40 ibdata1
-rw-r--r-- 1 root root 5.0M Apr 27 07:40 ib_logfile0
-rw-r--r-- 1 root root 5.0M Apr 27 07:40 ib_logfile1
drwx------ 2 root root 4.0K Apr 27 07:30 mysql
drwx------ 2 root root 4.0K Apr 27 07:30 performance_schema
drwx------ 2 root root 4.0K Apr 27 07:30 test
-rw-r----- 1 root root 27 Apr 27 07:30 xtrabackup_binlog_info
-rw-r--r-- 1 root root 29 Apr 27 07:40 xtrabackup_binlog_pos_innodb
-rw-r----- 1 root root 117 Apr 27 07:40 xtrabackup_checkpoints
-rw-r----- 1 root root 470 Apr 27 07:30 xtrabackup_info
-rw-r----- 1 root root 2.0M Apr 27 07:40 xtrabackup_logfile
4、恢复数据
[root@node1 ~]# rm -rf /data/* #删除数据文件

***不用启动数据库也可以还原*************

[root@node1 ~]# innobackupex --copy-back /extrabackup/2016-04-27_07-30-48/ #恢复数据, 记清使用方法

#########我们这里是编译安装的mariadb所以需要做一些操作##########
[root@node1 data]# killall mysqld

[root@node1 ~]# chown -R mysql:mysql ./*
[root@node1 ~]# ll /data/ #数据恢复
total 28704
-rw-rw---- 1 mysql mysql 16384 Apr 27 07:43 aria_log.00000001
-rw-rw---- 1 mysql mysql 52 Apr 27 07:43 aria_log_control
-rw-rw---- 1 mysql mysql 18874368 Apr 27 07:43 ibdata1
-rw-rw---- 1 mysql mysql 5242880 Apr 27 07:43 ib_logfile0
-rw-rw---- 1 mysql mysql 5242880 Apr 27 07:43 ib_logfile1
-rw-rw---- 1 mysql mysql 264 Apr 27 07:43 mysql-bin.000001
-rw-rw---- 1 mysql mysql 19 Apr 27 07:43 mysql-bin.index
-rw-r----- 1 mysql mysql 2166 Apr 27 07:43 node1.anyisalin.com.err


[root@node1 data]# service mysqld restart
MySQL server PID file could not be found! [FAILED]
Starting MySQL.. [ OK ]

MariaDB [(none)]> SHOW DATABASES; #查看数据库, 已经恢复
+--------------------+
| Database |
+--------------------+
| information_schema |
| employees |
| mysql |
| performance_schema |
| test |
+--------------------+
5 rows in set (0.00 sec

```

# 基于mysql5.6优化

```bash
今日小技巧：
1）内存利用方面：
innodb_buffer_pool_size
这个是Innodb最重要的参数，和MyISAM的key_buffer_size有相似之处，但也是有差别的。
这个参数主要缓存innodb表的索引，数据，插入数据时的缓冲。
该参数分配内存的原则：
这个参数默认分配只有8M，可以说是非常小的一个值。
如果是一个专用DB服务器，那么他可以占到内存的70%-80%。
这个参数不能动态更改，所以分配需多考虑。分配过大，会使Swap占用过多，致使Mysql的查询特慢。
如果你的数据比较小，那么可分配是你的数据大小+10%左右做为这个参数的值。
例如：数据大小为50M,那么给这个值分配innodb_buffer_pool_size＝64M
设置方法，在my.cnf文件里：
innodb_buffer_pool_size=4G

2）关于日志方面
innodb_log_file_size
作用：指定在一个日志组中，每个log的大小。
结合innodb_buffer_pool_size设置其大小，25%-100%。避免不需要的刷新。
注意：这个值分配的大小和数据库的写入速度，事务大小，异常重启后的恢复有很大的关系。一般取256M可以兼顾性能和recovery的速度。
分配原则：几个日值成员大小加起来差不多和你的innodb_buffer_pool_size相等。上限为每个日值上限大小为4G.一般控制在几个Log文件相加大小在2G以内为佳。具体情况还需要看你的事务大小，数据大小为依据。
说明：这个值分配的大小和数据库的写入速度，事务大小，异常重启后的恢复有很大的关系。
设置方法：在my.cnf文件里：
innodb_log_file_size = 256M

innodb_log_files_in_group
作用：指定你有几个日值组。
分配原则：　一般我们可以用2-3个日值组。默认为两个。
设置方法：在my.cnf文件里：
innodb_log_files_in_group=3

innodb_log_buffer_size：
作用：事务在内存中的缓冲，也就是日志缓冲区的大小， 默认设置即可，具有大量事务的可以考虑设置为16M。
如果这个值增长过快，可以适当的增加innodb_log_buffer_size
另外如果你需要处理大理的TEXT，或是BLOB字段，可以考虑增加这个参数的值。
设置方法：在my.cnf文件里：
innodb_log_buffer_size=3M

3）文件IO分配，空间占用方面
innodb_file_per_table
作用：使每个Innodb的表，有自已独立的表空间。如删除文件后可以回收那部分空间。默认是关闭的，建议打开（innodb_file_per_table=1）
分配原则：只有使用不使用。但DB还需要有一个公共的表空间。
设置方法：在my.cnf文件里：
innodb_file_per_table=1

innodb_file_io_threads
作用：文件读写IO数，这个参数只在Windows上起作用。在Linux上只会等于4，默认即可！
设置方法：在my.cnf文件里：
innodb_file_io_threads=4

innodb_open_files
作用：限制Innodb能打开的表的数据。
分配原则：这个值默认是300。如果库里的表特别多的情况，可以适当增大为1000。innodb_open_files的大小对InnoDB效率的影响比较小。但是在InnoDBcrash的情况下，innodb_open_files设置过小会影响recovery的效率。所以用InnoDB的时候还是把innodb_open_files放大一些比较合适。
设置方法：在my.cnf文件里：
innodb_open_files=800

innodb_data_file_path
指定表数据和索引存储的空间，可以是一个或者多个文件。最后一个数据文件必须是自动扩充的，也只有最后一个文件允许自动扩充。这样，当空间用完后，自动扩充数据文件就会自动增长（以8MB为单位）以容纳额外的数据。
例如： innodb_data_file_path=/disk1/ibdata1:900M;/disk2/ibdata2:50M:autoextend 两个数据文件放在不同的磁盘上。数据首先放在ibdata1 中，当达到900M以后，数据就放在ibdata2中。
设置方法，在my.cnf文件里：
innodb_data_file_path =ibdata1:1G;ibdata2:1G;ibdata3:1G;ibdata4:1G;ibdata5:1G;ibdata6:1G:autoextend

innodb_data_home_dir
放置表空间数据的目录，默认在mysql的数据目录，设置到和MySQL安装文件不同的分区可以提高性能。
设置方法，在my.cnf文件里：（比如mysql的数据目录是/data/mysql/data，这里可以设置到不通的分区/home/mysql下）
innodb_data_home_dir = /home/mysql

```



# redis配置文件详解

```bash
今日小技巧：
redis配置文件详解：
[root@redis ~]# vim /etc/redis.conf
15 ################################## INCLUDES ###################################
30 # include /path/to/local.conf #包含子配置文件
33 ################################ GENERAL #####################################
37 daemonize yes #是否运行为守护进程
41 pidfile ar/run/redis.pid #pid文件位置
45 port 6379 #监听端口
54 tcp-backlog 511 #本地接收缓冲满了，缓存在tcp队列
65 bind 127.0.0.1 #指明监听的地址
71 # unixsocket /tmp/redis.sock #sock文件位置
72 # unixsocketperm 700 #sock文件权限
75 timeout 0 #客户端超时，0表示不超时
99 loglevel notice #指定日志级别，推荐级别大些
104 logfile ar/log/redis/redis.log #指定日志文件位置
114 # syslog-facility local0 #如果使用syslog接受日志，这里设置syslog设备
119 databases 16 #指定数据库数量
121 ################################ 快照 ################################
141 # save "" #关闭save操作
143 save 900 1 #900秒内，一个键发生变化做一次sava
144 save 300 10 #300秒内，十个键发生变化做一次sava
145 save 60 10000 #60秒内，一万个键发生变化做一次sava
160 stop-writes-on-bgsave-error yes #在进行快照备份时，一旦检测到快照发生错误，是否停止
166 rdbcompression yes #是否对文件进行压缩
175 rdbchecksum yes #是否对rdb的镜像文件做校验码检测
178 dbfilename dump.rdb #指明db文件名
188 dir arb/redis/ #指明db文件目录
190 ################################# 复制 #################################
206 # slaveof &lt;masterip&gt; &lt;masterport&gt; #指定redis主节点的ip地址和端口
213 # masterauth &lt;master-password&gt; #指定redis主节点密码（如果没有则不指定）
226 slave-serve-stale-data yes #当slave丢失master或者同步正在进行时，如果发生对slave的服务请求：设置为yes则slave正常提供服务，设置为no，则salve返回client错误
242 slave-read-only yes #设置从库只读
291 # repl-ping-slave-period 10 #slave发送pings到master的间隔时间
303 # repl-timeout 60 #IO超时时间
331 # repl-backlog-size 1mb #同步的backlog（主库连接从库的队列）大小
340 # repl-backlog-ttl 3600 #backlog生存时间
355 slave-priority 100 #slave优先级
371 # min-slaves-to-write 3 #设置slave小于几个master停止写入操作
372 # min-slaves-max-lag 10 #设置slave落后master指定秒，master停止写入操作
379 ################################## 安全 ###################################
392 # requirepass foobared #设置redis连接密码
403 # rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52 #重命名CONFIG命令
408 # rename-command CONFIG "" #关闭CONFIG命令
413 ################################### 限制 ####################################
424 # maxclients 10000 #设置最大连接数
449 # maxmemory &lt;bytes&gt; #使用内存的限制，redis到达限制会删除key，就算key未过期
472 # maxmemory-policy noeviction #内存限制算法，使用此算法删除key
485 ############################## AOF持久化###############################
505 appendonly no #是否启用aof功能
509 appendfilename "appendonly.aof" #aof文件名
535 appendfsync everysec #每次收到写命令，立即写入到磁盘
557 no-appendfsync-on-rewrite no #每秒钟写一次
576 auto-aof-rewrite-percentage 100 #appendfsyn不会执行写操作，而是有操作系统决定
577 auto-aof-rewrite-min-size 64mb #当前的aof大小是上次重写文件的2倍时，则自动启动日志重写过程
601 aof-load-truncated yes #当前aof启动日志过程的最小值
603 ################################ LUA 脚本 ###############################
619 lua-time-limit 5000 #lua脚本执行时间，单位为毫秒
621 ################################ 集群 ###############################
633 # cluster-enabled yes #打开redis集群
641 # cluster-config-file nodes-6379.conf #redis集群配置文件
647 # cluster-node-timeout 15000 #集群节点超时时间，单位毫秒
692 # cluster-slave-validity-factor 10 #slave节点检测因素，开始failover的超时时限是通过factor与timeout的乘积来确定的
711 # cluster-migration-barrier 1 #设置master只有在关联多少slave时才会触发迁移过程
724 # cluster-require-full-coverage yes #如果某一些key space没有被集群中任何节点 覆盖，集群将停止接受写入
729 ################################## 慢查询日志###################################
747 slowlog-log-slower-than 10000 #它决定要对执行时间大于多少微妙（microsecond，1秒=1，000，000微妙）的查询进行记录
751 slowlog-max-len 128 #最多能保存多少条日志，slow log本身是一个FIFO队列。
#如果需要查看慢查询日志，需要使用如下命令
127.0.0.1:6379&gt; SLOWLOG get
753 ################################ 高级配置 ##############################
818 notify-keyspace-events "" #键空间通知，选项为空字符串功能关闭（默认）。
825 hash-max-ziplist-entries 512 #哈希对象保存的键值对数量小于 512 个，会采用线性紧凑格式存储。
826 hash-max-ziplist-value 64 #哈希对象保存的所有键值对的键和值的字符串长度都小于64字节，就会采用线性紧凑存储来节省空间。
831 list-max-ziplist-entries 512 #list保存的键值对数量小于 512 个，会采用线性紧凑格式存储。
832 list-max-ziplist-value 64 #lisi保存的所有键值对的键和值的字符串长度都小于64字节，就会采用线性紧凑存储来节省空间。
839 set-max-intset-entries 512 #限制set编码最大上限
844 zset-max-ziplist-entries 128 # zset保存的键值对数量小于 512 个，使用线性紧凑格式存储
845 zset-max-ziplist-value 64 # zset保存的所有键值对的键和值的字符串长度都小于64字节，就会采用线性紧凑存储来节省空间。
859 hll-sparse-max-bytes 3000 #稀疏格式最大字节限制
879 activerehashing yes #默认值yes，用来控制是否自动重建hash。Active rehashing每100微秒使用1微秒cpu时间排序，以重组Redis的hash表。重建是通过一种lazy方式，写入hash表的操作越多，需要执行rehashing的步骤也越多，如果服务器当前空闲，那么rehashing操作会一直执行。如果对实时性要求较高，难以接受redis时不时出现的2微秒的延迟，则可以设置activerehashing为no，否则建议设置为yes，以节省内存空间。
914 client-output-buffer-limit normal 0 0 0 #显示分配的缓冲区大小，防止内存无节制分配。参数的默认值都为0，意思是不做任何限制。
915 client-output-buffer-limit slave 256mb 64mb 60 #限制从库的复制客户端，缓冲区硬限制为256M，软限制64M，软限制超时时间为60秒（超过这个时间会断开和客户端的连接）
916 client-output-buffer-limit pubsub 32mb 8mb 60 #显示发布订阅的客户端，缓冲区硬限制为32M，软限制为8M，软限制超时时间为60秒（超过这个时间会断开和客户端的连接）
933 hz 10 #redis执行后台任务（关闭客户端连接，清楚未被请求过的过期key）的频率
939 aof-rewrite-incremental-fsync yes #当一个子进程重写aof文件时，如果启用下面这个选项，则文件没生成32M数据会被同步。

```

#  集中式存储与分布式存储区别

```bash
今日小技巧：
：
1、共享存储：NAS，SAN，数据集中存储在一个设备上
2、分布式两种存储方式：
1.分布式一般专门有一个服务器存元数据(元数据节点)，其他节点存储数据元数据节点会分配好数据到各个服务器上 再对数据进行冗余--datablock
2.无专门提供的元数据节点时，那每一个节点都存储了整个集群的完整的元数据，和一部分数据

```

# mongo数据模拟恢复

http://myapp.img.qiniu.mykernel.cn/MongoDB%20Oplog%E6%95%B0%E6%8D%AE%E6%81%A2%E5%A4%8D.html



#  分布式文件系统的常见实现

```bash
今日小技巧：

1、GFS：google file system 海量数据高效存储
2、HDFS: Hadoop Distribute filesystem
元数据都放在元数据节点上，但是这些元数据都是存储在内存中，所以存储文件数量有限，而且整个系统的单点所在，对于存储海量小数据不是很理想，但是试用于一些文件数不多的大文件
3、淘宝开源的存储文件系统
TFS：Taobao file system
开源将元数据存储于关系型数据库或其他高性能存储中，从而能维护海量的文件元数据，
但是带来的劣势：
1.依靠外部的组件来实现文件系统，依赖的组件变多，组件不在完整
2.放在外部存储中，没有放在内存中性能高
4、GlasterFS：
去中心化设计(没有专门的元数据节点)，即无专门提供的元数据节点，每一个节点都存储了整个集群的完整的元数据，和一部分数据
5、ceph：
整合进linux内核当中的分布式文件系统，已经被收录进内核
6、MooseFS：mfs
MogileFS：经过测表明MogileFS的性能比MooseF高

```

# 运维工单的需求有哪些？

```bash
https://www.52wiki.cn/project-2/doc-935/

```

# MogileFS特性

```bash
今日小技巧：
MogileFS特性：
mogilefs的高可用不包括数据库 所以数据库的高可用要自己做

1.基于http 存储 所以非常适用于web站点中
2.数据存储冗余
3.自动文件复制
4.简单名称空间(可取名的范围就叫命名空间，MogileFS没有目录的概念)
5.不共享任何东西 shared-nothing
6.自动提供冗余，不需要raid设备
7.不能追加写，随机写，只能整个文件替换
8.Tracker client传输，管理数据复制，删除，查询，修复及其监控
9.数据通过HTTP/WEBDAV服务上传到Storage node(数据存储时的服务为Mogstored)
10.mysql存储mogilefs的元数据

```

# CentOS 跑K8s，内核建议在4.x以上

分享个技巧  CentOS 跑K8s，内核建议在4.x以上，默认的3.10在频繁创建Pod的场景中（例如CronJob），容易触发Cgroup内存泄露的bug

#  zabbix重要组件说明

今日小技巧：

zabbix重要组件说明：
1）zabbix server:负责接收agent发送的报告信息的核心组件，所有配置、统计数据及操作数据都由它组织进行；
2）database storage：专用于存储所有配置信息，以及由zabbix收集的数据；
3）web interface：zabbix的GUI接口；
4）proxy：可选组件，常用于监控节点很多的分布式环境中，代理server收集部分数据转发到server，可以减轻server的压力；
5）agent：部署在被监控的主机上，负责收集主机本地数据如cpu、内存、数据库等数据发往server端或proxy端；

# 主机自动重启原因定位

```bash
https://www.cnblogs.com/doctormo/p/12619485.html
```

# zabbix常用的监控架构平台

```bash
今日小技巧：
1、server-agentd模式：
这个是最简单的架构了，常用于监控主机比较少的情况下。
2、server-proxy-agentd模式：
这个常用于比较多的机器，使用proxy进行分布式监控，有效的减轻server端的压力。

```

# prometheus的rate与irate内部是如何计算的

```bash
https://zhangguanzhang.github.io/2020/07/30/prometheus-rate-and-irate/
```

# 二篇zabbix实战

```bash
今天分享二篇zabbix实战：
zabbix监控进程的实战：https://www.cnblogs.com/skyflask/articles/8007162.html
zabbix实现业务数据的监控：https://www.cnblogs.com/skyflask/articles/8439115.html
```

# 基于docker的jenkins部署

```bash
基于docker的jenkins部署
docker run -d -u root -p 8080:8080 -v jenkins_home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkinsci/blueocean
```

# Logstash 介绍

```bash
今日小技巧：
LogStash由JRuby语言编写，基于消息（message-based）的简单架构，并运行在Java虚拟机（JVM）上。不同于分离的代理端（agent）或主机端（server），LogStash可配置单一的代理端（agent）与其它开源软件结合，以实现不同的功能。

LogStash的四大组件
Shipper：发送事件（events）至LogStash；通常，远程代理端（agent）只需要运行这个组件即可；
Broker and Indexer：接收并索引化事件；
Search and Storage：允许对事件进行搜索和存储；
Web Interface：基于Web的展示界面
正是由于以上组件在LogStash架构中可独立部署，才提供了更好的集群扩展性。
```

# 分布式系统的三个指标

```bash
1、Partition tolerance，中文叫做"分区容错"。
大多数分布式系统都分布在多个子网络。每个子网络就叫做一个区（partition）。分区容错的意思是，区间通信可能失败。比如，一台服务器放在中国，另一台服务器放在美国，这就是两个区，它们之间可能无法通信
2、Consistency 中文叫做"一致性"。意思是，写操作之后的读操作，必须返回该值。举例来说，某条记录是 v0，用户向 G1 发起一个写操作，将其改为 v1。
3、Availability 中文叫做"可用性"，意思是只要收到用户的请求，服务器就必须给出回应。
用户可以选择向 G1 或 G2 发起读操作。不管是哪台服务器，只要收到请求，就必须告诉用户，到底是 v0 还是 v1，否则就不满足可用性
```

# k8s优化方向

```bash
https://zhuanlan.zhihu.com/p/111244925
```

# 最基础的监控类

```bash
这还不好说？给你个最基础的监控类的https://www.cnblogs.com/zhaojiedi1992/p/zhaojiedi_liunx_65_prometheus_alertmanager_rule.html
```

# 加固系统 

http://blog.mykernel.cn/2020/11/30/linux%E5%AE%89%E5%85%A8%E5%8A%A0%E5%9B%BA/

```bash
1. 账号：ssh：禁用root远程，添加新的root权限账号。
2. 登入失败处理：pam模块，root密码3次尝试锁定15分钟
3. history保存行数及命令时间
4. 周期更新密码
```

# ZooKeeper特征

```bash

ZooKeeper是简单的：ZooKeeper的核心是一个精简文件系统，它提供一些简单的操作和一些额外的抽象操作，例如：排序和通知。
ZooKeeper是富有表现力的：ZooKeeper的原语操作是一组构件（building block），可用于实现很多协调数据结构和协议，例如：分布式队列、分布式锁和一组同级节点中的leader选举（leader election）。
ZooKeeper具有高可用性：ZooKeeper运行在集群上，被设计成具有较高的可用性，因此应用程序可以完全依赖它。ZooKeeper可以帮助系统避免出现单点故障，从而构建一个可靠的应用。
ZooKeeper采用松耦合交互方式：在ZooKeeper支持的交互过程中，参与者之间不需要彼此了解。例如：ZooKeeper可以被用作一个rendezvous机制，让进程在不了解其他进程或网络状况的情况下能够彼此发现并进行交互。参与协调的各方甚至可以不必同时存在，因为一个进程在ZooKeeper中留下一条消息，在该进程结束后，另外一个进程还可以读取这条消息。
ZooKeeper是一个资源库：ZooKeeper提供了一个关于通用协调模式实现和方法的开源共享存储库，能使程序员免于编写这类通用的协议。所有人都能够对这个资源库进行添加和改进，随着时间的推移，会使每个人都从中收益。
ZooKeeper是高性能的：在它的诞生地Yahoo!公司，对于以写为主的工作负载来说，ZooKeeper的基准吞吐量已超过每秒10000个操作；对于常规的以读为主的工作负载来说，吞吐量更是高出好几倍。

```

# sudo

![image-20210318163337961](http://myapp.img.mykernel.cn/image-20210318163337961.png)

![image-20210318163349536](http://myapp.img.mykernel.cn/image-20210318163349536.png)

# ZooKeeper典型使用场景

```bash
今日小技巧：
1、ZooKeeper典型使用场景
数据发布于订阅
特性、用法
发布与订阅即所谓的配置管理，顾名思义就是将数据发布到zk节点上，供订阅者动态获取数据，实现配置信息的集中式管理和动态更新。例如：全局的配置信息、地址列表等。

2、命名服务
特性、用法
这个主要是作为分布式命名服务，通过调用zk的create node api，能够很容易创建一个全局唯一的path，可以将这个path作为一个名称。

3、分布通知
特性、用法
ZooKeeper中特有的watcher注册于异步通知机制，能够很好的实现分布式环境下不同系统之间的通知与协调，实现对数据变更的实时处理。使用方法通常是不同系统都对zk上同一个znode进行注册，监听znode的变化（包括znode本身内容及子节点内容），其中一个系统update了znode，那么另一个系统能够收到通知，并做出相应处理。
使用ZooKeeper来进行分布式通知和协调能够大大降低系统之间的耦合

4、分布式锁
特性、用法
分布式锁，主要得益于ZooKeeper保证数据的强一致性，即zk集群中任意节点（一个zk server）上系统znoe的数据一定相同

5、集群管理
特性、用法
集群机器监控：这通常用于那种对集群中机器状态、机器在线率有较高要求的场景，能够快速对集群中机器变化做出响应。
Master选举：在分布式环境中，相同的业务应用分布在不同的机器上，有些业务逻辑（例如一些耗时的计算、网络IO处理），往往需要让整个集群中的某一台机器进行执行，其余机器可以共享这个结果，这样可以减少重复劳动、提高性能。利用ZooKeeper的强一致性，能够保证在分布式高并发情况下节点创建的全局唯一性。

```

# Kubernetes是什么？我(们)为什么使用？

```bash
今日小技巧：
Kubernetes是什么？我(们)为什么使用？
Kubernetes是Google(基于borg)开源的容器集群管理系统，其提供应用部署、维护、 扩展机制等功能，利用Kubernetes能方便地管理跨机器运行容器化的应用，
其主要功能如下：
1、使用Docker对应用程序包装(package)、实例化(instantiate)、运行(run)。
2、以集群的方式运行、管理跨机器的容器。
3、解决Docker跨机器容器之间的通讯问题。
4、好处多多
```

# nginx 4层也可以透传源IP

```bash
https://docs.nginx.com/nginx/admin-guide/load-balancer/using-proxy-protocol/
```

# StatefulSet

```bash
今日小技巧：
StatefulSet是为了解决有状态服务的问题，其应用场景包括：

1、稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据，基于PVC来实现
2、稳定的网络标志，即Pod重新调度后其PodName和HostName不变，基于Headless Service（即没有Cluster IP的Service）来实现
3、有序部署，有序扩展，即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依次进行（即从0到N-1，在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态），基于init containers来实现
4、有序收缩，有序删除（即从N-1到0）
```

# iftop命令

```bash
https://mp.weixin.qq.com/s?__biz=MzI3MTI2NzkxMA==&mid=2247494944&idx=1&sn=6d0b004e9ce784330026d18ff5100ea2&chksm=eac6cc09ddb1451f6472accd9354e591e70ea17b4e49876053627958260b50c03060ec08c058&mpshare=1&scene=23&srcid=0322zHo9lYuglqWBRbQsd9Mw&sharer_sharetime=1616373403788&sharer_shareid=526a33875b341a963104be96ad05b723#rd
```

#  K8s认证策略（Authentication strategies）

```bash
今日小技巧：
Kubernetes的用户可以使用客户端证书、Bearer Token、身份验证代理或HTTP基本认证，通过身份验证插件来验证API请求。
比如，当HTTP请求到达API server，插件尝试将以下的属性与请求进行关联：
1、Username：用户名，标识最终用户的字符串。通常，Username的值可能像“kube-admin”或者“jane@example.com”。
2、UID：用户的唯一标识标识。
3、Groups：用户组组名。
4、Extra fields：记录用户其他信息的属性。
```

# K8s Calico介绍

```bash
今日小技巧：
K8s Calico介绍
Calico 是一种容器之间互通的网络方案。在虚拟化平台中，比如 OpenStack、Docker 等都需要实现 workloads 之间互连，但同时也需要对容器做隔离控制，就像在 Internet 中的服务仅开放80端口、公有云的多租户一样，提供隔离和管控机制。而在多数的虚拟化平台实现中，通常都使用二层隔离技术来实现容器的网络，这些二层的技术有一些弊端，比如需要依赖 VLAN、bridge 和隧道等技术，其中 bridge 带来了复杂性，vlan 隔离和 tunnel 隧道则消耗更多的资源并对物理环境有要求，随着网络规模的增大，整体会变得越加复杂。我们尝试把 Host 当作 Internet 中的路由器，同样使用 BGP 同步路由，并使用 iptables 来做安全访问策略，最终设计出了 Calico 方案。

```

# tomcat+redis实现session共享

```bash
https://www.alittletiger.com/852.html   
```

# Calico网络模型主要工作组件

```bash
今日小技巧：

Calico网络模型主要工作组件：
1.Felix：运行在每一台 Host 的 agent 进程，主要负责网络接口管理和监听、路由、ARP 管理、ACL 管理和同步、状态上报等。
2.etcd：分布式键值存储，主要负责网络元数据一致性，确保Calico网络状态的准确性，可以与kubernetes共用；
3.BGP Client（BIRD）：Calico 为每一台 Host 部署一个 BGP Client，使用 BIRD 实现，BIRD 是一个单独的持续发展的项目，实现了众多动态路由协议比如 BGP、OSPF、RIP 等。在 Calico 的角色是监听 Host 上由 Felix 注入的路由信息，然后通过 BGP 协议广播告诉剩余 Host 节点，从而实现网络互通。
4.BGP Route Reflector：在大型网络规模中，如果仅仅使用 BGP client 形成 mesh 全网互联的方案就会导致规模限制，因为所有节点之间俩俩互联，需要 N^2 个连接，为了解决这个规模问题，可以采用 BGP 的 Router Reflector 的方法，使所有 BGP Client 仅与特定 RR 节点互联并做路由同步，从而大大减少连接数

```

#  etcd, raft, paxos

```bash
etcd 用wal 格式记录集群日志，性能好得很
raft强一致性协议，国内有一些软件就是采用这种来做的方案

raft 原理简单 。 看paxos原理，失眠都可以治疗好
mysql MGR就用 paxos
```



# K8s-Flannel网络工作原理

```bash

今日小技巧：
K8s-Flannel网络工作原理
Flannel实质上是一种“覆盖网络(overlay network)”，也就是将TCP数据包装在另一种网络包里面进行路由转发和通信，目前已经支持UDP、VxLAN、AWS VPC和GCE路由等数据转发方式。
默认的节点间数据通信方式是UDP转发。

数据从源容器中发出后，经由所在主机的docker0虚拟网卡转发到flannel0虚拟网卡，这是个P2P的虚拟网卡，flanneld服务监听在网卡的另外一端。
Flannel通过Etcd服务维护了一张节点间的路由表，详细记录了各节点子网网段 。
源主机的flanneld服务将原本的数据内容UDP封装后根据自己的路由表投递给目的节点的flanneld服务，数据到达以后被解包，然后直接进入目的节点的flannel0虚拟网卡，然后被转发到目的主机的docker0虚拟网卡，最后就像本机容器通信一下的有docker0路由到达目标容器。
```

# iptables 的认识及使用 实现“增删改查”

```bash
https://blog.csdn.net/weixin_42191545/article/details/109402215
```

# 命令限制网卡速度

```bash
Linux的话你研究下这个wondershaper
```

# kibana报警

```bash
https://blog.csdn.net/whg18526080015/article/details/73812400
```

# 集群网络架构

```bash
今日小技巧：
集群网络架构是 K8s 中比较复杂的，最让用户头痛的方面之一。K8s 拥有众多的 CNI 插件
网络插件 性能 隔离策略 开发者
kube-router 最高 支持
calico 2 支持
canal 3 支持
flannel 3 无 CoreOS
romana 3 支持
Weave 3 支持 Weaveworks

说明：
Flannel: 最成熟、最简单的选择
Calico: 性能好、灵活性最强，目前的企业级主流
Canal: 将Flannel提供的网络层与Calico的网络策略功能集成在一起。
Weave: 独有的功能，是对整个网络的简单加密，会增加网络开销
Kube-router: kube-router采用lvs实现svc网络,采用bgp实现pod网络.
```

#  Prometheus

```bash
今日小技巧：
Prometheus是一套开源的监控&报警&时间序列数据库的组合,起始是由SoundCloud公司开发的。成立于2012年，之后许多公司和组织接受和采用prometheus,他们便将它独立成开源项目，并且有公司来运作.该项目有非常活跃的社区和开发人员，目前是独立的开源项目，任何公司都可以使用它，2016年，Prometheus加入了云计算基金会，成为kubernetes之后的第二个托管项目.google SRE的书内也曾提到跟他们BorgMon监控系统相似的实现是Prometheus。现在最常见的Kubernetes容器管理系统中，通常会搭配Prometheus进行监控。

```

# Grafana

```bash
今日小技巧：
Grafana 是一个开源的时序性统计和监控平台，支持例如 elasticsearch、graphite、influxdb 等众多的数据源，并以功能强大的界面编辑器著称。
Grafana作为通用的可视化工具，支持将多个数据源的数据整合到单个可视化面板中。
因此现在很多企业使用prometheus存k8s监控数据，然后使用Grafana展示各种报表数据。
```

# 衡量磁盘性能方法

```bash
https://www.wabks.com/post/570.html 
```

# 抢先看！Kubernetes v1.21 新特性一览

```bash
https://mp.weixin.qq.com/s?__biz=MzA3MjY1MTQwNQ==&mid=2649853438&idx=1&sn=15b18e649b553114dcfc3c9cab816b59&chksm=871e2a1cb069a30ad5e3b3765562a92fd494040143dbf7aab2cb4e0d97bdfc1a6581d84d7272&mpshare=1&scene=23&srcid=0401vERqnIB5RNRFhn34NIeU&sharer_sharetime=1617270973466&sharer_shareid=526a33875b341a963104be96ad05b723#rd
```



# Iaas、Paas、Saas介绍

```bash
今日小技巧：
Iaas、Paas、Saas介绍
云计算的经典理论上讲三大层：Iaas、Paas、Saas，分布是Infrastructure、Platform、Software as a service。其中的Infrastructure指的是硬件资源虚拟化；Platform指的是软件平台，是应用软件运行的基础平台。
Saas层是直接面向业务用户的
Iaas层是应用运行的底层物理资源
中间的Paas是应用运行的标准平台，它包括基础数据库服务、基础中间件服务、基础开发框架和开发套件、应用部署、管理和运维服务。通过Paas平台这一层，Saas层更专注于自身的业务实现，运行平台和运行中间件由Paas层提供。
```

#  mysql

```bash
全库哪个索引用的次数少，前三
SELECT
 OBJECT_NAME,
 INDEX_NAME,
 COUNT_FETCH,
 COUNT_INSERT,
 COUNT_UPDATE,
 COUNT_DELETE 
FROM
 PERFORMANCE_SCHEMA.table_io_waits_summary_by_index_usage 
ORDER BY
 SUM_TIMER_WAIT 
 LIMIT 3;
```

```bash
全库哪个索引从来没有使用过
SELECT
 OBJECT_SCHEMA,
 OBJECT_NAME,
 INDEX_NAME 
FROM
 PERFORMANCE_SCHEMA.table_io_waits_summary_by_index_usage 
WHERE
 INDEX_NAME IS NOT NULL 
 AND COUNT_STAR = 0 
 AND OBJECT_SCHEMA <> 'mysql' 
ORDER BY
 OBJECT_SCHEMA,
 OBJECT_NAME;
```

# tcping

```bash
给大家分享个小工具，tcping小工具是用于tcp监控的，也可以看到ping 值，即使机房禁PING，服务器禁PING了，也可以通过它来监控服务器的情况。除了ping ，它还有一个功能，监听端口的状态。使用方法一会给大家截图举个例子
```

# java性能分析工具

```bash
https://github.com/oldratlee/useful-scripts/blob/dev-2.x/docs/java.md#-show-busy-java-threads
```

[Linux安全强化](https://mp.weixin.qq.com/s?__biz=MzAxMTkwODIyNA==&mid=2247532568&idx=1&sn=5381a4075a417d12d7ca9894a80e3f00&chksm=9bbbe5f7accc6ce1c39b24d658b15d11a406b2ad23bffeadcb3f57fa7be35214aea411970b84&mpshare=1&scene=24&srcid=0518oHU1hfmxZhW2jWKC9TbK&sharer_sharetime=1621268268140&sharer_shareid=739eb6a8ece84f23406462f4a3912621#rd)

# K8s上部署和优化gitlab 

```bash
  https://tech.souyunku.com/?p=38532
```

# 隐藏进程分析

https://www.cnblogs.com/FengGeBlog/p/14326794.html
