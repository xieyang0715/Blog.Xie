---
title: 使用nvm管理node版本
date: 2021-11-05 15:57:14
tags:
- nodejs
photos:
- http://myapp.img.mykernel.cn/nvm.png
---



# 前言

博客的hexo是javascript编写的，需要node来执行



华为的nodejs镜像: https://mirrors.huaweicloud.com/home



<!--more-->
# nvm 镜像

## 安装nvm

https://github.com/nvm-sh/nvm#troubleshooting-on-linux

```bash
export NVM_DIR="$HOME/.nvm" && (   git clone git@github.com:nvm-sh/nvm.git "$NVM_DIR";   cd "$NVM_DIR";   git checkout `git describe --abbrev=0 --tags --match "v[0-9]*" $(git rev-list --tags --max-count=1)`; ) && \. "$NVM_DIR/nvm.sh"
```

> 注意走: git@, 克隆快，需要主机的pub放在github上

## 配置nvm登陆终端加载

`~/.bashrc`

```bash
export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" # This loads nvm
```

## 配置华为云镜像

`~/.bashrc`

```bash
export NVM_IOJS_ORG_MIRROR=https://repo.huaweicloud.com/iojs/
```

## 使用nvm

> 参考: https://www.linode.com/docs/guides/how-to-install-use-node-version-manager-nvm/

### 查看安装的node

```bash
nvm ls
```

### 查看所有可安装的node版本

```bash
nvm ls-remote
```

###  安装版本

```bash
nvm install node
```

> 名称: node
>
> 版本: 主版本 14 或次版本 14.15

### 切换指定版本

```bash
# 查看版本
nvm ls

# 切换
nvm use node
```

> 名称: node
>
> 版本: 主版本 14 或次版本 14.15

### 别名

```bash
[root@024ca694084b apps]# nvm alias maintainer v14
maintainer -> v14 (-> v14.18.1)

```

```bash
[root@024ca694084b apps]# nvm alias maintainer v14
maintainer -> v14 (-> v14.18.1)
[root@024ca694084b apps]# nvm ls
maintainer -> v14 (-> v14.18.1)
```

