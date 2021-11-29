---
title: nerdctl类似docker管理containerd
date: 2021-09-01 14:29:24
tags:
- docker
- kubernetes
---



# 安装containerd

docker源就带有containerd, 故参考docker源安装: `https://developer.aliyun.com/mirror/docker-ce`

- centos

```bash
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3
sudo sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
# Step 4: 更新并安装Docker-CE
sudo yum makecache fast
```

- ubuntu

```bash
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装Docker-CE
sudo apt-get -y update
```



安装containerd

- centos

  ```bash
  yum install -y containerd
  ```

- ubuntu

  ```bash
  apt install -y containerd
  ```

  

# 配置containerd

参考: `https://github.com/containerd/containerd/blob/main/docs/cri/config.md`

仅需要修改`sandbox_image` 为镜像地址, 因为新版本的kubernetes, 不支持docker了，而kubelet配置的pause是dockershim支持的方式。所以新版本直接把Pause配置在containerd上



# 使用nerdctl管理containerd

https://github.com/containerd/nerdctl/

帮助

```bash
[root@ip-10-0-0-91 ~]# nerdctl -h
```

查看namespace

```bash
[root@ip-10-0-0-91 ~]# nerdctl namespace ls
NAME       CONTAINERS    IMAGES    VOLUMES
default    0             1         0
k8s.io     17            25        0
```

查看镜像、容器、...

```bash
[root@ip-10-0-0-91 ~]# nerdctl -n k8s.io images
[root@ip-10-0-0-91 ~]# nerdctl -n k8s.io ps
```

# 添加别名

`~/.bashrc`

```bash
alias docker-compose='nerdctl compose'
alias docker='nerdctl'
```



