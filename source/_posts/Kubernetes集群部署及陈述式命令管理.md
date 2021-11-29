---
title: Kubernetes集群部署及陈述式命令管理
date: 2021-01-13 13:46:58
tags:
toc: true
---



 

# 部署前考虑环境

## 测试环境

```bash
1）1 master, 1 etcd
2）3 nodes
nfs或glusterfs...存储
```

单机、学习使用：minikube



## 生产环境

```bash
1）高可用etcd：并且对etcd备份，即使k8s崩溃了，可以直接恢复。
2）高可用master
	kube-apiserver无状态：
		方案一：keepalived vip 流动实现 冗余 
		方案二：lvs或haproxy或nginx反代，并借助于keepalived对代理服务器进行冗余 
	kube-scheduler及kube-controller-manager各自只能一个活动实例，但可以有多个备用，各自自带leader选举，默认启用状态。
	
3）多node主机，数量越多冗余能力越强。

ceph, glusterfs, iscsi, fc san及各种云存储。
```



# 部署工具

常用部署环境

- 公有云：aws、gce、azure等； 
  - 自托管k8s环境，买公有云服务才提供这个服务
- 私有云：openstack和vsphere等； 
- baremetal环境：物理服务器或独立的虚拟机等； 
  - 就算我们有openstack, 也当裸环境使用
  - vmware 也可以使用
  - 树莓派 也可以安装

部署工具

- **kubeadm** k8s官方的部署工具, alpha/beta版本
  - centos: kubeadm(静态pod), kubelet(管理pod，通过docker引擎), docker引擎
- kops 亚马逊云计算环境中
- kubespray  ansible部署
- kontena pharos     不成熟的工具
- ...

其它二次封装的常用发行版

- rancher            labs 
- tectonic           coreos
- openshift        readhat公司二次发行





# 部署方式

## 二进制部署

所有组件运行为守护进程

