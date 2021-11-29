---
title: 购买ACK标准版集群
date: 2021-08-11 15:55:30
tags:
---

# 容器集群购买地址

https://cs.console.aliyun.com/?spm=5176.12818093.ProductAndService--ali--widget-home-product-recent.dre2.5adc16d0npVAf0#/k8s



# 参数

专有网络、虚拟交换机：要和相同业务的主机相同。

生产要使用同地域。一般一个地域有多个可用区，不同可用区间是时延低的线路。但是内网不通。

![image-20210811160406188](http://myapp.img.mykernel.cn/image-20210811160406188.png)

> ACK托管、标准、地域

![image-20210811160427792](http://myapp.img.mykernel.cn/image-20210811160427792.png)

![image-20210811160509906](http://myapp.img.mykernel.cn/image-20210811160509906.png)

![image-20210811160525900](http://myapp.img.mykernel.cn/image-20210811160525900.png)

# 选择ecs

进入ecs购买页，查看推送的**独享型**主机，属于哪个区。

然后在k8s选择这个区，自然有主机。

![image-20210811162507151](http://myapp.img.mykernel.cn/image-20210811162507151.png)

# k8s上选择节点

![image-20210811162548544](http://myapp.img.mykernel.cn/image-20210811162548544.png)

> 2个，必须。高可用。
>
> ESSD 120G. iops 5万。
>
> centos 7.9, 默认这个

# 组件配置

默认即可

![image-20210811162724037](http://myapp.img.mykernel.cn/image-20210811162724037.png)

# 确定是否按量

![image-20210811162807619](http://myapp.img.mykernel.cn/image-20210811162807619.png)

> 累积一个月 830元。

