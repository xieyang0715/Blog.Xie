---
title: 几种IO模型的原理
date: 2020-10-09 11:43:53
tags: plan
toc: true
---



# 1. 系统IO模型

调用发起后：存在阻塞和非阻塞的状态

调用返回前：存在通知机制(异步)和没有通知(同步)



![image-20201013144559435](http://myapp.img.mykernel.cn/image-20201013144559435.png)

<!--more-->

# 2. 网络IO模型

## 2.1 同步阻塞IO模型

![image-20201013145304281](http://myapp.img.mykernel.cn/image-20201013145304281.png)

整个IO请求过程，用户线程被阻塞，不能做任何事件, 对CPU资源利用率不够。

优点：程序简单，在阻塞等待数据期间进程/线程挂起，基本不会占用CPU资源。

**缺点**：每个连接需要独立的进程/线程处理, 当并发请求量大时，为了维护程序, 内存、线程切换开销较大，apache的prefork使用的这种模式。





## 2.2 同步非阻塞IO模型

用户线程发起IO请求立即返回，但是没有数据，需要轮询发起IO请求，直到数据到达，才读到数据。

**缺点**：不断轮询，不断的IO请求，不断的系统调用，用户和内核态不断切换。

如果过于频繁的重试干耗CPU, 如果等待时长了，用户体验不好，延迟大。所以很少使用这个模型。

![image-20201013145846393](http://myapp.img.mykernel.cn/image-20201013145846393.png)



## 2.3 IO多路复用(事件驱动)

select,poll,epoll,rtsig,kqueue,/dev/poll,eventport

![image-20201013150135881](http://myapp.img.mykernel.cn/image-20201013150135881.png)



apache的prefork = 主进程 + 多进程/单线程 + select, apache的work = 主进程 + 多进程/多线程 + poll

IO多路复用，就是select, poll, epoll， 也叫事件驱动IO(event driven io)。

进程不在阻塞, 而是由select完成阻塞，而且select可以同时阻塞多个请求, 进程只管处理select返回的数据。

![image-20201013150606383](http://myapp.img.mykernel.cn/image-20201013150606383.png)



## 2.4 信号驱动 

apache的event = 主 + 多进程/ 多线程 + 信号驱动

signal-driven io ，用户进程通过sigaction系统调用注册一个信号处理程序，主程序继续执行，当IO操作准备好，内核通知触发一个SIGIO信号。

优点：等待数据达到期间，进程不阻塞，主进程可以继续派任务给进程。

**缺点**：信号IO在大量IO操作时，可能会因为信号队列溢出导致没法通知。

![image-20201013151123594](http://myapp.img.mykernel.cn/image-20201013151123594.png)

## 2.5 异步IO(AIO)

iocp

epoll

用户进程发起aio_read后，直接返回给用户进程，数据准备好后，内核直接复制给用户进程, 然后kernel通知进程已经准备好。IO两个阶段完全非阻塞。

优点：发起调用后，期间进程可以接新请求，内核可以处理事物，相互不影响，可以实现较大的同时实现较高的IO复用，因此异步非阻塞使用最多的一种通信方式，**nginx是异步非阻塞**。

![image-20201013151452402](http://myapp.img.mykernel.cn/image-20201013151452402.png)



# 3. select/poll/epoll区别

apache的prefork(select), work(poll);

nginx(epoll)

```nginx
event {
    use epoll;
}
```



水平触发：多次通知，需要关心**数据是否取完**避免重复通知，效率较低。select, poll

边缘触发：一次通知，需要关心**数据是否取走**避免数据丢失，效率较高。 epoll



![image-20201013151612543](http://myapp.img.mykernel.cn/image-20201013151612543.png)





# 4. epoll优势

1. Linux 2.6内核提出的select和poll的增强版本

2. 水平触发

3. 没有最大并发数限制，能打开FD上限**/proc/sys/fs/file-max**，并结合**worker_rlimit_nofile**参数

4. FD增加，不像select/poll的轮询FD，使用哈希表，效率提升。

5. 使用mmap(memory mapping), epoll可以直接从磁盘往用户空间加载数据。