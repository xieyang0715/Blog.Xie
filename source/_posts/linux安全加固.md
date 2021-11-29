---
title: linux安全加固
date: 2020-11-30 06:20:18
tags: 马哥企业教练
toc: true
---

按照等保三级进行检测并加固。需要的同学可以收集一下，学习shell的同学也可以看看

```bash
1. 账号：ssh：禁用root远程，添加新的root权限账号。
2. 登入失败处理：pam模块，root密码3次尝试锁定15分钟
3. history保存行数及命令时间
4. 周期更新密码
```

# 账号

openssh

```bash
PasswordAuthentication yes       # 可以使用密码
PubkeyAuthentication yes         # 可以使用公钥
PermitRootLogin prohibit-password # 禁止root登陆
```

新root账号

```bash
useradd weizhixiu
echo 'weizhixiu:xxxx' | chpasswd
groupmod -a chpasswd wheel # wheel组内用户有root权限
```



# 登入失败处理

限制登入失败三次，普通账号锁定5分钟，root账号锁定5分钟

```bash
sed -i '/^#%PAM-1.0/a\auth required pam_tally2.so deny=3 unlock_time=300 even_deny_root root_unlock_time=300' /etc/pam.d/sshd
```



#  shell命令操作

```bash
#  HISTSIZE has been set to 10000
sed -i 's/^HISTSIZE=.*$/HISTSIZE=10000/g' /etc/profile 
# HISTTIMEFORMAT has been set to "Number-Time-User-Command"
echo 'export HISTTIMEFORMAT="%F %T `whoami` "' >> /etc/profile 
echo TMOUT=300 >> /etc/profile
```

> 在xshell连接ssh server时会自动发送openssh的心跳, 所以配置openssh-server超时没有用。

```bash
# openssh配置超时，无效
TCPKeepAlive no
ClientAliveInterval 1m
ClientAliveCountMax 1
```

