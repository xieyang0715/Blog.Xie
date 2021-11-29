---
title: salt基本使用
date: 2021-07-12 09:57:21
tags:
- salt
---

# 介绍

配置管理、执行命令

# 安装

```bash
pip install salt
```



# 配置并启动salt

## master

```bash
root@78190ee71646:~# cat /etc/salt/master 
file_roots:
  base:
  - /tmp/minion/

root@78190ee71646:~# salt-master -l debug

```

## minion

```bash
root@20c2055b5199:~# cat /etc/salt/minion
master: 172.17.0.6
id: min1

root@20c2055b5199:~# salt-minion -l debug

```

# master接受salt

```bash
salt-key -L
```

```bash
 salt-key -a min1
```

# master操作salt

## ping

```bash
salt '*' test.ping # 所有主机
salt 'min*' test.ping # 匹配主机
salt 'min1' test.ping # 指定主机
```

## cmd

```bash
salt 'min*' cmd.run 'pwd'
```

powershell 脚本

```bash
salt 'min*' cmd.script salt://powershell/scriptname.ps1 shell=powershell
```

## 文件 file.

### 新建  file.touch

```bash
salt 'min*' file.touch '/tmp/salt_files/new.txt'
```

### 写入 file.write

```bash
salt 'min*' file.write '/tmp/salt_file' 'SaltSTack demo'
```

## 目标主机信息 grains.items

```bash
salt 'min1' grains.items

```

## 拷贝文件 cp.get_file

```bash
salt 'min1' cp.get_file 'salt://salt.txt' '/tmp/salt_files/salt.txt'
```

> salt://salt.txt 文件就是master配置base目录就是基础目录

## 分发文件

```bash
salt 'min1' cp.file 源路径 目标路径
```

