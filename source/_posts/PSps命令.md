---
title: ps命令分析占用高的进程
date: 2021-03-19 02:34:43
tags:
- ps
- 个人日记
---



# 进程相关
<!--more-->

```bash
PS_OPT="state=SimpleProcessState,stat,user=username,uid=UserId,pid,f=ProcessFlags,ppid=ParentPid,tty=TTY,ni=NICE,pri=Priority,sgi_p=ExecOnProcessor,psr=OnProcessor,lstart:30=CreateProcessTime,etime:30=ElapsedTimeSinceProcessStart,%cpu,cputime:20=CpuUse,%mem,vsz:20=TotalMem,rsz:20=OnlyInMemory\(kb\),command"

# 所有进程(与任意终端相关或无关)
root@harbor01:~# ps axo $PS_OPT

# 与任意终端相关的进程
root@harbor01:~# ps ao $PS_OPT


# 与当前终端相关的进程
root@master01:~# ps -To $PS_OPT

# 运行状态的进程
root@master01:~# ps -ro $PS_OPT
```

## 选项

```bash
#--columns n Set screen width. 
ps axo $PS_OPT --columns 20
ps axo $PS_OPT --columns 10

#e      Show the environment after the command.      
ps eaxo $PS_OPT


#f      ASCII art process hierarchy (forest). # 进程权
ps faxo $PS_OPT

#h      No header.
ps haxo $PS_OPT

# 排序 --sort user顺序。 --user -user逆序 
# 使用 %cpu, %mem, cputime, vsz, rss, lstart, etime, psr, sgi_p, s, ni, pri 均可以
## 基于用户
ps -axo  $PS_OPT   --sort user
## 基于用户id
ps -axo  $PS_OPT   --sort uid
## 基于cpu
ps -axo  $PS_OPT   --sort -%cpu
## 基于内存
ps -axo  $PS_OPT   --sort -%mem
```



## 过滤

```bash
# 获取PID为42的进程
root@master01:~# ps -o $PS_OPT 42
root@master01:~# ps -o $PS_OPT -p 42
# 用户
root@master01:~# ps -o $PS_OPT -u nobody

# 命令 bash
root@master01:~# ps -o $PS_OPT -C bash
root@master01:~# ps -o $PS_OPT -C nfsd

# tty
root@master01:~# ps -o $PS_OPT -t tty1

# uid, gid, ...
root@master01:~# ps -o $PS_OPT -u 1
root@master01:~# ps -o $PS_OPT -g 1
```



# 运行或IO过程的进程，占用CPU使用最多的进程

进程处于运行态或不可中断睡眠，占用内存和CPU最多的进程

```bash
ps -axo  $PS_OPT   --sort -%cpu,-%mem | awk '{if (NR==1 || $1 ~ "^[RD]") {print}}'

# 使用脚本
PS_OPT="state=SimpleProcessState,stat,user=username,uid=UserId,pid,f=ProcessFlags,ppid=ParentPid,tty=TTY,ni=NICE,pri=Priority,sgi_p=ExecOnProcessor,psr=OnProcessor,lstart:30=CreateProcessTime,etime:30=ElapsedTimeSinceProcessStart,%cpu,cputime:20=CpuUse,%mem,vsz:20=TotalMem,rsz:20=OnlyInMemory\(kb\),comm"
while true; do sleep 1; ps -axo  $PS_OPT   --sort -%cpu,-%mem | awk '{if (NR==1 || $1 ~ "^[RD]") {print}}' | grep -v $$; echo ; done
```



# PROCESS FLAGS（f)

```bash
ps o f
1    forked but didn't exec
4    used super-user privileges
               
```



# PROCESS STATE CODES(stat,state,s)

