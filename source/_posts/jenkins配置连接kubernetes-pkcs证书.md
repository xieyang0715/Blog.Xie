---
title: "jenkins配置连接kubernetes pkcs证书"
date: 2021-09-16 13:19:03
tags:
- jenkins
- kubernetes
---



# 前言

在jenkins的传统项目中，需要连接k8s去更新镜像，通常需要一个kubernetes插件，并配置k8s的证书和地址，和k8s密钥

以下就通过kubeconfig文件来快速生成pcks文件



<!--more-->
# jenkins配置

## 凭据

![image-20210916132146265](http://myapp.img.mykernel.cn/image-20210916132146265.png)

## 项目中引用

构建环境中，可以配置kubectl的配置

![image-20210916132303716](http://myapp.img.mykernel.cn/image-20210916132303716.png)

# 生成pkcs文件

如果我们假设有了一个~/.kube/config文件

```bash
CONFIG=".kube/config"
echo -n $(cat $CONFIG | grep certificate-authority-data | cut -d: -f2) | base64 -d > my-ca-cert.crt
echo -n $(cat $CONFIG | grep client-certificate-data | cut -d: -f2) | base64 -d > my-client.crt
echo -n $(cat $CONFIG | grep client-key-data | cut -d: -f2) | base64 -d > my-client.key
openssl pkcs12 -export -out newokcs12-for-jenkins-connection.pfx -inkey my-client.key -in  my-client.crt -certfile my-ca-cert.crt 
```

