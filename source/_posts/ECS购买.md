---
title: ECS购买
date: 2020-10-20 06:14:18
tags:
toc: true
---

初创公司前期只需要5-6台服务器，不需要自建机房就需要使用公有云。

<!--more-->

# 1. aliyun主页

aliyun主页: https://www.aliyun.com/

有点类似淘宝电商网站主页

![image-20201020141715749](http://myapp.img.mykernel.cn/image-20201020141715749.png)



# 2. 控制台

完成：产品购买, 日常运维



点击右上脚的控制台即进入以下页面

![image-20201020141915946](http://myapp.img.mykernel.cn/image-20201020141915946.png)



# 3. 两大利器

## 3.1. 工单 

提工单，要有针对性，问题要描述准确

![image-20201020154538162](http://myapp.img.mykernel.cn/image-20201020154538162.png)

![image-20201020154550784](http://myapp.img.mykernel.cn/image-20201020154550784.png)

![image-20201020154611503](http://myapp.img.mykernel.cn/image-20201020154611503.png)

在*处填写需要的信息, 直接提交

![image-20201020154201728](http://myapp.img.mykernel.cn/image-20201020154201728.png)



## 3.2. 帮助文档

以ECS示例

![image-20201020154634565](http://myapp.img.mykernel.cn/image-20201020154634565.png)

![image-20201020154655676](http://myapp.img.mykernel.cn/image-20201020154655676.png)

![image-20201020154726546](http://myapp.img.mykernel.cn/image-20201020154726546.png)

一般有最佳实践

![image-20201020154845180](http://myapp.img.mykernel.cn/image-20201020154845180.png)

# 4. 购买ECS

进入控制台页面

![image-20201020155538175](http://myapp.img.mykernel.cn/image-20201020155538175.png)

创建实例

![image-20201020155558500](http://myapp.img.mykernel.cn/image-20201020155558500.png)

## 4.1 自定义购买

![image-20201020155647546](http://myapp.img.mykernel.cn/image-20201020155647546.png)

## 4.2 付费模式

按量付费(1天大概10多块)：做实验

包年包月：公司买比较划算

![image-20201020155750382](http://myapp.img.mykernel.cn/image-20201020155750382.png)

## 4.3 推荐区

选择有优惠的区

![image-20201020155943260](http://myapp.img.mykernel.cn/image-20201020155943260.png)

## 4.4 实例

- 实验：2C4G
- 场景化选型：生产

### 4.4.1 实验

![image-20201020160435165](http://myapp.img.mykernel.cn/image-20201020160435165.png)

一次购买实例个数

![image-20201020160501005](http://myapp.img.mykernel.cn/image-20201020160501005.png)

### 4.4.2 生产有场景化选型

![image-20201020160052012](http://myapp.img.mykernel.cn/image-20201020160052012.png)

![image-20201020160125226](http://myapp.img.mykernel.cn/image-20201020160125226.png)

## 4.5 镜像

一般公共镜像选择centos或ubuntu

![image-20201020161106931](http://myapp.img.mykernel.cn/image-20201020161106931.png)



## 4.6 存储

生产需要数据盘，实验不需要专用的数据盘

| 性能类别 | ESSD云盘                                                     | SSD云盘                                    | 高效云盘              | 普通云盘                                                     |
| -------- | ------------------------------------------------------------ | ------------------------------------------ | --------------------- | ------------------------------------------------------------ |
| 应用场景 | 大型OLTP数据库：如MySQL、PostgreSQL、Oracle、SQL Server等关系型数据库 NoSQL数据库：如MongoDB、HBase、Cassandra等非关系型数据库 ElasticSearch分布式日志：ELK（Elasticsearch、Logstash和Kibana）日志分析等 | I/O密集型应用 中小型关系数据库 NoSQL数据库 | 开发与测试业务 系统盘 | 数据不被经常访问或者低I/O负载的应用场景 需要低成本并且有随机读写I/O的应用环境 |

![image-20201020160543635](http://myapp.img.mykernel.cn/image-20201020160543635.png)

## 4.7 下一步

![image-20201020160953020](http://myapp.img.mykernel.cn/image-20201020160953020.png)

## 4.8 网络和安全组

![image-20201020161210454](http://myapp.img.mykernel.cn/image-20201020161210454.png)

## 4.9 网络

前往控制台创建网络

![image-20201020161233293](http://myapp.img.mykernel.cn/image-20201020161233293.png)

![image-20201020161259481](http://myapp.img.mykernel.cn/image-20201020161259481.png)

![image-20201020161709739](http://myapp.img.mykernel.cn/image-20201020161709739.png)

![image-20201020161724396](http://myapp.img.mykernel.cn/image-20201020161724396.png)

网络片刷新选择最新的网络

![image-20201020161755149](http://myapp.img.mykernel.cn/image-20201020161755149.png)



## 4.10 公网IP

1. 购买时，分配公网地址, 公网IP和机器绑定的。机器释放，公网IP也释放。
2. 弹性网卡，想用就绑定可以上公网。不想上公网, 删除网卡即可。



>  用户流量不是直接从这个公网IP进入。用户一般从SLB接入。公网IP作用方便登陆ssh, 也可以一个主机配置公网IP，用openvpn接入, 其他主机不买公网IP省钱。

> 按固定带宽使用

![image-20201020162231901](http://myapp.img.mykernel.cn/image-20201020162231901.png)

## 4.11 安全组

公网IP的防火墙

用户接入从前端的SLB，HTTPS在前端卸载，到后端只有HTTP

![image-20201020162315835](http://myapp.img.mykernel.cn/image-20201020162315835.png)

## 4.12 下一步

![image-20201020162408599](http://myapp.img.mykernel.cn/image-20201020162408599.png)

## 4.13 系统配置

使用自定义密码，创建后配置需要重启机器

实例名称，最好给一个标识

主机名：无所谓

	RAM
	   主账号特别高的管理权限
	   将部分权限分配给不同的人，就通过RAM添加子账号。
![image-20201020163237699](http://myapp.img.mykernel.cn/image-20201020163237699.png)

## 4.14 资源分组

默认， 可以分成资源组，不同账号所见不同的资源组

![image-20201020162555629](http://myapp.img.mykernel.cn/image-20201020162555629.png)

## 4.15 所选配置

确定为按量付费的ubuntu系统，高效云盘，2C,4G

公网固定带宽2M，交换机IP 172.31.0.0/24

自定义密码

![image-20201020162733546](http://myapp.img.mykernel.cn/image-20201020162733546.png)

## 4.16 生成模板或脚本

只要不是包年，都是后付费

![image-20201020162903926](http://myapp.img.mykernel.cn/image-20201020162903926.png)

![image-20201020163001452](http://myapp.img.mykernel.cn/image-20201020163001452.png)

## 4.17 设置自动释放时间

![image-20201020163302431](http://myapp.img.mykernel.cn/image-20201020163302431.png)

## 4.18 购买实例

![image-20201020163336275](http://myapp.img.mykernel.cn/image-20201020163336275.png)

## 4.19 自动进入控制台

![image-20201020164641344](http://myapp.img.mykernel.cn/image-20201020164641344.png)

# 5. 连接ecs

复制上面的公网ip, 使用xshell连接

![image-20201020164734461](http://myapp.img.mykernel.cn/image-20201020164734461.png)

![image-20201020170038322](http://myapp.img.mykernel.cn/image-20201020170038322.png)