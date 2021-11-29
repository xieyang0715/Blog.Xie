---
title: nginx的proxy_connect_timeout参数调整
date: 2020-11-16 08:12:12
tags: 
- 生产小经验
- nginx
toc: true
---



公司对app有后台管理服务, 通过ELK可以观察到后台管理响应时长大多数为60s左右, 开发说这个接口是多个查询导致查询慢，但是用户并不会使用这个接口，只是我们运营在使用。查询60s时，nginx会响应502。因为nginx反代到后端默认超时为60s，后端查询需要一定时长，所以就断开了。

所以添加如下选项

```nginx
location / {
	proxy_pass http://upstream_server;
	proxy_connect_timeout 900s; # proxy与server建立连接超时

	proxy_read_timeout 900s;    # 连接建立后的读操作超时
    proxy_send_timeout 900s;    # 连接建立后的写操作超时
}
```





