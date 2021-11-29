---
title: nginx的location传递
date: 2021-03-17 08:18:00
tags:
- nginx
---



#   locatoin /a -> /

`api.youninyouqu.com/service-user/user/login -> http://localhost:30081/user/login`

```nginx
server {
        listen 443 ssl http2;
        server_name api.youninyouqu.com;

		location /service-user {
                proxy_pass http://localhost:30081/;

                proxy_set_header   Host             $host;
                proxy_set_header   X-Real-IP        $remote_addr;
                proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
}

# 前端访问接口是 /service-user/user/login
# 后台输出接口是 /user/login
```

#  location /a -> /a

 `api.youninyouqu.com/service-user/user/login -> http://localhost:30081/service-user/user/login`

```
server {
        listen 443 ssl http2;
        server_name api.youninyouqu.com;

		location /service-user {
                proxy_pass http://localhost:30081;

                proxy_set_header   Host             $host;
                proxy_set_header   X-Real-IP        $remote_addr;
                proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
}


# 前端访问接口是 /service-user/user/login
# 后台输出接口是 /service-user/user/login
```