```bash

       Here are the different values that the s, stat and state output specifiers (header "STAT" or "S") will display to describe the state of a process:

               D    uninterruptible sleep (usually IO) # 正在进行IO，不可以被调度到CPU运行。即vmstat的b
               I    Idle kernel thread
               R    running or runnable (on run queue) # 运行
               S    interruptible sleep (waiting for an event to complete) # 可以被调度到CPU上运行。即vmstat的r
               T    stopped by job control signal
               t    stopped by debugger during the tracing
               W    paging (not valid since the 2.6.xx kernel)
               X    dead (should never be seen)
               Z    defunct ("zombie") process, terminated but not reaped by its parent

       For BSD formats and when the stat keyword is used, additional characters may be displayed:

               <    high-priority (not nice to other users)
               N    low-priority (nice to other users)
               L    has pages locked into memory (for real-time and custom IO)
               s    is a session leader
               l    is multi-threaded (using CLONE_THREAD, like NPTL pthreads do)
               +    is in the foreground process group
               
               
root@master01:~# ps o stat,f,cmd
STAT F CMD
Ss+  4 /sbin/agetty -o -p -- \u --noclear tty1 linux
Ss   4 -bash
R+   0 ps o stat,f,cmd
Ss+  4 -bash

STAT S F CMD
Ss+  S 4 /sbin/agetty -o -p -- \u --noclear tty1 linux
Ss   S 4 -bash
R+   R 0 ps o stat,s,f,cmd
Ss+  S 4 -bash


root@master01:~# vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b 交换 空闲 缓冲 缓存   si   so    bi    bo   in   cs us sy id wa st
 0  0 138756 634708 945952 4191436    0    0     8    65    1   16  7  3 85  5  0
```

# STANDARD FORMAT SPECIFIERS (o ...)

## cpu和内存使用百分比 (%cpu,%mem,pcpu,pmem)

```bash
root@master01:~# ps o pcpu,pmem,cmd k-pcpu,-pmem
%CPU %MEM CMD
 0.0  0.1 -bash
 0.0  0.1 -bash
 0.0  0.0 /sbin/agetty -o -p -- \u --noclear tty1 linux
 0.0  0.0 ps o pcpu,pmem,cmd k-pcpu,-pmem


root@master01:~# ps o %cpu
%CPU
 0.0
 0.0
 0.0
 0.0
root@master01:~# ps o %mem
%MEM
 0.0
 0.0
 0.1
 0.1


```

## cpu总时间 (cputime)

```bash
 cputime     TIME      cumulative CPU time, "[DD-]hh:mm:ss"  format.  (alias time)
```

## 代码部分占用 内存 (trs)

```bash
       trs         TRS       text resident set size, the amount of
                             physical memory devoted to executable code.


root@master01:~# ps v
  PID TTY      STAT   TIME  MAJFL   TRS   DRS   RSS %MEM COMMAND
 1196 tty1     Ss+    0:00      2    49 17898  1736  0.0 /sbin/agetty -o -p -- \u --noclear tty1 linux
12293 pts/1    R+     0:00      0   109 30574  1532  0.0 ps v
17936 pts/1    Ss     0:00      0  1036 30855 12416  0.1 -bash
30769 pts/0    Ss+    0:00      0  1036 30855 12652  0.1 -bash
root@master01:~# ps o trs,drs
 TRS   DRS
  49 17898
 109 30574
1036 30855
1036 30855

```



## 非代码的数据部分占用内存 (drs)

```bash
       drs         DRS       data resident set size, the amount of
                             physical memory devoted to other than
                             executable code.
                             
                             
root@master01:~# ps v
  PID TTY      STAT   TIME  MAJFL   TRS   DRS   RSS %MEM COMMAND
 1196 tty1     Ss+    0:00      2    49 17898  1736  0.0 /sbin/agetty -o -p -- \u --noclear tty1 linux
16580 pts/1    R+     0:00      0   109 30574  1528  0.0 ps v
17936 pts/1    Ss     0:00      0  1036 30855 12416  0.1 -bash
30769 pts/0    Ss+    0:00      0  1036 30855 12652  0.1 -bash
root@master01:~# ps o drs
  DRS
17898
30574
30855
30855
root@master01:~# 

```

## 进程的虚拟内存,总内存 (vsz)

