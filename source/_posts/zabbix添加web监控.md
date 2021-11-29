---
title: zabbix添加web监控
date: 2020-12-09 03:24:26
tags: 生产小经验
toc: true
---



使用zabbix添加web监控kibana(basic)认证页面出问题

# 1. 以模板添加web监控

添加一个模板，配置所属的组

后期需要报警的主机，直接链接这个模板就可以实现报警

![image-20201209113146606](http://myapp.img.mykernel.cn/image-20201209113146606.png)

# 2. 添加场景

前期将更新间隔尽量调低, 1s一次方便查看

后期要修改为1m

![image-20201209112544754](http://myapp.img.mykernel.cn/image-20201209112544754.png)

<!--more-->

# 3. 添加basic认证

![image-20201209112641062](http://myapp.img.mykernel.cn/image-20201209112641062.png)

# 4. 配置监控的页面

注意，不要打开获取头信息，否则会报错

![image-20201209112721258](http://myapp.img.mykernel.cn/image-20201209112721258.png)

# 5. web监测页面

![image-20201209112825682](http://myapp.img.mykernel.cn/image-20201209112825682.png)

# 6. 报警

![image-20201209112911178](http://myapp.img.mykernel.cn/image-20201209112911178.png)

![image-20201209112928578](http://myapp.img.mykernel.cn/image-20201209112928578.png)



找到自已模板的位置 zabbix server组内的haproxy-web模板，测试为rspcode是响应状态码

![image-20201209113002278](http://myapp.img.mykernel.cn/image-20201209113002278.png)

最后一次是200响应

![image-20201209113045856](http://myapp.img.mykernel.cn/image-20201209113045856.png)

# 7. 测试报警

将kibana的认证密码添加11， 更新间隔为1s

![image-20201209113329428](http://myapp.img.mykernel.cn/image-20201209113329428.png)

![image-20201209113344368](http://myapp.img.mykernel.cn/image-20201209113344368.png)

![image-20201209113412758](http://myapp.img.mykernel.cn/image-20201209113412758.png)