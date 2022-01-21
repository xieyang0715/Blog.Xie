---
title: ssh免密登录
date: 2022-01-21 14:30:46
categories: ssh
toc: true
---

A主机通过ssh免密访问B主机  

A主机生成rsa rsa.pub

```
cd ~/.ssh/
ssh-keygen -t rsa（不用输入密码）
```

在B上的命令

```
touch /root/.ssh/authorized_keys (如果已经存在这个文件, 跳过这条)
cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
(将公钥rsa.pub的内容追加到authorized_keys 中)
```