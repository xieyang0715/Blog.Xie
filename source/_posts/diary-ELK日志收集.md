---
title: ELK日志收集
date: 2021-03-08 10:42:55
tags:
- 个人日记
- elk
---

# 部署elasticsearch
<!--more-->

es分为data, master, ingest节点，我们只需要使用默认配置，集群内部实现这些逻辑

由于本地没有足够的主机，所以使用单机的es

## 单机es 7.11.1

https://www.elastic.co/cn/downloads/past-releases#elasticsearch

直接选择最新的  [RPM X86_64](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.11.1-x86_64.rpm)[sha](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.11.1-x86_64.rpm.sha512)

```bash
[root@elk ~]# rpm -ivh elasticsearch-7.11.1-x86_64.rpm 
```

```bash
[root@elk ~]# install -dv /data/elasticsearch/
[root@elk ~]# mount /dev/sda5 /data/elasticsearch/
[root@elk ~]# ls /data/elasticsearch/
lost+found

```

```diff
[root@elk ~]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Thu Jun 18 21:44:15 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=12d1c266-1a85-46dd-8a9e-165bc5a8b9cc /boot                   xfs     defaults        0 0
/dev/mapper/centos-home /home                   xfs     defaults        0 0
+/dev/sda5               /data/elasticsearch/    ext4    defaults        0 0
```

配置

```bash
cluster.name: elk-cluster
node.name: 192.168.0.171 # node.name必须存在在  discovery.seed_hosts cluster.initial_master_nodes 中，主机名就也写主机名
path.data: /data/elasticsearch/es_data
path.logs: /data/elasticsearch/es_logs  
bootstrap.memory_lock: true
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["192.168.0.171"]
cluster.initial_master_nodes: ["192.168.0.171"]
gateway.recover_after_nodes: 1
action.destructive_requires_name: true
http.cors.enabled: true #开启支持跨域访问
http.cors.allow-origin: "*" #指定允许访问范围
```

```bash
[root@elk ~]# install -dv -o elasticsearch -g elasticsearch /data/elasticsearch/es_data /data/elasticsearch/es_logs
```

启动

```bash
[root@elk ~]# systemctl restart elasticsearch
```

测试访问9200

```bash
[root@elk ~]# curl localhost:9200
{
  "name" : "192.168.0.171",
  "cluster_name" : "elk-cluster",
  "cluster_uuid" : "1AxrlLZNQmml8YNf4zKE1Q",
  "version" : {
    "number" : "7.11.1",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "ff17057114c2199c9c1bbecc727003a907c0db7a",
    "build_date" : "2021-02-15T13:44:09.394032Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

查看Elasticsearch健康状态 (*表示ES集群的master主节点)

```bash
[root@elk ~]# curl -XGET 'http://192.168.0.171:9200/_cat/nodes?v'
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role  master name
10.244.6.0           21          59   3    0.47    0.46     0.35 cdhilmrstw *      192.168.0.171



[root@elk ~]# curl -XGET 'http://192.168.0.171:9200/_cat/health'
1614942494 11:08:14 elk-cluster green 1 1 6 6 0 0 0 0 - 100.0%


[root@elk ~]# curl -XGET 'http://192.168.0.171:9200/_cluster/health?pretty'
{
  "cluster_name" : "elk-cluster",
  "status" : "green",       #为 green 则代表健康没问题，如果是 yellow 或者 red 则是集群有问题
  "timed_out" : false,        #是否有超时
  "number_of_nodes" : 1,         #集群中的节点数量
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 6,
  "active_shards" : 6,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0 #集群分片的可用性百分比，如果为0则表示不可用
}

```

> https://www.cnblogs.com/kevingrace/p/10671063.html

### 部署ElasticSearch Web管理工具

https://github.com/lmenezes/cerebro

```bash
# releases处下载
[root@elk ~]# rpm -ivh cerebro-0.9.3-1.noarch.rpm 
准备中...                          ################################# [100%]
Creating system group: cerebro
Creating system user: cerebro in cerebro with cerebro user-daemon and shell /bin/false
正在升级/安装...
   1:cerebro-0.9.3-1                  ################################# [100%]
Created symlink from /etc/systemd/system/multi-user.target.wants/cerebro.service to /usr/lib/systemd/system/cerebro.service.


[root@elk ~]# java -Duser.dir=/usr/share/cerebro  -jar /usr/share/cerebro/lib/cerebro.cerebro-0.9.3-launcher.jar
[info] p.c.s.AkkaHttpServer - Listening for HTTP on /0:0:0:0:0:0:0:0:9000
```

首页填 http://192.168.0.171:9200

![image-20210305191927931](http://myapp.img.mykernel.cn/image-20210305191927931.png)





## 集群es

直接拿下生成的es的yaml

| ip           | 角色 |
| ------------ | ---- |
| 172.16.0.222 | es1  |
| 172.16.0.223 | es2  |
| 172.16.0.224 | es3  |

```diff
# 222
[root@k8s-master1 ~]# cat /etc/elasticsearch/elasticsearch.yml 
cluster.name: weizhixiu
+node.name: node-1
path.data: /mnt/elasticsearch/data
path.logs: /mnt/elasticsearch/log
bootstrap.memory_lock: true
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
cluster.initial_master_nodes: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
gateway.recover_after_nodes: 2
action.destructive_requires_name: true
http.cors.enabled: true
http.cors.allow-origin: "*"
```

```diff
# 223
[root@k8s-master2 ~]# cat /etc/elasticsearch/elasticsearch.yml 
cluster.name: weizhixiu
+node.name: node-2
path.data: /mnt/elasticsearch/data
path.logs: /mnt/elasticsearch/log
bootstrap.memory_lock: true
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
cluster.initial_master_nodes: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
gateway.recover_after_nodes: 2
action.destructive_requires_name: true
http.cors.enabled: true #开启支持跨域访问
http.cors.allow-origin: "*" #指定允许访问范围
```

```diff
[root@k8s-master3 ~]# cat /etc/elasticsearch/elasticsearch.yml 
cluster.name: weizhixiu
+node.name: node-3
path.data: /mnt/elasticsearch/data
path.logs: /mnt/elasticsearch/logs
bootstrap.memory_lock: true
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
cluster.initial_master_nodes: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
gateway.recover_after_nodes: 2
action.destructive_requires_name: true
http.cors.enabled: true #开启支持跨域访问
http.cors.allow-origin: "*" #指定允许访问范围
```



# 部署logstash

## 部署jdk

```bash
#!/bin/bash
#此版本适合Ubuntu 18.04.4 LTS，其他版本未测试



#下载
cd /usr/local/src

[[ ! -f jdk-8u221-linux-x64.tar.gz ]] && wget http://qiniu.mykernel.cn/jdk-8u221-linux-x64.tar.gz

cd /usr/local/src && rm -rf  jdk1.8.0_221

#解压
tar xvf jdk-8u221-linux-x64.tar.gz

#软连接
ln -sv /usr/local/src/jdk1.8.0_221 /usr/local/jdk

#编辑配置文件
echo "
export JAVA_HOME=/usr/local/jdk
export TOMCAT_HOME=/apps/tomcat
export PATH=\$JAVA_HOME/bin:\$JAVA_HOME/jre/bin:\$TOMCAT_HOME/bin:\$PATH
export CLASSPATH=.\$CLASSPATH:\$JAVA_HOME/lib:\$JAVA_HOME/jre/lib:\$TOMCAT_HOME/lib/tools.jar">>/etc/profile

#读取配置文件
source /etc/profile

#版本检测
java -version
```

## 部署logstash 7.11.1

- [RPM X86_64](https://artifacts.elastic.co/downloads/logstash/logstash-7.11.1-x86_64.rpm)[sha](https://artifacts.elastic.co/downloads/logstash/logstash-7.11.1-x86_64.rpm.sha512)

### 测试标准输入、输出

```bash
[root@elk ~]# rpm -ivh logstash-7.11.1-x86_64.rpm 

~]# /usr/share/logstash/bin/logstash -e 'input { stdin{} } output { stdout{ codec => rubydebug }}' #标准 输入 和输出
hello

{
    "@timestamp" => 2021-03-05T11:40:22.916Z,       # 当前 事件的 发生时间
      "@version" => "1",                            # 事件 版本号 一个事件就是一个 ruby 对象
       "message" => "how",                          # 消息 的 具体内容
          "host" => "elk.youwyouqu.io"              # 标记事件发生 在 哪里
}

```



### 测试输出到文件

```bash
~]# /usr/share/logstash/bin/logstash -e 'input { stdin{} } output { file { path => "/tmp/log-%{+YYYY.MM.dd}messages.gz" } }' 
 
 
# 验证
[root@elk ~]# cat /tmp/log-2021.03.05messages.gz 
{"@version":"1","@timestamp":"2021-03-05T11:47:26.516Z","message":"hello","host":"elk.youwyouqu.io"}
```



### 测试输出到elasticsearch

```bash


~]# /usr/share/logstash/bin/logstash -e 'input { stdin{} } output { elasticsearch { hosts => [" 192.168.0.171:9200"] index => "mytest-%{+YYYY.MM.dd}" } }' 
 
hello
```

验证数据

![image-20210305195229441](http://myapp.img.mykernel.cn/image-20210305195229441.png)

```bash
[root@elk ~]# ls /data/elasticsearch/es_data/nodes/0/indices/
-2rQPgsURfOUb92NM3GjbQ  imBHodTsSp6d7T05Mj3aLQ  LzxADPztTleC4eu3fHimeg  w2-0ff4hS4aCZrSix3185w  z1Ug-8v0TgueFWWDKzaJ2A
FVy58KrbQrWLRxoHr-RL3A  lA6f2U9IQ0KSK_V_BQfbxQ  -MLJP-UbSqSEB1ekIu8TLg  x6bWEDYhTg2WchNIrbzCBQ
```

# 部署kibana 7.11.1

https://www.elastic.co/cn/downloads/past-releases#kibana

- [RPM 64-BIT](https://artifacts.elastic.co/downloads/kibana/kibana-7.11.1-x86_64.rpm)[sha](https://artifacts.elastic.co/downloads/kibana/kibana-7.11.1-x86_64.rpm.sha512)

## 单机kibana

```bash
[root@elk ~]# rpm -ivh kibana-7.11.1-x86_64.rpm 
```

配置

```bash
[root@elk ~]# cat /etc/kibana/kibana.yml
server.port: 5601
server.host: "192.168.0.171"
elasticsearch.hosts: ["http://192.168.0.171:9200"]
#elasticsearch.username: ""
#elasticsearch.password: ""
i18n.locale: "zh-CN"
```

查看状态

http://192.168.0.171:5601/status

访问kibana

![image-20210305192200836](http://myapp.img.mykernel.cn/image-20210305192200836.png)

添加上一步的数据

![image-20210305195636374](http://myapp.img.mykernel.cn/image-20210305195636374.png)

![image-20210305195658982](http://myapp.img.mykernel.cn/image-20210305195658982.png)

查看数据

![image-20210305195917607](http://myapp.img.mykernel.cn/image-20210305195917607.png)

## 集群kibana

由于是http请求，就和nginx一样启动多个即可



# logstash日志收集

## 收集系统日志

```bash
[root@elk ~]# cat /etc/logstash/conf.d/system-log.conf
input {
  file {
    path => "/var/log/messages"         # 日志路径 
    type => "systemlog"                 # 事件类型
    start_position => "beginning"       # 首次收集日志的位置, 默认end; benginning将导入旧数据。此选项仅在此文件没有被logstash读取过的文件，如果读取过了，此选项将失效，由记录的位置开始读取。
	stat_interval => "3"                # 查看文件是否修改
  }

  file {
    path => "/var/log/secure"         # 日志路径 
    type => "securelog"                 # 事件类型
    start_position => "beginning"       # 首次收集日志的位置
	stat_interval => "3"                # 查看文件是否修改
  }
}


