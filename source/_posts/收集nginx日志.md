---
title: ELK收集nginx日志
date: 2020-12-23 08:06:12
tags: 
- nginx
- elk
---

 elasticsearch-7.6.1-x86_64.rpm  

filebeat-7.6.1-x86_64.rpm 

kibana-7.6.1-x86_64.rpm  

logstash-7.6.1.rpm

# nginx配置日志格式

```bash
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
 '"tcp_xff":"$proxy_protocol_addr",'
 '"referer":"$http_referer",'
 '"user-agent":"$http_user_agent",'
 '"method":"$request",'
 '"request-body":"$request_body",'
# 以下是公司要求获取的自定义首部, 如果使用filebeat自带的模块，要收集自定义的日志, 就比较麻烦。
# '"deviceno":"$http_deviceno",'
# '"model":"$http_model",'
# '"buildversion":"$http_buildversion",'
# '"Authorization":"$http_Authorization",'
 '"status":"$status"}';
 
map $time_iso8601 $logdate {
      '~^(?<ymd>\d{4}-\d{2}-\d{2})' $ymd;
      default 'date-not-found';
}

access_log  /var/log/nginx/access-$logdate.log  access_json;
```

# filebeat收集nginx/jar日志

filebeat收集nginx日志

```bash
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access-*.log
  exclude_lines: ["^DBG","^$"]
  fields:
    type: k8s-nginx-ingress


filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1

output.redis:
  hosts: ["srv-devops-redis.kube-system.svc.weizhixiu.local:6379"]
  key: "k8s-nginx"
  db: 1
  timeout: 5
  password: "xxxxxxx"

```

filebeat收集java日志

```bash
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /apps/logs/app.log
  fields:
    type: k8s-MICROSERVICE

  multiline.pattern: '^\d+-\d+-\d+'
  multiline.negate: true
  multiline.match: "after"

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1

output.redis:
  hosts: ["srv-devops-redis.kube-system.svc.weizhixiu.local:6379"]
  key: "k8s-java"
  db: 1
  timeout: 5
  password: "xxxx"
```



# logstash读redis，输出至elasticsearch

```bash
input {
	redis {
		data_type => "list"
		key => "k8s-nginx"
		host => "172.16.0.222"
		port => "6379"
		db => "1"
		password => "xxxx"
        codec => "json"
	}
	redis {
		data_type => "list"
		key => "k8s-java"
		host => "172.16.0.222"
		port => "6379"
		db => "1"
		password => "xxxx"
        codec => "json"
	}
}
filter {
	if [fields][type] == "k8s-ingress-nginx" {
		 json {
			source => "message"
			remove_field => ["message"]
		}
		json {
			source => "request-body"
			#remove_field => ["request-body"]
		}
		 geoip {
		     # 如果前端获取的日志的nginx前端是http转发：xff；前端是tcp转发: tcp_xff; 如果没有前端: clientip.
			 source => "tcp_xff" 
			 target => "geoip"
			 database => "/etc/logstash/GeoLite2-City_20200317/GeoLite2-City.mmdb"
			 add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
			 add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}" ]
		 }
		 mutate {
		 convert => [ "[geoip][coordinates]", "float"]
		} 
	} 
}
output {

    if [fields][type] == "k8s-ingress-nginx" {
         elasticsearch {
             hosts => ["172.16.0.222:9200"]
             index => "logstash-nginx-k8s-ingress-nginx-%{+YYYY.MM.dd}"
         }
    }
}
```

# kibana从elasticsearch读取，展示

```bash
server.port: 5601
server.host: "127.0.0.1"
elasticsearch.hosts: ["http://172.16.0.222:9200","http://172.16.0.223:9200","http://172.16.0.224:9200"]
i18n.locale: "zh-CN"
logging.dest: /var/log/kibana.log
```

