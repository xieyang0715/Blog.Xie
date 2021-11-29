---
title: bash获取命令行参数
date: 2020-11-20 07:48:46
tags: 日常挖掘
toc: true
---



根据编译时传递参数, 解析参数的脚本, 摘录如下，方便日后查询

<!--more-->

# 1. nginx

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-11-20
#FileName：             kvm.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************

#变量默认值
opt=
help=no

for option
do
    opt="$opt `echo $option | sed -e \"s/\(--[^=]*=\)\(.* .*\)/\12/\"`"
    case "$option" in
        -*=*) value=`echo "$option" | sed -e s/[-_a-zA-Z0-9]*=//` ;;
           *) value="" ;;
    esac
    case "$option" in
        --help)                          help=yes                   ;;  
        # 选项
        --with-zlib-asm=*)               ZLIB_ASM="$value"          ;;
        *)
            echo "$0: error: invalid option \"$option\""
            exit 1
        ;;
    esac
done

if [ "$help" == "yes" ]; then
cat <<EOF
# 帮助
kvm.sh

EOF
fi
```

# 2. haproxy

```bash
USAGE="Usage: ${0##*/} [-f] [-b branch] [-d date] [-o oldver] [-n newver]"
FORCE=
OUTPUT=
BRANCH=
HTML=
DATE=
YEAR=
OLD=
NEW=
DIR=

die() {
    [ "$#" -eq 0 ] || echo "$*" >&2
    exit 1
}

err() {
    echo "$*" >&2
}

quit() {
    [ "$#" -eq 0 ] || echo "$*"
    exit 0
}

while [ -n "$1" -a -z "${1##-*}" ]; do
    case "$1" in
        -d)        DATE="$2"      ; shift 2 ;;
        -b)        BRANCH="$2"    ; shift 2 ;;
        -f)        FORCE=1        ; shift   ;;
        -o)        OLD="$2"       ; shift 2 ;;
        -n)        NEW="$2"       ; shift 2 ;;
        -h|--help) quit "$USAGE" ;;
        *)         die  "$USAGE" ;;
    esac
done

if [ $# -gt 0 ]; then
    die "$USAGE"
fi

```



# 3. getopts

仅支持短选项 

```bash
# 默认配置
USER=root
HOST=localhost
PORT=3306

# 帮助信息
function Usage(){
cat <<EOF
  command [-H] [-p] [-u] -d <需要备份的数据库> [-t <需要备份的表>] [-h 帮助]
        -H 127.0.0.1, 主机, option
        -p 3306, 端口, option
        -u root, 用户, option
        -d test, 库
        -t t1, 表名
        -h help
EOF
  exit -1
}

# 仅支持短选项 , -d 2 -t 1
while getopts "d:t:H:p:u:h" opt
do
   case $opt in
        d)
        DBNAME=${OPTARG}
        ;;
        t)
        TABLENAME=${OPTARG}
        ;;
        H)
        HOST=${OPTARG:-$HOST}
        ;;
        p)
        PORT=${OPTARG:-$PORT}
        ;;
        u)
        USER=${OPTARG:-$USER}
        ;;
        ?)
        Usage
        ;;
   esac
done
```



# 4. 自行组合

```bash
# 默认配置
DISK_PATH=/var/lib/libvirt/images/xxx.qcow2
DISK_SIZE=10G
VIRT_TYPE=kvm
NAME=vm1
RAM=512
VCPUS=1
CDROM=/usr/local/src/xxx.iso
BRIDGE1=br0
BRIDGE2=

##############

trap 'echo  -e "\n退出前操作\nerror line: $LINENO,error cmd: $BASH_COMMAND\n ${USAGE}"; rm -fr $DISK_PATH; virsh shutdown $NAME; virsh undefine $NAME' ERR

USAGE="
使用帮助:
		${0##*/} -d 磁盘路径 -s 磁盘大小 -t 类型 -n 虚拟机名称 -r 内存大小MB  -v CPU核心数 -c ISO文件路径 -b 网卡名 [-f 网卡名]
		使用示例: ${0##*/} -d /var/lib/libvirt/images/xxx.qcow2 -s 10G -t kvm|qemu -n vm1 -r 512  -v 1 -c /usr/local/src/ -b br0 -f br1"
die() {
    [ "$#" -eq 0 ] || echo "$*" >&2
    exit 1
}
err() {
    echo "$*" >&2
}


while getopts "d:s:t:n:r:v:c:b:f:" opt
do
   case $opt in
        d)
        DISK_PATH=${OPTARG}
        ;;
        s)
        DISK_SIZE=${OPTARG}
        ;;
        t)
        VIRT_TYPE=${OPTARG}
        ;;
        n)
        NAME=${OPTARG}
        ;;
        r)
        RAM=${OPTARG}
        ;;
        v)
        VCPUS=${OPTARG}
        ;;
        c)
        CDROM=${OPTARG}
        ;;
        b)
        BRIDGE1=${OPTARG}
        ;;
        f)
        BRIDGE2=${OPTARG}
        ;;
        ?)
		die "$USAGE"	
        ;;
   esac
done
```

