---
title: 基于kubeadm实现高可用kubernetes集群
date: 2021-02-07 09:35:56
tags:
---



# 前言



k8s核心优势

- [x] 基于yaml文件实现容器自动创建、删除
- [x] 快速扩容、动态扩容
- [x] 简单的代码升级和回滚



非k8s组件

- etcd
  - coreos, k8s默认使用存储系统，所有k8s集群数据。使用raft协议完成分布式协作。
    - https://github.com/etcd-io/etcd

k8s组件

- master
  - controller创建 https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-proxy/
  - scheduler调度  https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-scheduler/
    - 默认策略就可以，基于资源调度。
  - api 网关 https://v1-18.docs.kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/
- node
  - kubelet 创建容器
    - kubelet提交资源. openstack也是组件placement收集资源反馈给控制端写给数据库。
    - 运行时：docker
    - 是k8s客户端
  - Kube-proxy 策略管理
    - 是k8s客户端，k8s service -> ipvs/iptables规则。用户访问请求转发到service。端口绑定到service的绑定，service通过label selector访问pod.
      - 前期3-4千ipables, 后期pod数量增长而增长，导致k8s响应时间慢。后期使用lvs.



- [x] 业务繁忙可以使用5个、7个master。etcd分布式奇数节点
- [x] 避免master创建pod、调度延迟，一般跑Pod。



pods: 最小容器单元，使用不同容器技术，kubelet命令调用不同的引擎启动容器即可。

- CSI container storage interface
- CVI volume
- CRI runtime

service ：访问pod的端点, k8s内部的负载均衡器

- nodeport
- loadbalancer
- externalname
- clusterip

controller: 创建pod, 解循环保证pod不宕机(健康状态)