output {
	if [type] == "systemlog" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "system-log-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
	if [type] == "securelog" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "secure-log-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
}

```

```bash
[root@elk ~]# chmod 644 /var/log/secure /var/log/messages
# 检查语法 
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/system-log.conf -t
OK

# 写日志
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/system-log.conf 
```

测试向messages中添加

```bash
# 换终端
[root@elk ~]# echo test >> /var/log/messages

# 观察到logstash输出
{
          "type" => "systemlog",
    "@timestamp" => 2021-03-08T07:10:52.593Z,
          "path" => "/var/log/messages",
          "host" => "elk.youwyouqu.io",
       "message" => "test",
      "@version" => "1"
}

[root@elk ~]# echo test >> /var/log/secure
{
          "type" => "securelog",
    "@timestamp" => 2021-03-08T07:11:46.606Z,
          "path" => "/var/log/secure",
          "host" => "elk.youwyouqu.io",
       "message" => "test",
      "@version" => "1"
}
```

通过此网站 http://www.kjson.com/ 验证一段json日志

```json
{"sitename":"hello","siteurl":"www.magedu.com","keyword":"DevOps, 云计算，Python","description":"hellohellohellohellohello"}
```

```bash
[root@elk ~]# echo '{"sitename":"hello","siteurl":"www.magedu.com","keyword":"DevOps, 云计算，Python","description":"hellohellohellohellohello"}' >> /var/log/messages
```

可以发现message输出

```bash
{
          "type" => "systemlog",
    "@timestamp" => 2021-03-08T07:19:13.865Z,
          "path" => "/var/log/messages",
          "host" => "elk.youwyouqu.io",
       "message" => "{\"sitename\":\"hello\",\"siteurl\":\"www.magedu.com\",\"keyword\":\"DevOps, 云计算，Python\",\"description\":\"hellohellohellohellohello\"}",
      "@version" => "1"
}
```





kibana查看索引

![image-20210308151319131](http://myapp.img.mykernel.cn/image-20210308151319131.png)

![image-20210308151334209](http://myapp.img.mykernel.cn/image-20210308151334209.png)![image-20210308151410711](http://myapp.img.mykernel.cn/image-20210308151410711.png)

![image-20210308152101713](http://myapp.img.mykernel.cn/image-20210308152101713.png)

## logstash收集java日志

https://www.elastic.co/guide/en/logstash/current/plugins-codecs-multiline.html

多行编解码器将折叠多行消息并将它们合并为单个事件。此编解码器的最初目标是允许将文件中的多行消息连接到单个事件中。例如，将Java异常和堆栈跟踪消息连接到单个事件中。

### 标准输出或文件

```bash
input {
  stdin {
    codec => multiline { 
      pattern => "^\["  #当遇到[开头的行时候将多行进行合并
      negate => "true" #true 为匹配成功进行操作， false 为不成功进行操作 
      what => "previous" #与以前 的行合并，如果是下面的行合并就是 next
    }
  }
}
filter { #日志过滤，如果所有的日志都过滤就写这里，如果只针对某一个过滤就写在 input 里面的日志输入里面
}
output {
	stdout{ codec => rubydebug }
	file { path => "/tmp/javalog-%{+YYYY.MM.dd}messages.gz" }
}
```

> 在输出中每经过一个插件，均会把数据流给一份。
>
> 输入codec => multiline 是插件，可以应用stdin,files插件中

```bash
[1
2
3
4
5
[12
{
          "tags" => [
        [0] "multiline"
    ],
    "@timestamp" => 2021-03-08T07:39:59.129Z,
          "host" => "elk.youwyouqu.io",
       "message" => "[1\n2\n3\n4\n5",
      "@version" => "1"
}
[123
{
    "@timestamp" => 2021-03-08T07:40:10.465Z,
          "host" => "elk.youwyouqu.io",
       "message" => "[12",
      "@version" => "1"
}
456 
[789
{
          "tags" => [
        [0] "multiline"
    ],
    "@timestamp" => 2021-03-08T07:40:22.869Z,
          "host" => "elk.youwyouqu.io",
       "message" => "[123\n456",
      "@version" => "1"
}
101112,13,14,15
[1357
{
          "tags" => [
        [0] "multiline"
    ],
    "@timestamp" => 2021-03-08T07:40:47.926Z,
          "host" => "elk.youwyouqu.io",
       "message" => "[789\n101112,13,14,15",
      "@version" => "1"
}
```

> 通过输出发现均能收集到[之前所有数据

### 输出到elasticsearch

```bash
[root@elk ~]# cat /etc/logstash/conf.d/java.conf
input {
  file {
    path => "/data/elasticsearch/es_logs/elk-cluster.log"
	type => "javalog"
	start_position => "beginning"
    codec => multiline { 
      pattern => "^\[" #这个pattern应该匹配您认为是一个指示符，该字段是多行事件的一部分。
      negate => "true"  
      what => "previous"
    }
  }
}
output {
	stdout{ codec => rubydebug }
	if [type] == "javalog" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "javalog-%{+YYYY.MM.dd}"
		}
	}
}

```

```bash
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/java.conf 
```

![image-20210308155110961](http://myapp.img.mykernel.cn/image-20210308155110961.png)![image-20210308155126545](http://myapp.img.mykernel.cn/image-20210308155126545.png)![image-20210308155159141](http://myapp.img.mykernel.cn/image-20210308155159141.png)

### 关于sincedb

```bash
[root@elk ~]# cat /var/lib/logstash/plugins/inputs/file/.sincedb_*
1356343 0 64768 527208 1604375141.152489 /tmp/09-53-32.log
1356343 0 64768 527208 1604374976.417862 /09-53-32.log
34267079 0 64768 470152 1604390438.54313 /tmp/test/07-30-30.log
70090422 0 64768 1391353 1597916574.0671659 /usr/local/nginx/logs/access.log

# 记录了文件inode, 位置
```

重复收集一个文件日志时，需要先删除sincedb

```bash
root@master05:~# rm -f /usr/share/logstash/data/plugins/inputs/file/.sincedb_be886e787cd3c64785be2782b22802bf
```



## logstash收集nginx日志

安装nginx

```bash
[root@elk ~]# wget -O /etc/yum.repos.d/epel-7.repo https://mirrors.aliyun.com/repo/epel-7.repo
[root@elk ~]# yum install -y nginx
[root@elk ~]# systemctl start nginx
[root@elk ~]# curl localhost
[root@elk ~]# tail /var/log/nginx/access.log 
::1 - - [08/Mar/2021:15:54:05 +0800] "GET / HTTP/1.1" 200 4833 "-" "curl/7.29.0" "-"

```

配置json日志输出 /etc/nginx/nginx.conf

```nginx
server {
	...
	log_format access_json escape=json '{"@timestamp":"$time_iso8601",'
     '"host":"$server_addr",'
     '"clientip":"$remote_addr",'
     '"size":$body_bytes_sent,'
     '"responsetime":$request_time,'
     '"upstreamtime":"$upstream_response_time",'
     '"upstreamhost":"$upstream_addr",'
     '"http_host":"$host",'
     '"url":"$uri",'
     '"domain":"$host",'
     '"xff":"$http_x_forwarded_for",'
     '"tcp-xff":"$proxy_protocol_addr",'
     '"referer":"$http_referer",'
     '"user-agent":"$http_user_agent",'
     '"status":"$status"}';

    access_log  /var/log/nginx/access.log  access_json;   
	...
}
```

```bash
[root@elk ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@elk ~]# nginx -s reload
[root@elk ~]# curl -I localhost/index.html
HTTP/1.1 200 OK
Server: nginx/1.16.1
Date: Mon, 08 Mar 2021 07:57:16 GMT
Content-Type: text/html
Content-Length: 4833
Last-Modified: Fri, 16 May 2014 15:12:48 GMT
Connection: keep-alive
ETag: "53762af0-12e1"
Accept-Ranges: bytes

[root@elk ~]# tail /var/log/nginx/access.log 
::1 - - [08/Mar/2021:15:54:05 +0800] "GET / HTTP/1.1" 200 4833 "-" "curl/7.29.0" "-"
{"@timestamp":"2021-03-08T15:57:16+08:00","host":"::1","clientip":"::1","size":0,"responsetime":0.000,"upstreamtime":"","upstreamhost":"","http_host":"localhost","url":"/index.html","domain":"localhost","xff":"","tcp-xff":"","referer":"","user-agent":"curl/7.29.0","status":"200"}
```

访问nginx日志

```bash
[root@elk conf.d]# cat /etc/logstash/conf.d/nginx.conf
input {
  file {
    path => "/var/log/nginx/access.log"         # 日志路径 
    type => "nginx-accesslog"                 # 事件类型
    start_position => "end"       # 首次收集日志的位置
	stat_interval => "3"                # 查看文件是否修改
	codec => json                   # 将json的每个元互相转换为一个事件
  }

}


output {
	if [type] == "nginx-accesslog" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-nginx-accesslog-%{+YYYY.MM.dd}"     # 地理位置索引，必须需要nginx的索引名为logstash起始
		}
		stdout{ codec => rubydebug }
	}
}
```

> https://www.elastic.co/guide/en/logstash/7.11/plugins-codecs-json.html

```bash
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf 
```

访问nginx http://192.168.0.171

控制台输出如下格式

```bash
{
            "size" => 82896,
      "@timestamp" => 2021-03-08T08:07:22.000Z,
             "url" => "/img/header-background.png",
        "clientip" => "192.168.0.33",
    "upstreamhost" => "",
    "upstreamtime" => "",
    "responsetime" => 0.004,
          "domain" => "192.168.0.171",
         "referer" => "http://192.168.0.171/",
            "path" => "/var/log/nginx/access.log",
            "type" => "nginx-accesslog",
         "tcp-xff" => "",
       "http_host" => "192.168.0.171",
             "xff" => "",
      "user-agent" => "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.25 Safari/537.36 Core/1.70.3861.400 QQBrowser/10.7.4313.400",
          "status" => "200",
            "host" => "192.168.0.171",
        "@version" => "1"
}
```

![image-20210308160903871](http://myapp.img.mykernel.cn/image-20210308160903871.png)![image-20210308161016228](http://myapp.img.mykernel.cn/image-20210308161016228.png)

## logstash收集tcp日志

通过logstash 的 tcp /udp 插件收集日志 通常用于在 向 elasticsearch 日志补录丢失的部分日志， 可以 将丢失的日志 写到 一个文件，然后通过 TCP 日志 收集方式直接 发送 给 lo gstash 然后 再写入 到 elasticsearch 服务器。

https://www.elastic.co/guide/en/logstash/current/input-plugins.html

```bash
[root@elk conf.d]# cat /etc/logstash/conf.d/tcp.conf
input {
	tcp {
		port => 9889
		type => "tcplog"
		mode => "server"
	}
}


output {
	if [type] == "tcplog" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "tcplog-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
}
```

```bash
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/tcp.conf 
```

另一个主机推送日志

```bash
root@master01:~# echo "nc test" | nc 192.168.0.171 9889


# logstash输出
{
          "host" => "192.168.0.27",
    "@timestamp" => 2021-03-08T08:15:41.218Z,
          "port" => 14543,
          "type" => "tcplog",
       "message" => "nc test",
      "@version" => "1"
}


# 发送文件
root@master01:~# nc 192.168.0.171 9889 < /etc/passwd

# logstash输出, 每行发送一个
{
          "host" => "192.168.0.27",
    "@timestamp" => 2021-03-08T08:16:22.833Z,
          "port" => 15017,
          "type" => "tcplog",
       "message" => "dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin",
      "@version" => "1"
}
{
          "host" => "192.168.0.27",
    "@timestamp" => 2021-03-08T08:16:22.833Z,
          "port" => 15017,
          "type" => "tcplog",
       "message" => "jack:x:1000:1000:jack,,,:/home/jack:/bin/bash",
      "@version" => "1"
}

# 通过伪设备发送
root@master01:~# echo "nc test"  > /dev/tcp/192.168.0.171/9889
{
          "host" => "192.168.0.27",
    "@timestamp" => 2021-03-08T08:18:24.251Z,
          "port" => 16395,
          "type" => "tcplog",
       "message" => "nc test",
      "@version" => "1"
}

```

kibana添加索引

![image-20210308161912697](http://myapp.img.mykernel.cn/image-20210308161912697.png)![image-20210308161937448](http://myapp.img.mykernel.cn/image-20210308161937448.png)

## 收集haproxy日志

```bash
[root@elk conf.d]# systemctl stop nginx # 先停了以上使用的nginx

[root@elk conf.d]# yum -y install haproxy
```

```diff
[root@elk conf.d]# cat /etc/haproxy/haproxy.cfg
global
    maxconn 100000 
    spread-checks 2 
    chroot /usr/local/haproxy 
    nbproc 4   
    cpu-map 1 0 
    cpu-map 2 1
    cpu-map 3 2
    cpu-map 4 3
    stats socket /var/lib/haproxy/haproxy.sock1 mode 600 level admin process 1 
    stats socket /var/lib/haproxy/haproxy.sock2 mode 600 level admin process 2 
    stats socket /var/lib/haproxy/haproxy.sock3 mode 600 level admin process 3 
    stats socket /var/lib/haproxy/haproxy.sock4 mode 600 level admin process 4 
    user haproxy          
    group haproxy         
    daemon                
    pidfile /var/lib/haproxy/haproxy.pid
+    log 127.0.0.1 local6 info 
defaults
    option http-keep-alive
    option forwardfor
    option redispatch              
    timeout http-keep-alive 1ms       
    mode http
    timeout connect 10s             
    timeout client 1m       
    timeout server 1m     
listen stats    
  bind :9999
+  log global 
  stats enable
  stats hide-version
  stats uri /haproxy
  stats realm HAPorxy\ Stats\ Page
  stats auth admin:123456
  stats auth haadmin:123456
  stats admin if TRUE
```

```bash
[root@elk conf.d]# install -dv /usr/local/haproxy
[root@elk conf.d]# systemctl restart haproxy
```

配置日志 `/etc/rsyslog.conf `

```bash

# Provides UDP syslog reception
$ModLoad imudp
$UDPServerRun 514

# Provides TCP syslog reception
$ModLoad imtcp
$InputTCPServerRun 514

local6.*                                                /var/log/haproxy.log

```

```bash
[root@elk conf.d]# systemctl restart haproxy rsyslog

[root@elk conf.d]# cat /var/log/haproxy.log  # 表示日志正常
Mar  8 16:33:32 localhost haproxy[22102]: Proxy stats started.
Mar  8 16:35:14 localhost haproxy[22336]: Proxy stats started. 
```

配置将日志写入logstash

```bash
local6.*                                                @@192.168.0.171:516
```

配置logstash

https://www.elastic.co/guide/en/logstash/current/plugins-inputs-syslog.html

```bash
[root@elk conf.d]# cat /etc/logstash/conf.d/haproxy.conf
input {
	syslog {
		port => 516
		type => "system-syslog"
	}
}


output {
	if [type] == "system-syslog" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "system-syslog-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
}
```

```bash
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/haproxy.conf 


# 重启haproxy
[root@elk conf.d]# systemctl restart haproxy
# 访问 http://192.168.0.171:9999/haproxy
```

```bash
# 控制台输出
{
        "@timestamp" => 2021-03-08T08:50:03.000Z,
    "facility_label" => "local6",
          "priority" => 182,
          "severity" => 6,
              "host" => "192.168.0.171",
              "type" => "system-syslog",
         "logsource" => "localhost",
          "@version" => "1",
               "pid" => "23902",
    "severity_label" => "Informational",
         "timestamp" => "Mar  8 16:50:03",
           "message" => "Connect from 192.168.0.33:57412 to 192.168.0.171:9999 (stats/HTTP)\n",
          "facility" => 22,
           "program" => "haproxy"
}
```

![image-20210308165106307](http://myapp.img.mykernel.cn/image-20210308165106307.png)![image-20210308165128960](http://myapp.img.mykernel.cn/image-20210308165128960.png)

​         

> 可以看出自动解析为syslog格式

## logstash->redis->logstash->es

用一台 服务器按照 部署 redis 服务， 专门用于日志缓存使用 用于 **web 服务器产生大量日志的场景** 例如下面的服务器内存即将被使用完毕，查看是因为 redis
服务保存 了大量的数据 没有 被读取 而 占用了大量的内存空间。

部署redis

```bash
root@master03:~# wget https://download.redis.io/releases/redis-3.2.12.tar.gz
root@master03:~# tar xvf redis-3.2.12.tar.gz -C /usr/local/src/
root@master03:~# cd /usr/local/src/redis-3.2.12/
root@master03:/usr/local/src/redis-3.2.12# make PREFIX=/apps/redis install
root@master03:~# install -dv /apps/redis/{bin,etc,logs,data,run}
root@master03:/usr/local/src/redis-3.2.12# cp redis.conf /apps/redis/etc/
cat >  /lib/systemd/system/redis.service <<'EOF'
[Unit]
Description=Redis persistent key-value database
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/apps/redis/bin/redis-server /apps/redis/etc/redis.conf --supervised systemd
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
Type=notify
User=redis
Group=redis
RuntimeDirectory=redis
RuntimeDirectoryMode=0755
# ubuntu必须设定
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
root@master03:/usr/local/src/redis-3.2.12# useradd -r redis
root@master03:/usr/local/src/redis-3.2.12# chown -R redis.redis /apps/redis/



root@master03:/usr/local/src/redis-3.2.12# grep '^[a-Z]' /apps/redis/etc/redis.conf
bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 1024
requirepass linux48
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /apps/redis/run/redis_6379.pid
loglevel notice
logfile "/apps/redis/logs/redis_6379.log"
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error no
rdbcompression yes
rdbchecksum yes
dbfilename dump_6379.rdb
dir /apps/redis/data/ 
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync yes
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay yes
repl-backlog-size 512mb
slave-priority 100
maxclients 100000
maxmemory 1044905984
appendonly yes
appendfilename "appendonly_6379.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
cluster-enabled no
cluster-config-file /apps/redis/etc/nodes-6379.conf
cluster-slave-validity-factor 10
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes

```

```bash
root@master03:/usr/local/src/redis-3.2.12# systemctl restart redis
root@master03:/usr/local/src/redis-3.2.12# lsof -ni:6379
COMMAND     PID  USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME
redis-ser 21426 redis    4u  IPv4 3249790      0t0  TCP *:6379 (LISTEN)


root@master03:/usr/local/src/redis-3.2.12# ln -sv /apps/redis/bin/* /usr/local/bin

root@master03:/usr/local/src/redis-3.2.12# redis-cli  -h 127.0.0.1
127.0.0.1:6379> select 0
(error) NOAUTH Authentication required.
127.0.0.1:6379> auth linux48
OK
127.0.0.1:6379> select 0
OK
127.0.0.1:6379> select 1
OK
127.0.0.1:6379[1]> select 16
(error) ERR invalid DB index
127.0.0.1:6379[1]> select 15
OK
127.0.0.1:6379[15]> dbsize # 几个key
(integer) 0
127.0.0.1:6379[15]> set m 1
OK
127.0.0.1:6379[15]> dbsize
(integer) 1
127.0.0.1:6379[15]> keys * # 列出key
1) "m"
127.0.0.1:6379[15]> flushdb # 清空key
OK
127.0.0.1:6379[15]> keys *
(empty list or set)
127.0.0.1:6379[15]> set m 2
OK
127.0.0.1:6379[15]> bgsave
Background saving started
127.0.0.1:6379[15]> exit
root@master03:/usr/local/src/redis-3.2.12# ls /apps/redis/data/
appendonly_6379.aof  dump_6379.rdb
```

### 配置logstash写redis

```bash
[root@elk conf.d]# cat /etc/logstash/conf.d/nginx.conf 
input {
  file {
    path => "/var/log/nginx/access.log"         # 日志路径 
    type => "nginx-accesslog"                 # 事件类型
    start_position => "end"       # 首次收集日志的位置
	stat_interval => "3"                # 查看文件是否修改
	codec => json                   # 将json的每个元互相转换为一个事件
  }
}

