---
title: jenkins自动构建博客
date: 2022-01-18 15:35:46
categories: jenkins
toc: true
---

&emsp;&emsp;  
说明：用户提交代码通过代码库 hook 到 Jenkins 容器自动构建代码，将 Jenkins 构建的文件打包到部署容器，然后通过 ssh 免密登录到服务器（sftp 打包和 ssh 登录通过放在 jenkins 容器中的脚本执行）。这里只有一个服务器，只是将 Jenkins、部署博客以容器的方式部署，也可以部署到不同的服务器上。  
开始的版本我是采用 jenkins 上的 Publish Over SSH 来打包发布的，在 2022 年 1 月由于发现了[CSRF](https://www.jenkins.io/security/advisory/2022-01-12/#SECURITY-2290)漏洞便不再支持使用了。

```
Publish Over SSH 1.22
Password stored in plain text
Path traversal vulnerability
CSRF vulnerability and missing permission checks
Stored XSS vulnerability
```

<!--more-->

架构图：  
![架构图](/images/blog/blogdepprocess.png)

# jenkins、github 建立连接，以便 jenkins 拉取代码

![](/images/blog/jenkinsgenerssh.png)

## 本地生成密钥

```
ssh-keygen -t rsa

执行以生成公钥私钥
公钥：id_rsa
私钥：id_rsa.pub

windows在 C:\Users\Administrator\.ssh
linxu一般在当前执行目录，我一般在/root/.ssh
```

## jenkins 添加 ssh 代码访问凭据

路径：系统管理/凭据管理/全局/添加凭据
这里以 ssh 的方式，添加 SSH Username with private key 凭据  
![](/images/blog/jenkinsaddsshprivatekey.png)

## github 添加公钥

路径：头像/设置/SSH and GPG keys  
将生成的 ssh 的公钥添加进去

# 添加 github hook 自动触发 jenkins 构建代码

## github 添加 hook 触发 token

路径：头像/Settings/Developer settings/New personal access token  
生成 token 后请直接复制，因为这个只出现一次

## jenkins 添加 Secret text

路径：系统管理/凭据管理/全局/添加凭据  
弹出的页面中，类型选择 Secret text，Secret 填入前面在 GitHub 上生成的 Personal access tokens，描述随便写一些描述信息  
然后在 jenkins 项目的 Use secret text(s) or file(s)中选择该绑定凭证
