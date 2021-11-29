---
title: linux基础入门大纲
date: 2020-11-03 07:01:22
tags: 生产小经验
---

马哥2015基础部分
<!--more-->


```bash

https://blog.51cto.com/sonlich/1951866
    帮助：type, enable, man, makewhatis, info 
        {}, [], |, <> 在帮助中代表的含义？
    终端快捷键：ctrl + a, ctrl + e, ctrl + u, ctrl + k
    less/vim/more快键键：ctrl +u, ctrl + d, ctrl + b, ctrl + f, gg, G, #G, q,    /string, ?string, N, n

https://blog.51cto.com/sonlich/1951918
    文本内容类型：
        ASCII
        directory
        symbolic
        ELF
        block
        character
        socket
        sticky
    文件内容类型显示: file
    文件的元数据查看和修改：stat,touch
    时间相关的命令：date, hwclock, cal
    开机、关机: halt,poweroff,reboot和shutdown
    文件查看工具: cat,tac,more,less，tail,head
    文本处理命令,wc,cut,sort,uniq，tr
    文本编辑命令，nano
    字符串的显示命令,echo，printf
    目录管理命令：pwd,cd,ls,mkdir,rmdir,mv,cp， install, tree

https://blog.51cto.com/sonlich/1951999
    bash的引用：命令引用、变量引用
        命令引用：``, $()
        
        变量引用：
            弱："$PATH","${PATH}"
            强：'$PATH'

    bash命令历史：history, HISTCONTROL={ignorespaces|ignoreboth|ignoredups}， HISTSIZE， HISTFILESIZE ,HISTTIMEFORMAT 
        调用命令历史：!#, !!, !string, !$或ecs+., ctrl + r搜索

    bash中的通配符： man 7 glob
        *
        ?
        []
            [pig]
            [a-z], [a-Z]
            [A-Z]
            [0-9]
            [^]
            [^A-Z0-9]
            ls p[^A-Z0-9]*.conf

            字符集：
                [:digit:] 所有数字

                [:lower:] 所有小宝字母

                [:upper:] 所有大写字母

                [:alpha:] 所有字母

                [:alnum:] 字母和数字

                [:space:] 所有空白

                [:punct:] 所有特殊符号


                [[:space:]] 任意单个空白

                [[:alpha:]] 任意单个字母，相当于 [a-z]

                [[:digit:]] 任意单个数字，相当于 [0-9]

    命令或路径补全
        tab
    命令行的展开
        ~
        ~username
        {1..9}
        {，，，} 承载一个以逗号分隔的列表，并以乘法结合律的规则分配。
    I/O重定向
        >, >>
        set -C 禁止将内容覆盖输出至已有文件，仅对当前shell有效. 强制覆盖输出, COMAND >| POS
        set +C 允许将内容覆盖输出至已有文件，仅对当前shell有效
        command 2> position , command 2>> pos
         
        标准输出和标准错误输出分别定向至不同的文件： ls /root > /tmp/var.out 2> /tmp/var.err
        标准输出和标准错误输出分别定向至相同的文件：ls /root > /tmp/var.out 2> /tmp/var.out; ls /root > /tmp/var.out 2>&1; ls /root &> /tmp/var.out

        <, <<

            cat, tr命令应用

    bash中的管道
         COMMAND1 | COMMAND2 | COMMAND3 | ... ...

    命令的别名
        alias

    bash的快捷键
        ^c  强制取消命令执行

        ^d  正常终止命令

        ^uk 

        ^ae 行首或行尾

        ^l 清屏

https://blog.51cto.com/sonlich/1952071
    用户名和密码保存的文件位置，文件也被称为"配置文件"
        /etc/passwd, /etc/group, /etc/shadow, /etc/gshadow文件格式
    

       /etc/passwd关联的命令
            useradd,userdel,usermod,chfn,chsh,finger,id,su命令
        /etc/group
            groupadd,groupdel,groupmod,gpasswd命令
        /etc/shadow, /etc/gshadow
            useradd,usermod,passwd,chage命令

https://blog.51cto.com/sonlich/1952080 权限
    解释ls -l 输出行的意思
        -rw-rw-r--  1 root   utmp                9984 Jan  2  2016 wtmp
    chmod,chown,chgrp命令
    mask, umask
    文件的权限: 666-umask. 目录: 777-umask


https://blog.51cto.com/sonlich/1952467
    grep


    grep练习
        1、显示/proc/meminfo文件中以大写s开头的行，要求使用两种方式

        2、显示/etc/passwd文件中不以/bin/bash结尾的行

        3、显示/etc/passwd文件中ID号最大的用户的用户名

        4、如果root用户存在，显示其默认的shell程序

        5、显示/etc/passwd文件中两位或三位数

        6、显示/etc/rc.d/rc.sysinit文件中，至少以一个空白字符开头的且后面存在非空白字符的行

        7、找出'netstat -tab'命令的结果中以'LISTEN'后跟0、1或多个空白字符结尾的行

        8、添加用户bash，testbash,basher以及nologin，要求nologin的shell为/sbin/nologin，而后找出/etc/passwd文件中用户名同shell名的行

https://blog.51cto.com/sonlich/1952624
    egrep

    egrep练习 ， 使用egrep实现grep的练习, 并完成以下内容
        9、 显示当前系统root,centos或user1用户的默认shell和uid                

        10、 找出/etc/rc.d/init.d/functions文件中(CentOS 6),某个单词后面跟一个小括号的行

        11、 echo 输出一个路径，使用egrep取出其基名，

        12、 找出ifcofnig命令结果中1-255中间的任意数值

        13、 找出ifconfig命令结果中的ip地址。


https://blog.51cto.com/sonlich/1952836 进程管理
    pstree
    变量类型：本地，环境，局部，位置，特殊变量
    变量引用
    变量销毁：unset
    显示环境变量：export, env, printenv
          PATH,SHELL,UID,PS1,HISTSIZE,HOME,PWD，OLDPWD
    只读变量： readonly, declare -r
    局部变量：local
    位置：$1, $2, ... $@, $*, $#
        shift命令
    特殊变量
        $?

https://blog.51cto.com/sonlich/1952923  
    算术运算
        $[expression]
        $((expression))
        let expression

https://blog.51cto.com/sonlich/1953085 条件判断
    1）返回状态码
    2）测试表达式
        test 测试表达式
        [ 测试表达式 ]
        [[ 测试表达式 ]]

        数值：-eq, -ne, -gt, -ge, -lt, -le
        字符：
            ==, !=
            a =~ pattern
            -z "string"
            -n "string"
    3）文件测试
        -a/-e
        -b,-c, -d, -f, -p, -h/-L,-S,-g,-u,-k,-r,-w,-x,-s,-N,-O,-G,-ef,-nt,-ot

    条件连接:
        ||
        &&
        !


    exit #


    写一个脚本，可接受一个文件路径作为参数，

    如果参数个数小于1，则提示用户"至少给出一个参数"，并立即退出

    如果参数个数不小于1，则显示第一个参数所指向的文件中的空白行数

https://blog.51cto.com/sonlich/1953252
    vim

https://blog.51cto.com/sonlich/1953703
    find命令

https://blog.51cto.com/sonlich/1953825
    suid, sgid, sticky权限


https://blog.51cto.com/sonlich/1953846 
    linux编程
        if单分支

https://blog.51cto.com/sonlich/1954050 磁盘管理(跳过)

https://blog.51cto.com/sonlich/1955162 脚本交互
    read
    bash -x, bash -n

https://blog.51cto.com/sonlich/1955942 压缩工具
    gzip/gunzip: .gz结尾

    bzip2/bunzip2: .bz2结尾

    xz/unxz: .xz后缀,.lzma和.raw后缀


    tar zcvf file.tar.gz
    tar jcvf file.tar.bz2
    tar Jcvf file.tar.xz


https://blog.51cto.com/sonlich/1955978 脚本
    if多分支, for循环

https://blog.51cto.com/sonlich/1956235 程序包管理(跳过)
    rpm, yum
    配置yum源
    自建yum仓库

https://blog.51cto.com/sonlich/1957674 网络基础
    

https://blog.51cto.com/sonlich/1957800 ip
    ip地址分类，必须学
    子网掩码，必须学
    配置路由，必须学
    路由选择：略
    网络接口命名：略
    命令行配置：略
    配置文件配置，必须学

https://blog.51cto.com/sonlich/1958340 进程管理
    pstree,ps,pgrep,pkill,pidof
    top,htop
    glance，pmap,vmstat,dstat
    kill
    job,bg,fg,nohup
    sar(内存),tsar,iosstat(磁盘IO),iftop(网络接口数据)

https://blog.51cto.com/sonlich/1959468 任务计划
    mail,at,batch,crond,sleep

    1）如果有自己喜欢的电影，公司服务器，晚上访问量小，带宽使用小，此时用个at让晚上下载或白天用batch命令，让内核决定什么时候下载。

    2）如何每天0点对数据库备份或etc目录备份。对于每天重复的事情crontab可以解决

    3)磁盘满了给root发mail


    4）如何实现秒级别的执行命令：在每分钟到达时，运行一个命令，需要60秒，就行了

    5）如何实现每7分钟运行一次任务?

    6）每4小时备份一次/etc目录至/backup目录中，保存的文件名称格式为“etc-yyyy-mm-dd-HH.tar.xz”；

    7）每周2, 4, 7备份/var/log/messages文件至/logs目录中，文件名形如“messages-yyyymmdd”；

    8）每两小时取出当前系统/proc/meminfo文件中以S或M开头的信息追加至/tmp/meminfo.txt文件中；

    9）工作日时间内，每小执行一次“ip addr show”命令；

https://blog.51cto.com/sonlich/1961362

    内核组成: uname命令       ，必须会

    内核：uname,mkinitrd,dracut，跳过

    模块: lsmod,modinfo,depmod,modprobe,insmod,rmmod， 学习

    /proc,sysctl，/sys,/dev，udevadm,hotplug命令，了解

    内核编译，跳过

https://blog.51cto.com/sonlich/1964037 脚本编程, for, while

    练习：100以内所有正整数之和

    练习: 100以内所有偶数和

    练习:添加10个用户

    练习:通过ping命令探测172.16.250.1-254范围内的所有主机的在线状态.用while循环

    练习：打印9x9乘法表

    练习：利用RANDOM生成10个随机数，输出这10个数字，并显示其中最大者和最小者


https://blog.51cto.com/sonlich/1964161 sed命令


https://blog.51cto.com/sonlich/1964166 脚本编程 until, for, case


https://blog.51cto.com/sonlich/1964474 函数

    定义
        function name {

        }
        name() {

        }
    返回
        return

https://blog.51cto.com/sonlich/1964833 systemd


https://blog.51cto.com/sonlich/1964938 编程
    数组
        索引数组
        关联数组
    引用
    切片
        字符
        数组

    字符取子串
    变量默认值

https://blog.51cto.com/sonlich/1965174 AWK
    
https://blog.51cto.com/sonlich/1965404 创建CA（跳过）

https://blog.51cto.com/sonlich/1965708 DNS（跳过）

https://blog.51cto.com/sonlich/1967397 openssh
    
https://blog.51cto.com/sonlich/1968933 http协议基础
    General
        Request URL:http://172.16.100.1/          
        Request Method:GET
        Status Code:200 OK
        Remote Address:172.16.100.1:80                   //服务器地址 
    Response Headers
    view source
        Accept-Ranges:bytes                                          
        Connection:close                                 // 服务器是非持久连接 KeepAlive off
        Content-Encoding:gzip                            // 实体格式：字符集,包含多种语言编码格式
        Content-Length:7725                              // 大小
        Content-Type:text/html; charset=UTF-8            // 类型
        Date:Sat, 09 Sep 2017 12:30:15 GMT               // 请求报文的创建时间
        ETag:"10c-6353-558c0da6c3922"                    // 实体的额外标签,基于标签的条件式请求
        Last-Modified:Sat, 09 Sep 2017 12:30:05 GMT      // 实体最近一次修改的时间    
        Server:Apache/2.2.15 (CentOS)                    // 服务器程序名、版本号
        Vary:Accept-Encoding                             // 服务器查看变化的首部
    Request Headers
    view source
        Accept:text/html,application/xhtml+xml,applicat // 客户端可接受的MIME类型
        Accept-Encoding:gzip, deflate, sdch             // 客户端可接受的压缩格式    
        Accept-Charset:                                 // 字符集               
        Accept-Language:zh-CN,zh;q=0.8                  // 客户端可接受的语言编码格式
        Cache-Control:max-age=0                         // 缓存控制
        Connection:keep-alive                           // 
        Host:172.16.100.1                               // 服务器主机                  
        User-Agent:Mozilla/5.0              // 用户代理


之后学习完这几个服务
    iptables, mysql, zabbix, nginx
```