output {
	if [type] == "nginx-accesslog" {
	    redis {
			data_type => "list"
			key => "nginx-accesslog"
			host => "192.168.0.79"
			port => 6379
			db => "13"
			password => "linux48"
		}
		stdout{ codec => rubydebug }
	}
}
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf 
```

```bash
[root@elk conf.d]# systemctl stop haproxy
[root@elk conf.d]# systemctl start nginx
```

访问nginx http://192.168.0.171/

```bash
# logstash已经 可以看到日志
{
            "size" => 3650,
            "path" => "/var/log/nginx/access.log",
    "upstreamtime" => "",
    "responsetime" => 0.0,
             "url" => "/404.html",
      "@timestamp" => 2021-03-08T09:15:38.000Z,
             "xff" => "",
      "user-agent" => "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.25 Safari/537.36 Core/1.70.3861.400 QQBrowser/10.7.4313.400",
          "domain" => "192.168.0.171",
       "http_host" => "192.168.0.171",
        "@version" => "1",
        "clientip" => "192.168.0.33",
    "upstreamhost" => "",
         "tcp-xff" => "",
            "type" => "nginx-accesslog", # TYPE
         "referer" => "",
          "status" => "404",
            "host" => "192.168.0.171"
}


# redis检验
root@master03:/usr/local/src/redis-3.2.12# redis-cli  -h 127.0.0.1
127.0.0.1:6379> auth linux48
OK
127.0.0.1:6379> select 13
OK
127.0.0.1:6379[13]> dbsize
(integer) 1
127.0.0.1:6379[13]> keys *
1) "nginx-accesslog"

