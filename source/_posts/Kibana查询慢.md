---
title: Kibana查询慢
date: 2020-11-19 03:45:40
tags: 生产小经验
toc: true
---



前期搭建单机的kibana, logstash, redis使用没有问题, 但是随着日志量的增长, 发现kibana越来越慢

![image-20201119121936740](http://myapp.img.mykernel.cn/image-20201119121936740.png)



将单机修改为集群

![image-20201119122013380](http://myapp.img.mykernel.cn/image-20201119122013380.png)

```bash
# kibana service配置
# /etc/systemd/system/kibana.service
[Unit]
Description=Kibana

[Service]
Type=simple
User=root
Group=root
# Load env vars from /etc/default/ and /etc/sysconfig/ if they exist.
# Prefixing the path with '-' makes it try to load, but if the file doesn't
# exist, it continues onward.
EnvironmentFile=-/etc/default/kibana
EnvironmentFile=-/etc/sysconfig/kibana
ExecStart=/usr/share/kibana/bin/kibana --allow-root
Restart=on-failure
RestartSec=3
WorkingDirectory=/
#StartLimitBurst=3
StartLimitInterval=60


[Install]
WantedBy=multi-user.target
```



> daemonset的nginx只需要配置反代到127.0.0.1:30789
>
> 并且nginx到后端的超时需要给的高，避免没有查出来返回502

```bash
http {
	upstream kibanas {
        server 127.0.0.1:30789;
    }
    server {
        listen       0.0.0.0:32678;
        server_name  kibana.youninyouqu.com;
        location / {
          proxy_pass http://kibanas;
          proxy_connect_timeout 900s; # 连接超时, 默认60s
          proxy_send_timeout 900s;    # 发送超时
          proxy_read_timeout 900s;    # 读响应超时
          proxy_set_header   Host    $host;
          proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Real-IP $remote_addr;
          auth_basic "Restricted Access";
          auth_basic_user_file /usr/local/nginx/conf/.htpasswd.users;
        }
}

```





elasticsearch内存调整, 原来1g, 现在调整到2g

```bash
#/etc/elasticsearch/jvm.options
-Xms2g
-Xmx2g
```