![image-20210113145512169](http://myapp.img.mykernel.cn/image-20210113145512169.png)

​	由于master不需要运行容器，所以不需要kubelet, docker.

​	 https://github.com/kubernetes/kubernetes/releases 下载二进制包, 注意k8s认证的版本, 目前最高19.03

## 静态pod部署

**kubeadm**命令

![image-20210113143025238](http://myapp.img.mykernel.cn/image-20210113143025238.png)

- 获取镜像，在gcr.io
- kubelet，kubeadm 使用rpm安装，rpm仓库也在google上，好在mirrors.aliyun.com已经镜像在[Kubernetes](http://mirrors.aliyun.com/kubernetes/)仓库中。kubelet, kubeadm, kube-proxy, kube-apiserver, ...





# NMT(Nginx+Tomcat+MariaDB)

![image-20210113142301090](http://myapp.img.mykernel.cn/image-20210113142301090.png)

> 用户 -> nginx service/deploy/pod -> tomcat servcie/deploy/pod -> mariadb service/deploy/pod 

运维工作：**发布、变更、故障处理**

​    发布：控制器：创建pod

​    变更 ：扩容或缩容，控制 器

​    故障处理：故障重启，控制器。   例如：Pod坏了，控制 器会自动恢复

​    以往的任务，统统由控制器实现，所以控制器是封装的运维工程师， 你不用7X24工作，控制器可以`controller loop`进行和解循环 7X24小时工作。

# 静态pod部署 kubernetes

学习环境搭建：

1. 测试环境

2. 部署工具 kubeadm
3. 静态pod部署

使用kubeadm命令部署, 参考部署文档 kubeadm(v2).txt

- [x] 系统环境， 各节点

- [x] 各节点安装程序包：docker, kubelet, kubeadm, kubectl

- [x] 启动服务
  
  - docker
  - kubelet
  
- [x] 初始化master, 添加node至Master

  - master
  - 添加node至master

  

## 系统环境

最小化安装系统，不添加中文语言支持，**centos 7.5 1804, 均为2核2G内核**

### 节点规划

| 节点名称 | ip           | 命令                                         |
| -------- | ------------ | -------------------------------------------- |
| master01 | 172.16.1.100 | `hostnamectl set-hostname master.magedu.com` |
| node01   | 172.16.1.101 | `hostnamectl set-hostname node01.magedu.com` |
| node02   | 172.16.1.102 | `hostnamectl set-hostname node02.magedu.com` |
| node03   | 172.16.1.103 | `hostnamectl set-hostname node03.magedu.com` |

### 各节点时间同步

- [x] chronyd服务，前提系统 可以访问互联网
- [ ] 自建时间服务器

```bash
# 安装chrony
yum -y install chrony

systemctl enable chronyd
systemctl restart chronyd # 过一会就自动同步
```

### 各节点主机名解析

- [ ] 内网dns完成主机名解析
- [x] host文件完成主机名解析

```bash
# /etc/hosts
172.16.1.100 master.magedu.com master
172.16.1.101 node01.magedu.com node01
172.16.1.102 node02.magedu.com node02
172.16.1.103 node03.magedu.com node03
```



### 各节点 关闭iptables, firewalld

kubernetes生成大量iptables规则，避免混淆

```bash
systemctl disable iptables  && systemctl stop iptables 
systemctl disable firewalld && systemctl stop  firewalld

```



### 各节点禁用selinux

```bash
setenforce 0
sed -i 's@^\(SELINUX=\).*@\1disabled@' /etc/sysconfig/selinux

# 验证
getenforce
```



### 各节点禁用swap设备

- [ ] 禁用swap

  ```bash
  # 查看swap
  free -m
  
  # 禁用swap
  swapoff -a
  
  # 编辑fstab, 注释swap类型
  ```

- [x] 不禁用swap设备，明确表示不禁用使用选项





### 各节点ipvs模型

service2种类型iptables, ipvs(k8s **1.11**之后的版本，各节点载入ipvs模块)

/etc/sysconfig/modules/目录下脚本，用于开机自动载入模块使用

```bash
# /etc/sysconfig/modules/ipvs.modules
#!/bin/bash
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs" 
#  /usr/lib/modules/ 存在多个内核版本。$(uname -r) 当前内核版本 kernel/net/netfilter/ipvs 里面 内核 模块名.ko.xz
for mod in $(ls $ipvs_mods_dir | grep -o "^[^.]*"); do 
# .ko结尾，去掉.ko
    /sbin/modinfo -F filename $mod  &> /dev/null # 探测模块存在
    if [ $? -eq 0 ]; then
        /sbin/modprobe $mod
    fi
done

# chmod +x /etc/sysconfig/modules/ipvs.modules
# /etc/sysconfig/modules/ipvs.modules
```

其他节点

```bash
for 1 2 3; do scp /etc/sysconfig/modules/ipvs.modules node0$i:/etc/sysconfig/modules/ipvs.modules; done

各节点加载
 /etc/sysconfig/modules/ipvs.modules

```



### 各节点禁用NetworkManager

networkmanager会自动生成一个domain在/etc/resolv.cnf中，k8s中会使用这个domain

```bash
systemctl disable NetworkManager --now
```

### 各节点重启

```bash
reboot
```



## 各节点安装程序包

### 所有节点安装docker

一定要最新的版本, runC漏洞，容器提权宿主机的权限

https://mirrors.aliyun.com/docker-ce/linux/

https://mirrors.aliyun.com/docker-ce/linux/centos/

docker的源

https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo 

http://mirrors.aliyun.com/repo/

centos 7的源

http://mirrors.aliyun.com/repo/Centos-7.repo

```bash
# 所有节点准备repo文件
cd /etc/yum.repos.d/
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 避免系统环境提供的docker依赖的包版本低，直接使用阿里云的base源
wget http://mirrors.aliyun.com/repo/Centos-7.repo

# 所有节点安装docker-ce
yum -y install docker-ce
# 当前docker版本
Installed:
  # 程序包名.架构      主版本号.次版本号.发行号-编译号.系统版本
  docker-ce.x86_64 3:20.10.2-3.el7    
```

k8s官方会对自已支持的docker版本进行**认证**，经过官方认证的docker版本才是稳定的.

https://blog.csdn.net/M82_A1/article/details/98872734



当然也可以直接使用最新版本的docker-ce, 忽略警告。

### 所有节点启动docker

- [x] docker镜像加速服务，本地网络不加速
- [x] forward链默认会禁用, 修改为accept

```diff
# /usr/lib/systemd/system/docker.service
+ Environment="HTTPS_PROXY=192.168.0.33:808"
+ Environment="NO_PROXY=127.0.0.0/8,172.16.0.0/16"
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
+ ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT


# 启动docker
systemctl daemon-reload
systemctl start docker

# 验证代理
docker info
+ HTTPS Proxy: 192.168.0.33:808
+ No Proxy: 127.0.0.0/8,172.16.0.0/16

# 验证iptables
[root@master yum.repos.d]# iptables -vnL FORWARD
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes) # ACCEPT

# 桥接参数
[root@master yum.repos.d]# sysctl -a | grep bridge
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1 # 需要为1
net.bridge.bridge-nf-call-iptables = 1  # 需要为1

# 如果不为1
# cat /etc/sysctl.d/k8s.cnf
+ net.bridge.bridge-nf-call-ip6tables = 1
+ net.bridge.bridge-nf-call-iptables = 1

# 重读参数
sysctl -p /etc/sysctl.d/k8s.cnf
```

其他节点，直接复制, 启动

```bash
[root@master yum.repos.d]# for i in 1 2 3; do scp /usr/lib/systemd/system/docker.service node0$i:/usr/lib/systemd/system/docker.service; done 
[root@master yum.repos.d]# for i in 1 2 3; do scp /etc/sysctl.d/k8s.cnf node0$i:/etc/sysctl.d/k8s.cnf; done

# 启动
systemctl daemon-reload && systemctl start docker

# 验证
docker info
 HTTPS Proxy: 192.168.0.33:808
 No Proxy: 127.0.0.0/8,172.16.0.0/16
```



所有节点的docker开启自启

```bash
systemctl enable docker
```

### 所有节点安装kubernetes程序包

#### yum仓库

http://mirrors.aliyun.com/kubernetes/

apt # 适用于ubuntu/debian

yum # 适用于centos/rehl

![image-20210114111631838](http://myapp.img.mykernel.cn/image-20210114111631838.png)

> doc目录
>
> [apt-key.gpg](http://mirrors.aliyun.com/kubernetes/yum/doc/apt-key.gpg)                                        13-Jan-2021 20:13                1974 [apt-key.gpg.asc](http://mirrors.aliyun.com/kubernetes/yum/doc/apt-key.gpg.asc)                                    13-Jan-2021 20:13                2752 [rpm-package-key.gpg](http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg)                                13-Jan-2021 20:13                 975 [yum-key.gpg](http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg)                                        13-Jan-2021 20:13                3654 
>
> rpm-package-key, yum-key实现程序包的签名验证



![image-20210114111640771](http://myapp.img.mykernel.cn/image-20210114111640771.png)

找到与平台版本相匹配:

centos7 -> el7

amd64 -> x86_64

unstable 非稳定版，不建议

所以选择`kubernetes-el7-x86_64/`

![image-20210114111829756](http://myapp.img.mykernel.cn/image-20210114111829756.png)

进入后，看到repodata, 这里就是repo文件的url的起始路径 

生成repo文件

```bash
# /etc/yum.repos.d/kuberntes.repo
[kubernetes]
name=Kubernetes Repository
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg 	 
	http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
```

> name仓库名
>
> baseurl 指向repodata
>
> gpgcheck签名验证
>
> gpgkey 签名验证的key, 在http://mirrors.aliyun.com/kubernetes/yum/doc/目录下, 直接复制对应的url



检查仓库

```bash 
 yum repolist

	kubernetes                   624/624

								 # 624个包		
```



#### 安装kube包

```bash
 yum list all | grep "^kube"
											# 版本
kubeadm.x86_64                              1.20.2-0                   kubernetes # 主要关注kubernetes的仓库，是我们自已建立的
kubectl.x86_64                              1.20.2-0                   kubernetes
kubelet.x86_64                              1.20.2-0                   kubernetes
kubernetes.x86_64                           1.5.2-0.7.git269f928.el7   extras   
kubernetes-client.x86_64                    1.5.2-0.7.git269f928.el7   extras   
kubernetes-cni.x86_64                       0.8.7-0                    kubernetes
kubernetes-master.x86_64                    1.5.2-0.7.git269f928.el7   extras   
kubernetes-node.x86_64                      1.5.2-0.7.git269f928.el7   extras  
```

```bash
yum -y install kubeadm kubectl kubelet
```



其他节点安装

```bash
for i in 1 2 3 ; do scp /etc/yum.repos.d/kuberntes.repo node0$i:/etc/yum.repos.d/kuberntes.repo; done
yum -y install kubeadm kubectl kubelet
```

#### kubelet

```bash
[root@master yum.repos.d]# rpm -ql kubelet
/etc/kubernetes/manifests
/etc/sysconfig/kubelet                        # 配置KUBELET_EXTRA_ARGS="--fail-swap-on=false", 避免swap未禁用, 而报错
/usr/bin/kubelet                              # 应用运行程序
/usr/lib/systemd/system/kubelet.service       # 会读取/etc/sysconfig/kubelet

```

#### kubeadm

```bash
[root@master yum.repos.d]# rpm -ql kubeadm
/usr/bin/kubeadm
/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf # 配置文件


[root@master ~]# kubeadm

Available Commands:
  alpha       Kubeadm experimental sub-commands # 实验
  certs       Commands related to handling kubernetes certificates
  completion  Output shell completion code for the specified shell (bash or zsh) # 自动补全
  config      Manage configuration for a kubeadm cluster persisted in a ConfigMap in the cluster # 配置
  help        Help about any command
  init        Run this command in order to set up the Kubernetes control plane # 初始化集群，仅master
  join        Run this on any machine you wish to join an existing cluster # 加入主节点
  reset       Performs a best effort revert of changes made to this host by 'kubeadm init' or 'kubeadm join'  # 无论master/node重置
  token       Manage bootstrap tokens  # 管理令牌
  upgrade     Upgrade your cluster smoothly to a newer version with this command # 版本升级


kubeadm help config
kubeadm  config  images --help
	list        Print a list of images kubeadm will use. The configuration file is used in case any images or image repositories are customized
  	pull        Pull images used by kubeadm

kubeadm config print --help # 获取帮助

[root@master ~]# kubeadm  config   print init-defaults # 获取初始化配置
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: master.magedu.com
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
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.20.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}

```

#### kubectl

```bash
[root@master yum.repos.d]# rpm -ql kubectl
/usr/bin/kubectl

以下验证步骤在k8s集群搭建好之后
# 执行依赖
家目录下有.kube目录，下面有config文件

# 主节点上, 获取当前配置
[root@master ~]# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED # 证书双向认证
    server: https://172.16.1.100:6443 # 连接k8s api地址
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
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED

```



## 初始化master, 添加node至Master

- [ ] yaml文件定义，复杂方式后面参考文档
- [x] kubeadm 传递选项, 简单方法

### master

获取初始化默认配置

会自动加载以yaml格式配置信息, 包含2种配置

- [ ] bootstrapTokens 令牌，先不管
- [x] apiServer

```bash
[root@master ~]# kubeadm  config   print init-defaults
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: master.magedu.com
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki # 证书
clusterName: kubernetes # 集群名
controllerManager: {}
dns:
  type: CoreDNS # dns
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io # 哪儿 加载镜像
kubeProxy:
  config:
    mode: "ipvs" # 直接初始化ipvs类型
    ipvs:
      ExcludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: "" #  默认rr, 可以指定wrr, wlc
      syncPeriod: 30s    
kubeletConfiguration:
  baseConfig:
    cgroupDriver: cgroupfs
    clusterDNS:
    - 10.96.0.10        # kubelet指定dns位置
    clusterDomain: cluster.local
    failSwapOn: false
    resolvConf: /etc/resolv.conf
    staticPodPath: /etc/kubernetes/manifests
kind: ClusterConfiguration
kubernetesVersion: v1.20.0 # 版本
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12 # 服务网络
  podSubnet: 10.244.0.0/16
scheduler: {}

```



#### 配置k8s网络

##### 节点网络

以上规划的节点是172.16.0.0/16

##### service网络

以上yaml文件serviceSubnet: 10.96.0.0/12, service创建时，会从此网络动态分配 

##### pod网络

配置pod时，k8s不管。K8S提供CNI接口，需要 3rd插件为pod提供网络, falnnel/calico，默认使用网络地址不一样。

- flannel,  默认10.244.0.0/16。**自定义时**：部署k8s指定网络地址 ，部署flannel指定地址
- calico, 默认192.168.0.0/16。**自定义时**：部署k8s指定网络地址，部署calico指定地址

k8s指定pod网络地址

- [x] 方式一：yaml文件

  flannel

  ```diff
  [root@master ~]# kubeadm  config   print init-defaults
  ...
  ---
  apiServer:
    timeoutForControlPlane: 4m0s
  apiVersion: kubeadm.k8s.io/v1beta2
  certificatesDir: /etc/kubernetes/pki
  clusterName: kubernetes
  controllerManager: {}
  dns:
    type: CoreDNS
  etcd:
    local:
      dataDir: /var/lib/etcd
  imageRepository: k8s.gcr.io
  kind: ClusterConfiguration
  kubernetesVersion: v1.20.0
  networking:
    dnsDomain: cluster.local
    serviceSubnet: 10.96.0.0/12
  + podSubnet: 10.244.0.0/16
  scheduler: {}
  
  ```

  calico

  ```diff
  [root@master ~]# kubeadm  config   print init-defaults
  ...
  ---
  apiServer:
    timeoutForControlPlane: 4m0s
  apiVersion: kubeadm.k8s.io/v1beta2
  certificatesDir: /etc/kubernetes/pki
  clusterName: kubernetes
  controllerManager: {}
  dns:
    type: CoreDNS
  etcd:
    local:
      dataDir: /var/lib/etcd
  imageRepository: k8s.gcr.io
  kind: ClusterConfiguration
  kubernetesVersion: v1.20.0
  networking:
    dnsDomain: cluster.local
    serviceSubnet: 10.96.0.0/12
  + podSubnet: 192.168.0.0/16
  scheduler: {}
  
  ```

  

- [x] 方式二：命令行选项`kubeadm --pod-network-cidr`



### kubeadm 命令行初始化master 

```bash
 kubeadm help init

 --kubernetes-version string            (default "stable-1") 应该和kubelet的版本一致

	[root@master ~]# rpm -q kubelet
	kubelet-1.20.2-0.x86_64
	所以版本是v1.20.2
    [root@master ~]# rpm -q kubeadm
    kubeadm-1.20.2-0.x86_64

--pod-network-cidr string  pod网络，现在使用最简单的网络插件，所以指定 --pod-network-cidr 10.244.0.0/16
	
--dry-run 第一次测试

--image-repository string (default "k8s.gcr.io") 国内有镜像时，使用镜像地址
	如果没有加速，就走国内的仓库。或者没有人把较新版本同步过来，就使用加速。

```



```bash
[root@master yum.repos.d]# kubeadm init --dry-run  --kubernetes-version="v1.20.2" --pod-network-cidr="10.244.0.0/16" 
[init] Using Kubernetes version: v1.20.2
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 20.10.2. Latest validated version: 19.03 # 这里表示k8s最新认证的docker版本，而当前安装的docker版本是20.10.2.没有认证不一定不兼容, 生产环境一定要使用19.03,即便使用19.03也要使用kubernetes 19.03.x最新版本.
	[WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR Swap]: running with swap on is not supported. Please disable swap
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher


# WARNING IsDockerSystemdCheck
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
systemctl restart docker

# WARNING Service-Kubelet
systemctl enable kubelet.service

# ERROR Swap  避免swap没有禁用报错
--ignore-preflight-errors=Swap 
```



初始化

```bash
[root@master yum.repos.d]# kubeadm init --dry-run  --kubernetes-version="v1.20.2" --pod-network-cidr="10.244.0.0/16"  --ignore-preflight-errors=Swap
[init] Using Kubernetes version: v1.20.2
[preflight] Running pre-flight checks
	[WARNING Swap]: running with swap on is not supported. Please disable swap
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 20.10.2. Latest validated version: 19.03
[preflight] Would pull the required images (like 'kubeadm config images pull')
[certs] Using certificateDir folder "/etc/kubernetes/tmp/kubeadm-init-dryrun484176618"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master.magedu.com] and IPs [10.96.0.1 172.16.1.100]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
 
```



测试正常后，正常跑会在互联网，通过代理下载镜像，在启动。避免等待，先拉下来镜像

#### 拉镜像

```bash
# 列出需要的镜像
kubeadm config images list

k8s.gcr.io/kube-apiserver:v1.20.2
k8s.gcr.io/kube-controller-manager:v1.20.2
k8s.gcr.io/kube-scheduler:v1.20.2
k8s.gcr.io/kube-proxy:v1.20.2
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns:1.7.0


# 拉镜像
kubeadm config images pull
[config/images] Pulled k8s.gcr.io/kube-apiserver:v1.20.2 # 表示开始托镜像了
```

其他节点快速安装镜像，可以将镜像导出，其他节点导入

#### 正常初始化master

```bash
kubeadm init  --kubernetes-version="v1.20.2" --pod-network-cidr="10.244.0.0/16"  --ignore-preflight-errors=Swap


Your Kubernetes control-plane has initialized successfully! # 主节点初始化成功

To start using your cluster, you need to run the following as a regular user: # 普通用户操作

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config # 管理员有权限, sudo。为了方便直接使用root用户执行
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster. # 部署网络插件
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at: # 部署清单
  https://kubernetes.io/docs/concepts/cluster-administration/addons/  # 清单在这里

Then you can join any number of worker nodes by running the following on each as root:
	
	# 此命令可以在node节点上直接加入集群
kubeadm join 172.16.1.100:6443 --token shlv3e.u8ub0d058pkcchfr \
    --discovery-token-ca-cert-hash sha256:eaf445126296783dc3b9b996b6f8faf03119ef6209b0a3a7b46656bbb35185cf 

```



获取集群节点

```bash
[root@master ~]# kubectl get nodes
NAME                STATUS     ROLES                  AGE     VERSION
                    # 只有flannel正常才ready
master.magedu.com   NotReady   control-plane,master   8m12s   v1.20.2
```

#### 部署flannel

flannel是coreos研发的，直接在coreos的github页面 https://github.com/coreos/flannel

For Kubernetes v1.17+ `kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`

执行以上命令后，会在当前集群自动部署flannel的pod

```bash
# 查看正在创建的pod
		# get 表示增删改查：查询
		# -n 指定名称空间
kubectl get pods -n kube-system
		
			
[root@master ~]# kubectl get pods -n kube-system
NAME                                        READY   STATUS              RESTARTS   AGE
	# 集群service 域名和ip提供动态注册
coredns-74ff55c5b-k6zwb                     0/1     Pending             0          14m
coredns-74ff55c5b-q4qrv                     0/1     ContainerCreating   0          14m
etcd-master.magedu.com                      1/1     Running             0          15m
	# apiserver
kube-apiserver-master.magedu.com            1/1     Running             0          15m
	# controller manager
kube-controller-manager-master.magedu.com   1/1     Running             0          15m
												   # 当flannel运行起来
kube-flannel-ds-4l4hp                       1/1     Running             0          39s
kube-proxy-fmj6w                            1/1     Running             0          14m
	# scheduler
kube-scheduler-master.magedu.com            1/1     Running             0          15m

# 以上全部是静态pod启动

# 查看node的状态
[root@master ~]# kubectl get nodes
NAME                STATUS   ROLES                  AGE   VERSION
					# 控制平面master已经就绪
master.magedu.com   Ready    control-plane,master   15m   v1.20.2

```

### 其它3个节点添加Node至master

- [ ] 安装kubeadm, kubectl

  ```bash
  for i in 1 2 3 ; do scp /etc/yum.repos.d/kuberntes.repo node0$i:/etc/yum.repos.d/kuberntes.repo; done
  yum -y install kubeadm kubectl kubelet
  ```

- [ ] swap忽略选项

  ```bash
  for i in 1 2 3; do scp /etc/sysconfig/kubelet node0${i}:/etc/sysconfig/kubelet; done
  ```

- [ ] 修复上面的报错选项

  ```bash
  # WARNING IsDockerSystemdCheck
  cat <<EOF | sudo tee /etc/docker/daemon.json
  {
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "100m"
    },
    "storage-driver": "overlay2",
    "storage-opts": [
      "overlay2.override_kernel_check=true"
    ]
  }
  EOF
  systemctl restart docker
  
  # WARNING Service-Kubelet
  systemctl enable kubelet.service
  ```

  

#### 方式一：镜像加速直接加入

```bash
kubeadm join 172.16.1.100:6443 --token shlv3e.u8ub0d058pkcchfr \
    --discovery-token-ca-cert-hash sha256:eaf445126296783dc3b9b996b6f8faf03119ef6209b0a3a7b46656bbb35185cf  --ignore-preflight-errors=Swap
    
    
[preflight] Running pre-flight checks
	[WARNING Swap]: running with swap on is not supported. Please disable swap
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 20.10.2. Latest validated version: 19.03
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.


# 在初始化过程中，也会托下来镜像
[root@node02 yum.repos.d]# docker images
REPOSITORY               TAG           IMAGE ID       CREATED         SIZE
k8s.gcr.io/kube-proxy    v1.20.2       43154ddb57a8   21 hours ago    118MB       # iptables/ipvs规则生成pod
quay.io/coreos/flannel   v0.13.1-rc1   f03a23d55e57   7 weeks ago     64.6MB      # flannel
k8s.gcr.io/pause         3.2           80d28bedfe5d   11 months ago   683kB       # pod的infra container

# 如果不想托，将镜像打包后，传送过来，docker load即可
```

#### 方式二：master打包，启动

其他节点可以master打包后，启动

```bash
# 打包 flannel, proxy, pause
#docker save  f03a23d55e57  80d28bedfe5d  43154ddb57a8  -o flannel-proxy-pause.tar
# 注意不带标签导出，导入时也没有标签
docker save k8s.gcr.io/kube-proxy:v1.20.2 quay.io/coreos/flannel:v0.13.1-rc1 k8s.gcr.io/pause:3.2 -o flannel-proxy-pause.tar

# 传输
scp flannel-proxy-pause.tar node02:

# 加载
 docker load -i flannel-proxy-pause.tar 
f00bc8568f7b: Loading layer [==================================================>]  53.89MB/53.89MB
6ee930b14c6f: Loading layer [==================================================>]  22.05MB/22.05MB
2b046f2c8708: Loading layer [==================================================>]  4.894MB/4.894MB
f6be8a0f65af: Loading layer [==================================================>]  4.608kB/4.608kB
3a90582021f9: Loading layer [==================================================>]  8.192kB/8.192kB
94812b0f02ce: Loading layer [==================================================>]  8.704kB/8.704kB
ef407ef15d1a: Loading layer [==================================================>]  39.49MB/39.49MB
Loaded image: k8s.gcr.io/kube-proxy:v1.20.2
ace0eda3e3be: Loading layer [==================================================>]  5.843MB/5.843MB
0a790f51c8dd: Loading layer [==================================================>]  11.42MB/11.42MB
db93500c64e6: Loading layer [==================================================>]  2.595MB/2.595MB
70351a035194: Loading layer [==================================================>]  45.68MB/45.68MB
cd38981c5610: Loading layer [==================================================>]   5.12kB/5.12kB
dce2fcdf3a87: Loading layer [==================================================>]  9.216kB/9.216kB
be155d1c86b7: Loading layer [==================================================>]   7.68kB/7.68kB
Loaded image: quay.io/coreos/flannel:v0.13.1-rc1
ba0dae6243cc: Loading layer [==================================================>]  684.5kB/684.5kB
Loaded image: k8s.gcr.io/pause:3.2

# 加入集群
kubeadm join 172.16.1.100:6443 --token shlv3e.u8ub0d058pkcchfr \
    --discovery-token-ca-cert-hash sha256:eaf445126296783dc3b9b996b6f8faf03119ef6209b0a3a7b46656bbb35185cf  --ignore-preflight-errors=Swap
```



主节点验证所有node均已经添加

```bash
[root@master ~]# kubectl get nodes
NAME                STATUS   ROLES                  AGE     VERSION
master.magedu.com   Ready    control-plane,master   29m     v1.20.2
node01.magedu.com   Ready    <none>                 5m37s   v1.20.2
node02.magedu.com   Ready    <none>                 5m23s   v1.20.2
node03.magedu.com   Ready    <none>                 5m19s   v1.20.2
```



#### 其他节点使用kubectl

由于kubectl需要 ~/.kube/config文件，加载集群，并认证。就直接拿master的config即可

k8s RBAC不同权限级别的多用户，这里的admin.conf就是管理员权限

```bash
# node节点添加目录
mkdir -p .kube


# master节点复制配置
for i in 1 2 3; do scp /etc/kubernetes/admin.conf node0$i:~/.kube/config; done

# node节点测试kubectl命令
[root@node01 ~]# kubectl get nodes
NAME                STATUS   ROLES                  AGE   VERSION
master.magedu.com   Ready    control-plane,master   47m   v1.20.2
node01.magedu.com   Ready    <none>                 23m   v1.20.2
node02.magedu.com   Ready    <none>                 22m   v1.20.2
node03.magedu.com   Ready    <none>                 22m   v1.20.2
```





# 陈述式命令管理

k8s容器编排工具，需要编排

- controller plane: 
  - api server https 6443, 请求api server需要两向认证(client和server都需要证书机构CA颁发证书)，用户需要操作k8s集群只能和api server通信。
  - Pod controller: 多个概念的统称 创建pod，出现问题得重建。
    - Deployment， 是类型
      1. ngx-deploy -> nginx pod. 需要service暴露pod端口
    - statefulset
  - scheduler : 创建的pod后，调度
- node: 
  - kubelet: 运行pod
  - kube-proxy 转换service为iptables/ipvs规则。监视k8s service资源变动，反映在当前节点上对应的iptables/ipvs规则。



api 交互方式: 

http://blog.mykernel.cn/2021/01/13/Kubernetes%E5%9F%BA%E7%A1%80%E6%A6%82%E5%BF%B5/#k8s-api

```bash 
[root@master ~]# ll /etc/kubernetes/pki/
total 56
-rw-r--r--. 1 root root 1277 Jan 14 18:34 apiserver.crt
-rw-r--r--. 1 root root 1135 Jan 14 18:34 apiserver-etcd-client.crt
-rw-------. 1 root root 1675 Jan 14 18:34 apiserver-etcd-client.key
-rw-------. 1 root root 1675 Jan 14 18:34 apiserver.key
-rw-r--r--. 1 root root 1143 Jan 14 18:34 apiserver-kubelet-client.crt
-rw-------. 1 root root 1679 Jan 14 18:34 apiserver-kubelet-client.key
-rw-r--r--. 1 root root 1066 Jan 14 18:34 ca.crt # CA cert
-rw-------. 1 root root 1675 Jan 14 18:34 ca.key # CA key
drwxr-xr-x. 2 root root  162 Jan 14 18:34 etcd
-rw-r--r--. 1 root root 1078 Jan 14 18:34 front-proxy-ca.crt
-rw-------. 1 root root 1675 Jan 14 18:34 front-proxy-ca.key
-rw-r--r--. 1 root root 1103 Jan 14 18:34 front-proxy-client.crt
-rw-------. 1 root root 1679 Jan 14 18:34 front-proxy-client.key
-rw-------. 1 root root 1675 Jan 14 18:34 sa.key
-rw-------. 1 root root  451 Jan 14 18:34 sa.pub
[root@master ~]# ll /etc/kubernetes/admin.conf
	# 地址，端口，ca和client证书信息 kubectl config view
-rw-------. 1 root root 5568 Jan 14 18:34 /etc/kubernetes/admin.conf # 持有CA颁发的client的证书，所以kubectl使用admin.conf可以与https通信
```



## 创建资源

获取kubectl帮助

```bash
[root@master ~]# kubectl -h
Basic Commands (Beginner 初级用户):
  create        创建资源
  expose        Take a replication controller, service, deployment or pod and expose it as a new Kubernetes Service
  run           Run a particular image on the cluster
  set           Set specific features on objects

Basic Commands (Intermediate 中级用户):
  explain       Documentation of resources
  get           查看
  edit          Edit a resource on the server
  delete        Delete resources by filenames, stdin, resources and names, or by resources and label selector


```



### k8s资源

在k8s集群资源中，里面分成多个名称空间

```bash 
[root@master ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   19h # 不指定名称空间，默认的名称空间
kube-node-lease   Active   19h 
kube-public       Active   19h  # 公共的
kube-system       Active   19h
```

pod是名称空间中的资源

```bash
[root@master ~]# kubectl get pod
No resources found in default namespace. # 默认在default名称空间
[root@master ~]# kubectl get pod -n kube-public
No resources found in kube-public namespace.
[root@master ~]# kubectl get pod -n kube-system # 指定在kube-system名称空间
NAME                                        READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-k6zwb                     1/1     Running   0          19h
coredns-74ff55c5b-q4qrv                     1/1     Running   0          19h
etcd-master.magedu.com                      1/1     Running   0          19h
kube-apiserver-master.magedu.com            1/1     Running   1          19h
kube-controller-manager-master.magedu.com   1/1     Running   1          19h
kube-flannel-ds-2s6p8                       1/1     Running   0          19h
kube-flannel-ds-4l4hp                       1/1     Running   0          19h
kube-flannel-ds-qqrw2                       1/1     Running   0          19h
kube-flannel-ds-vbw8m                       1/1     Running   0          19h
kube-proxy-fmj6w                            1/1     Running   0          19h
kube-proxy-j8vkr                            1/1     Running   0          19h
kube-proxy-ldvj9                            1/1     Running   0          19h
kube-proxy-nxcsj                            1/1     Running   0          19h
kube-scheduler-master.magedu.com            1/1     Running   1          19h


### 扩展
# 长格式信息展示, 类似ls -l
[root@master ~]# kubectl get pod -n kube-system -o wide
																			   # pod自已的pod  # pod运行在的节点   
NAME                                        READY   STATUS    RESTARTS   AGE   IP             NODE                NOMINATED NODE   READINESS GATES
coredns-74ff55c5b-k6zwb                     1/1     Running   0          19h   10.244.0.3     master.magedu.com   <none>           <none>
coredns-74ff55c5b-q4qrv                     1/1     Running   0          19h   10.244.0.2     master.magedu.com   <none>           <none>
etcd-master.magedu.com                      1/1     Running   0          19h   172.16.1.100   master.magedu.com   <none>           <none>
kube-apiserver-master.magedu.com            1/1     Running   1          19h   172.16.1.100   master.magedu.com   <none>           <none>

	# 注意：coredns 10.244.0.3 是flannel提供的网络， 但是下面是172.16.1.100是共享了宿主机的网络名称空间
	# docker网络模型closed, join, bridge, host中的host
```



### 创建资源

不知道有哪些资源

```bash
# 获取所有资源类型
kubectl api-resources
# 资源名                           #简写格式
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
bindings                                       v1                                     true         Binding
componentstatuses                 cs           v1                                     false        ComponentStatus

# 获取单个资源类型（完整格式）
[root@master ~]# kubectl get  deployments
No resources found in default namespace.
			#（简写格式）
[root@master ~]# kubectl get  deploy -n kube-system
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           20h 


# 可以创建哪些资源？
kubectl create -h
Available Commands:
  clusterrole         Create a ClusterRole.
  clusterrolebinding  Create a ClusterRoleBinding for a particular ClusterRole
  configmap           Create a configmap from a local file, directory or literal value
  cronjob             Create a cronjob with the specified name.
  deployment          #Create a deployment with the specified name.
  ingress             Create an ingress with the specified name.
  job                 Create a job with the specified name.
  namespace           #Create a namespace with the specified name
  poddisruptionbudget Create a pod disruption budget with the specified name.
  priorityclass       Create a priorityclass with the specified name.
  quota               Create a quota with the specified name.
  role                Create a role with single rule.
  rolebinding         Create a RoleBinding for a particular Role or ClusterRole
  secret              Create a secret using specified subcommand
  service             #Create a service using specified subcommand.
  serviceaccount      Create a service account with the specified name
```



以名称空间为例，简要说明资源的`获取信息或状态、创建、删除`

>  名称空间，资源逻辑组合，创建属于名称空间的资源，均属于它的名称空间。

```bash
# 获取名称空间
kubectl get ns
NAME              STATUS   AGE
default           Active   20h
kube-node-lease   Active   20h
kube-public       Active   20h
kube-system       Active   20h

# 创建
kubectl create namespace develop # 开发
kubectl create namespace testing # 测试
kubectl create namespace prod    # 产品
# 验证
kubectl get ns
NAME              STATUS   AGE
default           Active   20h
develop           Active   60s
kube-node-lease   Active   20h
kube-public       Active   20h
kube-system       Active   20h
testing           Active   48s

```

### 删除资源

引用一个资源

- [x] 名称空间 类型             `ns default`
- [x] 名称空间/类型 ....       `ns/default ns/kube-public`

删除非常危险，所有属于名称空间下的资源均被删除。

```bash
# 删除名称空间
		# 引用类型， 名称空间 类型, 名称空间/类型 ....,删除非常危险，所以有属于名称空间下的资源均被删除
# 写法一
kubectl delete namespace develop
# 写法二
kubectl delete ns/develop # 可以使用以上方法创建ns，重新删除
kubectl delete ns/testing ns/prod

# 验证
[root@master ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   20h
kube-node-lease   Active   20h
kube-public       Active   20h
kube-system       Active   20h
```



### 获取资源信息

- [x] 扩展格式
- [x] yaml格式
- [x] json格式：可以使用json过滤器或json模板进行过滤信息

```bash
# 获取所有ns
kubectl get ns
NAME              STATUS   AGE
default           Active   20h
kube-node-lease   Active   20h
kube-public       Active   20h
kube-system       Active   20h


# 获取单个ns信息
										  # 扩展格式
[root@master ~]# kubectl get ns/default -o wide
NAME      STATUS   AGE
default   Active   20h
										  # yaml格式
[root@master ~]# kubectl get ns/default -o yaml
apiVersion: v1          # api版本
kind: Namespace         # 类型是Namespace
metadata:               # ns元数据
  creationTimestamp: "2021-01-14T10:34:46Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:status:
        f:phase: {}
    manager: kube-apiserver
    operation: Update
    time: "2021-01-14T10:34:46Z"
  name: default
  resourceVersion: "200"
  uid: eaa3c369-2d77-49ea-8ead-6e202b032788
spec: 	# 用户期望的状态
  finalizers:
  - kubernetes
status: # 当前状态
  phase: Active
  
										  # json格式, 可以使用json过滤器或json模板进行过滤信息
[root@master ~]# kubectl get ns/default -o json
{
    "apiVersion": "v1",
    "kind": "Namespace",
    "metadata": {
        "creationTimestamp": "2021-01-14T10:34:46Z",
        "managedFields": [
            {
                "apiVersion": "v1",
                "fieldsType": "FieldsV1",
                "fieldsV1": {
                    "f:status": {
                        "f:phase": {}
                    }
                },
                "manager": "kube-apiserver",
                "operation": "Update",
                "time": "2021-01-14T10:34:46Z"
            }
        ],
        "name": "default",
        "resourceVersion": "200",
        "uid": "eaa3c369-2d77-49ea-8ead-6e202b032788"
    },
    "spec": {
        "finalizers": [
            "kubernetes"
        ]
    },
    "status": {
        "phase": "Active"
    }
}
```



k8s每一个资源 5个1级字段, k8s的标准格式，如上ns的yaml文件格式。

名称一样，值有所不同，个别资源可能不一样。 参考

http://blog.mykernel.cn/2021/01/13/Kubernetes%E5%9F%BA%E7%A1%80%E6%A6%82%E5%BF%B5/#%E5%A3%B0%E6%98%8E%E5%BC%8Fapi

```yaml
apiVersion: 
kind: 
metadata:
spec:
status:
```



### 获取资源状态

```bash
kubectl describe ns/default

Name:         default
Labels:       <none>
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.

```



### 创建控制器

可以控制pod运行起来

```bash
# 获取帮助
kubectl create deployment -h
Aliases deploy的别名
Examples 示例
Options 选项
      --allow-missing-template-keys=true: If true, ignore any errors in templates when a field or map key is missing in
the template. Only applies to golang and jsonpath output formats.
      --dry-run='none': # 测试 Must be "none", "server", or "client". If client strategy, only print the object that would be
sent, without sending it. If server strategy, submit server-side request without persisting the resource.
      --field-manager='kubectl-create': Name of the manager used to track field ownership.
      --image=[]: # 镜像 Image names to run.
  -o, --output='': Output format. One of:
json|yaml|name|go-template|go-template-file|template|templatefile|jsonpath|jsonpath-as-json|jsonpath-file.
      --port=-1: The port that this container exposes.
  -r, --replicas=1: Number of replicas to create. Default is 1.
      --save-config=false: # 创建命令保存至 annotation If true, the configuration of current object will be saved in its annotation. Otherwise, the
annotation will be unchanged. This flag is useful when you want to perform kubectl apply on this object in the future.
      --template='': Template string or path to template file to use when -o=go-template, -o=go-template-file. The
template format is golang templates [http://golang.org/pkg/text/template/#pkg-overview].
      --validate=true: If true, use a schema to validate the input before sending it

Usage 语法格式说明 

```

创建控制器

- [x] 控制器属于名称空间，**没有指定ns 属于默认名称空间**
- [x] 控制器会启动一个pod, **pod中只有一个容器**, 容器基于`nginx:1.14-alpine`镜像启动。

```bash
kubectl create deploy ngx-deploy --image=nginx:1.14-alpine
deployment.apps/ngx-deploy created

#验证
							 # all表示所有资源  # 没有指定名称空间表示default名称空间
[root@master ~]# kubectl get all
NAME                              READY   STATUS    RESTARTS   AGE
	# pod # 
pod/ngx-deploy-6b44c98c55-cfvrq   1/1     Running   0          2m50s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
	# service
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   20h

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
	# 1. deploy
deployment.apps/ngx-deploy   1/1     1            1           2m50s

NAME                                    DESIRED   CURRENT   READY   AGE
	# 2. 中间层
replicaset.apps/ngx-deploy-6b44c98c55   1         1         1       2m50s

```

获取控制器启动的pod

```bash
kubectl get pods -o wide

NAME                          READY   STATUS    RESTARTS   AGE    IP           NODE                NOMINATED NODE   READINESS GATES
																  # 容器ip, flannel分配 
ngx-deploy-6b44c98c55-cfvrq   1/1     Running   0          5m5s   10.244.1.2   node01.magedu.com   <none>           <none>
```

直接与pod中的nginx容器通信

```bash
[root@master ~]# curl 10.244.1.2
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

#### 连接pod

```bash
# 获取pod
[root@master ~]# kubectl get pod
NAME                          READY   STATUS    RESTARTS   AGE
ngx-deploy-6b44c98c55-cfvrq   1/1     Running   0          11m

# 连接pod
[root@master ~]# kubectl exec -it ngx-deploy-6b44c98c55-vpkbp -- sh
/ # 

```



#### 删除pod

用户陈述式命令创建的控制器，会指定用户期望的pod 1个。

当我们删除一个pod, 少了就补回来。

```bash

[root@master ~]# kubectl get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP           NODE                NOMINATED NODE   READINESS GATES
ngx-deploy-6b44c98c55-cfvrq   1/1     Running   0          6m38s   10.244.1.2   node01.magedu.com   <none>           <none>
								# 清理pod
[root@master ~]# kubectl delete pod/ngx-deploy-6b44c98c55-cfvrq
pod "ngx-deploy-6b44c98c55-cfvrq" deleted
[root@master ~]# 
[root@master ~]# kubectl get pods -o wide
NAME                          READY   STATUS              RESTARTS   AGE   IP       NODE                NOMINATED NODE   READINESS GATES
									 # 控制器会重建pod
ngx-deploy-6b44c98c55-vpkbp   0/1     ContainerCreating   0          16s   <none>   node02.magedu.com   <none>           <none>
[root@master ~]# kubectl get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE                NOMINATED NODE   READINESS GATES
																 #地址不一样	   # 运行在一个新的节点之上
ngx-deploy-6b44c98c55-vpkbp   1/1     Running   0          23s   10.244.2.2   node02.magedu.com   <none>           <none>

```

现在访问10.244.2.2 

```bash
[root@master ~]# curl 10.244.2.2 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

### 创建service

以上pod重建，ip 变了。客户端怎么知道ip变了？我们要对用户**透明**

要透明，就加中间层`service`



负载均衡的调度

- [x] 调度器有ip,port。哪怕不监听，ipvsadm 需要指定端口，后面哪个服务。
  - [ ] iptables: `dnat -d 10.96.1.7 --dport 80 -j DNAT --to-destination 10.244.2.2:80`
- [x] 被调度的主机有ip, port。

> lb和后端ip/port可以不一致



调度器的规则在每个节点都有



```bash
# 帮助
[root@master ~]# kubectl create service -h
Create a service using specified subcommand.

Aliases:
service, svc

Available Commands: # 以下是几种service类型
  clusterip    #Create a ClusterIP service. 最简单，使用最多的一种类型
  externalname Create an ExternalName service.
  loadbalancer Create a LoadBalancer service.
  nodeport     Create a NodePort service.

Usage:
  kubectl create service [flags] [options]

```

#### clusterip

- [x]  --tcp=5678:8080 本地service服务的端口:被代理的后端pod的端口

```bash
[root@master ~]# kubectl create service clusterip -h
Create a ClusterIP service with the specified name.

Examples:
  # Create a new ClusterIP service named my-cs
  kubectl create service clusterip my-cs --tcp=5678:8080
  
  # Create a new ClusterIP service named my-cs (in headless mode)
  kubectl create service clusterip my-cs --clusterip="None"
Options:
      --allow-missing-template-keys=true: If true, ignore any errors in templates when a field or map key is missing in
the template. Only applies to golang and jsonpath output formats.
      --clusterip='': # 分配service ip Assign your own ClusterIP or set to 'None' for a 'headless' service (no loadbalancing).
      --dry-run='none': # 干跑 Must be "none", "server", or "client". If client strategy, only print the object that would be
sent, without sending it. If server strategy, submit server-side request without persisting the resource.
      --field-manager='kubectl-create': Name of the manager used to track field ownership.
  -o, --output='': Output format. One of:
json|yaml|name|go-template|go-template-file|template|templatefile|jsonpath|jsonpath-as-json|jsonpath-file.
      --save-config=false: If true, the configuration of current object will be saved in its annotation. Otherwise, the
annotation will be unchanged. This flag is useful when you want to perform kubectl apply on this object in the future.
      --tcp=[]: # 本地service服务的端口:被代理的后端pod的端口 Port pairs can be specified as '<port>:<targetPort>'.
      --template='': Template string or path to template file to use when -o=go-template, -o=go-template-file. The
template format is golang templates [http://golang.org/pkg/text/template/#pkg-overview].
      --validate=true: If true, use a schema to validate the input before sending it

Usage:
  kubectl create service clusterip NAME [--tcp=<port>:<targetPort>] [--dry-run=server|client|none] [options]
```



```bash
# 创建
kubectl create service clusterip ngx-svc --tcp=80:80

#验证
[root@master ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP   20h
						 # 地址动态分配,属于10.96.0.0/12
ngx-svc      ClusterIP   10.111.86.0   <none>        80/TCP    11s

# 状态     
[root@master ~]# kubectl describe svc/ngx-svc
Name:              ngx-svc
Namespace:         default
Labels:            app=ngx-svc
Annotations:       <none>
Selector:          app=ngx-svc
Type:              ClusterIP
IP Families:       <none>
IP:                10.100.14.82
IPs:               10.100.14.82
Port:              80-80  80/TCP    # sevice port
TargetPort:        80/TCP           # pod port
Endpoints:         <none>           # 后端pod地址
Session Affinity:  None
Events:            <none>


```

没有指定pod, 现在创建service指定pod

创建与deploy同的service, 会自动关联deploy创建的pod

```bash
# 删除svc
kubectl delete svc/ngx-svc


# 创建与deploy同的service, 会自动关联deploy创建的pod
kubectl create service clusterip ngx-deploy --tcp=80:80

# 状态信息
[root@master ~]# kubectl describe svc/ngx-deploy
Name:              ngx-deploy
Namespace:         default
Labels:            app=ngx-deploy
Annotations:       <none>
Selector:          app=ngx-deploy
Type:              ClusterIP
IP Families:       <none>
IP:                10.108.15.150
IPs:               10.108.15.150
Port:              80-80  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.2.2:80   # pod已经自动关联ip
Session Affinity:  None
Events:            <none>

# iptables规则
[root@master ~]# iptables -S -t nat | grep 10.108.15.150
-A KUBE-SERVICES ! -s 10.244.0.0/16 -d 10.108.15.150/32 -p tcp -m comment --comment "default/ngx-deploy:80-80 cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.108.15.150/32 -p tcp -m comment --comment "default/ngx-deploy:80-80 cluster IP" -m tcp --dport 80 -j KUBE-SVC-DMTVCZI7L444G2GF
```

访问service的ip

```bash
[root@master ~]# curl 10.108.15.150
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>



```

##### 验证pod变动

所有pod变动，会反映到api server

controller会重建pod

service会随之做相应的改变 

```bash
# 删除pod
kubectl delete pod/ngx-deploy-6b44c98c55-vpkbp

# 获取pod
[root@master ~]# kubectl get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE                NOMINATED NODE   READINESS GATES
																 # ip变化
ngx-deploy-6b44c98c55-22j8m   1/1     Running   0          33s   10.244.1.3   node01.magedu.com   <none>           <none>

# svc状态信息
[root@master ~]# kubectl describe svc/ngx-deploy
Name:              ngx-deploy
Namespace:         default
Labels:            app=ngx-deploy
Annotations:       <none>
Selector:          app=ngx-deploy
Type:              ClusterIP
IP Families:       <none>
IP:                10.108.15.150
IPs:               10.108.15.150
Port:              80-80  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.3:80       # pod的ip自动更新
Session Affinity:  None
Events:            <none>

# nginx访问
[root@master ~]# curl 10.108.15.150
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

##### service重建

service ip会变化，所以我们要使用service的名称来引用service.

```bash
[root@master ~]# kubectl get svc/ngx-deploy
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
ngx-deploy   ClusterIP   10.108.15.150   <none>        80/TCP    9m36s
[root@master ~]# kubectl delete svc/ngx-deploy
service "ngx-deploy" deleted

[root@master ~]# kubectl create svc clusterip ngx-deploy --tcp=80:80
service/ngx-deploy created
[root@master ~]# kubectl get svc/ngx-deploy
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
						 # ip变化
ngx-deploy   ClusterIP   10.107.111.213   <none>        80/TCP    3s

# 仍关联至pod
[root@master ~]# kubectl describe svc/ngx-deploy
Name:              ngx-deploy
Namespace:         default
Labels:            app=ngx-deploy
Annotations:       <none>
Selector:          app=ngx-deploy
Type:              ClusterIP
IP Families:       <none>
IP:                10.107.111.213
IPs:               10.107.111.213
Port:              80-80  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.3:80 # pod的ip
Session Affinity:  None
Events:            <none>

```

##### 名称引用service

当service重建后，service的域名和Ip会注册到coredns

```bash
[root@master ~]# curl ngx-deploy
curl: (6) Could not resolve host: ngx-deploy; Unknown error

解析不了，因为默认dns是互联网上的dns
[root@master ~]# cat /etc/resolv.conf
# Generated by NetworkManager
search magedu.com
nameserver 223.6.6.6

而以上的service名称，应该是k8s集群之上默认运行的dns, coredns提供
[root@master ~]# kubectl get pods -n kube-system
NAME                                        READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-k6zwb                     1/1     Running   0          21h
coredns-74ff55c5b-q4qrv                     1/1     Running   0          21h


coredns pod暴露的服务
[root@master ~]# kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
						# dns服务
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   21h

将以上coredns的ip作为dns测试解析k8s域名
[root@master ~]# cat /etc/resolv.conf
# Generated by NetworkManager
search magedu.com
nameserver 10.96.0.10

[root@master ~]# ping ngx-deploy
ping: ngx-deploy: Name or service not known


报错是因为，我们使用相对路径，所以就搜索就加上了magedu.com域名。搜索时我们加上完整路径 
[root@master ~]# ping ngx-deploy.default
ping: ngx-deploy.default: Name or service not known
[root@master ~]# ping ngx-deploy.default.svc.cluster.local.
											# 解析的是svc的ip
PING ngx-deploy.default.svc.cluster.local (10.107.111.213) 56(84) bytes of data.
^C
--- ngx-deploy.default.svc.cluster.local ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2000ms
					  # service名称，名称空间，固定服务， 固定字符串, kubeadm --service-dns-domain指定
[root@master ~]# curl ngx-deploy.default.svc.cluster.local.
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>


# 现在根据以上重建，重建service，之后发现ip改变。但是请求这个域名始终可以访问
curl ngx-deploy.default.svc.cluster.local

```



##### controller扩容或缩容、service负载均衡

默认iptables随机调度

如果使用ipvs，可以指定调度算法

##### 扩容

```bash
									# 用户名/项目名:版本
kubectl create deploy myapp --image=ikubernetes/myapp:v1

# 起来了
[root@master ~]# kubectl get deploy
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
myapp        1/1     1            1           59s
ngx-deploy   1/1     1            1           49m
[root@master ~]# kubectl get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE                NOMINATED NODE   READINESS GATES
myapp-7d4b7b84b-x8bp4         1/1     Running   0          83s   10.244.3.2   node03.magedu.com   <none>           <none>


# 请求pod
[root@master ~]# curl 10.244.3.2
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
[root@master ~]# curl 10.244.3.2/hostname.html
myapp-7d4b7b84b-x8bp4

# service
[root@master ~]# kubectl create service clusterip myapp --tcp=80:80
service/myapp created
[root@master ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   21h
myapp        ClusterIP   10.99.91.102     <none>        80/TCP    6s
ngx-deploy   ClusterIP   10.107.242.140   <none>        80/TCP    6m25s

# 请求svc域名
[root@master ~]# curl myapp.default.svc.cluster.local.
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
[root@master ~]# curl myapp.default.svc.cluster.local./hostname.html
myapp-7d4b7b84b-x8bp4



# deploy完成发布、变更(扩容、缩容)、故障处理。
# deploy扩容、缩容
[root@master ~]# kubectl scale -h
  kubectl scale [--resource-version=version] [--current-replicas=count] --replicas=COUNT (-f FILENAME | TYPE NAME)
#扩容
[root@master ~]# kubectl scale --replicas=3 deployment myapp
deployment.apps/myapp scaled
#验证deploy
[root@master ~]# kubectl get pods -o wide
NAME                          READY   STATUS         RESTARTS   AGE     IP           NODE                NOMINATED NODE   READINESS GATES
																					#自动关联3个节点
myapp-7d4b7b84b-5fj6p         0/1     ErrImagePull   0          36s     10.244.2.3   node02.magedu.com   <none>           <none>
myapp-7d4b7b84b-x8bp4         1/1     Running        0          5m48s   10.244.3.2   node03.magedu.com   <none>           <none>
myapp-7d4b7b84b-z9bh4         1/1     Running        0          35s     10.244.3.3   node03.magedu.com   <none>           <none>
# 验证service
[root@master ~]# kubectl describe svc/myapp
Name:              myapp
Namespace:         default
Labels:            app=myapp
Annotations:       <none>
Selector:          app=myapp
Type:              ClusterIP
IP Families:       <none>
IP:                10.99.91.102
IPs:               10.99.91.102
Port:              80-80  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.2.3:80,10.244.3.2:80,10.244.3.3:80 #自动关联至3个节点

# 再次访问时, service是负载均衡的
[root@master ~]# curl myapp.default.svc.cluster.local/hostname.html
myapp-7d4b7b84b-z9bh4
[root@master ~]# curl myapp.default.svc.cluster.local/hostname.html
myapp-7d4b7b84b-5fj6p
[root@master ~]# curl myapp.default.svc.cluster.local/hostname.html
myapp-7d4b7b84b-5fj6p


```

##### 缩容

```bash
[root@master ~]# kubectl scale --replicas=2 deployment myapp
deployment.apps/myapp scaled

# 控制器保证目标状态只有2个
[root@master ~]# kubectl get pods -o wide
NAME                          READY   STATUS        RESTARTS   AGE    IP           NODE                NOMINATED NODE   READINESS GATES
myapp-7d4b7b84b-5fj6p         1/1     Running       0          5m9s   10.244.2.3   node02.magedu.com   <none>           <none>
myapp-7d4b7b84b-x8bp4         1/1     Running       0          10m    10.244.3.2   node03.magedu.com   <none>           <none>
myapp-7d4b7b84b-z9bh4         0/1     Terminating   0          5m8s   <none>       node03.magedu.com   <none>           <none>
ngx-deploy-6b44c98c55-22j8m   1/1     Running       0          28m    10.244.1.3   node01.magedu.com   <none>           <none>

# service关联2个
[root@master ~]# kubectl describe svc/myapp
Name:              myapp
Namespace:         default
Labels:            app=myapp
Annotations:       <none>
Selector:          app=myapp
Type:              ClusterIP
IP Families:       <none>
IP:                10.99.91.102
IPs:               10.99.91.102
Port:              80-80  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.2.3:80,10.244.3.2:80 # service关联2个
Session Affinity:  None
Events:            <none>

# 负载均衡至2个
[root@master ~]# curl myapp.default.svc.cluster.local/hostname.html
myapp-7d4b7b84b-x8bp4
[root@master ~]# curl myapp.default.svc.cluster.local/hostname.html
myapp-7d4b7b84b-5fj6p
[root@master ~]# curl myapp.default.svc.cluster.local/hostname.html
myapp-7d4b7b84b-5fj6p

```

#### node port

在创建service时指定nodeport,  节点端口类型

即在每个节点上创建iptables/ipvs规则





```bash
# 先删除以上的deploy/service
[root@master ~]# kubectl delete deploy/myapp
[root@master ~]# kubectl delete svc/myapp
```



deploy

```bash
kubectl create deploy myapp --image=ikubernetes/myapp:v1
```

service

```bash
kubectl create service nodeport -h

								# 与deploy名称一致
kubectl create service nodeport myapp --tcp=80:80

# 状态
[root@master ~]# kubectl describe svc/myapp
Name:                     myapp
Namespace:                default
Labels:                   app=myapp
Annotations:              <none>
Selector:                 app=myapp
Type:                     NodePort
IP Families:              <none>
IP:                       10.106.127.86
IPs:                      10.106.127.86
Port:                     80-80  80/TCP      # service端口
TargetPort:               80/TCP             # pod端口
NodePort:                 80-80  32224/TCP   # 宿主机的端口
Endpoints:                
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

# 信息
[root@master ~]# kubectl get svc/myapp
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
												 # service端口:宿主机的端口
myapp   NodePort   10.106.127.86   <none>        80:32224/TCP   42s


```

请求集群任意主机的端口：`32224`

![image-20210115160950753](http://myapp.img.mykernel.cn/image-20210115160950753.png)

![image-20210115161015848](http://myapp.img.mykernel.cn/image-20210115161015848.png)

![image-20210115161025266](http://myapp.img.mykernel.cn/image-20210115161025266.png)

![image-20210115161040619](http://myapp.img.mykernel.cn/image-20210115161040619.png)

iptables/ipvs规则， 宿主机端口会转发至service, 之后到pod

```bash
[root@master ~]# iptables  -t nat  -vnL KUBE-NODEPORTS 
Chain KUBE-NODEPORTS (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    2   104 KUBE-MARK-MASQ  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/myapp:80-80 */ tcp dpt:32224
    2   104 KUBE-SVC-IJFIQBSXQ4XPPRV7  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/myapp:80-80 */ tcp dpt:32224
```

规则的变动，由宿主机的kube-proxy完成 

```bash
[root@master ~]# ps -ef | grep kube-proxy
root     29511 29490  0 Jan14 ?        00:00:12 /usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/config.conf --hostname-override=master.magedu.com
```



现在就可以使用陈述式命令启动一个tomcat, nginx, 只是对应的配置需要存储卷



## 围绕pod的资源

![image-20210115161828863](http://myapp.img.mykernel.cn/image-20210115161828863.png)

上面是控制器

中间是service

下面是volume是容器跨节点必须的组件



# restful api操作

api server可以接受 kubectl, dashboard, api操作

api详细见: http://blog.mykernel.cn/2021/01/13/Kubernetes%E5%9F%BA%E7%A1%80%E6%A6%82%E5%BF%B5/#k8s-api



## 获取pod

获取上面链接中api支持的所有版本

```bash
kubectl api-versions
# v1 pod属于v1


# 获取pod
curl http://localhost:6443/v1
curl: (7) Failed connect to localhost:6443; Connection refused
由于api工作在https协议，需要认证

# 可以架设一个代理， 参考：https://kubernetes.io/zh/docs/tasks/access-application-cluster/access-cluster/
kubectl proxy --port=8080

# 获取v1版本
[root@node01 ~]# curl http://localhost:8080/v1
{
  "paths": [
    "/apis",
    "/apis/",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1",
    "/apis/apiextensions.k8s.io/v1beta1",
    "/healthz",
    "/healthz/etcd",
    "/healthz/log",
    "/healthz/ping",
    "/healthz/poststarthook/crd-informer-synced",
    "/healthz/poststarthook/generic-apiserver-start-informers",
    "/healthz/poststarthook/priority-and-fairness-config-consumer",
    "/healthz/poststarthook/priority-and-fairness-filter",
    "/healthz/poststarthook/start-apiextensions-controllers",
    "/healthz/poststarthook/start-apiextensions-informers",
    "/livez",
    "/livez/etcd",
    "/livez/log",
    "/livez/ping",
    "/livez/poststarthook/crd-informer-synced",
    "/livez/poststarthook/generic-apiserver-start-informers",
    "/livez/poststarthook/priority-and-fairness-config-consumer",
    "/livez/poststarthook/priority-and-fairness-filter",
    "/livez/poststarthook/start-apiextensions-controllers",
    "/livez/poststarthook/start-apiextensions-informers",
    "/metrics",
    "/openapi/v2",
    "/readyz",
    "/readyz/etcd",
    "/readyz/informer-sync",
    "/readyz/log",
    "/readyz/ping",
    "/readyz/poststarthook/crd-informer-synced",
    "/readyz/poststarthook/generic-apiserver-start-informers",
    "/readyz/poststarthook/priority-and-fairness-config-consumer",
    "/readyz/poststarthook/priority-and-fairness-filter",
    "/readyz/poststarthook/start-apiextensions-controllers",
    "/readyz/poststarthook/start-apiextensions-informers",
    "/readyz/shutdown",
    "/version"
  ]
}



# 获取default名称空间下的pod
								   # 集群下是名称空间    # pod属于名称空间
curl http://localhost:8080/api/v1/namespaces/default/pods

```