127.0.0.1:6379[13]> LPOP nginx-accesslog # 第一个元素，和上面的logstash一致
"{\"size\":3650,\"path\":\"/var/log/nginx/access.log\",\"upstreamtime\":\"\",\"responsetime\":0.0,\"url\":\"/404.html\",\"@timestamp\":\"2021-03-08T09:15:38.000Z\",\"xff\":\"\",\"user-agent\":\"Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.25 Safari/537.36 Core/1.70.3861.400 QQBrowser/10.7.4313.400\",\"domain\":\"192.168.0.171\",\"http_host\":\"192.168.0.171\",\"@version\":\"1\",\"clientip\":\"192.168.0.33\",\"upstreamhost\":\"\",\"tcp-xff\":\"\",\"type\":\"nginx-accesslog\",\"referer\":\"\",\"status\":\"404\",\"host\":\"192.168.0.171\"}"
# 注意type
```

### logstash读redis

```bash
[root@elk conf.d]# cat /etc/logstash/conf.d/redis-ngx-es.conf
input {
	    redis {
			data_type => "list"
			key => "nginx-accesslog"
			host => "192.168.0.79"
			port => 6379
			db => "13"
			password => "linux48"

			codec => json                   # 将json的每个元互相转换为一个事件
		}
}

output {
	if [type] == "nginx-accesslog" {
		stdout{ codec => rubydebug }
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-redis-ngx-accesslog-%{+YYYY.MM.dd}"     # 地理位置索引，必须需要nginx的索引名为logstash起始
		}
	}
}

```

```bash
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/redis-ngx-es.conf 
```

可以发现一下把redis数据读完了

```bash
{
    "upstreamhost" => "",
             "url" => "/404.html",
            "size" => 3650,
         "referer" => "",
            "type" => "nginx-accesslog",
        "clientip" => "192.168.0.33",
    "responsetime" => 0.0,
          "status" => "404",
            "host" => "192.168.0.171",
        "@version" => "1",
      "@timestamp" => 2021-03-08T09:15:43.000Z,
             "xff" => "",
    "upstreamtime" => "",
          "domain" => "192.168.0.171",
         "tcp-xff" => "",
            "path" => "/var/log/nginx/access.log",
       "http_host" => "192.168.0.171",
      "user-agent" => "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.25 Safari/537.36 Core/1.70.3861.400 QQBrowser/10.7.4313.400"
}
...
root@master03:/usr/local/src/redis-3.2.12# redis-cli  -h 127.0.0.1
127.0.0.1:6379> auth linux48
OK
127.0.0.1:6379[13]> keys *
(empty list or set)
```

查看kibana

![image-20210308172355566](http://myapp.img.mykernel.cn/image-20210308172355566.png)

![image-20210308172521860](http://myapp.img.mykernel.cn/image-20210308172521860.png)

注：测试没有 问题 之后 请 将 logstash 使用服务的方式正常启动

```bash
[root@elk ~]# chown -R logstash.logstash  /var/lib/logstash/.lock  /var/lib/logstash/queue /var/lib/logstash/dead_letter_queue /var/log/logstash/
[root@elk ~]# systemctl start logstash
[root@elk ~]# systemctl status logstash
# 会有各种读日志权限问题，所以只有root启动
```

```diff
[root@elk ~]# cat /etc/systemd/system/logstash.service
[Unit]
Description=logstash

[Service]
Type=simple
+User=root
+Group=root
# Load env vars from /etc/default/ and /etc/sysconfig/ if they exist.
# Prefixing the path with '-' makes it try to load, but if the file doesn't
# exist, it continues onward.
EnvironmentFile=-/etc/default/logstash
EnvironmentFile=-/etc/sysconfig/logstash
ExecStart=/usr/share/logstash/bin/logstash "--path.settings" "/etc/logstash"
Restart=always
WorkingDirectory=/
Nice=19
LimitNOFILE=16384

# When stopping, how long to wait before giving up and sending SIGKILL?
# Keep in mind that SIGKILL on a process can cause data loss.
TimeoutStopSec=infinity

