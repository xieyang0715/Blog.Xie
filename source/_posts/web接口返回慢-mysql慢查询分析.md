---
title: 'web接口返回慢, mysql慢查询分析'
date: 2020-12-30 09:03:23
tags: 
- 生产小经验
- mysql
---





今天运营在导出报表，导出10分钟没有结果，开发给出sql语句，在navicat上执行了20分钟没有结果。

#  结束mysql的查询语句

http://blog.mykernel.cn/2020/12/03/%E7%94%9F%E4%BA%A7app%E4%B8%8D%E5%8F%AF%E7%94%A8502/#2-mysql-%E6%9F%A5%E7%9C%8B%E6%85%A2%E6%97%A5%E5%BF%97%E3%80%81%E7%9B%91%E6%8E%A7



#  explain分析sql

https://www.cnblogs.com/xuanzhi201111/p/4175635.html

https://segmentfault.com/a/1190000008131735

https://cloud.tencent.com/developer/article/1093229

[52条sql优化](https://mp.weixin.qq.com/s?__biz=MjM5MzgyODQxMQ==&mid=2650375466&idx=2&sn=f9ddb709ed799f14a625dc56bbd0d172&chksm=be9c3e7e89ebb768eea6644ccc8ce876a120c294d05db89e6e31703f32e73970f2ae841d188e&mpshare=1&scene=23&srcid=1230nxXwP55UR6b23x7A3bFG&sharer_sharetime=1609336897070&sharer_shareid=87634a93e07b7b3e9e7ee0c0a1173c37#rd)

[mysql性能调优](https://www.oschina.net/group/skill#/detail/2366266)

SQL Select语句完整的执行顺序： 

1、from子句组装来自不同数据源的数据； 

2、where子句基于指定的条件对记录行进行筛选； 

3、group by子句将数据划分为多个分组； 

4、使用聚集函数进行计算； 

5、使用having子句筛选分组； 

6、计算所有的表达式； 

7、select 的字段；

8、使用order by对结果集进行排序。

[SQL](https://www.cnblogs.com/HDK2016/p/6884191.html)查询处理的步骤序号：

(1) FROM <left_table>

(3) <join_type> JOIN <right_table>

(2) ON <join_condition>

(4) WHERE <where_condition>

(5) GROUP BY <group_by_list>

(6) WITH {CUBE | ROLLUP}

(7) HAVING <having_condition>

(8) SELECT

(9) DISTINCT

(9) ORDER BY <order_by_list>

(10) <TOP_specification> <select_list>



![image-20201230175119076](http://myapp.img.mykernel.cn/image-20201230175119076.png)

1. **id**

   id如果相同，可以认为是一组，从上往下顺序执行；在所有组中，id值越大，优先级越高，越先执行

2. **select_type**

   **查询中每个select子句的类型**

   - PRIMARY(查询中若包含任何复杂的子部分,最外层的select被标记为PRIMARY)
   - DERIVED(派生表的SELECT, FROM子句的子查询)
   - DEPENDENT SUBQUERY(子查询中的第一个SELECT，取决于外面的查询)

3. **table**

   这一行的数据是关于哪张表的，有时不是真实的表名字,看到的是derivedx(x是个数字,我的理解是第几步执行的结果)



4. **type**

   常用的类型有： **ALL, index, range, ref, eq_ref, const, system, NULL（从左到右，性能从差到好）**

   - ALL：Full Table Scan， MySQL将遍历全表以找到匹配的行
   - eq_ref: 类似ref，多表连接中使用primary key或者 unique key作为关联条件
   - ref: 表示**table**表连接匹配条件，即哪些列或常量被用于查找索引列上的值

5. **possible_keys**

   指出MySQL能使用哪个索引在表中找到记录，查询涉及到的字段上若存在索引，则该**索引将被列出，但不一定被查询使用**

6. **Key**

   **key列显示MySQL实际决定使用的键（索引）**

   如果没有选择索引，键是NULL。要想强制MySQL使用或忽视possible_keys列中的索引，在查询中使用FORCE INDEX、USE INDEX或者IGNORE INDEX。

7. **key_len**

   **表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度（key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的）**

   不损失精确性的情况下，长度越短越好 

8. **ref**

   **表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值**

9. **rows**

    **表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数**

10. **Extra**

    **该列包含MySQL解决查询的详细信息,有以下几种情况：**

    - Using where: mysql服务器将在存储引擎检索行后再进行过滤。就是先读取整行数据，再按 where 条件进行检查，符合就留下，不符合就丢弃。
    - Using index：这发生在对表的请求列都是同一索引的部分的时候，返回的列数据只使用了索引中的信息，而没有再去访问表中的行记录。是性能高的表现。
    - not exists: MYSQL优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行， 就不再搜索了。
    - using filesort: ORDER BY ,GROUP BY使用了非索引字段



# soar优化

官方地址：https://github.com/XiaoMi/soar

web地址：https://github.com/xiyangxixian/soar-web

```bash
docker run -d --name soar-web -p 5077:5077 becivells/soar-web
```



