---
title: Linux常用命令
date: 2022-01-18 11:30:46
categories: Linux
toc: true
---

显示系统内核版本与系统架构等信息：

```
uname -a
```

ubuntu 查看版本：

```
cat /etc/issue
```

<!--more-->
centos 查看版本：

```
cat /etc/centos-release
```

Linux 查看 cpu 相关信息，包括型号、主频、内核信息等:

```
cat /proc/cpuinfo
```

上传下载文件：

```
安装：apt install lrzsz
上传：rz -E
下载：se start.sh
```

查找文件/文件夹：

```
find init.d*  *通配符
```

<!--more-->

安装 vim：

```
执行：apt-get update
再次执行：apt-get install vim
```

显示当前目录：

```
pwd
```

删除文件/文件夹：

```
rm name
```

删除当前目录下所有文件：

```
rm -rf *
```

递归删除目录及目录下所有文件：

```
rm -rf /data
```

将 /home/html/ 这个目录下所有文件和文件夹打包为当前目录下的 html.zip

```
zip -q -r html.zip /home/html
```

把一个文件 abc.txt 和一个目录 dir1 压缩成为 yasuo.zip：

```
zip -q -r yasuo.zip abc.txt dir1
```

把/home 目录下面的 mydata.zip 解压到 mydatabak 目录里面：

```
unzip mydata.zip -d mydatabak
```

复制 dir1 目录到 dir2 目录：

```
cp -R dir1 dir2/
```

ss 获取 socket 统计信息，它显示的内容和 netstat 类似

```
ss -tlpn
-p, --processes 显示监听端口的进程(Ubuntu 上需要 sudo)
-l, --listening 只显示处于监听状态的端口
-t, --tcp 显示 TCP 协议的 sockets
```

查看进程：

```
ps -ef|grep nginx  在上面带（-e）参数的ps命令输出中，选择所有进程，（-a）选择除会话前导外的所有进程，并使用（-f）参数输出完整格式列表（即 -eaf）
```

杀死进程 2072：

```
kill -QUIT 2072
```

查看时间：

```
date -R
```

修改时区：

```
timedatectl set-timezone Asia/Shanghai
timedatectl set-time 2021-05-18
timedatectl set-time 9:30
```

pidof 命令用于查询某个指定服务进程的 PID 号码值：

```
pidof [参数] 服务名称
pidof nginx
```

查看 80 端口占用情况

```
lsof -i:80
```

关闭占用程序

```
kill -9 pid号
```

终止某个指定名称的服务所对应的全部进程:

```
killall [参数] 服务名称
killall nginx
```

让 bash 将一个字串作为完整的命令来执行

```
sh -c
如：docker exec -it blog sh -c 'rm -rf /111/*'
```

**[Linux 基础](https://www.linuxprobe.com/)**  
**[Linux 命令大全](https://www.linuxcool.com/)**