[Install]
WantedBy=multi-user.target
```

```bash
[root@elk ~]# systemctl daemon-reload
[root@elk ~]# systemctl restart logstash
```



## filebeat->redis->logstash->es (架构1)

Filebeat 是 轻量级单用途的日志收集工具， 用于 在 没有安装 java 的服务器上专门收集日志， 可以 将日志转发到 logstash 、 elasticsearch 或 redis 及 kafka 等场景中进行下一步处理。

### filebeat收集系统日志

下载地址 https://www.elastic.co/cn/downloads/past-releases/filebeat-7-11-1

```bash
[root@elk ~]# wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.11.1-x86_64.rpm
[root@elk ~]# yum install ./filebeat-7.11.1-x86_64.rpm  -y
```

#### 输出到文件

```diff
[root@elk ~]# grep -v '#' /etc/filebeat/filebeat.yml | grep -v '^$'
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/*.log
    - /var/log/messages
  exclude_lines: ['^DBG']
  exclude_files: ['.gz$']
  fields:
    type: system-log-filebeat
    level: debug
    review: 1
- type: filestream
  enabled: false
  paths:
    - /var/log/*.log
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
+output.file:
  path: "/tmp/"
  filename: filebeat
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
```



https://www.elastic.co/guide/en/beats/filebeat/current/file-output.html

```bash
 systemctl restart filebeat
[root@elk ~]# tail -f /tmp/filebeat
```

#### 输出到redis

```diff
[root@elk ~]# grep -v '#' /etc/filebeat/filebeat.yml | grep -v '^$'
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/*.log
    - /var/log/messages
  exclude_lines: ['^DBG']
  exclude_files: ['.gz$']
  fields:
    type: system-log-filebeat
    level: debug
    review: 1
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
  exclude_lines: ['^DBG']
  exclude_files: ['.gz$']
  fields:
    type: nginxaccess-log-filebeat
    level: debug
    review: 1
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
+output.redis:
  hosts: ["192.168.0.79"]
  password: "linux48"
  key: "systemlog-filebeat"
  db: 13
  timeout: 5
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
```

```bash
 systemctl restart filebeat
 
 
 root@master03:/usr/local/src/redis-3.2.12# redis-cli  -h 127.0.0.1
127.0.0.1:6379> auth linux48
OK
127.0.0.1:6379> select 13
OK
127.0.0.1:6379[13]> dbsize
(integer) 1
127.0.0.1:6379[13]> keys *
1) "systemlog-filebeat"
```

https://www.elastic.co/guide/en/beats/filebeat/current/redis-output.html

### 配置logstash读redis写es

```bash
[root@elk ~]# cat /etc/logstash/conf.d/system-log.conf
input {
	    redis {
			data_type => "list"
			key => "systemlog-filebeat"
			host => "192.168.0.79"
			port => 6379
			db => "13"
			password => "linux48"

			#codec => json                   # 将json的每个元互相转换为一个事件# 由于以上直接filebeat写的redis, 所以不能json化
		}  

  file {
    path => "/var/log/secure"         # 日志路径 
    type => "securelog"                 # 事件类型
    start_position => "beginning"       # 首次收集日志的位置
	stat_interval => "3"                # 查看文件是否修改
  }
}


output {
	if [fields][type] == "system-log-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "system-log-filebeat-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
	if [fields][type] == "nginxaccess-log-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-nginxaccess-log-filebeat-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
	if [type] == "securelog" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "secure-log-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
}
```

```bash
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/system-log.conf 
[root@elk conf.d]# echo '1122AAA1111111123213' >> /var/log/messages

{
         "input" => {
        "type" => "log"
    },
          "host" => {
                 "name" => "elk.youwyouqu.io",
                   "ip" => [
            [ 0] "192.168.0.171",
            [ 1] "192.168.0.3",
            [ 2] "fe80::2cf:e0ff:fe4c:7dc1",
            [ 3] "172.19.0.1",
            [ 4] "172.21.0.1",
            [ 5] "172.18.0.1",
            [ 6] "172.20.0.1",
            [ 7] "fe80::42:7ff:fea4:2c11",
            [ 8] "172.17.0.1",
            [ 9] "fe80::42:7cff:fe0e:55",
            [10] "172.22.0.1",
            [11] "fe80::42:11ff:fedd:eb1e",
            [12] "10.244.6.0",
            [13] "fe80::3403:c1ff:fea2:c89b",
            [14] "fe80::424:cbff:fe21:25e7",
            [15] "fe80::5c39:9aff:fe06:42be"
        ],
                  "mac" => [
            [0] "00:cf:e0:4c:7d:c1",
            [1] "02:42:16:c3:c7:fe",
            [2] "02:42:94:ef:84:e3",
            [3] "02:42:bc:3d:fa:10",
            [4] "02:42:07:a4:2c:11",
            [5] "02:42:7c:0e:00:55",
            [6] "02:42:11:dd:eb:1e",
            [7] "36:03:c1:a2:c8:9b",
            [8] "06:24:cb:21:25:e7",
            [9] "5e:39:9a:06:42:be"
        ],
                   "id" => "3c8fc19cc4d44b57954359dc1938f400",
             "hostname" => "elk.youwyouqu.io",
        "containerized" => false,
                   "os" => {
                "name" => "CentOS Linux",
            "platform" => "centos",
              "kernel" => "3.10.0-1127.el7.x86_64",
              "family" => "redhat",
             "version" => "7 (Core)",
            "codename" => "Core"
        },
         "architecture" => "x86_64"
    },
           "ecs" => {
        "version" => "1.6.0"
    },
         "agent" => {
                "name" => "elk.youwyouqu.io",
                "type" => "filebeat",
                  "id" => "c4c629c6-5970-4ef7-9d7a-65c1d96a8548",
        "ephemeral_id" => "30cee5a6-0bc4-4a66-9e37-7677f61a2a93",
            "hostname" => "elk.youwyouqu.io",
             "version" => "7.11.1"
    },
           "log" => {
        "offset" => 278608903,
          "file" => {
            "path" => "/var/log/messages"
        }
    },
    "@timestamp" => 2021-03-08T10:09:39.794Z,
        "fields" => {
         "level" => "debug",
        "review" => 1,
          "type" => "system-log-filebeat"
    },
      "@version" => "1",
       "message" => "1122AAA1111111123213"
}


{
         "input" => {
        "type" => "log"
    },
        "fields" => {
        "review" => 1,
          "type" => "nginxaccess-log-filebeat",
         "level" => "debug"
    },
           "ecs" => {
        "version" => "1.6.0"
    },
         "agent" => {
        "ephemeral_id" => "d0e5da5a-5520-4abb-a8e8-32221cf1d7fd",
                "type" => "filebeat",
                "name" => "elk.youwyouqu.io",
            "hostname" => "elk.youwyouqu.io",
             "version" => "7.11.1",
                  "id" => "c4c629c6-5970-4ef7-9d7a-65c1d96a8548"
    },
    "@timestamp" => 2021-03-08T10:22:37.822Z,
          "host" => {
                  "mac" => [
            [0] "00:cf:e0:4c:7d:c1",
            [1] "02:42:16:c3:c7:fe",
            [2] "02:42:94:ef:84:e3",
            [3] "02:42:bc:3d:fa:10",
            [4] "02:42:07:a4:2c:11",
            [5] "02:42:7c:0e:00:55",
            [6] "02:42:11:dd:eb:1e",
            [7] "36:03:c1:a2:c8:9b",
            [8] "06:24:cb:21:25:e7",
            [9] "5e:39:9a:06:42:be"
        ],
         "architecture" => "x86_64",
             "hostname" => "elk.youwyouqu.io",
                 "name" => "elk.youwyouqu.io",
        "containerized" => false,
                   "os" => {
              "family" => "redhat",
              "kernel" => "3.10.0-1127.el7.x86_64",
                "name" => "CentOS Linux",
            "platform" => "centos",
             "version" => "7 (Core)",
            "codename" => "Core"
        },
                   "id" => "3c8fc19cc4d44b57954359dc1938f400",
                   "ip" => [
            [ 0] "192.168.0.171",
            [ 1] "192.168.0.3",
            [ 2] "fe80::2cf:e0ff:fe4c:7dc1",
            [ 3] "172.19.0.1",
            [ 4] "172.21.0.1",
            [ 5] "172.18.0.1",
            [ 6] "172.20.0.1",
            [ 7] "fe80::42:7ff:fea4:2c11",
            [ 8] "172.17.0.1",
            [ 9] "fe80::42:7cff:fe0e:55",
            [10] "172.22.0.1",
            [11] "fe80::42:11ff:fedd:eb1e",
            [12] "10.244.6.0",
            [13] "fe80::3403:c1ff:fea2:c89b",
            [14] "fe80::424:cbff:fe21:25e7",
            [15] "fe80::5c39:9aff:fe06:42be"
        ]
    },
           "log" => {
          "file" => {
            "path" => "/var/log/nginx/access.log"
        },
        "offset" => 21463
    },
       "message" => "{\"@timestamp\":\"2021-03-08T18:22:35+08:00\",\"host\":\"192.168.0.171\",\"clientip\":\"192.168.0.33\",\"size\":0,\"responsetime\":0.000,\"upstreamtime\":\"\",\"upstreamhost\":\"\",\"http_host\":\"192.168.0.171\",\"url\":\"/index.html\",\"domain\":\"192.168.0.171\",\"xff\":\"\",\"tcp-xff\":\"\",\"referer\":\"\",\"user-agent\":\"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.66 Safari/537.36\",\"status\":\"304\"}",
      "@version" => "1"
}

# json解码失败
```

![image-20210308181107161](http://myapp.img.mykernel.cn/image-20210308181107161.png)

![image-20210308181114284](http://myapp.img.mykernel.cn/image-20210308181114284.png)



![image-20210308181136533](http://myapp.img.mykernel.cn/image-20210308181136533.png)

nginx

![image-20210308182442382](http://myapp.img.mykernel.cn/image-20210308182442382.png)

![image-20210308182512904](http://myapp.img.mykernel.cn/image-20210308182512904.png)



### filebeat收集nginx日志

直接在pod中测试，先缩减pod为1，进入pod调整filebeat日志

![image-20210314095541133](http://myapp.img.mykernel.cn/image-20210314095541133.png)

![image-20210314095549582](http://myapp.img.mykernel.cn/image-20210314095549582.png)

```diff
/ # cat /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml 
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /apps/nginx/logs/access.log
  exclude_lines: ['^DBG','kube-probe/1.20','Prometheus/2.25.0']
  exclude_files: ['.gz$']
  fields:
    type: nginx-accesslog-filebeat
    level: debug
    review: 1
- type: log
  enabled: true
  paths:
    - /apps/nginx/logs/error.log
  exclude_lines: ['^DBG','kube-probe/1.20','Prometheus/2.25.0']
  exclude_files: ['.gz$']
  fields:
    type: nginx-errorlog-filebeat
    level: debug
    review: 1
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
+output.redis:
  hosts: ["192.168.0.79"]
  password: "linux48"
  key: "ngx-filebeat"
  db: 5
  timeout: 5
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~

```

重载配置

```bash
/ # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 {run.sh} /bin/sh /usr/local/bin/run.sh
   14 root      0:00 nginx: master process /apps/nginx/sbin/nginx
   15 root      0:16 nginx: worker process
   16 root      0:15 nginx: worker process
   17 root      0:17 nginx: worker process
   18 root      0:08 tail -f /etc/hosts
   19 root      0:16 nginx: worker process
   20 root      0:18 nginx: worker process
   21 root      0:16 nginx: worker process
   22 root      0:16 nginx: worker process
   23 root      0:14 nginx: worker process
   57 root      0:00 sh
   69 root      0:08 /usr/local/filebeat-7.11.1-linux-x86_64/filebeat --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c /usr/local/filebeat-7.
   82 root      0:00 sh
   90 root      0:00 sh
   97 root      0:00 ps -ef
/ # kill -9 69


# 前台启动
/ # /usr/local/filebeat-7.11.1-linux-x86_64/filebeat   --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c  /usr/local/filebeat-7.11.1-linux-x86_
64/filebeat.yml --path.data /usr/local/filebeat-7.11.1-linux-x86_64/data/ --path.home /usr/local/filebeat-7.11.1-linux-x86_64/ --path.logs /usr/local/f
ilebeat-7.11.1-linux-x86_64/logs -e
```

Redis查看5库的KEY

![image-20210314095756143](http://myapp.img.mykernel.cn/image-20210314095756143.png)

chrome请求网页, 显示一共37个nginx请求。

![image-20210314095819134](http://myapp.img.mykernel.cn/image-20210314095819134.png)

查看filebeat日志，已经连接redis

![image-20210314095837242](http://myapp.img.mykernel.cn/image-20210314095837242.png)



查看库

![img](http://myapp.img.mykernel.cn/clip_image002.jpg)

正常

 

### 配置logstash直接读redis, 解析和写es

```bash
root@localhost:~# cp  /etc/logstash/conf.d/nginx.conf{,.bak}
```



#### 先以json读,写标准输出和文件

```diff
root@localhost:~# cat /etc/logstash/conf.d/nginx.conf
input {
	    redis {
			data_type => "list"
+			key => "ngx-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
+			db => "5"
			password => "linux48"
			codec => json                   # 将json的每个元互相转换为一个事件
		}  
}

filter {
	if [fields][type] == "nginx-accesslog-filebeat" {
		geoip {
			source => "xff"
			target => "geoip"
			database => "/etc/logstash/GeoLite2-City_20210302/GeoLite2-City.mmdb"
			add_field => ["[geoip][coordinates]","%{[geoip][longitude]}"]
			add_field => ["[geoip][coordinates]","%{[geoip][latitude]}"]
		}
		mutate {
			convert => ["[geoip][coordinates]","float"]
		}
	}
}

output {
	if [fields][type] == "nginx-accesslog-filebeat" {
+		stdout{ codec => rubydebug }
+		file { path => "/tmp/ngx-access-log-%{+YYYY.MM.dd}" }
	}
	if [fields][type] == "nginx-errorlog-filebeat" {
		stdout{ codec => rubydebug }
		file { path => "/tmp/ngx-error-log-%{+YYYY.MM.dd}" }
	}
}
root@localhost:~# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf
```

结果，ngx日志文件, 数目正常

```bash
root@localhost:~# wc -l /tmp/ngx-access-log-2021.03.14 
37 /tmp/ngx-access-log-2021.03.14
```

![image-20210314100138924](http://myapp.img.mykernel.cn/image-20210314100138924.png)

但是标准输出的nginx日志并未解析

#### json模块解析nginx日志

```diff
root@localhost:~# cat /etc/logstash/conf.d/nginx.conf
input {
	    redis {
			data_type => "list"
			key => "ngx-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "5"
			password => "linux48"
			codec => json                   # 将json的每个元互相转换为一个事件
		}  
}

filter {
	if [fields][type] == "nginx-accesslog-filebeat" {
+	    json {
+			source => "message"
+		}
		geoip {
			source => "xff"
			target => "geoip"
			database => "/etc/logstash/GeoLite2-City_20210302/GeoLite2-City.mmdb"
			add_field => ["[geoip][coordinates]","%{[geoip][longitude]}"]
			add_field => ["[geoip][coordinates]","%{[geoip][latitude]}"]
		}
		mutate {
			convert => ["[geoip][coordinates]","float"]
+			 remove_field => ["message"]
		}
	}
}

output {
	if [fields][type] == "nginx-accesslog-filebeat" {
		stdout{ codec => rubydebug }
		file { path => "/tmp/ngx-access-log-%{+YYYY.MM.dd}" }
	}
	if [fields][type] == "nginx-errorlog-filebeat" {
		stdout{ codec => rubydebug }
		file { path => "/tmp/ngx-error-log-%{+YYYY.MM.dd}" }
	}
}

#再次启动logstash
root@localhost:~# rm -f /tmp/ngx-access-log-2021.03.14
root@localhost:~# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf > /tmp/logstash.log

#再次来37个请求
root@localhost:~# less /tmp/logstash.log
```

![image-20210314100340898](http://myapp.img.mykernel.cn/image-20210314100340898.png)

可以观察到多了message字段 

更新配置文件

```diff
root@localhost:~# cat /etc/logstash/conf.d/nginx.conf
input {
	    redis {
			data_type => "list"
			key => "ngx-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "5"
			password => "linux48"
			codec => json                   # 将json的每个元互相转换为一个事件
		}  
}

filter {
	if [fields][type] == "nginx-accesslog-filebeat" {
	    json {
			source => "message"
		}
		geoip {
			source => "xff"
			target => "geoip"
			database => "/etc/logstash/GeoLite2-City_20210302/GeoLite2-City.mmdb"
			add_field => ["[geoip][coordinates]","%{[geoip][longitude]}"]
			add_field => ["[geoip][coordinates]","%{[geoip][latitude]}"]
		}
		mutate {
			convert => ["[geoip][coordinates]","float"]
+			 remove_field => ["message"]
		}
	}
}

output {
	if [fields][type] == "nginx-accesslog-filebeat" {
		stdout{ codec => rubydebug }
		file { path => "/tmp/ngx-access-log-%{+YYYY.MM.dd}" }
	}
	if [fields][type] == "nginx-errorlog-filebeat" {
		stdout{ codec => rubydebug }
		file { path => "/tmp/ngx-error-log-%{+YYYY.MM.dd}" }
	}
}

root@localhost:~# rm -f /tmp/ngx-access-log-2021.03.14
root@localhost:~# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf > /tmp/logstash.log 
```

现在再请求,Message字段已经移除，并且nginx日志个数 

![image-20210314100512502](http://myapp.img.mykernel.cn/image-20210314100512502.png)

```bash
root@localhost:~# wc -l /tmp/ngx-access-log-2021.03.14 
37 /tmp/ngx-access-log-2021.03.14  # 正常
```





## filebeat->logstash->redis->logstash->es (架构2)

### filbeat 写 logstash

```diff
[root@elk conf.d]#  grep -v '#' /etc/filebeat/filebeat.yml | grep -v '^$'
filebeat.inputs:
+- type: log
  enabled: true
  paths:
    - /var/log/*.log
    - /var/log/messages
  exclude_lines: ['^DBG']
  exclude_files: ['.gz$']
  fields:
    type: system-log-filebeat
    level: debug
    review: 1
+- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
  exclude_lines: ['^DBG']
  exclude_files: ['.gz$']
  fields:
    type: nginxaccess-log-filebeat
    level: debug
    review: 1
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
+output.logstash:
  hosts: ["192.168.0.171:5045","192.168.0.172:5045"]
  worker: 1
  compression_level: 3
  loadbalance: true
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
```

### logstash读filebeat日志

```diff
[root@elk ~]# cat /etc/logstash/conf.d/system-log.conf
input {
+  beats {
+	port => 5045
+   codec => "json" 
+  }
  file {
    path => "/var/log/secure"           # 日志路径 
    type => "securelog"                 # 事件类型
    start_position => "beginning"       # 首次收集日志的位置
	stat_interval => "3"                # 查看文件是否修改
  }
}


output {
	if [fields][type] == "system-log-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "system-log-filebeat-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
	if [fields][type] == "nginxaccess-log-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-nginxaccess-log-filebeat-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
	if [type] == "securelog" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "secure-log-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
}

```

```bash
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/system-log.conf 
[root@elk conf.d]# systemctl restart filebeat

```

访问nginx查看日志

```bash
{
    "responsetime" => 0.0,
            "size" => 0,
    "upstreamhost" => "",
          "domain" => "192.168.0.171",
             "xff" => "",
          "fields" => {
        "review" => 1,
          "type" => "nginxaccess-log-filebeat",
         "level" => "debug"
    },
         "referer" => "",
           "input" => {
        "type" => "log"
    },
         "tcp-xff" => "",
           "agent" => {
        "ephemeral_id" => "3e6d30e1-2663-49f1-82b9-544cd6a52e94",
                "name" => "elk.youwyouqu.io",
                "type" => "filebeat",
                  "id" => "c4c629c6-5970-4ef7-9d7a-65c1d96a8548",
             "version" => "7.11.1",
            "hostname" => "elk.youwyouqu.io"
    },
        "clientip" => "192.168.0.33",
       "http_host" => "192.168.0.171",
            "host" => {
                   "ip" => [
            [ 0] "192.168.0.171",
            [ 1] "192.168.0.3",
            [ 2] "fe80::2cf:e0ff:fe4c:7dc1",
            [ 3] "172.19.0.1",
            [ 4] "172.21.0.1",
            [ 5] "172.18.0.1",
            [ 6] "172.20.0.1",
            [ 7] "fe80::42:7ff:fea4:2c11",
            [ 8] "172.17.0.1",
            [ 9] "fe80::42:7cff:fe0e:55",
            [10] "172.22.0.1",
            [11] "fe80::42:11ff:fedd:eb1e",
            [12] "10.244.6.0",
            [13] "fe80::3403:c1ff:fea2:c89b",
            [14] "fe80::424:cbff:fe21:25e7",
            [15] "fe80::5c39:9aff:fe06:42be"
        ],
        "containerized" => false,
                 "name" => "elk.youwyouqu.io",
         "architecture" => "x86_64",
                  "mac" => [
            [0] "00:cf:e0:4c:7d:c1",
            [1] "02:42:16:c3:c7:fe",
            [2] "02:42:94:ef:84:e3",
            [3] "02:42:bc:3d:fa:10",
            [4] "02:42:07:a4:2c:11",
            [5] "02:42:7c:0e:00:55",
            [6] "02:42:11:dd:eb:1e",
            [7] "36:03:c1:a2:c8:9b",
            [8] "06:24:cb:21:25:e7",
            [9] "5e:39:9a:06:42:be"
        ],
                   "id" => "3c8fc19cc4d44b57954359dc1938f400",
             "hostname" => "elk.youwyouqu.io",
                   "os" => {
              "family" => "redhat",
            "platform" => "centos",
                "name" => "CentOS Linux",
              "kernel" => "3.10.0-1127.el7.x86_64",
             "version" => "7 (Core)",
            "codename" => "Core"
        }
    },
        "@version" => "1",
      "user-agent" => "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.25 Safari/537.36 Core/1.70.3861.400 QQBrowser/10.7.4313.400",
            "tags" => [
        [0] "beats_input_codec_json_applied"
    ],
             "ecs" => {
        "version" => "1.6.0"
    },
             "url" => "/index.html",
    "upstreamtime" => "",
          "status" => "304",
             "log" => {
        "offset" => 23634,
          "file" => {
            "path" => "/var/log/nginx/access.log"
        }
    },
      "@timestamp" => 2021-03-08T10:45:22.928Z
}
```

![image-20210308184656684](http://myapp.img.mykernel.cn/image-20210308184656684.png)

### logstash写redis

```diff
[root@elk ~]# cat /etc/logstash/conf.d/system-log.conf
input {
  beats {
	port => 5045
	codec => "json"
  }
  file {
    path => "/var/log/secure"           # 日志路径 
    type => "securelog"                 # 事件类型
    start_position => "beginning"       # 首次收集日志的位置
	stat_interval => "3"                # 查看文件是否修改
  }
}


output {
	if [fields][type] == "system-log-filebeat" {
+	    redis {
+			data_type => "list"
+			key => "systemlog-filebeat"
+			host => "192.168.0.79"
+			port => 6379
+			db => "13"
+			password => "linux48"
+		}
+		stdout{ codec => rubydebug }
	}
	if [fields][type] == "nginxaccess-log-filebeat" {
+	    redis {
+			data_type => "list"
+			key => "nginx-accesslog"
+			host => "192.168.0.79"
+			port => 6379
+			db => "13"
+			password => "linux48"
+		}
+		stdout{ codec => rubydebug }
	}
	if [type] == "securelog" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "secure-log-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
}
```

访问nginx验证key

```bash
127.0.0.1:6379[13]> keys *
1) "nginx-accesslog"
```

### logstash读redis再写es

```bash
[root@elk ~]# cat /etc/logstash/conf.d/nginx.conf
input {
	    redis {
			data_type => "list"
			key => "systemlog-filebeat"
			host => "192.168.0.79"
			port => 6379
			db => "13"
			password => "linux48"

			codec => json                   # 将json的每个元互相转换为一个事件
		}  
	    redis {
			data_type => "list"
			key => "nginx-accesslog"
			host => "192.168.0.79"
			port => 6379
			db => "13"
			password => "linux48"

			codec => json                   # 将json的每个元互相转换为一个事件
		}  
}

output {
	if [fields][type] == "system-log-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "filebeat-logstash-redis-logstash-es-system-log-filebeat-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
	if [fields][type] == "nginxaccess-log-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-filebeat-logstash-redis-logstash-es-nginxaccess-log-filebeat-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
}
```

```bash
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf 
```

现在kibana加索引

![image-20210308190207967](http://myapp.img.mykernel.cn/image-20210308190207967.png)

![image-20210308190221650](http://myapp.img.mykernel.cn/image-20210308190221650.png)

## logstash解析客户端ip

https://dev.maxmind.com/geoip/geoip2/geolite2/

![image-20210310074118264](http://myapp.img.mykernel.cn/image-20210310074118264.png)

![image-20210310074314424](http://myapp.img.mykernel.cn/image-20210310074314424.png)

邮件重置密码

![image-20210310074456569](http://myapp.img.mykernel.cn/image-20210310074456569.png)

下载https://www.maxmind.com/en/accounts/514602/geoip/downloads

![image-20210310074718225](http://myapp.img.mykernel.cn/image-20210310074718225.png)

### 配置地址位置索引库

elasticsearch索引名必须以 `logstash-`开头

在第二个logstash完成此步骤

```bash
root@localhost:~# tar xvf GeoLite2-City_20210302.tar.gz -C /etc/logstash/
GeoLite2-City_20210302/
GeoLite2-City_20210302/README.txt
GeoLite2-City_20210302/COPYRIGHT.txt
GeoLite2-City_20210302/GeoLite2-City.mmdb
GeoLite2-City_20210302/LICENSE.txt

# 位置
root@localhost:~# ls /etc/logstash/GeoLite2-City_20210302/GeoLite2-City.mmdb 
/etc/logstash/GeoLite2-City_20210302/GeoLite2-City.mmdb

```

配置Logstash的nginx的访问日志

```bash
root@localhost:~# cat /etc/logstash/conf.d/nginx.conf
input {
	    redis {
			data_type => "list"
			key => "nginxlog-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "12"
			password => "linux48"
			codec => json                   # 将json的每个元互相转换为一个事件
		}  
}

filter {
	if [fields][type] == "nginx-accesslog-filebeat" {
		geoip {
			source => "clientip"
			target => "geoip"
			database => "/etc/logstash/GeoLite2-City_20210302/GeoLite2-City.mmdb"
			add_field => ["[geoip][coordinates]","%{[geoip][longitude]}"]
			add_field => ["[geoip][coordinates]","%{[geoip][latitude]}"]
		}
		mutate {
			convert => ["[geoip][coordinates]","float"]
		}
	}
}

output {
	if [fields][type] == "nginx-accesslog-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-nginx-accesslog-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
	if [fields][type] == "nginx-errorlog-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-nginx-errorlog-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
}
```

> https://www.elastic.co/guide/en/logstash/current/plugins-filters-geoip.html
>
> https://www.elastic.co/guide/en/logstash/master/plugins-filters-mutate.html

在4.8的架构中，把第二个logstash先移除

第一个logstash启动nginx日志

```bash
 /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/beats-nginx.conf 
```

在redis中查询KEY

```bash
# 有Key先清空
127.0.0.1:6379[11]> SELECT 12
OK
127.0.0.1:6379[12]> KEYS *
1) "nginxlog-filebeat"
127.0.0.1:6379[12]> LLEN nginxlog-filebeat
(integer) 227
127.0.0.1:6379[12]> FLUSHDB
OK
127.0.0.1:6379[12]> LLEN nginxlog-filebeat
(integer) 0
```

F12, 无缓存模式请求 http://test.youwoyouqu.io/

![image-20210310080240186](http://myapp.img.mykernel.cn/image-20210310080240186.png)

```bash
# redis
127.0.0.1:6379[12]> LLEN nginxlog-filebeat
(integer) 34
127.0.0.1:6379[12]> 
```

现在接上第二个logstash

```bash
root@master05:~# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf

#观察控制台输出
{
        "@version" => "1",
         `  "geoip" => {},`
	...
          "domain" => "test.youwoyouqu.io",
             "xff" => "192.168.0.33",
       `    "tags" => [` 
      `  [0] "beats_input_codec_json_applied",` 
      `   [1] "_geoip_lookup_failure"` 
    ],
}
# 注意报错
```

> 注意：**4.9.2**就是测试geoip是否正常，如果正常，只要此步骤在线上，就可以正常解析



### 配置logstash读file, 写stdout

直接在第二个logstash从文件读数据，写到控制台

```bash
# 准备日志, 从nginx-controller-pod中拷贝`正常和错误`日志
{"@timestamp":"2021-03-10T08:02:49+08:00","host":"10.244.5.30","clientip":"192.168.0.27","size":1399,"responsetime":0.000,"upstreamtime":"","upstreamhost":"","http_host":"test.youwoyouqu.io","url":"/static/js/chunk-8a24a96a.36e5ec41.js","domain":"test.youwoyouqu.io","xff":"192.168.0.33","tcp-xff":"","referer":"http://test.youwoyouqu.io/","user-agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.66 Safari/537.36","status":"200"}
{"@timestamp":"2021-03-10T08:02:49+08:00","host":"10.244.5.30","clientip":"192.168.0.27","size":861,"responsetime":0.000,"upstreamtime":"","upstreamhost":"","http_host":"test.youwoyouqu.io","url":"/static/css/chunk-b69033aa.2dc92603.css","domain":"test.youwoyouqu.io","xff":"192.168.0.33","tcp-xff":"","referer":"http://test.youwoyouqu.io/","user-agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.66 Safari/537.36","status":"200"}
{"@timestamp":"2021-03-10T08:02:05+08:00","host":"10.244.5.30","clientip":"192.168.0.27","size":736,"responsetime":0.000,"upstreamtime":"","upstreamhost":"","http_host":"test.youwoyouqu.io","url":"/index.html","domain":"test.youwoyouqu.io","xff":"192.168.0.33","tcp-xff":"","referer":"","user-agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.66 Safari/537.36","status":"200"}
2021/03/09 18:04:32 [error] 26#26: *5242 open() "/apps/home/videoManagement/level_2_2/android_20210303/1134285463163637760.jpg" failed (2: No such file or directory), client: 192.168.0.27, server: test.youwoyouqu.io, request: "GET /videoManagement/level_2_2/android_20210303/1134285463163637760.jpg HTTP/1.1", host: "test.youwoyouqu.io", referrer: "http://test.youwoyouqu.io/videoManagement/level_2_2/auditPassList"
# 把日志的clientip的值修改为公网ip, 写入一个日志文件`/var/log/test.log`

```



```bash
root@localhost:~# cp /etc/logstash/conf.d/nginx.conf{,.bak}
root@localhost:~# vim /etc/logstash/conf.d/nginx.conf 
input {
  file {
    path => "/var/log/test.log"         # 日志路径 
    type => "accesslog"                 # 事件类型
    start_position => "beginning"       # 首次收集日志的位置
	stat_interval => "3"                # 查看文件是否修改
	codec => json                       # 解析json
  }
}

filter {
	if [type] == "accesslog" {
		geoip {
			source => "clientip"
			target => "geoip"
			database => "/etc/logstash/GeoLite2-City_20210302/GeoLite2-City.mmdb"
			add_field => ["[geoip][coordinates]","%{[geoip][longitude]}"]
			add_field => ["[geoip][coordinates]","%{[geoip][latitude]}"]
		}
		mutate {
			convert => ["[geoip][coordinates]","float"]
		}
	}
}

output {
	if [type] == "accesslog" {
		stdout{ codec => rubydebug }
	}
	if [fields][type] == "nginx-errorlog-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-nginx-errorlog-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
}

```

启动

```bash
root@master05:~# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf 


#观察输出，4个输出，错误日志
{
    "@timestamp" => 2021-03-10T00:18:54.965Z,
          "type" => "accesslog",
      "@version" => "1",
          "tags" => [
        [0] "_jsonparsefailure", # JSON解析失败
        [1] "_geoip_lookup_failure" # geoip解析失败
    ],
          "host" => "localhost",
       "message" => "2021/03/09 18:04:32 [error] 26#26: *5242 open() \"/apps/home/videoManagement/level_2_2/android_20210303/1134285463163637760.jpg\" failed (2: No such file or directory), client: 192.168.0.27, server: test.youwoyouqu.io, request: \"GET /videoManagement/level_2_2/android_20210303/1134285463163637760.jpg HTTP/1.1\", host: \"test.youwoyouqu.io\", referrer: \"http://test.youwoyouqu.io/videoManagement/level_2_2/auditPassList\"",
          "path" => "/var/log/test.log"
}


# 正常日志正常
{
      "@timestamp" => 2021-03-10T00:02:49.000Z,
          "status" => "200",
            "path" => "/var/log/test.log",
           "geoip" => {
           "region_code" => "MI",
           "coordinates" => [
            [0] -83.5248,
            [1] 43.0352
        ],
             "city_name" => "Davison",
           "postal_code" => "48423",
         "country_code2" => "US",
                    "ip" => "12.168.0.27",
          "country_name" => "United States",
              "latitude" => 43.0352,
             "longitude" => -83.5248,
         "country_code3" => "US",
              "timezone" => "America/Detroit",
              "dma_code" => 513,
              "location" => {
            "lon" => -83.5248,
            "lat" => 43.0352
        },
           "region_name" => "Michigan",
        "continent_code" => "NA"
    },
    "responsetime" => 0.0,
             "xff" => "192.168.0.33",
            "host" => "10.244.5.30",
            "type" => "accesslog",
             "url" => "/static/js/chunk-8a24a96a.36e5ec41.js",
      "user-agent" => "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.66 Safari/537.36",
        "@version" => "1",
         "tcp-xff" => "",
    "upstreamtime" => "",
       "http_host" => "test.youwoyouqu.io",
            "size" => 1399,
        "clientip" => "12.168.0.27",
    "upstreamhost" => "",
          "domain" => "test.youwoyouqu.io",
         "referer" => "http://test.youwoyouqu.io/"
}
```

### kibana添加地理位置索引

参考：http://blog.mykernel.cn/2020/10/30/kibana%E6%B7%BB%E5%8A%A0%E5%B8%B8%E7%94%A8%E5%9B%BE%E5%BD%A2/

## logstash写mysql

写入数据库的目的是 用于 持久化保存重要 数据 ，比如 状态码 、 客户端 IP 、客户端 浏览器版本 等等 ，用于后期 按月做数据统计等。



### 准备数据库

```bash
MySQL [(none)]> create database elk character set utf8 collate utf8_bin;
Query OK, 1 row affected (0.00 sec)
MySQL [(none)]> grant all privileges on elk.* to 'elk'@'%' identified by '123456';
Query OK, 0 rows affected (0.00 sec)

CREATE TABLE elklog (
	id INTEGER NOT NULL COMMENT '自增ID' AUTO_INCREMENT, 
	host VARCHAR(128) COMMENT '主机', 
	`status` INTEGER COMMENT '状态码', 
	clientip VARCHAR(128) COMMENT '客户端IP', 
	`UserAgent` VARCHAR(512) COMMENT '客户端useragent', 
	create_time TIMESTAMP NOT NULL COMMENT '创建时间', 
	PRIMARY KEY (id)
)
```

测试数据库连接

![image-20210310084303044](http://myapp.img.mykernel.cn/image-20210310084303044.png)

### logstash 配置 mysql connector java 包

MySQL Connector/J 是 MySQL 官方 JDBC 驱动程序 JDBC Java Data Base Connectivity,java 数据库连接）是一种用于执行 SQL 语句的 Java API ，可以为多种
关系数据库提供统一访问，它由一组用 Java 语言编写的类和接口组成。
官方下载
地址： https://dev.mysql.com/downloads/

![image-20210310083913510](http://myapp.img.mykernel.cn/image-20210310083913510.png)

![image-20210310084058030](http://myapp.img.mykernel.cn/image-20210310084058030.png)

![image-20210310084119478](http://myapp.img.mykernel.cn/image-20210310084119478.png)

```bash
root@localhost:~# dpkg -i mysql-connector-java_8.0.23-1ubuntu18.04_all.deb 
root@localhost:~# dpkg -L mysql-connector-java
/.
/usr
/usr/share
/usr/share/doc
/usr/share/doc/mysql-connector-java
/usr/share/doc/mysql-connector-java/CHANGES.gz
/usr/share/doc/mysql-connector-java/INFO_BIN
/usr/share/doc/mysql-connector-java/INFO_SRC
/usr/share/doc/mysql-connector-java/LICENSE.gz
/usr/share/doc/mysql-connector-java/README
/usr/share/doc/mysql-connector-java/changelog.Debian.gz
/usr/share/doc/mysql-connector-java/copyright
/usr/share/java
/usr/share/java/mysql-connector-java-8.0.23.jar # jar包
```

### logstash输出到mysql

https://github.com/theangryangel/logstash-output-jdbc/blob/master/README.md

第二个logstash中完成

```bash
# 列出logstash已经安装的插件
root@localhost:~# /usr/share/logstash/bin/logstash-plugin list

# 配置源
root@localhost:~# apt install gem ruby -y
## 查看
# gem source list


# 不走代理,安装logstash-output-jdbc
root@localhost:~# gem sources --add http://gems.ruby-china.com/ --remove https://rubygems.org/
root@localhost:~# /usr/share/logstash/bin/logstash-plugin install  logstash-output-jdbc
Using bundled JDK: /usr/share/logstash/jdk
OpenJDK 64-Bit Server VM warning: Option UseConcMarkSweepGC was deprecated in version 9.0 and will likely be removed in a future release.
Validating logstash-output-jdbc
Installing logstash-output-jdbc
Installation successful # 安装成功


#走代理安装
[root@elk conf.d]# export https_proxy=http://192.168.0.33:808
[root@elk conf.d]# /usr/share/logstash/bin/logstash-plugin install  logstash-output-jdbc
Using bundled JDK: /usr/share/logstash/jdk
OpenJDK 64-Bit Server VM warning: Option UseConcMarkSweepGC was deprecated in version 9.0 and will likely be removed in a future release.
Validating logstash-output-jdbc
Installing logstash-output-jdbc
Installation successful

```

配置Logstash

```diff
root@localhost:~# cat /etc/logstash/conf.d/nginx.conf
input {
	    redis {
			data_type => "list"
			key => "nginxlog-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "12"
			password => "linux48"
			codec => json                   # 将json的每个元互相转换为一个事件
		}  
}

filter {
	if [fields][type] == "nginx-accesslog-filebeat" {
		geoip {
			source => "clientip"
			target => "geoip"
			database => "/etc/logstash/GeoLite2-City_20210302/GeoLite2-City.mmdb"
			add_field => ["[geoip][coordinates]","%{[geoip][longitude]}"]
			add_field => ["[geoip][coordinates]","%{[geoip][latitude]}"]
		}
		mutate {
			convert => ["[geoip][coordinates]","float"]
		}
	}
}

output {
	if [fields][type] == "nginx-accesslog-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-nginx-accesslog-%{+YYYY.MM.dd}"
		}
+		jdbc {
+           driver_jar_path => "/usr/share/java/mysql-connector-java-8.0.23.jar"
+			connection_string => "jdbc:mysql://192.168.0.171/elk?user=elk&password=123456&useUnicode=true&characterEncoding=UTF8"
+			statement => ["INSERT INTO elklog(http_host,clientip,status,user-agent) VALUES(?,?,?,?)", "http_host","clientip","status","AgentVersion"]
+		}
		stdout{ codec => rubydebug }
	}
	if [fields][type] == "nginx-errorlog-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-nginx-errorlog-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
}
```

> 字段需要和nginx定义的字段一致

第二个logstash还未启动，直接先访问网页生成数据

```bash
127.0.0.1:6379[12]> DBSIZE
(integer) 1
127.0.0.1:6379[12]> KEYS *
1) "nginxlog-filebeat"
127.0.0.1:6379[12]> LLEN nginxlog-filebeat
(integer) 34
```

启动logstash

```bash
root@localhost:~# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf > /tmp/logstash.log
# 另一个终端查看/tmp/logstash.log日志，方便排错
```

mysql日志

![image-20210310093711204](http://myapp.img.mykernel.cn/image-20210310093711204.png)

# Kibana作图

[kibana绘图](http://blog.mykernel.cn/2020/10/30/kibana%E6%B7%BB%E5%8A%A0%E5%B8%B8%E7%94%A8%E5%9B%BE%E5%BD%A2/)

## kibana分享

nginx基于域名统一地址，然后

![image-20210310115743610](http://myapp.img.mykernel.cn/image-20210310115743610.png)

# elk日志清理

elasticsearch存储的日志较多的时候会影响搜索性能，因此建议定期清理

## 获取索引

![image-20210311080130267](http://myapp.img.mykernel.cn/image-20210311080130267.png)

## 测试删除

观察本地环境索引不多，甚至可以保留很多天

所以保留5天

```bash
# 测试删除第2天的数据

## 获取
[root@elk prometheus-2.25.0.linux-amd64]# curl -s -XGET 'http://127.0.0.1:9200/_cat/indices/?v' | grep $(date -d'-1 days' +%Y.%m.%d)  | awk '{printf  "/usr/bin/curl -XDELETE http://127.0.0.1:9200/%s\n",$3}'
/usr/bin/curl -XDELETE http://127.0.0.1:9200/logstash-nginx-errorlog-2021.03.10
/usr/bin/curl -XDELETE http://127.0.0.1:9200/wzx-admin-gateway-filebeat-2021.03.10
/usr/bin/curl -XDELETE http://127.0.0.1:9200/video-server-filebeat-2021.03.10
/usr/bin/curl -XDELETE http://127.0.0.1:9200/auth-server-filebeat-2021.03.10
/usr/bin/curl -XDELETE http://127.0.0.1:9200/video-comment-server-filebeat-2021.03.10
/usr/bin/curl -XDELETE http://127.0.0.1:9200/logstash-nginx-accesslog-2021.03.10
/usr/bin/curl -XDELETE http://127.0.0.1:9200/user-server-filebeat-2021.03.10
/usr/bin/curl -XDELETE http://127.0.0.1:9200/wzx-mobile-gateway-filebeat-2021.03.10


## 删除
[root@elk prometheus-2.25.0.linux-amd64]# curl -s -XGET 'http://127.0.0.1:9200/_cat/indices/?v' | grep $(date -d'-1 days' +%Y.%m.%d)  | awk '{printf  "/usr/bin/curl -XDELETE http://127.0.0.1:9200/%s\n",$3}' | bash
{"acknowledged":true}{"acknowledged":true}{"acknowledged":true}{"acknowledged":true}{"acknowledged":true}{"acknowledged":true}{"acknowledged":true}{"acknowledged":true}[root@elk prometheus-2.25.0.linux-amd64]# 
# 每个true对应删除成功


## 获取, 昨日索引为空
[root@elk prometheus-2.25.0.linux-amd64]# curl -s -XGET 'http://127.0.0.1:9200/_cat/indices/?v' | grep $(date -d'-1 days' +%Y.%m.%d)  | awk '{printf  "/usr/bin/curl -XDELETE http://127.0.0.1:9200/%s\n",$3}' 
```

## cron脚本

编写shell脚本，每天凌晨1点完成清理倒数第10天索引(本地环境数据量小)

```bash
[root@elk prometheus-2.25.0.linux-amd64]# cat /opt/scripts/delete_es.sh
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    1062670898
#Date:                  2021-03-11
#FileName：             /opt/scripts/delete_es.sh
#URL:                   http://www.magedu.com
#Description：          script
#Copyright (C):        2021 All rights reserved
#********************************************************************
curl -s -XGET 'http://127.0.0.1:9200/_cat/indices/?v' | grep $(date -d'-10 days' +%Y.%m.%d)  | awk '{printf  "/usr/bin/curl -XDELETE http://127.0.0.1:9200/%s\n",$3}' | bash
[root@elk prometheus-2.25.0.linux-amd64]# chmod +x /opt/scripts/delete_es.sh
```

```bash
[root@elk prometheus-2.25.0.linux-amd64]# cat /var/spool/cron/root 
# delete elasticsearch index
00 01 * * * /opt/scripts/delete_es.sh > /dev/null
```

# elk集群健康状态检测

https://github.com/justwatchcom/elasticsearch_exporter

```bash
# es节点上
[root@elk conf.d]# docker run --rm --network host -p 9114:9114 justwatch/elasticsearch_exporter:1.1.0
#访问 localhost:9114/metrics
```

prometheus配置 

https://prometheus.io/download/

```bash
[root@elk prometheus-2.25.0.linux-amd64]# grep  -Ev '^#|^[[:space:]]*$|^[[:space:]]*#' prometheus.yml 
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
alerting:
  alertmanagers:
  - static_configs:
    - targets:
rule_files:
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
    
  - job_name: 'elasticsearch-cluster'
    static_configs:
    - targets: ['localhost:9114']
```

grafana配置

https://grafana.com/grafana/download

直接启动

https://grafana.com/grafana/dashboards/13562

下载模块，修改$database 为 ${DS_PROMETHEUS-SYS-01} 

 "timezone": "Asia/Shanghai",

https://www.datadoghq.com/blog/elasticsearch-unassigned-shards/



# logstash监控java错误日志

```bash
input {
  beats {
    port => 5046
  }
}

output {
    if [message] =~ /(exception|ERROR)/ {
        email {
            port => 587
            address => "smtp.qq.com"
            username => "1062670898@qq.com" 
            password => "scyhdklulyrmbdci"
            authentication => "plain"
            contenttype => ""
            from => "1062670898@qq.com"
            subject => "测试环境 %{[fields][type]}错误告警"
            to => "2192383945@qq.com,wangshenaperlin@163.com,1028402351@qq.com"       # 开发邮箱                                
            use_tls => true
            via => "smtp"
            domain => "smtp.qq.com"
            body => "%{[message]}"
            debug => true
        }
    }
}
```

