---
title: kibana添加常用图形
date: 2020-10-30 09:06:50
tags: 
- 生产小经验
- kibana
toc: true
---





效果如下

优势

- qps衡量访问指标：可以查看业务并发状态
- pv: 总url数, url请求比
- uv: 总源地址，源地址地图索引，每个ip请求次数：防攻击
- status, client事务时长，nginx事务时长
- user-agent, 按比重兼容浏览器

![image-20210310105727859](http://myapp.img.mykernel.cn/image-20210310105727859.png)





<!--more-->

# 1. 添加和保存图形界面

![image-20201030173830006](http://myapp.img.mykernel.cn/image-20201030173830006.png)

![image-20201030173843650](http://myapp.img.mykernel.cn/image-20201030173843650.png)





![image-20201030173927707](http://myapp.img.mykernel.cn/image-20201030173927707.png)

# 2. 每秒请求量



![image-20201102171139386](http://myapp.img.mykernel.cn/image-20201102171139386.png)

![image-20201102180019593](http://myapp.img.mykernel.cn/image-20201102180019593.png)



![image-20201102173814675](http://myapp.img.mykernel.cn/image-20201102173814675.png)

## 2.1 验证

```bash
# 日志格式, 默认. 取出每条日志的小时，分钟，秒。对这个做排序，统计就知道每秒有多少个请求了。
awk '{print $4}' access.log |cut -c14-21|sort|uniq -c|sort -nr|head -n 100
```

![image-20201102180212709](http://myapp.img.mykernel.cn/image-20201102180212709.png)

通过以上图片可以看出, 每秒500个左右的请求，从Kibana图上有一定差距。

filebeat -> redis -> logstash -> elasticsearch -> kibana

1. 将logstash直接从redis读，写入文件，不过滤。观察输出文件每秒请求数和前端nginx的每秒请求数是否一致。确定redis, filebeat正常。 正常
2. 将fileter加上, 测试少了一个数据, 确定为规则问题
3. 将那段时间的日志取出，在本地跑对应规则，观察错误 

![image-20201103162155974](http://myapp.img.mykernel.cn/image-20201103162155974.png)

![image-20201103162209319](http://myapp.img.mykernel.cn/image-20201103162209319.png)

# 3. map地图

![image-20201102171123364](http://myapp.img.mykernel.cn/image-20201102171123364.png)

![image-20201102171008684](http://myapp.img.mykernel.cn/image-20201102171008684.png)、

![image-20201102171020026](http://myapp.img.mykernel.cn/image-20201102171020026.png)



# 5. PV统计(总共的http请求数量)

默认metric计数所有url请求总数, 即PV数量







![image-20201102171156118](http://myapp.img.mykernel.cn/image-20201102171156118.png)



![image-20201102171210934](http://myapp.img.mykernel.cn/image-20201102171210934.png)

## 5.1 验证

从每s请求量之和计算总数, 为一致, 表示没有问题。

![image-20201102180630087](http://myapp.img.mykernel.cn/image-20201102180630087.png)

# 5. 添加UV统计

![image-20201102171251708](http://myapp.img.mykernel.cn/image-20201102171251708.png)

![image-20201102171318168](http://myapp.img.mykernel.cn/image-20201102171318168.png)

# 添加用户统计

nginx中收集标识一个用户的字段，基于uv统计，添加metrics.

# 所有http请求的总响应大小

![image-20201102171354106](http://myapp.img.mykernel.cn/image-20201102171354106.png)

![image-20201102171414914](http://myapp.img.mykernel.cn/image-20201102171414914.png)

单个http请求，响应大小估算

```bash
>>> 2241518341 / 72127026
31 bytes
```



## 一段时间内平均单个http请求的响应大小



![image-20201225152938612](http://myapp.img.mykernel.cn/image-20201225152938612.png)

# 响应状态码统计

![image-20201102171636333](http://myapp.img.mykernel.cn/image-20201102171636333.png)

![image-20201102171729801](http://myapp.img.mykernel.cn/image-20201102171729801.png)

![image-20201102171746055](http://myapp.img.mykernel.cn/image-20201102171746055.png)

![image-20201102171759286](http://myapp.img.mykernel.cn/image-20201102171759286.png)

# 响应时长

client发起一次事务的总响应时长 reponsetime



![image-20201102171830714](http://myapp.img.mykernel.cn/image-20201102171830714.png)

![image-20201102171847202](http://myapp.img.mykernel.cn/image-20201102171847202.png)

![image-20201102171913382](http://myapp.img.mykernel.cn/image-20201102171913382.png)

![image-20201102172028618](http://myapp.img.mykernel.cn/image-20201102172028618.png)



![image-20201102172056200](http://myapp.img.mykernel.cn/image-20201102172056200.png)

## 后端响应慢

upstreamtime: nginx反代一次事务的时长

![image-20210310103848302](http://myapp.img.mykernel.cn/image-20210310103848302.png)

![image-20210310103922076](http://myapp.img.mykernel.cn/image-20210310103922076.png)

# 9. 请求首行统计

request-method, 在nginx日志中配置为

```nginx
 '"request-method":"$request",
```



![image-20201102172817915](http://myapp.img.mykernel.cn/image-20201102172817915.png)

![image-20201102172843270](http://myapp.img.mykernel.cn/image-20201102172843270.png)

![image-20201102172859863](http://myapp.img.mykernel.cn/image-20201102172859863.png)



![image-20201102173106060](http://myapp.img.mykernel.cn/image-20201102173106060.png)

# 10. IP占比统计

![image-20201102173147765](http://myapp.img.mykernel.cn/image-20201102173147765.png)

![image-20201102173157995](http://myapp.img.mykernel.cn/image-20201102173157995.png)

# 11. ip请求量

![image-20201216144153236](http://myapp.img.mykernel.cn/image-20201216144153236.png)

![image-20201216144204434](http://myapp.img.mykernel.cn/image-20201216144204434.png)

![image-20201225153334001](http://myapp.img.mykernel.cn/image-20201225153334001.png)



# 12. 统计user-agent

查看80%的人使用的浏览器类型，前端写代码要兼容 

饼图

![image-20201223160432728](http://myapp.img.mykernel.cn/image-20201223160432728.png)

![image-20201223160444650](http://myapp.img.mykernel.cn/image-20201223160444650.png)

# 生产示例

## 1. 根据响应时长分析接口

根据长响应时间在discover界面搜索相应接口, 让开发去解决这些接口后面的逻辑，`responsetime : "60"`

http://liangcheng.mykernel.cn/2020/10/28/%E6%8E%92%E6%9F%A5%E7%BD%91%E7%AB%99%E6%85%A2/

