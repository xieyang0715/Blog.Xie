---
title: "kvm io测试 sysbench命令工具和KDiskMark图形工具"
date: 2021-03-03 13:42:22
tags:
- kvm
---

# 命令测试

https://my.oschina.net/u/4266471/blog/4053494

```bash
sysbench --test=fileio --num-threads=50 --file-total-size=2G --file-test-mode=rndrw prepare  #准备测试
sysbench --test=fileio --num-threads=50 --file-total-size=2G --file-test-mode=rndrw run      #开始测试
sysbench --test=fileio --num-threads=50 --file-total-size=2G --file-test-mode=rndrw cleanup  #清除测试文件
```





# 图形测试

https://github.com/JonMagon/KDiskMark

![img](https://raw.githubusercontent.com/JonMagon/KDiskMark/master/assets/images/kdiskmark.png)

ubuntu

```bash
add-apt-repository ppa:jonmagon/kdiskmark
apt update
apt install kdiskmark
```





运行

```bash
LANG=en kdiskmark
```



## 机械

机械盘测试不动了 / 空间

![image-20210317094149364](http://myapp.img.mykernel.cn/image-20210317094149364.png)

## 固态盘测试

![image-20210317100217382](http://myapp.img.mykernel.cn/image-20210317100217382.png)

## SSD m.2 Nvme ，虚拟机中

![image-20210317122509136](http://myapp.img.mykernel.cn/image-20210317122509136.png)

## ceph分布式 rbd挂载测试

![image-20210406115911728](http://myapp.img.mykernel.cn/image-20210406115911728.png)

## nfs挂载测试

![image-20210406121140738](http://myapp.img.mykernel.cn/image-20210406121140738.png)

## ceph分布式 cephfs挂载测试

![image-20210406151248515](http://myapp.img.mykernel.cn/image-20210406151248515.png)

## ceph分布式 fuse用户空间文件系统

![image-20210406150403757](http://myapp.img.mykernel.cn/image-20210406150403757.png)