```bash
vsz         VSZ       virtual memory size of the process in KiB
                             (1024-byte units).  Device mappings are
                             currently excluded; this is subject to
                             change.  (alias vsize).
                             
root@master01:~# ps o vsz,rsz
   VSZ   RSZ
 17948  1736
 30684  1528
 31892 12416
 31892 12652

```



## 不能换进swap的空间RSS/RSZ (rsz,rss)

```bash
       rss         RSS/RSZ             resident set size, the non-swapped physical
                             memory that a task has used (in kilobytes).
                             (alias rssize, rsz).
                             
root@master01:~# ps v
  PID TTY      STAT   TIME  MAJFL   TRS   DRS   RSS %MEM COMMAND
 1196 tty1     Ss+    0:00      2    49 17898  1736  0.0 /sbin/agetty -o -p -- \u --noclear tty1 linux
17936 pts/1    Ss     0:00      0  1036 30855 12416  0.1 -bash
27937 pts/1    R+     0:00      0   109 30574  1528  0.0 ps v
30769 pts/0    Ss+    0:00      0  1036 30855 12652  0.1 -bash
root@master01:~# ps o rss,cmd
  RSS CMD
 1736 /sbin/agetty -o -p -- \u --noclear tty1 linux
12416 -bash
 1528 ps o rss,cmd
12652 -bash

```



# 进程创建的时间 (lstart,start,start_time,stime)

```bash
root@master01:~# ps o lstart:30,cmd
                       STARTED CMD
      Wed Mar  3 19:47:10 2021 /sbin/agetty -o -p -- \u --noclear tty1 linux
      Fri Mar 19 11:48:02 2021 ps o lstart:30,cmd
      Fri Mar 19 11:17:52 2021 -bash
      Thu Mar 18 21:05:02 2021 -bash
root@master01:~# ps o start:30,cmd
                       STARTED CMD
                        Mar 03 /sbin/agetty -o -p -- \u --noclear tty1 linux
                      11:48:30 ps o start:30,cmd
                      11:17:52 -bash
                      21:05:02 -bash
root@master01:~# ps o start_time:30,cmd
                         START CMD
                        3月03 /sbin/agetty -o -p -- \u --noclear tty1 linux
                         11:48 ps o start_time:30,cmd
                         11:17 -bash
                        3月18 -bash
root@master01:~# ps o stime:30,cmd
                         STIME CMD
                        3月03 /sbin/agetty -o -p -- \u --noclear tty1 linux
                         11:48 ps o stime:30,cmd
                         11:17 -bash
                        3月18 -bash
```

## 进程创建至今经过的时间 (etime,etimes)

```bash
# 经过时间
root@master01:~# ps o drs,etime,lstart,cmd
  DRS     ELAPSED                  STARTED CMD
17898 15-16:32:17 Wed Mar  3 19:47:10 2021 /sbin/agetty -o -p -- \u --noclear tty1 linux
30574       00:00 Fri Mar 19 12:19:27 2021 ps o drs,etime,lstart,cmd
30855    01:01:35 Fri Mar 19 11:17:52 2021 -bash
30855    15:14:25 Thu Mar 18 21:05:02 2021 -bash
# 经过的秒数
root@master01:~# ps o drs,etimes,lstart,cmd
  DRS ELAPSED                  STARTED CMD
17898 1355540 Wed Mar  3 19:47:10 2021 /sbin/agetty -o -p -- \u --noclear tty1 linux
30574       0 Fri Mar 19 12:19:30 2021 ps o drs,etimes,lstart,cmd
30855    3698 Fri Mar 19 11:17:52 2021 -bash
30855   54868 Thu Mar 18 21:05:02 2021 -bash
```



# 进程当前在哪个处理器 (psr)

```bash
       psr         PSR       processor that process is currently
                             assigned to.
                             
                             
root@master01:~# ps o psr
PSR
  1
  0
  1
  1

```

## 进程当前在哪个处理器执行 (sgi_p)

