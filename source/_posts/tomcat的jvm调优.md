---
title: tomcat的jvm调优
date: 2020-12-17 09:11:38
tags:
- java
---

JDK: jvm、库、工具 

jvm：java的虚拟机, java是最流行编程语言，企业级开发、大数据方向。

# 1. jvm组成

1. **heap堆**：不断生成、消亡。 频繁创建对象，变量、数组，清理垃圾是后招，减少垃圾才是先招。高手写java性能好。
2. 栈：threads自已的东西。
3. program counter: 计数器
4. method area：加载类的信息
5. native internal: 其他调用接口

![5977533-ec4ba756d6653df3.webp](http://myapp.img.mykernel.cn/5977533-ec4ba756d6653df3.webp.jpg)

<!--more-->

# 2. gc类型

垃圾回收器，主要是*heap* ，通过标准，不需要覆盖。空间满了就是归整。

1. python回收：标记，当引用计数为0就表示不使用了，可以清除。引用计数不为0，表示已使用，不可以清除。
2. java标记清除，将未使用的内存标记为未使用的状态，新数据来了就直接覆盖内存。 https://my.oschina.net/javazhiyin/blog/4900631
   - 复制：内存对半分，在使用的一半内存中基于标记将已用的复制到另一半完全没有使用的内存中，将原来的一半全部标记为未使用。
     - 空间浪费1半
     - 在复制时，浪费cpu
   4. 标记后，将多个归整成连续的。
   5. 分代收集（主流），1.7之前：新生代和老年代



memcached不挪内存，将内存部分区域划分成固定大小的格式，数据占据格子，小了放在略大的格子中，浪费就浪费了。用完了整个内存，在找最近最少使用的直接覆盖。



# 3. STW：stop world

表示在gc过程中，jvm虚拟机运行的程序中的所有线程(work表示工作)都会停止，“世界停止了”



# 4. gc分代

![image-20201217173957955](http://myapp.img.mykernel.cn/image-20201217173957955.png)

回收1：eden满了，将已用复制到s0，eden标识未用。

回收2：eden和s0一起，将已用复制到s1，eden,s0标识未用。

回收3:  eden和s1一直，复制到s0, eden和s1标识未用。



老年代很难触发垃圾回收：1. 它内存足够大，2. 需要经过多次回收将反复挪动的放在老年代，所以比较稳定。如果老年代满了就真的STW，时间长。



## 4.1. full gc

老年代和新第一代一起gc, permanent不变。



## 4.2. minor gc

老年代gc前，新生代会先gc, 将未使用的和本身对比有相同的直接标识。



## 4.3 full gc触发

1. 老年代满了, 所以要给够内存
2. system.gc()
3. 新生代满了

## 4.4 minor gc触发

新生代满了



## 4.5 减少stw

1. 并行gc回收。
2. 减少gc次数。



并发：这段时间内有这么多Io要做，怎么解决？

并行：某一个时间点有这么多事。



# 5. 回收线程

gc：单线程、时间长、只有在gc在工作，stw结束，工作线程才继续 。

gc：并行，时间短，gc和work同时工作。



# 6. 垃圾收集

1. 新生和老年都是串行
2. 一并一串
3. 新并老CMS（仅标记），直到触发条件才归整：
   - 满了
   - 标记几次后
   - 进行full收集时

# 7. gc调优

年轻代或持久代-XX:*

年轻和老年：-Xms, -Xmm(最大不应超过32G)



```bash
# java -jar service-pay-0.0.1-SNAPSHOT.jar --spring.profiles.active=pre -server -Xms512m -Xmx1024m  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5
 

 -Xms 初始
 -Xmx 最大
 -server  VM运行在server模式，为在服务器端最大化程序运行速度而优化
 CMS模式：-XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5
```



```bash
tomcat connect的线程调整
```





# 8. 工具

jstats打印线程