- replicaset
- deployment 滚动升级
- 负责集群的资源管理、资源重建、replicas=、namespace隔离pod(harbor project隔离镜像、...）、resourcequota、serviceaccount

api:  提交资源保存在etcd集群中

- etcd集群
- 无状态：前端的haproxy可以轮循
- restful接口， http协议，传递json



亲和性

一个部门，多个项目，一般多个部门，部门之间的硬件资源是独立的。开发是独立的，运维来说只有一套kubernetes, 这一套项目运行数百上千的程序，让指定项目的pod跑在指定的服务器：

- nodeName, nodeSelector
- node亲和、pod亲和

例如：

a项目的硬件：group=linux39

b项目的硬件：group=linux40

启动linux39的应用时，就nodeSelector到自已的硬件的标签或node硬亲和

openstack也是一样，分成不同的项目



反亲和

有些节点不跑容器，这些节点加标签，与此标签相关节点反亲和

管理端使用污点，不跑pod



Namespace

A项目的pod在a namespace

B项目的Pod在b namespace

> 同ns直接使用名称调用
>
> 不同ns使用全名访问，甚至可以设定访问策略，不允许别人访问



Resourcequota

容器不添加资源限制会无限制的使用资源，java: 2G, java使用完了资源，就出现慢、访问超时

- CPU: 1000m对应1核。2000m(相当于毫秒)对应2核(秒)
- RAM：2048Mi





serviceaccount

- web界面多账号
  - A开发仅是A项目中有权限。A项目对B项目没有权限。
- 其他账号：内置的用户账号



<!--more-->
# 部署k8s

- [x] 规划服务器的节点

- [x] 部署

  

## 规划节点

### 保证高可用

机器不多，1个节点也可以: 2G

![image-20210207182904531](http://myapp.img.mykernel.cn/image-20210207182904531.png)



haproxy负载均衡准备2个，部署时，只使用一个haproxy的VIP，后期扩展haproxy+keepalived还是这个VIP

![image-20210207183121026](http://myapp.img.mykernel.cn/image-20210207183121026.png)

harbor保存镜像，前期一个节点，一个VIP入口，后期可以添加配置一个harbor, 直接配置harbor复制功能，还是这个VIP入口

106

准备node节点，越多越冗余，配置：96G/128G

107



### 规划网段

172.16.0.0/16

| ip           | 主机    | 非高可用 |
| ------------ | ------- | -------- |
| 172.16.3.101 | master1 | *        |
| 172.16.3.102 | master2 | *        |
| 172.16.3.103 | master3 | *        |
| 172.16.3.104 | ha1     | *        |
| 172.16.3.105 | ha2     |          |
| 172.16.3.106 | harbor1 | *        |
| 172.16.3.107 | harbor2 |          |
| 172.16.3.108 | node1   | *        |
| 172.16.3.109 | node2   |          |
| 172.16.3.110 | node3   |          |
| 172.16.3.111 | node4   |          |
| 172.16.3.112 | node5   |          |
| 172.16.3.113 | node6   |          |
| 172.16.3.114 | node7   |          |
| 172.16.3.115 | node8   |          |

```bash
 for i in {101..115}; do bash safe_clone_kvm.sh -i /VMs/template/ubuntu-1804-1C1G.qcow2 -d /VMs/kubernetes/172.16.3.$i  -t kvm  -n 172.16.3.$i -r 4096  -v 4 -b vmbr0; done
 
# 前期
 for i in 101 102 103 104 106 108; do bash safe_clone_kvm.sh -i /VMs/template/ubuntu-1804-1C1G.qcow2 -d /VMs/kubernetes/172.16.3.$i  -t kvm  -n 172.16.3.$i -r 4096  -v 4 -b vmbr0; done
 
# 把所有ip配置正常，reboot后ip正常
 
# 快照
for i in 101 102 103 104 106 108; do virsh snapshot-create-as --name NewOS --domain  172.16.3.$i; done
[root@centos7-iaas kvm]# for i in {4..11}; do virsh snapshot-create-as --name NewOS 192.168.0.$i ;done

# 启动
[root@centos7-iaas ~]#  for i in 101 102 103 104 106 108; do virsh start 172.16.3.$i; done
 [root@centos7-iaas kvm]# for i in {4..11}; do virsh start 192.168.0.$i ;done

```

参考：[kvm虚拟机操作](http://blog.mykernel.cn/2021/01/13/kvm%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%93%8D%E4%BD%9C/)

### 安装组件

![image-20210208080840509](http://myapp.img.mykernel.cn/image-20210208080840509.png)

master: controller, scheduler api server

node: kubelet, docker, kube-proxy

## 选择版本

目前最新版本v1.20.2版本，不使用。

[v1.19.7](https://github.com/kubernetes/kubernetes/releases/tag/v1.19.7)，相对比较新，示例

[v1.18.15](https://github.com/kubernetes/kubernetes/releases/tag/v1.18.15)，相对比较新，示例

[v1.17.17](https://github.com/kubernetes/kubernetes/releases/tag/v1.17.17)，相对比较新，示例

1.16.8，之后k8s api发生变化，之前的yaml文件的功能有差异

升级，带来新功能和bug的修复。



1年4个大版本。





系统选型

- centos 8
- ubuntu 强烈建议

## 安装方式

- [x] ansible: `kubeasz`：公司内使用，后期维护方便。

  - [x] 运维：**使用场景**：后期维护（加速添加宿主机：员工离职率高，你搭建好了，得可以快速搭建出来）、稳定性

    > 添加master、添加node、后期集群升级：当前k8s的bug或需要新功能

- [x] 手动二进制:  官方原始二进制，官方脚本。部署集群慢、后期维护不方便，你可以很快加节点，运行不行。

  - [x] 官方的源码目录有`cluster/scripts`脚本
  - [x] 全手工式安装 `cfssl工具`

- [x] kubeadm：后期维护方便，公司使用的人不多。 https://github.com/kubernetes/kubeadm

  - [x] K8S 1.9+，生产可用。
  - [x] 部署简单，官方出品，**使用场景**：`开发环境、测试环境、稳定性要求不高`

- [x] apt-get/yum



学习路径：手工 -> ansible playbook



## 升级方式

升级：bug或新功能。

> 1.9 -> 1.11
>
> 升级方式一：
>
> 1）测试环境安装1.9, 跑pod, 监控pod访问。
>
> 2）升级过程中不能影响业务中断
>
> 3）流动升级：
>
> ​	3.1)  haproxy下线一个master, 升级master版本，相同证书即可。
>
> ​    3.2）haproxy下线第二个master, 升级master版本，相同证书即可。
>
> ​    3.3） haproxy挂上2个master, 下线另一个master, 先不升级，查看上线的2个master是否有问题。没有问题，先把node节点滚动升级。
>
> ​    3.4）node: kubectl 驱逐pod, 当前节点不跑容器后，打上污点，升级成最新的，把node节点加入master，然后让k8s平衡pod
>
> ​     3.5）其他node相同方式，
>
> 升级方式二：
> 1）新部署k8s 1.11集群
>
> 2）把业务在新版k8s上跑起来，然后下线上一套集群。一般公司没有这么多物理机给你用。
>
> 3）测试业务没有问题，直接在入口上把k8s集群切过去。





为了演示kubernetes升级，当前安装1.17.3, 最新版本是1.17.4, 可以演示升级。

小版本是修复 bug, 大版本是添加新功能。

## 安装步骤 

- 基础环境、内核参数`ipv4.ip_forward`，容器需要源地址转换
- 部署harbor
- master安装kubeadm, kubelet, docker, kubectl
  - kubeadm拉集群
  - kubelet, docker 启动静态pod, 但是不调度
  - kubectl 需要用户在上面与api通信，管理集群。仅有运维有master的root权限
- 所有node: kubeadm, kubelet, docker.
  - kubeadm 加入集群
  - kubelet, docker 启动pod, 调度
  - kubectl不需要用户在上面与api通信，管理集群。
- master init
- node加入
- pod通信
- 部署dashboard
- 集群升级
- harbor 公司内部拉镜像，业务镜像。上课演示可以不做，镜像来自外网。



## 上线步骤

- 开发写好代码，给领导/测试发邮件，申请上线。
- 测试对代码进行上线测试，测试告诉运维什么时间对代码升级。**运维点jenkins对代码升级**
  - jenkins 触发job 
    1. 连接专用镜像服务器，拉代码、测试、编译、传到harbor。
    2. 连接k8s master服务器（haproxy），执行代码升级指令。
    3. master controller滚动升级，kubelet自动拉harbor镜像, 重建镜像。

![image-20210211100007403](http://myapp.img.mykernel.cn/image-20210211100007403.png)

![image-20210211100117265](http://myapp.img.mykernel.cn/image-20210211100117265.png)

## 安装harbor

harbor配置域名之后，在内部DNS添加A记录。

node节点拉镜像时，使用内部dns解析至harbor主机。

k8s上的pod，默认指向内部的coredns, 如果需要连接外部的mysql域名，先在coredns解析，如果 coredns解析不了，就转发至内部的dns.



```bash
root@ubuntu-template:~# git clone -b slcnux https://codeup.aliyun.com/5f9133e01858a17210467f88/linux_scripts.git
Cloning into 'linux_scripts'...
Username for 'https://codeup.aliyun.com': slcnux
Password for 'https://slcnux@codeup.aliyun.com': 
remote: Enumerating objects: 918, done.
remote: Counting objects: 100% (918/918), done.
remote: Total 918 (delta 0), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (918/918), 39.04 MiB | 8.85 MiB/s, done.
root@ubuntu-template:~# 


# 去掉一些配置
538 set_chinese_ubuntu
540 set_openssh_server_ubuntu $PORT $ALLOW_ROOT_LOGIN $ALLOW_PASS_LOGIN
541 set_root_passwd_ubuntu "$ROOT_PASS"
543 set_ethX_ubuntu
544 set_ip_ubuntu "$IPADDR" "$NETMASK" "$GATEWAY" "$DNS1"

root@ubuntu-template:~/linux_scripts# sed -i -e '538d' -e '540d' -e '541d' -e '543d' -e '544d'   template/linux_template_install.sh 
# 配置主机名，vim特性，资源优化
bash linux_template_install.sh --hostname=harbor01.mykernel.cn --author=songliangcheng --qq=2192383945 --desc="A test toy"
hostnamectl set-hostname `cat /etc/hostname`

```

安装docker, docker-compose

```bash
root@harbor01:~/linux_scripts/docker-ce# bash install-docker.sh 
root@ubuntu-template:~/linux_scripts/docker-ce# apt install docker-compose -y

```

安装harbor

```bash
root@harbor01:~# tar xvf harbor-offline-installer-v2.0.6.tgz -C /usr/local/
harbor/harbor.v2.0.6.tar.gz
harbor/prepare
harbor/LICENSE
harbor/install.sh
harbor/common.sh
harbor/harbor.yml.tmpl
root@harbor01:~# cd /usr/local/harbor/

root@harbor01:/usr/local/harbor# cp harbor.yml.tmpl harbor.yml
root@harbor01:/usr/local/harbor# install -dv certs

# cd certs

curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o cfssl
chmod +x cfssl
curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o cfssljson
chmod +x cfssljson
curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o cfssl-certinfo
chmod +x cfssl-certinfo

mkdir cert
cd cert
../cfssl print-defaults config > config.json
../cfssl print-defaults csr > csr.json

```

准备openssl ca证书的申请 

```diff
root@harbor01:/usr/local/harbor/certs/cert# cat ca-csr.json 
{
+    "CN": "harbor-ca",
    "hosts": [
    ],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "China",
            "L": "Sichuan",
            "ST": "ChengDu",
            "O": "system:Huayang"
        }
    ]
}

```

签发ca证书

```bash
../cfssl gencert -initca ca-csr.json | ../cfssljson -bare  harbor-ca
```

准备harbor的配置

```diff
root@harbor01:/usr/local/harbor/certs/cert# cat config.json 
{
    "signing": {
        "default": {
+            "expiry": "867000h"
        },
        "profiles": {
            "k8s-harbor": {
+                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
+                    "server auth",
-                    "client auth"
                ]
            }
        }
    }
}

```

> 证书是服务端证书，主要给harbor使用

准备harbor的证书签署请求

```diff
root@harbor01:/usr/local/harbor/certs/cert# cat csr.json 
{
+	"CN": "harbor",
	"hosts": [
+		"192.168.0.106",
+		"192.168.0.107",
+		"192.168.0.248",
+		"127.0.0.1",
+		"harbor.magedu.com",
+		"harbor.mykernel.cn"
	],
	"key": {
		"algo": "ecdsa",
		"size": 256
	},
	"names": [{
		"C": "China",
		"L": "Sichuan",
		"ST": "ChengDu",
		"O": "system:Huayang"
	}]
}
```

> hosts里面的地址就是连接harbor的可能地址
>
> 2个harbor的主机Ip，用户可能直连
>
> vip，nginx反代至2个harbor
>
> 127.0.0.1本地连接
>
> 下面2个域名是解析到VIP
>
> CN 是**浏览器**拿证书的CN当作网站的访问地址。



签发harbor证书

```bash
 ../cfssl gencert -ca=harbor-ca.pem -ca-key=harbor-ca-key.pem      --config=config.json -profile=k8s-harbor      csr.json | ../cfssljson -bare harbor
```

获取证书路径

```bash
root@harbor01:/usr/local/harbor/certs/cert# readlink -f harbor.pem 
/usr/local/harbor/certs/cert/harbor.pem
root@harbor01:/usr/local/harbor/certs/cert# readlink -f harbor-key.pem 
/usr/local/harbor/certs/cert/harbor-key.pem
```

配置harbor

`root@harbor01:/usr/local/harbor# grep '^[^#]' harbor.yml`

```diff
+hostname: harbor.mykernel.cn
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
+  certificate: /usr/local/harbor/certs/cert/harbor.pem
+  private_key: /usr/local/harbor/certs/cert/harbor-key.pem
+harbor_admin_password: 123456
```

> 如果密码写错了，修改密码，需要 `rm -fr /data/`

```bash
root@harbor01:/usr/local/harbor# ./install.sh 
```

![image-20210212160428520](http://myapp.img.mykernel.cn/image-20210212160428520.png)



```bash
root@harbor01:/usr/local/harbor/certs/cert# pwd
/usr/local/harbor/certs/cert
root@harbor01:/usr/local/harbor/certs/cert# sz harbor-ca.pem 
```

windows中将pem后缀修改为.crt后缀，并加载ca证书

![image-20210212174719867](http://myapp.img.mykernel.cn/image-20210212174719867.png)

![image-20210212174915397](http://myapp.img.mykernel.cn/image-20210212174915397.png)

### docker-ce证书

```bash
root@harbor01:/usr/local/harbor/certs/cert# scp harbor-ca.pem 192.168.0.103:/etc/docker/certs.d/harbor.mykernel.cn/ca.crt
```

103上

```bash
root@ubuntu-template:/etc/docker/certs.d/harbor.mykernel.cn# ls
ca.crt
```

```bash
root@ubuntu-template:/etc/docker/certs.d/harbor.mykernel.cn# docker login harbor.mykernel.cn
Username: admin
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

```bash
root@ubuntu-template:/etc/docker/certs.d# docker pull goharbor/harbor-portal:v2.0.6
v2.0.6: Pulling from goharbor/harbor-portal
2246cc853ebe: Pull complete 
6378cebd6d38: Pull complete 
a4f1aaf65efe: Pull complete 
11309c07b938: Pull complete 
d88bef848c29: Pull complete 
40875b466bd8: Pull complete 
e9678db29ce8: Pull complete 
14e9d997cd8c: Pull complete 
780c1db249e8: Pull complete 

root@ubuntu-template:/etc/docker/certs.d# docker tag  goharbor/harbor-portal:v2.0.6 harbor.mykernel.cn/pub/test:v4

添加主机解析记录
root@ubuntu-template:/etc/docker/certs.d# echo 192.168.0.106   harbor.mykernel.cn >> /etc/hosts
root@ubuntu-template:/etc/docker/certs.d/harbor.mykernel.cn# docker push harbor.mykernel.cn/pub/test:v4
The push refers to repository [harbor.mykernel.cn/pub/test]
0de754b914e6: Pushed 
b317e6a84b08: Pushed 
ff79adc23c6a: Pushed 
74a6d0b95d3b: Pushed 
0d73bea5837b: Pushed 
93908b71a428: Pushed 
cf10c2945c50: Pushed 
aa31f773c326: Pushed 
16c66899afe2: Layer already exists 
v4: digest: sha256:c6e795a9f8562497c03c32ca10124b5c4c9ea5342224aaaa7dfa998441f90358 size: 2200
```

手工测试将mykernel.cn修改magedu.com也正常。但是修改为magedu.cn就不正常。验证hosts属性的值就是访问证书对的服务。

### containerd证书

```bash
crictl pull harbor.youwoyouqu.io/baseimage/nginx-photon:v2.2.0


# ubuntu
root@harbor01:/opt/cert# scp harbor-ca.pem 192.168.0.78:/usr/share/ca-certificates/harbor-ca.crt
# centos
 scp harbor-ca.pem 192.168.0.171:/etc/pki/ca-trust/source/anchors/harbor-ca.crt

```

containerd节点

```bash
# ubuntu
echo 'harbor-ca.crt' >>  /etc/ca-certificates.conf
update-ca-certificates
systemctl restart containerd

# centos
update-ca-trust
systemctl restart containerd
```

```bash
root@master01:~# crictl pull harbor.youwoyouqu.io/baseimage/nginx-photon:v2.2.0
root@master01:~# crictl images
IMAGE                                                                         TAG                 IMAGE ID            SIZE
harbor.youwoyouqu.io/baseimage/nginx-photon                                   v2.2.0              39fcd9da1a471       16.9MB
```



## haproxy1

```bash
git clone -b slcnux https://codeup.aliyun.com/5f9133e01858a17210467f88/linux_scripts.git
slcnux
root@ubuntu-template:~/linux_scripts# sed -i -e '538d' -e '540d' -e '541d' -e '543d' -e '544d'   linux_template_install.sh 
bash linux_template_install.sh --hostname=haproxy01.mykernel.cn --author=songliangcheng --qq=2192383945 --desc="A test toy"
hostnamectl set-hostname `cat /etc/hostname`
```

安装haproxy, keepalived配置vip即可

```bash
apt update
root@haproxy01:~# apt install haproxy keepalived -y
```

```bash
root@haproxy01:~# cp /usr/share/doc/keepalived/samples/keepalived.conf.vrrp /etc/keepalived/keepalived.conf
```

```diff
root@haproxy01:~# cat /etc/keepalived/keepalived.conf 
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    garp_master_delay 10
    smtp_alert
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
+		192.168.0.248 dev eth0 label eth0:0
    }
}
```

```bash
root@haproxy01:~# systemctl restart keepalived
root@haproxy01:~# systemctl enable keepalived
Synchronizing state of keepalived.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable keepalived
```

```diff
root@haproxy01:~# cat /etc/haproxy/haproxy.cfg 
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# Default ciphers to use on SSL-enabled listening sockets.
	# For more information, see ciphers(1SSL). This list is from:
	#  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
	# An alternative list with additional directives can be obtained from
	#  https://mozilla.github.io/server-side-tls/ssl-config-generator/?server=haproxy
	ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS
	ssl-default-bind-options no-sslv3

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http



+listen apiserver-6443
+    bind 192.168.0.248:6443
+    mode tcp
+    server 192.168.0.101 192.168.0.101:6443  check inter 3s fall 2 rise 5
+    server 192.168.0.102 192.168.0.102:6443  check inter 3s fall 2 rise 5
+    server 192.168.0.103 192.168.0.103:6443  check inter 3s fall 2 rise 5


+listen harbor-443
+    bind 172.16.3.248:443
+    mode tcp
+    server 172.16.3.106 172.16.3.106:443  check inter 3s fall 2 rise 5
+    server 172.16.3.107 172.16.3.107:443  check inter 3s fall 2 rise 5
```

> 由于harbor的证书dns里面有172.16.3.248, 所以可以直接这个通信
>
> ```diff
> root@harbor01:/data# cat /opt/harbor/certs/cert/csr.json 
> {
> 	"CN": "harbor",
> 	"hosts": [
> 		"172.16.0.106",
> 		"172.16.0.107",
> +		"172.16.3.248",
> 		"127.0.0.1",
> 		"harbor.magedu.com",
> 		"harbor.mykernel.cn"
> 	],
> 	"key": {
> 		"algo": "ecdsa",
> 		"size": 256
> 	},
> 	"names": [{
> 		"C": "China",
> 		"L": "Sichuan",
> 		"ST": "ChengDu",
> 		"O": "system:Huayang"
> 	}]
> }
> ```
>
> 

```bash
root@haproxy01:~# systemctl restart haproxy
root@haproxy01:~# systemctl enable haproxy
Synchronizing state of haproxy.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable haproxy
root@haproxy01:~# systemctl status haproxy
```

测试docker通过vip通信

```diff
+root@harbor01:/data# docker push 172.16.3.248:7443/library/nginx-photon:v2.0.7
The push refers to repository [172.16.3.248:7443/library/nginx-photon]
+Get https://172.16.3.248:7443/v2/: x509: certificate signed by unknown authority



+root@harbor01:/data# install -dv /etc/docker/certs.d/172.16.3.248:7443
+root@harbor01:/data# cp /opt/harbor/certs/cert/harbor-ca.pem /etc/docker/certs.d/172.16.3.248:7443/ca.crt
+root@harbor01:/data# docker push 172.16.3.248:7443/library/nginx-photon:v2.0.7
The push refers to repository [172.16.3.248:7443/library/nginx-photon]
32157259d7b9: Layer already exists 
16c66899afe2: Layer already exists 
+unauthorized: unauthorized to access repository: library/nginx-photon, action: push: unauthorized to access repository: library/nginx-photon, action: push
+root@harbor01:/data# docker login 172.16.3.248:7443/library/nginx-photon:v2.0.7
Username: admin
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
+root@harbor01:/data# docker push 172.16.3.248:7443/library/nginx-photon:v2.0.7
The push refers to repository [172.16.3.248:7443/library/nginx-photon]
32157259d7b9: Layer already exists 
16c66899afe2: Layer already exists 
v2.0.7: digest: sha256:8c9b94cbd8d92de2df0a7323dd7c3c589894e342499f04ca471ab8912c8dc763 size: 740
```

另一个harbor节点，可以通过陈怀森同学的博客完成配置, 由于haproxy会自动检测后端主机的port, 所以当从harbor上线，将自动负载均衡

http://www.chsblogs.com/docker/2725.html#toc-10



## master和node

### 安装容器运行时

安装CHANGLOG中指定的docker版本，但是github说dockershim已经废弃，使用的containerd替代，kubelet使用容器运行时和容器服务只需要一个套接字，并传递2个选项。所以使用kubernetes指定版本的containerd即，containerd依赖的网络插件版本也要对。

https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#dockershim-deprecation

https://kubernetes.io/blog/2020/12/02/dockershim-faq/#why-is-dockershim-being-deprecated

Docker 本身当前未实现 CRI，Dockershim 连接docker和kubelet

在 Kubernetes 1.22 之前不会将其删除，这意味着在 2021 年后期，没有 Dockershim 的最早版本将是 1.23。我们将与供应商和其他生态系统组织密切合作，以确保平稳过渡，并将随着形势的发展进行评估。

https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/



https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/

在 Kubernetes 1.5 版本中，我们自豪地介绍了[容器运行时](https://github.com/kubernetes/kubernetes/blob/242a97307b34076d5d8f5bbeb154fa4d97c9ef1d/docs/devel/container-runtime-interface.md)接口 （CRI） -- 一个插件接口，它使 kubelet 能够使用各种容器运行时，而无需重新编译。

Kubelet 的概述使用 gRPC 框架通过 Unix 套接字与容器运行时（或运行时的 CRI 填充程序）进行通信，其中 kubelet 充当客户端，CRI 填充程序充当服务器。

![img](http://myapp.img.mykernel.cn/Image 2016-12-19 at 17.13.16.png)

协议缓冲区[API 包括](https://github.com/kubernetes/kubernetes/blob/release-1.5/pkg/kubelet/api/v1alpha1/runtime/api.proto)两个 gRPC 服务，即映像服务和运行时服务。映像服务提供 RPC，用于从存储库中提取映像、检查和删除映像。运行时服务包含用于管理 Pod 和容器生命周期的 RPC，以及用于与容器交互的调用（执行/附加/端口转发）。管理映像和容器（例如 Docker 和 rkt）的单片容器运行时可以使用单个套接字同时提供这两个服务。

The sockets can be set in Kubelet by --container-runtime-endpoint and --image-service-endpoint flags.

安装containerd 版本要对

![image-20210211093952184](http://myapp.img.mykernel.cn/image-20210211093952184.png)

[Release containerd 1.4.1 · containerd/containerd · GitHub](https://github.com/containerd/containerd/releases/tag/v1.4.1)

![image-20210211094120518](http://myapp.img.mykernel.cn/image-20210211094120518.png)











安装kubeadm ,使用[aliyun源](https://developer.aliyun.com/mirror/kubernetes)

![image-20210211101517427](http://myapp.img.mykernel.cn/image-20210211101517427.png)

[Installing kubeadm | Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)



https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd

![image-20210211101929627](http://myapp.img.mykernel.cn/image-20210211101929627.png)



所以直接安装docker 19.03即可



安装docker

1. [Container runtimes | Kubernetes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker)
2. [aliyun镜像docker](https://developer.aliyun.com/mirror/docker-ce)

配置镜像加速不需要配置，因为node节点是从harbor中获取镜像，不需要从外网获取镜像。配置镜像加速，是在镜像构建服务器

安装kubeadm

1. 官方

   https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl

   您需要确保它们与您希望 kubeadm 为你安装的 Kubernetes 控制平面的版本相匹配

   *支持*kubelet 和控件平面之间的一个次要版本偏斜，但 kubelet 版本可能永远不会超过 API 服务器版本, 

   For example, the kubelet running 1.7.0 should be fully compatible with a 1.8.0 API server, 

2. [aliyun镜像](https://developer.aliyun.com/mirror/kubernetes)

   安装什么版本的k8s, 每个节点安装的kubeadm, kubelet 需要使用对应版本，master节点的kubectl也一样。

   安装v1.20时，先安装小版本，后面方便升级

   docker版本，根据官方CHANGLOG来决定



```bash
git clone -b slcnux https://codeup.aliyun.com/5f9133e01858a17210467f88/linux_scripts.git
slcnux
bash linux-temp.sh --hostname=master01.mykernel.cn --author=songliangcheng --qq=2192383945 --desc="A test toy" --resourceslimit=1 --kernelparams=1 --basepkgs=1 --chinese=1 --eth0=0 --umirror=1 # 172.16.3.101
bash linux-temp.sh --hostname=master02.mykernel.cn --author=songliangcheng --qq=2192383945 --desc="A test toy" --resourceslimit=1 --kernelparams=1 --basepkgs=1 --chinese=1 --eth0=0 --umirror=1 # 172.16.3.102
bash linux-temp.sh --hostname=master03.mykernel.cn --author=songliangcheng --qq=2192383945 --desc="A test toy" --resourceslimit=1 --kernelparams=1 --basepkgs=1 --chinese=1 --eth0=0 --umirror=1 # 172.16.3.103

```


安装containerd v1.4.1, 由k8s `v1.20.x` CHANGLOG

https://github.com/containerd/containerd/releases/tag/v1.4.1

https://github.com/containerd/cri/blob/release/1.4/docs/installation.md

https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd



/etc/systemd/system/containerd.service

```diff
# 
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
+ExecStartPre=-/sbin/modprobe br_netfilter
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=1048576
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target

```

```diff
root@master03:~# cat /etc/crictl.yaml 
+runtime-endpoint: unix:///run/containerd/containerd.sock

```

mkdir /etc/containerd

root@master03:~# cat /etc/containerd/config.toml 

```diff
version = 2
+root = "/var/lib/containerd"
state = "/run/containerd"
plugin_dir = ""
disabled_plugins = []
required_plugins = []
oom_score = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  tcp_address = ""
  tcp_tls_cert = ""
  tcp_tls_key = ""
  uid = 0
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216

[ttrpc]
  address = ""
  uid = 0
  gid = 0

[debug]
  address = ""
  uid = 0
  gid = 0
  level = ""

[metrics]
  address = ""
  grpc_histogram = false

[cgroup]
  path = ""

[timeouts]
  "io.containerd.timeout.shim.cleanup" = "5s"
  "io.containerd.timeout.shim.load" = "5s"
  "io.containerd.timeout.shim.shutdown" = "3s"
  "io.containerd.timeout.task.state" = "2s"

[plugins]
  [plugins."io.containerd.gc.v1.scheduler"]
    pause_threshold = 0.02
    deletion_threshold = 0
    mutation_threshold = 100
    schedule_delay = "0s"
    startup_delay = "100ms"
  [plugins."io.containerd.grpc.v1.cri"]
    disable_tcp_service = true
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    stream_idle_timeout = "4h0m0s"
    enable_selinux = false
    selinux_category_range = 1024
+    sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2"
    stats_collect_period = 10
    systemd_cgroup = false
    enable_tls_streaming = false
    max_container_log_line_size = 16384
    disable_cgroup = false
    disable_apparmor = false
    restrict_oom_score_adj = false
    max_concurrent_downloads = 3
    disable_proc_mount = false
    unset_seccomp_profile = ""
    tolerate_missing_hugetlb_controller = true
    disable_hugetlb_controller = true
    ignore_image_defined_volumes = false
    [plugins."io.containerd.grpc.v1.cri".containerd]
      snapshotter = "overlayfs"
      default_runtime_name = "runc"
      no_pivot = false
      disable_snapshot_annotations = false
      discard_unpacked_layers = false
      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        runtime_type = ""
        runtime_engine = ""
        runtime_root = ""
        privileged_without_host_devices = false
        base_runtime_spec = ""
      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        runtime_type = ""
        runtime_engine = ""
        runtime_root = ""
        privileged_without_host_devices = false
        base_runtime_spec = ""
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          runtime_engine = ""
          runtime_root = ""
          privileged_without_host_devices = false
          base_runtime_spec = ""
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      max_conf_num = 1
      conf_template = ""
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
+          endpoint =  ["https://registry-1.docker.io", "https://docker.mirrors.ustc.edu.cn",  "http://hub-mirror.c.163.com"]
    [plugins."io.containerd.grpc.v1.cri".image_decryption]
      key_model = ""
    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""
  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"
  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"
  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "shared"
  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false
  [plugins."io.containerd.runtime.v1.linux"]
    shim = "containerd-shim"
    runtime = "runc"
    runtime_root = ""
    no_shim = false
    shim_debug = false
  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]
  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]
  [plugins."io.containerd.snapshotter.v1.devmapper"]
    root_path = ""
    pool_name = ""
    base_image_size = ""
    async_remove = false

```



```bash
export https_proxy=http://192.168.0.33:808
wget https://github.com/containerd/containerd/releases/download/v1.4.1/containerd-1.4.1-linux-amd64.tar.gz
tar --no-overwrite-dir -C /usr/local/ -xvzf containerd-1.4.1-linux-amd64.tar.gz
wget https://github.com/opencontainers/runc/releases/download/v1.0.0-rc93/runc.amd64
install runc.amd64  /usr/local/bin/runc
```

```bash
systemctl enable containerd --now
```



> 其他节点安装containerd, 同上
>
> 管理镜像和容器参考： https://kubernetes.io/zh/docs/tasks/debug-application-cluster/crictl/#docker-cli-%E5%92%8C-crictl-%E7%9A%84%E6%98%A0%E5%B0%84
>
> https://podman.io/getting-started/installation
>
> 安装podman
>
> ```bash
> export https_proxy=http://192.168.0.33:808
> . /etc/os-release
> echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
> curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/Release.key | apt-key add -
> apt-get update
> apt-get -y install podman
> # (Ubuntu 18.04) Restart dbus for rootless podman
> systemctl --user restart dbus
> ```
>
> > `sudo`不会走代理





   ### master安装


  ```bash
apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt update

apt-cache madison kubeadm
kubeadm |  1.20.2-00 | https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
kubeadm |  1.20.1-00 | https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages


 apt install kubeadm=1.20.1-00 kubelet=1.20.1-00 kubectl=1.20.1-00 -y
  ```





#### master节点集群初始化

仅需要一个master节点初始化，就将数据写入etcd, 其他节点在初始化，不一定初始化成功。

后期仅需要将master加入master



#### kubeadm命令



   ```
kubeadm --help

初始化集群的帮助

控制平面，就是VIP或VIP的域名。


alpha --help	 测试命令

completion  --help 命令补全
         # kubeadm completion bash >> /etc/profile.d/kubeadm.sh
         # source /etc/profile.d/kubeadm.sh
         
         
config --help 配置, 会管理在configmap中
	# kubeadm config print init-default 
		api版本
		bootstraptoken
		service网段
		dnsdomain
		k8s 版本
	# 	kubeadm config print init-defaults > kubeadm-v1.20.1.yaml
	# kubeadm init --config=kubeadm-v1.20.1.yaml
	
init --help 启动
join node/master节点加入至master
	apiserver地址和flags
reset 还原节点至kubeadm init/join产生的环境变化。不使用kubernetes时，就需要清空

token 管理bootstraptoken

upgrade 升级k8s版本
   ```

#### 初始化集群的init命令

[kubeadm init | Kubernetes](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/)

```bash
--apiserver-advertise-address string 对于每个master节点，这个地址是master物理地址
--apiserver-bind-port int32     默认值：6443
--control-plane-endpoint string      整个集群连接api, 将会使用这个地址。应该为VIP地址或 可以解析到VIP的域名； 这个VIP默认就具备可以负载均衡调度至3个master节点的6443端口
--cri-socket string                  要连接的 CRI 套接字的路径。如果为空，则 kubeadm 将尝试自动检测此值；仅当安装了多个 CRI 或具有非标准 CRI 插槽时，才使用此选项。存在docker/podman/containerd时，必须指定某个cri
--ignore-preflight-errors stringSlice  例如：'IsPrivilegedUser,Swap'。取值为 'all' 时将忽略检查中的所有错误。系统是否符合安装k8s, 某些检查忽略。例如，有swap时k8s启动不了，但是这个启动并不影响k8s使用，可以忽略。
--image-repository string     默认值："k8s.gcr.io" ，获取controller-manager, scheduler, kublet, api-server, kube-proxy，pause镜像的默认地址。可以使用registry.cn-hangzhou.aliyuncs.com/google_containers 国内镜像
--kubernetes-version string     默认值："stable-1"，默认是次新版本，我们使用kubeadm v1.17.2,所以也使用这个版本的k8s

k8s 保证主机名惟一，区分角色

--pod-network-cidr string 指定pod网段，并且controller会自动给每个节点分配一个子网。但是后期使用flannel/calico需要和这个网段一致。

--service-cidr string     默认值："10.96.0.0/12"，这个是pod的服务地址。一般业务的服务不多，但是服务中的pod很多。ip数量：21位：2^(32-21) -2 = 2046 个ip , 20位掩码，有4096个地址。

--service-dns-domain string     默认值："cluster.local"，不使用默认值，一般使用公司的类似的域名后缀。linux39.local/magedu.local. 这个域名是由coredns来解析service的A记录。

--token string 这个令牌用于建立控制平面节点与工作节点间的双向通信。格式为 [a-z0-9]{6}\.[a-z0-9]{16} - 示例：abcdef.0123456789abcdef。token格式，前6位.后16位

--token-ttl duration     默认值：24h0m0s k8s的token过期时间。太长时间，别人可以拿着你的token来接入k8s

--upload-certs 管理k8s证书，这个不用管，因为证书他会自动帮你创建。



# 全局的日志一般不用 记录
一般在/var/log/

```

kubeadm init引导集群，也可以使用config文件

获取引导时使用的镜像

```bash
kubeadm config  --kubernetes-version=v1.20.1 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers images pull
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.1
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.1
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.1
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.1
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0
```

配置swap

```diff
root@master01:~# cat > /etc/default/kubelet <<EOF
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
EOF

scp /etc/default/kubelet 172.16.3.102:/etc/default/kubelet
scp /etc/default/kubelet 172.16.3.103:/etc/default/kubelet
```

配置vip, 指向haproxy, haproxy已经配置负载到3个api server

```diff
root@master01:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu-template.magedu.local	ubuntu-template

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

+172.16.3.248 kubernetes-api.mykernel.cn


scp /etc/hosts 172.16.3.102:/etc/hosts
scp /etc/hosts 172.16.3.103:/etc/hosts
```

桥模块

```bash
modprobe br_netfilter
sysctl -p
```



##### 初始化方式一

```bash
#https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md
export KUBE_PROXY_MODE=ipvs

kubeadm init   --apiserver-advertise-address=0.0.0.0 --apiserver-bind-port=6443 --control-plane-endpoint=kubernetes-api.mykernel.cn:6443 --ignore-preflight-errors=swap    --kubernetes-version=v1.20.1 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --service-dns-domain=linux48.local --upload-certs 

```

> pod地址和service地址，使用172.16.0.0/12, 172.26.0.0/12 就出问题了。原因如下：
>
> ![image-20210215110439605](http://myapp.img.mykernel.cn/image-20210215110439605.png)
>
> 这是同一个网段

##### 方式二

```diff
kubeadm config print init-defaults > kubeadm-v1.20.1.yaml
root@master01:~# cat kubeadm-v1.20.1.yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
 # token有效期
+  ttl: 48h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
    # 本机
+  advertiseAddress: 172.16.3.101
  bindPort: 6443
nodeRegistration:
+  criSocket:/run/containerd/containerd.sock
  name: master01.mykernel.cn
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
+controlPlaneEndpoint: kubernetes-api.mykernel.cn:6443
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
+imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
+kubernetesVersion: v1.20.1
networking:
+  dnsDomain: linux48.local
+  serviceSubnet: 10.96.0.0/12
+  podSubnet: 10.244.0.0/16
kubeProxy:
  config:
    mode: ipvs
```

```bash
#https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md
export KUBE_PROXY_MODE=ipvs

kubeadm init --config=kubeadm-v1.20.1.yaml --ignore-preflight-errors=swap
```



```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

# 网络插件
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

# 加master
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join kubernetes-api.mykernel.cn:6443 --token ii95np.5abqfzygo2pgwglb \
    --discovery-token-ca-cert-hash sha256:9365f413933730cb79efcc11a917e954a27600cd0a391db47cc5d825e96a067e \
    --control-plane --certificate-key a12c8228e1435f023358f86484b1952107fe29f94deed820474107024a91233d

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

# 加work
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join kubernetes-api.mykernel.cn:6443 --token ii95np.5abqfzygo2pgwglb \
    --discovery-token-ca-cert-hash sha256:9365f413933730cb79efcc11a917e954a27600cd0a391db47cc5d825e96a067e 

```

> 记录以上返回信息

配置配置

```bash
root@master01:~# kubectl config view --kubeconfig=/etc/kubernetes/admin.conf
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://kubernetes-api.mykernel.cn:6443 # 连接haproxy -> 多个Master
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin # 操作kubernetes的管理员账户
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED

```

> 由于是管理员权限，所以文件不要在Node节点上，不然所有Node管理权限

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config # 证书信息	
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
root@master02:~# kubectl get pods -A
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
kube-system   coredns-54d67798b7-j6s6m                       0/1     Pending   0          3m21s
kube-system   coredns-54d67798b7-z8fg9                       0/1     Pending   0          3m21s
kube-system   etcd-master02.mykernel.cn                      1/1     Running   0          3m29s
kube-system   kube-apiserver-master02.mykernel.cn            1/1     Running   0          3m29s
kube-system   kube-controller-manager-master02.mykernel.cn   1/1     Running   0          3m29s
kube-system   kube-proxy-k2gbb                               1/1     Running   0          3m22s
kube-system   kube-scheduler-master02.mykernel.cn            1/1     Running   0          3m29s
```



#### 添加master

初始化，安装kubeadm, kubelet配置swap, hosts域名解析

```bash
# master1
scp -rp /etc/containerd/  172.16.3.103:/etc/
scp -rp /etc/crictl.yaml   172.16.3.103:/etc/
scp -rp /etc/systemd/system/containerd.service   172.16.3.103:/etc/systemd/system/containerd.service
scp containerd-1.4.1-linux-amd64.tar.gz 172.16.3.103:
scp /etc/default/kubelet 172.16.3.103:/etc/default/kubelet
scp /etc/hosts 172.16.3.103:/etc/hosts
scp -rp /usr/local/bin/runc 172.16.3.103:/usr/local/bin/runc


# 其他master节点
tar --no-overwrite-dir -C /usr/local/ -xvzf containerd-1.4.1-linux-amd64.tar.gz
systemctl enable containerd --now

modprobe net_filter
sysctl -p
```

```bash
apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt update
apt install kubeadm=1.20.1-00 kubelet=1.20.1-00 kubectl=1.20.1-00 -y
```

拉镜像

```bash
kubeadm config  --kubernetes-version=v1.20.1 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers images pull
```

获取key

```bash
kubeadm init phase upload-certs --upload-certs
```

加入集群

```bash
kubeadm join kubernetes-api.mykernel.cn:6443 --token abcdef.0123456789abcdef \
     --discovery-token-ca-cert-hash sha256:8c413e8b0908c5c1badf5fbfce2c0af8af5d311a3a019b65028dc807abd0a777 \
     --control-plane  --certificate-key   c2d8ea68065a8781cbf500c6e985c698370b5b25275ecf9de1027a1b295956f8 --ignore-preflight-errors=swap

To start administering your cluster from this node, you need to run the following as a regular user:

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.

```

生成配置

```bash
root@master01:~# kubectl get pods -A
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
kube-system   coredns-54d67798b7-j6s6m                       0/1     Pending   0          13m
kube-system   coredns-54d67798b7-z8fg9                       0/1     Pending   0          13m
kube-system   etcd-master01.mykernel.cn                      1/1     Running   0          26s
kube-system   etcd-master02.mykernel.cn                      1/1     Running   2          13m
kube-system   kube-apiserver-master01.mykernel.cn            1/1     Running   6          7m58s
kube-system   kube-apiserver-master02.mykernel.cn            1/1     Running   6          13m
kube-system   kube-controller-manager-master01.mykernel.cn   1/1     Running   0          7m58s
kube-system   kube-controller-manager-master02.mykernel.cn   1/1     Running   1          13m
kube-system   kube-proxy-k2gbb                               1/1     Running   0          13m
kube-system   kube-proxy-zktpv                               1/1     Running   0          7m59s
kube-system   kube-scheduler-master01.mykernel.cn            1/1     Running   0          7m58s
kube-system   kube-scheduler-master02.mykernel.cn            1/1     Running   1          13m

```

> 按以上方法添加master3

```bash
root@master01:~# kubectl get pods -A -w
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
kube-system   coredns-54d67798b7-gnlk8                      0/1     Pending   0          15m
kube-system   coredns-54d67798b7-swfgv                      0/1     Pending   0          15m
kube-system   etcd-master01.magedu.com                      1/1     Running   0          15m
kube-system   etcd-master02.magedu.com                      1/1     Running   0          11m
kube-system   etcd-master03.magedu.com                      1/1     Running   0          118s
kube-system   kube-apiserver-master01.magedu.com            1/1     Running   0          15m
kube-system   kube-apiserver-master02.magedu.com            1/1     Running   0          11m
kube-system   kube-apiserver-master03.magedu.com            1/1     Running   0          2m
kube-system   kube-controller-manager-master01.magedu.com   1/1     Running   1          15m
kube-system   kube-controller-manager-master02.magedu.com   1/1     Running   0          11m
kube-system   kube-controller-manager-master03.magedu.com   1/1     Running   0          2m
kube-system   kube-proxy-9nf9g                              1/1     Running   0          15m
kube-system   kube-proxy-j246b                              1/1     Running   0          11m
kube-system   kube-proxy-ln7q2                              1/1     Running   0          2m1s
kube-system   kube-scheduler-master01.magedu.com            1/1     Running   1          15m
kube-system   kube-scheduler-master02.magedu.com            1/1     Running   0          11m
kube-system   kube-scheduler-master03.magedu.com            1/1     Running   0          2m
```

### 配置网络插件

https://github.com/coreos/flannel

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
 grep image kube-flannel.yml 
        image: quay.io/coreos/flannel:v0.13.1-rc2
        image: quay.io/coreos/flannel:v0.13.1-rc2
        
        
"DirectRouting": true                         
```

```bash
root@master01:~# docker pull quay.io/coreos/flannel:v0.13.1-rc2
root@master01:~# docker tag quay.io/coreos/flannel:v0.13.1-rc2 harbor.mykernel.cn/pub/flannel:v0.13.1-rc2 # 因为证书中包含此dns
root@master01:~# docker push harbor.mykernel.cn/pub/flannel:v0.13.1-rc2
The push refers to repository [harbor.mykernel.cn/pub/flannel]
Get https://harbor.mykernel.cn/v2/: dial tcp: lookup harbor.mykernel.cn: no such host
```

把harbor证书分发给master

```diff
master1添加hosts解析记录
cat /etc/hosts
+172.16.3.248  kubernetes-api.mykernel.cn harbor.mykernel.cn                  
```

```bash
root@master01:~# scp /etc/hosts 172.16.3.102:/etc/hosts
root@master01:~# scp /etc/hosts 172.16.3.103:/etc/hosts
```

准备证书（docker)

```bash
root@master01:~# mkdir -pv /etc/docker/certs.d/harbor.mykernel.cn
# harbor分发
root@harbor01:~# scp -rp /opt/harbor/certs/cert/harbor-ca.pem 172.16.3.101:/etc/docker/certs.d/harbor.mykernel.cn/ca.crt
# master01分给其他master
root@master01:~# scp -rp /etc/docker/certs.d/ 172.16.3.102:/etc/docker
root@master01:~# scp -rp /etc/docker/certs.d/ 172.16.3.103:/etc/docker
```

准备证书(containerd)

```bash
scp -rp /opt/harbor/certs/cert/harbor-ca.pem 172.16.3.101:/usr/share/ca-certificates/harbor-ca.crt
echo 'harbor-ca.crt' >> /etc/ca-certificates.conf
update-ca-certificates
systemctl restart containerd
```



再次推送镜像

```bash
root@master01:~# docker login harbor.mykernel.cn/pub/flannel:v0.13.1-rc2
Username: admin
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded


root@master01:~# docker push harbor.mykernel.cn/pub/flannel:v0.13.1-rc2
```

测试master2拉镜像

```bash
root@master02:~# docker pull harbor.mykernel.cn/pub/flannel:v0.13.1-rc2
v0.13.1-rc2: Pulling from pub/flannel
df20fa9351a1: Pull complete 
0fbfec51320e: Pull complete 
734a6c0a0c59: Pull complete 
95bcce43aaee: Pull complete 
f5cb02651392: Pull complete 
071b96dd834b: Pull complete 
164ad1e8ed3f: Pull complete 
Digest: sha256:dd85a6f06d285ef8c82ec86af5045bf7bad701f77528c15d8325152293059c22
Status: Downloaded newer image for harbor.mykernel.cn/pub/flannel:v0.13.1-rc2
harbor.mykernel.cn/pub/flannel:v0.13.1-rc2

```

提交 kube-flannel

```bash
root@master01:~# sed 's@image: quay.io/coreos/flannel:v0.13.1-rc2@image: harbor.mykernel.cn/pub/flannel:v0.13.1-rc2@g'  kube-flannel.yml | kubectl apply -f -
```





```bash
root@master01:~# kubectl get pods -A
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
kube-system   coredns-54d67798b7-gnlk8                      1/1     Running   0          20m
kube-system   coredns-54d67798b7-swfgv                      1/1     Running   0          20m
kube-system   etcd-master01.magedu.com                      1/1     Running   0          20m
kube-system   etcd-master02.magedu.com                      1/1     Running   0          16m
kube-system   etcd-master03.magedu.com                      1/1     Running   0          7m6s
kube-system   kube-apiserver-master01.magedu.com            1/1     Running   0          20m
kube-system   kube-apiserver-master02.magedu.com            1/1     Running   0          16m
kube-system   kube-apiserver-master03.magedu.com            1/1     Running   0          7m8s
kube-system   kube-controller-manager-master01.magedu.com   1/1     Running   1          20m
kube-system   kube-controller-manager-master02.magedu.com   1/1     Running   0          16m
kube-system   kube-controller-manager-master03.magedu.com   1/1     Running   0          7m8s
kube-system   kube-flannel-ds-kd25d                         1/1     Running   0          88s
kube-system   kube-flannel-ds-qrxb5                         1/1     Running   0          88s
kube-system   kube-flannel-ds-vgtrg                         1/1     Running   0          88s
kube-system   kube-proxy-9nf9g                              1/1     Running   0          20m
kube-system   kube-proxy-j246b                              1/1     Running   0          16m
kube-system   kube-proxy-ln7q2                              1/1     Running   0          7m9s
kube-system   kube-scheduler-master01.magedu.com            1/1     Running   1          20m
kube-system   kube-scheduler-master02.magedu.com            1/1     Running   0          16m
kube-system   kube-scheduler-master03.magedu.com            1/1     Running   0          7m8s
```

#### 激活ipvs

```bash
root@harbor01:~# kubectl edit cm -n kube-system kube-proxy
 44     mode: "ipvs"
```

```bash
root@harbor01:~# kubectl delete pods -n kube-system -l k8s-app=kube-proxy
```

检验

```bash
root@master01:~# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 172.16.100.71:6443           Masq    1      0          0         
  -> 172.16.100.72:6443           Masq    1      0          0         
  -> 172.16.100.73:6443           Masq    1      0          0         
TCP  10.96.0.10:53 rr
  -> 10.244.1.2:53                Masq    1      0          0         
  -> 10.244.2.2:53                Masq    1      0          0         
TCP  10.96.0.10:9153 rr
  -> 10.244.1.2:9153              Masq    1      0          0         
  -> 10.244.2.2:9153              Masq    1      0          0         
UDP  10.96.0.10:53 rr
  -> 10.244.1.2:53                Masq    1      0          0         
  -> 10.244.2.2:53                Masq    1      0          0         
```



### 添加node节点

初始化

```bash
bash linux-temp.sh --hostname=node01.mykernel.cn --author=songliangcheng --qq=2192383945 --desc="A test toy" --resourceslimit=1 --kernelparams=1 --basepkgs=1 --chinese=1 --eth0=0 --umirror=1


modprobe br_netfilter
sysctl -p
```

安装docker

```bash
# master1
scp -rp /etc/containerd/  172.16.3.108:/etc/
scp -rp /etc/crictl.yaml   172.16.3.108:/etc/
scp -rp /etc/systemd/system/containerd.service   172.16.3.108:/etc/systemd/system/containerd.service
scp containerd-1.4.1-linux-amd64.tar.gz 172.16.3.108:
scp /etc/default/kubelet 172.16.3.108:/etc/default/kubelet
scp /etc/hosts 172.16.3.108:/etc/hosts
scp -rp /usr/local/bin/runc 172.16.3.108:/usr/local/bin/runc


# 其他node节点
tar --no-overwrite-dir -C /usr/local/ -xvzf containerd-1.4.1-linux-amd64.tar.gz
systemctl enable containerd --now
```

安装kubeadm

```bash
apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt update
 apt install kubeadm=1.20.1-00 kubelet=1.20.1-00  -y
```

配置ipvs

```bash
apt install -y ipset ipvsadm

cat > /etc/modules-load.d/10-k8s-modules.conf <<EOF
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
EOF
for i in br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack_ipv4; do modprobe $i; done
```



配置https

参考 安装harbor

```bash
kubeadm join kubernetes-api.mykernel.cn:6443 --token vcqd39.d495ou4a6rsqrl2x     --discovery-token-ca-cert-hash sha256:8b403c31fd8cd5980594d710a5cdd669723b8d73d8c2b2a326f3728b15782e58  --ignore-preflight-errors=Swap
```



### 部署dashboard

https://github.com/kubernetes/dashboard

release 显示dashboard兼容性

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.1.0/aio/deploy/recommended.yaml
```

配置nodePort接入流量

```bash
 kubectl edit svc -n kubernetes-dashboard kubernetes-dashboard 
service/kubernetes-dashboard edited


root@master01:~# kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.104.45.47   <none>        8000/TCP        2m3s
kubernetes-dashboard        NodePort    10.103.7.5     <none>        443:31012/TCP   2m4s

```

访问任意一个节点的31012

https://192.168.0.101:31012

配置https证书参考 [dashboard https](http://blog.mykernel.cn/2021/01/29/%E8%AE%A4%E8%AF%81%E3%80%81%E6%8E%88%E6%9D%83%E5%8F%8A%E5%87%86%E5%85%A5%E6%8E%A7%E5%88%B6/#dashboard-https)

### 部署监控prometheus operator

### 部署自定义指标

## 升级kubernetes 1.20.1至1.20.4

版本有bug或功能不满足业务（1.9 -> 1.11，使用ipvs）



测试、开发环境不管业务，不担心影响用户访问

生产环境：

- 升级时间：低峰期，凌晨2:00
- 升级方式：
  - ansible, 升级简单
  - kubeadm, 不复杂 
  - 手工：需要考虑怎么签发证书 `/etc/kubernetes/pki`



### kubeadm 升级kubernetes

#### 安装master的kubeadm

先安装kubeadm为目的版本的k8s，升级到k8s为1.20.4， 所以先升级kubeadm为1.20.4。

然后再用1.20.4的kubeadm升级k8s，先升级master的scheduler, controller, apiserver版本

```bash
# 所有master
apt-cache madison kubeadm # 只是跨当前版本的小版本
   kubeadm |  1.20.4-00 | https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
   kubeadm |  1.20.2-00 | https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
   kubeadm |  1.20.1-00 | https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
   kubeadm |  1.20.0-00 | https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages

# master1
apt install kubeadm=1.20.4-00 -y
root@master01:~# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.4"

# master2
apt install kubeadm=1.20.4-00  -y
root@master02:~# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.4"
# master3
apt install kubeadm=1.20.4-00  -y
root@master03:~# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.4"
```

升级集群至新版本

```bash
kubeadm upgrade --help
  apply       Upgrade your Kubernetes cluster to the specified version
  diff        # 测试 Show what differences would be applied to existing static pod manifests. See also: kubeadm upgrade apply --dry-run
  node        # 升级node Upgrade commands for a node in the cluster
  plan        # 查看版本升级计划 Check which versions are available to upgrade to and validate whether your current cluster is upgradeable. To skip the internet check, pass in the optional [version] parameter



```

#### 升级master2, master3

```bash
# 计划
#[upgrade/versions] Cluster version: v1.20.1 # 先检查当前版本
#[upgrade/versions] kubeadm version: v1.20.4 # kubeadm版本
COMPONENT   CURRENT       AVAILABLE
kubelet     4 x v1.20.1   v1.20.4

# 每个节点都检查

kubeadm upgrade plan # master2
kubeadm upgrade plan # master3


# 升级前，查看所有pod的镜像
root@master02:~# kubectl get pods -n kube-system -o=jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}' |sort
coredns-54d67798b7-cgp2j:	registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0, 
coredns-54d67798b7-l9mrm:	registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0, 
etcd-master01.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0, 
etcd-master02.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0, 
etcd-master03.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0, 
kube-apiserver-master01.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.1, 
kube-apiserver-master02.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.1, 
kube-apiserver-master03.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.1, 
kube-controller-manager-master01.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.1, 
kube-controller-manager-master02.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.1, 
kube-controller-manager-master03.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.1, 
kube-flannel-ds-7wk87:	harbor.mykernel.cn/pub/flannel:v0.13.1-rc2, 
kube-flannel-ds-dv57p:	harbor.mykernel.cn/pub/flannel:v0.13.1-rc2, 
kube-flannel-ds-krc7l:	harbor.mykernel.cn/pub/flannel:v0.13.1-rc2, 
kube-flannel-ds-mffzm:	harbor.mykernel.cn/pub/flannel:v0.13.1-rc2, 
kube-proxy-6dnnt:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.1, 
kube-proxy-fwjmc:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.1, 
kube-proxy-lfrwr:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.1, 
kube-proxy-mt7n8:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.1, 
kube-scheduler-master01.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.1, 
kube-scheduler-master02.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.1, 
kube-scheduler-master03.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.1, 


# 升级

```

先在haproxy上摘下master2, master3。就升级master2, master3。api请求不会向master2/3转发。升级完成就上线Master2/master3，观察一段时间在升级master1.

```diff
root@haproxy01:~# vim /etc/haproxy/haproxy.cfg 
listen apiserver-6443
    bind 172.16.3.248:6443
    mode tcp
    server 172.16.3.101 172.16.3.101:6443  check inter 3s fall 2 rise 5
+    #server 172.16.3.102 172.16.3.102:6443  check inter 3s fall 2 rise 5
+    #server 172.16.3.103 172.16.3.103:6443  check inter 3s fall 2 rise 5
listen harbor-443
    bind 172.16.3.248:443
    mode tcp
    server 172.16.3.106 172.16.3.106:443  check inter 3s fall 2 rise 5
    server 172.16.3.107 172.16.3.107:443  check inter 3s fall 2 rise 5



+root@haproxy01:~# systemctl restart haproxy
```

同时升级master2, master3. 因为api请求不会转发至master2/master3

```diff
+ kubeadm upgrade apply v1.20.4 # master2
root@master02:~# kubeadm upgrade apply v1.20.4
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade/version] You have chosen to change the cluster version to "v1.20.4"
[upgrade/versions] Cluster version: v1.20.1
[upgrade/versions] kubeadm version: v1.20.4
+[upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]: y
[upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
[upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
+[upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull' #拉镜像
...
[kubelet] Creating a ConfigMap "kubelet-config-1.20" in namespace kube-system with the configuration for the kubelets in the cluster
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy
+[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.20.4". Enjoy!

+[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.


+ kubeadm upgrade apply v1.20.4 # master3
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade/version] You have chosen to change the cluster version to "v1.20.4"
[upgrade/versions] Cluster version: v1.20.1
[upgrade/versions] kubeadm version: v1.20.4
+[upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]: y
[upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
[upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
+[upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull' #拉镜像
...
[kubelet] Creating a ConfigMap "kubelet-config-1.20" in namespace kube-system with the configuration for the kubelets in the cluster
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy
+[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.20.4". Enjoy!

+[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

验证pod版本

```bash
root@master02:~# kubectl get pods -n kube-system -o=jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}' |sort

coredns-54d67798b7-cgp2j:	registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0, 
coredns-54d67798b7-l9mrm:	registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0, 
etcd-master01.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0, 
etcd-master02.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0, 
etcd-master03.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0, 
kube-apiserver-master01.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.1, 
kube-apiserver-master02.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.4, 
kube-apiserver-master03.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.4, 
kube-controller-manager-master01.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.1, 
kube-controller-manager-master02.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.4, 
kube-controller-manager-master03.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.4, 
kube-flannel-ds-7wk87:	harbor.mykernel.cn/pub/flannel:v0.13.1-rc2, 
kube-flannel-ds-dv57p:	harbor.mykernel.cn/pub/flannel:v0.13.1-rc2, 
kube-flannel-ds-krc7l:	harbor.mykernel.cn/pub/flannel:v0.13.1-rc2, 
kube-flannel-ds-mffzm:	harbor.mykernel.cn/pub/flannel:v0.13.1-rc2, 
kube-proxy-cxqgx:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.4, 
kube-proxy-fwjmc:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.1, 
kube-proxy-hh8g7:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.4, 
kube-proxy-lfrwr:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.1, 
kube-scheduler-master01.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.1, 
kube-scheduler-master02.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.4, 
kube-scheduler-master03.magedu.local:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.4, 

```

> 现在master组件 scheduler, controller, api server ,kube-proxy, 版本已经 变化
>
> master1的版本还未变化
>
> node1的kube-proxy版本未变化

升级完成之后验证pod运行正常

```bash
kubectl get pods -A
```

#### 升级master1

由于升级会先拉镜像，所以在master1，可以先拉镜像

```bash
root@master01:~# kubeadm config images pull --kubernetes-version=v1.20.4 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.4
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.4
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.4
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.4
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0

```



```bash
kubeadm upgrade plan # master1
```

升级前，haproxy把master2/master3挂上，把master1摘除，来升级master1

```diff
vim  /etc/haproxy/haproxy.cfg


listen apiserver-6443
    bind 172.16.3.248:6443
    mode tcp
+    #server 172.16.3.101 172.16.3.101:6443  check inter 3s fall 2 rise 5
+    server 172.16.3.102 172.16.3.102:6443  check inter 3s fall 2 rise 5
+    server 172.16.3.103 172.16.3.103:6443  check inter 3s fall 2 rise 5
listen harbor-443
    bind 172.16.3.248:443
    mode tcp
    server 172.16.3.106 172.16.3.106:443  check inter 3s fall 2 rise 5
    server 172.16.3.107 172.16.3.107:443  check inter 3s fall 2 rise 5
+root@haproxy01:~# systemctl restart haproxy

```

升级master1

```bash
root@master01:~#  kubeadm upgrade apply v1.20.4
```

升级之后验证pod

```bash
root@master02:~# kubectl get pods -A
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
kube-system   coredns-54d67798b7-trnqp                       1/1     Running   1          80m
kube-system   coredns-54d67798b7-vx2gp                       1/1     Running   0          80m
kube-system   etcd-master01.mykernel.cn                      1/1     Running   4          80m
kube-system   etcd-master02.mykernel.cn                      1/1     Running   0          12m
kube-system   etcd-master03.mykernel.cn                      1/1     Running   0          8m16s
kube-system   kube-apiserver-master01.mykernel.cn            1/1     Running   0          3m37s
kube-system   kube-apiserver-master02.mykernel.cn            1/1     Running   0          12m
kube-system   kube-apiserver-master03.mykernel.cn            1/1     Running   0          8m1s
kube-system   kube-controller-manager-master01.mykernel.cn   1/1     Running   0          3m6s
kube-system   kube-controller-manager-master02.mykernel.cn   1/1     Running   0          11m
kube-system   kube-controller-manager-master03.mykernel.cn   1/1     Running   0          7m31s
kube-system   kube-flannel-ds-9kjj8                          1/1     Running   0          37m
kube-system   kube-flannel-ds-dxpdc                          1/1     Running   2          61m
kube-system   kube-flannel-ds-jhr9l                          1/1     Running   0          61m
kube-system   kube-flannel-ds-rcnvj                          1/1     Running   0          61m
kube-system   kube-proxy-6v2t9                               1/1     Running   0          8m50s
kube-system   kube-proxy-dw5b4                               1/1     Running   0          7m21s
kube-system   kube-proxy-j9pkz                               1/1     Running   0          10m
kube-system   kube-proxy-rvxvp                               1/1     Running   0          6m27s
kube-system   kube-scheduler-master01.mykernel.cn            1/1     Running   0          2m42s
kube-system   kube-scheduler-master02.mykernel.cn            1/1     Running   0          11m
kube-system   kube-scheduler-master03.mykernel.cn            1/1     Running   0          6m49s
```

验证pod版本

```bash
kubectl get pods -n kube-system -o=jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}' |sort
```

```bash
root@master02:~# kubectl get pods -n kube-system -o=jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}' |sort

coredns-54d67798b7-trnqp:	registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0, 
coredns-54d67798b7-vx2gp:	registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0, 
etcd-master01.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0, 
etcd-master02.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0, 
etcd-master03.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0, 
kube-apiserver-master01.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.4, 
kube-apiserver-master02.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.4, 
kube-apiserver-master03.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.4, 
kube-controller-manager-master01.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.4, 
kube-controller-manager-master02.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.4, 
kube-controller-manager-master03.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.4, 
kube-flannel-ds-9kjj8:	harbor.mykernel.cn/pub/flannel:v0.13.1-rc2, 
kube-flannel-ds-dxpdc:	harbor.mykernel.cn/pub/flannel:v0.13.1-rc2, 
kube-flannel-ds-jhr9l:	harbor.mykernel.cn/pub/flannel:v0.13.1-rc2, 
kube-flannel-ds-rcnvj:	harbor.mykernel.cn/pub/flannel:v0.13.1-rc2, 
kube-proxy-6v2t9:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.4, 
kube-proxy-dw5b4:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.4, 
kube-proxy-j9pkz:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.4, 
kube-proxy-rvxvp:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.4, 
kube-scheduler-master01.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.4, 
kube-scheduler-master02.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.4, 
kube-scheduler-master03.mykernel.cn:	registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.4, 
```

> 所有版本均变化

将haproxy的api地址全部挂上 api 

```bash
listen apiserver-6443
    bind 172.16.3.248:6443
    mode tcp
    server 172.16.3.101 172.16.3.101:6443  check inter 3s fall 2 rise 5
    server 172.16.3.102 172.16.3.102:6443  check inter 3s fall 2 rise 5
    server 172.16.3.103 172.16.3.103:6443  check inter 3s fall 2 rise 5

```

```bash
root@haproxy01:~# systemctl restart haproxy
```



#### 升级master的kubelet, kubectl

升级前kubelet前

```bash
root@master02:~# kubectl get node
NAME                   STATUS   ROLES                  AGE   VERSION
master01.mykernel.cn   Ready    control-plane,master   80m   v1.20.1
master02.mykernel.cn   Ready    control-plane,master   77m   v1.20.1
master03.mykernel.cn   Ready    control-plane,master   76m   v1.20.1
node01.mykernel.cn     Ready    <none>                 41m   v1.20.1
```

```bash
root@master01:~# apt install kubelet=1.20.4-00 kubectl=1.20.4-00
root@master02:~# apt install kubelet=1.20.4-00 kubectl=1.20.4-00
root@master03:~# apt install kubelet=1.20.4-00 kubectl=1.20.4-00
```

查看node

```bash
root@master02:~# kubectl get node
NAME                   STATUS   ROLES                  AGE   VERSION
master01.mykernel.cn   Ready    control-plane,master   84m   v1.20.4
master02.mykernel.cn   Ready    control-plane,master   82m   v1.20.4
master03.mykernel.cn   Ready    control-plane,master   80m   v1.20.4
node01.mykernel.cn     Ready    <none>                 45m   v1.20.1
```

kubelet成了1.17.4就意味着启动docker的kubelet版本变化，意味着每个节点的版本变化





#### 升级node

在各node节点执行

```diff
root@node01:~# kubeadm upgrade node 
[upgrade] Reading configuration from the cluster...
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks
[preflight] Skipping prepull. Not a control plane node.
[upgrade] Skipping phase. Not a control plane node.
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.

```

然后在node节点安装指定版本的kubeadm kubelet即可

```bash
root@node01:~# apt install kubeadm=1.20.4-00 kubelet=1.20.4-00
```

# haproxy-nginx-tomcat

https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/

跑nginx, 内外网通信，能调用tomcat即可



## 制作nginx

```bash
docker pull nginx:1.14.2
root@master01:~# docker tag nginx:1.14-alpine harbor.mykernel.cn/pub/nginx:1.14-alpine
root@master01:~# docker push harbor.mykernel.cn/pub/nginx:1.14-alpine
The push refers to repository [harbor.mykernel.cn/pub/nginx]
076c58d2644f: Pushed 
b2cbae4b8c15: Pushed 
5ac9a5170bf2: Pushed 
a464c54f93a9: Pushed 
1.14-alpine: digest: sha256:a3a0c4126587884f8d3090efca87f5af075d7e7ac8308cffc09a5a082d5f4760 size: 1153
root@master01:~# 
```

```diff
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# cat nginx.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
+        image: harbor.mykernel.cn/pub/nginx:1.14-alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
+  name: nginx-service
+  labels:
+    app: nginx-service-label
+  namespace: default
spec:
  type: NodePort
  selector:
+    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30007
      name: http

```

```bash
# kubectl apply -f nginx.yml 
```

## 访问nginx

http://172.16.3.108:30007/

可以发现通了，进入不同nginx分配不同的页面

```bash
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# kubectl exec -it nginx-deployment-8498cbd48d-cml62 bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
OCI runtime exec failed: exec failed: container_linux.go:370: starting container process caused: exec: "bash": executable file not found in $PATH: unknown
command terminated with exit code 126
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# kubectl exec -it nginx-deployment-8498cbd48d-cml62 sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
/ # cat /etc/issue
Welcome to Alpine Linux 3.9
Kernel \r on an \m (\l)

/ # ps -ef | grep nginx
    1 root      0:00 nginx: master process nginx -g daemon off;
    7 nginx     0:00 nginx: worker process
   22 root      0:00 grep nginx
/ # echo pod1 > /usr/share/nginx/html/index.html 



root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# kubectl exec -it nginx-deployment-8498cbd48d-ktzkd sh 
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
/ # echo pod2 > /usr/share/nginx/html/index.html

root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# kubectl exec -it nginx-deployment-8498cbd48d-sjr5q sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
/ # echo pod3 > /usr/share/nginx/html/index.html

```

http://172.16.3.108:30007/ 

现在不断请求这个页面

```bash
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# curl http://172.16.3.101:30007/
pod3
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# curl http://172.16.3.101:30007/
pod2
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# curl http://172.16.3.101:30007/
pod2
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# curl http://172.16.3.101:30007/
pod2
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# curl http://172.16.3.101:30007/
pod2
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# curl http://172.16.3.101:30007/
pod1

可以发现pod是轮循的，用户不能这样访问
```

为了测试方便将nginx运行一个副本

```diff
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# cat nginx.yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
+  replicas: 1
  selector:
    matchLabels:
...
```

```bash
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# kubectl apply -f nginx.yml 
```



## haproxy接入k8s流量

由于用户访问80/443所以在haproxy上直接反代至这些nodePort

防火墙只能反代到一个内网ip, 直接反代给haproxy就可以了

```bash
listen http-80
    bind 172.16.3.249:80
    mode tcp
    server 172.16.3.108 172.16.3.108:30007  check inter 3s fall 2 rise 5
    server 172.16.3.109 172.16.3.109:30007  check inter 3s fall 2 rise 5
    server 172.16.3.110 172.16.3.110:30007  check inter 3s fall 2 rise 5

```

```bash
root@haproxy01:~# systemctl restart haproxy
```

配置keepalived

```diff
 25     virtual_ipaddress {
 26         172.16.3.248 dev eth0 label eth0:0
+ 27         172.16.3.249 dev eth0 label eth0:1                                                                                                                                                                                                                                                                             
 28     }
 29 }
root@haproxy01:~# systemctl restart keepalived
```

这样haproxy就是k8s项目的入口，在nginx做动静分离，多域名识别

现在直接访问vip http://172.16.3.249/

最后直接通过域名解析至这个vip即可

```bash
172.16.3.249 www.linux48.com
```

直接访问 www.linux48.com



## 制作tomcat

访问www.linux48.com/app1就转发到tomcat

```bash
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# docker pull tomcat
latest: Pulling from library/tomcat
b9a857cbf04d: Pull complete 
d557ee20540b: Pull complete 
3b9ca4f00c2e: Download complete 
667fd949ed93: Downloading [=========================>                         ]  25.97MB/51.83MB
661d3b55f657: Download complete 
511ef4338a0b: Download complete 
a56db448fefe: Downloading [===========================>                       ]  108.7MB/196.8MB
00612a99c7dc: Download complete 
326f9601c512: Download complete 
c547db74f1e1: Download complete 

```

测试run

```bash
docker run -it --rm -p 8080:8080 tomcat 

# 换终端
root@master01:~# docker exec -it elastic_northcutt bash
root@0c9340cc3caf:/usr/local/tomcat# pwd
/usr/local/tomcat
root@0c9340cc3caf:/usr/local/tomcat# cd webapps/
root@0c9340cc3caf:/usr/local/tomcat/webapps# ls
root@0c9340cc3caf:/usr/local/tomcat/webapps# mkdir app
root@0c9340cc3caf:/usr/local/tomcat/webapps# echo 'tomcat app' > app/index.html

# 测试访问
http://172.16.3.101:8080/app/
返回 tomcat app
```

测试正常后，把tomcat跑在k8s中

```bash
root@master01:/usr/local/src/kubeadm-linux39/tomcat-dockerfile# pwd
/usr/local/src/kubeadm-linux39/tomcat-dockerfile
root@master01:/usr/local/src/kubeadm-linux39/tomcat-dockerfile# ls
Dockerfile  app
root@master01:/usr/local/src/kubeadm-linux39/tomcat-dockerfile# cat app/index.html 
tomcat app images page
root@master01:/usr/local/src/kubeadm-linux39/tomcat-dockerfile# cat Dockerfile 
FROM tomcat

ADD ./app /usr/local/tomcat/webapps/

```

创建harbor项目`linux48`, 公开

```bash
root@master01:/usr/local/src/kubeadm-linux39/tomcat-dockerfile# docker build -t harbor.mykernel.cn/linux48/tomcat:app .
Sending build context to Docker daemon  3.584kB
Step 1/2 : FROM tomcat
 ---> 040bdb29ab37
Step 2/2 : ADD ./app /usr/local/tomcat/webapps/app/
 ---> 8d02dc820826
Successfully built 8d02dc820826
Successfully tagged harbor.mykernel.cn/linux48/tomcat:app
```

测试运行

```bash
root@master01:/usr/local/src/kubeadm-linux39/tomcat-dockerfile# docker run -it --rm -p 8080:8080 harbor.mykernel.cn/linux48/tomcat:app
# 访问 http://172.16.3.101:8080/app/
```

正常后，上传至harbor

```bash
root@master01:/usr/local/src/kubeadm-linux39/tomcat-dockerfile# docker push  harbor.mykernel.cn/linux48/tomcat:app
The push refers to repository [harbor.mykernel.cn/linux48/tomcat]
ccc98b8885cb: Pushed 
9ddc8cd8299b: Pushed 
c9132b9a3fc8: Pushed 
8e2e6f2527c7: Pushed 
500f722b156b: Pushing [=========>                                         ]  63.52MB/323.8MB
7a9b35031285: Pushed 
7496c5e8691b: Pushing [==================================================>]  11.53MB
aa7af8a465c6: Pushing [====>                                              ]  14.21MB/145.5MB
ef9a7b8862f4: Pushing [===================================>               ]  12.35MB/17.54MB
a1f2f42922b1: Pushing [=====================>                             ]  7.091MB/16.47MB
4762552ad7d8: Waiting 
```

制作yml

```bash
root@master01:/usr/local/src/kubeadm-linux39/tomcat-yml# cp ../nginx-yml/nginx.yml ./tomcat.yml 
```

> 全文替换nginx -> tomcat
>
> 端口8080, nodeport惟一

```diff
root@master01:/usr/local/src/kubeadm-linux39/tomcat-yml# cat tomcat.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  labels:
    app: tomcat
spec:
+  replicas: 1
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
      - name: tomcat
+        image: harbor.mykernel.cn/linux48/tomcat:app
        ports:
+        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  labels:
    app: tomcat-service-label
  namespace: default
spec:
  type: NodePort
  selector:
    app: tomcat
  ports:
+    - port: 8080
+      targetPort: 8080
+      nodePort: 30008
      name: http
```

```bash
root@master01:/usr/local/src/kubeadm-linux39/tomcat-yml# kubectl apply -f tomcat.yml 
```

## 访问tomcat

http://172.16.3.108:30008/app

一旦访问通了，就应该关闭这个端口，因为tomcat是不对外的，需要撤消端口

```bash
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  labels:
    app: tomcat-service-label
  namespace: default
spec:
#  type: NodePort
  selector:
    app: tomcat
  ports:
    - port: 8080
      targetPort: 8080
#      nodePort: 30008
      name: http

```

```bash
root@master01:/usr/local/src/kubeadm-linux39/tomcat-yml# kubectl apply -f tomcat.yml 
```

## nginx转发tomcat

nginx打镜像，转发至tomcat

为了测试方便，先在容器中修改

```bash
root@master01:/usr/local/src/kubeadm-linux39# kubectl get pods 
NAME                                 READY   STATUS              RESTARTS   AGE
nginx-deployment-8498cbd48d-cml62    1/1     Running             0          41m
tomcat-deployment-64b89b7794-nvc9k   0/1     ContainerCreating   0          9m40s


root@master01:/usr/local/src/kubeadm-linux39# kubectl exec -it nginx-deployment-8498cbd48d-cml62  -- sh
/ # wget -O - -q tomcat-service:8080/app/index.html
tomcat app images page


vi /etc/nginx/conf.d/default.conf 


location /app {
	proxy_pass http://tomcat-service:8080;
}


/ # nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
/ # nginx -s reload
2021/02/23 07:25:50 [notice] 52#52: signal process started

```

访问viphttp://www.linux48.com/app/ 

返回 tomcat app images page



把node01节点上的nginx配置拿下来

```bash
root@node01:~# docker cp 0a9630691148:/etc/nginx/conf.d/default.conf  ./
```



```bash
root@master01:/usr/local/src/kubeadm-linux39/nginx-dockerfile# cat Dockerfile
FROM nginx:1.14-alpine


ADD default.conf /etc/nginx/conf.d/default.conf
```

生成镜像

```bash
root@master01:/usr/local/src/kubeadm-linux39/nginx-dockerfile# docker build -t harbor.mykernel.cn/linux48/nginx:v1 .
Sending build context to Docker daemon  4.096kB
Step 1/2 : FROM nginx:1.14-alpine
 ---> 8a2fb25a19f5
Step 2/2 : ADD default.conf /etc/nginx/conf.d/default.conf
 ---> 8c46341174e4
Successfully built 8c46341174e4
Successfully tagged harbor.mykernel.cn/linux48/nginx:v1


root@master01:/usr/local/src/kubeadm-linux39/nginx-dockerfile# docker push harbor.mykernel.cn/linux48/nginx:v1
The push refers to repository [harbor.mykernel.cn/linux48/nginx]
1f27779db7f7: Pushing [==================================================>]  4.608kB
076c58d2644f: Mounted from pub/nginx 
b2cbae4b8c15: Mounted from pub/nginx 
5ac9a5170bf2: Mounted from pub/nginx 
a464c54f93a9: Mounted from pub/nginx 

```

更新yaml文件

```diff
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# cat nginx.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
+        image: harbor.mykernel.cn/linux48/nginx:v1
```

```bash
root@master01:/usr/local/src/kubeadm-linux39/nginx-yml# kubectl apply -f nginx.yml 
```











