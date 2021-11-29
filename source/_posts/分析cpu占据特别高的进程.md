---
title: 分析cpu占据特别高的java进程
date: 2020-11-26 02:51:22
tags: 日常挖掘
toc: true
---

《高性能Linux服务器运维实战》高俊峰





演示如下

![分析cpu占据特别高的java进程](http://myapp.img.mykernel.cn/分析cpu占据特别高的java进程.gif)

<!--more-->

# 1. 获取占用高cpu的进程id(PID)

```bash
# top -d 1
```



# 2. 获取进程中占用高cpu的线程id(TID)

```bash
# top -d 1 -H -p PID
```





# 3. 依据jstack分析线程id

```bash
#打印16进制
# printf "%x\n" $TID

#线程执行的输出
# jstack $PID  | sed -n "/${TID}/,/^[\"]/p"

```



将异常给开发即可



