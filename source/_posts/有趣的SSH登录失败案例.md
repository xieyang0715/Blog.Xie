---
title: 有趣的SSH登录失败案例
date: 2020-09-27 08:55:28
tags:  马哥企业教练
toc: true
---

注：根据视频内容，后期做的一个课件。在此做个记录分享给大家。



之前自己遇到过一个案例，就是ssh主机不能成功登录进去，自己也是尝试了很多方法，最后总结出的一个相关的解决思路，于是分享给大家。

<!--more-->

## 1、问题背景

使用xshell工具无论如何都不能登录到主机中，密码是正确的，但是一直提示不对，从而登录不进去。但是通过虚拟机的控制台管理终端倒是可以登录系统内部。

## 2、问题排查

### 2.1、确保密码正确性

生产中密码是较为复杂的，建议重新修改为简单的密码尝试。  可以通过控制台管理终端登录到系统内部。此时就进去这个系统内部然后修改密码。修改密码之后尝试重新登录。

如果还是登录失败，那看来密码是其他方面的原因，此时尝试着查看ssh的日志，看看有什么打印的。

SSH的日志文件是哪个呢？

### 2.2、排除网络干扰因素

关于网络这块，大家立马就能想到是不是防火墙导致的原因。防火墙大致有`firewalld、selinux、iptables`这些。很明显其实并不是这样子，因为如果是防火墙的原因，那么导致的结果就是ssh的时候直接就被拒绝了。

 现在这台电脑是没有防火墙的， 可以很确信的告诉大家，但是现在 如果 加上一条防火墙规则，大家再来看看效果。

`iptables -I INPUT -p tcp  --dport 22 -j REJECT`

可以看到直接报错的就是Connection refused的错误。现在清空一下`iptables`规则。



当然，这里提个醒，云服务器都带有安全组，这个也是需要对特定端口开放才可以。



###  2.3、检查配置文件

检查配置文件主要是一下几点。

#### 2.3.1、检查主配置文件

找一台可以正常SSH的电脑，然后对比一下主配置文件`/etc/ssh/sshd_config`。

发现没有什么异常，配置文件几乎是一样的。

#### 2.3.2、检查是否存在白名单

比如配置文件可能存在`AllowUsers    root@192.168.1.1`，那么这样的就表示只允许特定的主机从特定的地址进行访问，其他地址则不能访问了。

当然很明显 这里是没有这些配置的， 可以找个其他的机器来测试一下给大家看看效果。

比如这样的配置，大家可以来看看

```bash
AllowUsers root@x.x.x.x
DenyUsers All
```

那么现在  测试一下。注意重启sshd服务。

那么从结果上可以看出非常符合  这台有问题的机器的一个现象， 可以ssh连接这台机器，但是 就是登录上去，一直让 输入密码，然而 的密码是没有错误的。

现象是一样的，但是出问题的地方很可能不一样，因为很明显 这个有问题的机器上并没有这些配置。

那么除此之外还有其他设置白名单的地方吗？

>  **其他白名单**

不知道大家是否了解过`/etc`下面的`hosts.allow、 hosts.deny`

在`/etc/hosts.allow`文件添加一些允许访问的地址

```bash
sshd:192.168.50.132
```

还需要在`/etc/hosts.deny`文件添加所有禁止访问

```bash
sshd:All
```

也不用重启sshd，  再来看看效果

### 2.4、允许空密码登录

通过上面检查  发现一直报密码错误的问题。所以  干脆直接把root密码清空尝试一下。为什么尝试使用空密码登录呢？主要就是为了判断系统的ssh验证过程。如果 现在使用空密码登录 看看会报什么样的错误。  不妨试一试。

允许空密码登录很简单，修改一下主配置文件。

```bash
PermitEmptyPasswords no
```

然后清空root用户的密码。

```bash
passwd -d root
```

或者直接在/etc/shadow文件中，把root用户的密码这一列删除掉。

重启sshd服务，然后  再来尝试登录一下。

本机以及外部测试依然提示密码不对， 现在都没有密码了怎么还是提示密码错误类问题，报的错是不相关的。

那么在这里，遇到类似的觉得莫名其妙的问题，那说明这个问题是跟其他地方有关系，所以  应该聚焦到ssh登录过程这一点。而不再是尝试修改密码啊，查看防火墙啊等等相关设置。



对于空密码来说提示密码错误说明实际上并没有达到密码验证的阶段，因为 已经拿着 的密码去向服务端进行验证了可还是直接报密码错误，说明服务端压根没有进行验证就直接给 报了错误，但是一旦验证了 的密码相信肯定是没有问题的。所以  要把问题聚焦到ssh的登录过程，研究一下为什么不验证 的密码呢？



ssh的一个登录过程，

PAM，全英文单词比较长，他是一个验证模块，简单来说，就是提供了一组身份验证，密码验证的统一抽象接口，开发程序员可以使用这些API接口来实现与安全性相关的功能，例如：

- 用户验证
- 数据加密
- LDAP账户集成



那最简单的解决方法是什么呢？就是直接把PAM验证这块的模块功能给去掉。

### 2.4、取消pam登录模块

怎么去掉pam登录模块呢？很简单，直接修改ssh的配置文件，然后把这个参数改为no

```bash
UsePAM no
```

这个参数默认就是打开的，并且是yes，也就是使用pam登录模块。改为no

然后重启ssh服务

发现这个xshell工具直接就可以连接到系统上面去了。

不过禁用pam登录模块是不建议的。所以  还是从pam模块本身来解决问题。所以 应该查看一下ssh的pam模块，来看看配置文件/etc/pam.d/sshd文件，看看里面的内容。

这个文件不应该是空的，既然如此，看来只能从其他电脑里面拷贝一份这个文件的内容，放在pam模块得目录下才行，咱 来试一试。

## 3、问题解决

### 3.1、解决pam模块问题

pam模块在`/etc/pam.d`目录下，这个目录下有好几种不同的pam模块，而对于 本节课来讲， 着重关注的是ssh的相关模块，ssh的模块配置文件是sshd，  来看看里面的内容。

```bash
#%PAM-1.0
auth	   required	pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```

这里面有四大部分，  来总结一下：

- auth验证模块
- account账户管理模块
- session会话管理模块
- password密码管理模块
- 

第一个auth，是认证的意思，这种类别主要是用来检验用户的身份，这种类别通常是需要使用密码来校验的。所以后续接的模块用来校验用户的身份。

第二个account模块，这个模块大部分用于进行授权的，这种类别主要校验用户是否具有正确的权限。举个例子，比如你持有正确的密码来登录系统，可是你这个密码过期了，还能登录吗？肯定是不能的

第三个password模块，这个模块主要就是用来修改密码的

第四个session模块，这是会话的意思，session管理的就是用户在这次登录期间，PAM所给予的环境设置。这个类别通常是用于记录用户登录与注销时的信息，比如，如果你尝尝使用su或者sudo命令的话，那么应该在`/var/log/secure`日志里面发现很多关于PAM的说明，记载的数据大致就是`session open、session close`的信息。





即PAM检验的优先级高于PasswordAuthentication（密码验证）
