---
title: 2021-02-18 大淘客面试总结
date: 2021-02-20 06:07:41
tag:
- 个人日记
---



18号那天，太阳比较好，刚过年完，来成都的第二天，便有幸得到一次面试的机会，学到很多东西。



面试前，大概背了几遍前2份工作及相关的项目，能较有逻辑的简要的说出来。到那家公司，先填写面试表格，....

 

后面进入一个办公室，等待面试，很幸运第一个来面试的是技术大佬，刚上来，没有让自我介绍：

- [x] Nginx模块
- [x] 系统加固
- [x] ELK收集日志的流程，问是不是直接fluentd/filebeat写入到es中？
- [x] 前后端开发MVC怎么写的？
- [x] k8s几个节点，多少个pod，用户访问业务，从域名解析到k8s pod的流程？
- [x] kibana分析时，qps、nginx关注的字段？
- [x] 页面一个图标慢怎么处理？

第二轮面试，大约在几分钟后，又来了一个：

- [x] mysql安装时，使用什么选项？

  - [x] 数据库位置、内存、连接数、单表空间、解析域名
  - [x] 数据怎么存储的？1级索引，2级索引
  - [x] 3范式 https://www.cnblogs.com/linjiqin/archive/2012/04/01/2428695.html
    - [ ] 原子性，字段惟一
    - [ ] 行惟一，当前行在所有行是惟一的
    - [ ]  每一列数据都和主键直接相关，间接相关走外键。

- [x] mysql慢日志，explain

  - [x] 全表扫描和全文索引

- [x] mongo加索引，什么选项不影响业务 backgroup=true https://www.runoob.com/mongodb/mongodb-indexing.html

- [x] redis 哪些情况会慢？

  - [x] key长，集中过期，http://kaito-kidd.com/2020/07/03/redis-latency-analysis/

- [x] redis慢查询优化：https://cloud.tencent.com/developer/article/1683794

- [x] activemq/rabbitmq区别：

  https://www.php.cn/faq/453825.html

  java, 集群依赖zookeeper

  erlang, 天生集群，有镜像模式和复制模式





<!--more-->