```bash
sgi_p       P    processor that the process is currently executing on. Displays "*" if the process is not currently running or runnable.
                       
                       
root@master01:~# ps o psr,sgi_p,cmd
PSR P CMD
  1 * /sbin/agetty -o -p -- \u --noclear tty1 linux
  1 1 ps o psr,sgi_p,cmd
  1 * -bash
  1 * -bash

# 分配在CPU1但是没有执行
                
```



# 进程状态 (s,stat,state)

```bash
       s           S         minimal state display (one character).  See
                             section PROCESS STATE CODES for the
                             different values.  See also stat if you
                             want additional information displayed.
                             (alias state).
                             
                             
root@master01:~# ps o state,psr,cmd
S PSR CMD
S   1 /sbin/agetty -o -p -- \u --noclear tty1 linux
S   1 -bash
S   1 -bash
R   1 ps o state,psr,cmd

```



# 命令 (command,cmd)

```bash
cmd         CMD       see args.  (alias args, command).

root@master01:~# ps o lstart,command
                 STARTED COMMAND
Wed Mar  3 19:47:10 2021 /sbin/agetty -o -p -- \u --noclear tty1 linux
Fri Mar 19 12:12:02 2021 ps o lstart,command
Fri Mar 19 11:17:52 2021 -bash
Thu Mar 18 21:05:02 2021 -bash

```

# nice值 (nice,ni)

```bash
       ni          NI        nice value.  This ranges from 19 (nicest)
                             to -20 (not nice to others), see nice(1).
                             (alias nice).
```

## 优先级 (pri,priority)

```bash
       pri         PRI       priority of the process.  Higher number
                             means lower priority.
```





# 同学分享

N49001 广州 学习使我快乐

```bash
1)使用ps命令找出占用内存资源最多的20个进程：
ps aux | head -1;
ps aux |grep -v PID |sort -rn -k +4 | head -20ps -aux --sort -pmem | head -n 20


2)查看进程占用的实际物理内存，这个也包含了共享库等：
ps -eo size,pid,user,command --sort -size | awk '{ hr=$1/1024 ; printf("%13.2f Mb ",hr) } { for ( x=4 ; x<=NF ; x++ ) { printf("%s ",$x) } print "" }' |cut -d "" -f2 | cut -d "-" -f1


3)通过python脚本观察私有内存和共享内存：python ps_mem.py -p 999脚本位置在https://raw.githubusercontent.com/pixelb/ps_mem/master/ps_mem.py，直接cp即可

4)通过smaps观察私有内存，这个和python脚本得出的数字差不多，单位是KB

ps uxf -p 999 | sort -nk6 | tail -n 1 | tee >( cat - >&2) | awk '{system("cat /proc/"$2"/smaps")}' | grep ^Private | awk '{A+=$2} END{print A}'

5)通过smem命令观察，这个得出来的USS和python脚本以及smaps差不多，USS == Private 
smem -k | grep -w '/usr/pgsql-13/bin/postgres'


6)追查内存使用情况，用到脚本：

#/bin/bash
for PROC in `ls /proc/|grep "^[0-9]"`
do
   if [ -f /proc/$PROC/statm ]; then
   	TEP=`cat /proc/$PROC/statm | awk '{print ($2)}'`
	RSS=`expr $RSS + $TEP`
   fi
done
RSS=`expr $RSS \* 4`
PageTable=`grep PageTables /proc/meminfo | awk '{print $2}'`
SlabInfo=`cat /proc/slabinfo | awk 'BEGIN{sum=0;}{sum=sum+$3*4;}END{print sum/1024/1024}'`

echo $RSS"KB",$PageTable"KB",$SlabInfo"MB"
printf "rss+pagetable+slabinfo=%sMB\n" `echo $RSS/1024 + $PageTable/1024 + $SlabInfo|bc`
free -m
```

# 老师分享

https://www.cnblogs.com/FengGeBlog/p/14326794.html



# top命令

> cnblogs: https://www.cnblogs.com/sparkdev/p/8176778.html
