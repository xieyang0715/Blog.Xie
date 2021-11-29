---
title: httrack镜像网站
tags:
  - 工具
photos:
  - http://myapp.img.mykernel.cn/header_title_4.png
date: 2021-11-26 13:04:00
---

# 前言

有一些非常好的网站，避免不可访问，故可以为其做一份镜像，类似于阿里云的镜像centos官方

`httrack`工具, 可以把网站的资源(image, html, css, js...)从远程服务器下载下来，保存在本地，方便我们离线浏览。

[官方站点](https://www.httrack.com/)



<!--more-->

# 使用注意事项

> https://www.httrack.com/html/abuse.html

下载过快会给站点压力

带宽限制

连接限制

大小限制

使用时间限制

工作时间不要下载

检查镜像的速率

对于较大的镜像先问问站点主人

确保你有版权拷贝这个站点

你拷贝这个站点只为私人使用？不要在线镜像站点，除非你有权限。

不要窃取私人信息、邮件

# 一个web站点防止消耗大量的带宽

1. 站点发布通告，不要占用大量的带宽。仅对好人有效。

2. 使用robots.txt文件。

   `robots.txt`

   ```
   User-agent: *
   Disallow: /bigfolder
   ```

3. 禁用离线浏览器的user-agent

   nginx过滤 http首部 `user-agent`

   容易篡改这个首部

4. 限制每ip或每文件的带宽

   Qos限制

5. 全部使用小文件，不使用大文件。

6. 禁用占用大量带宽的ips

   bad: 对于动态ip无效，非常不友好 

   怎么做：防火墙禁用ip

7. 限制占用大量带宽的ips

   bad: 对于动态ip无效，难维护

   怎么做：路由Qos, 

8. 使用技术手段隐藏URLS

   高效

   bad: 有效的技术手段让你的website非常重，用户不友好。用户只想离线website. 高级用户仍然可以发现url并抓取他们。no-javascript的浏览器将不会工作

   大多数浏览器不能渲染javascript

   你可以替换

   ```html
   <a href="bigfile.zip">Foo</a>
   ```

   替换成

   ```html
    <script language="javascript">
       <!--
       document.write('<a h' + 're' + 'f="');
       document.write('bigfile' + '.' + 'zip">');
       // -->
       </script>
       Foo
       </a>
   ```

9. 使用技术手段让离线浏览器变慢

   bad: 高级用户可以避免，也难维护，这个页面会消耗CPU

   通过一个空链接指向一个`cgi`, 它带有非常长的延迟

   ```php
     <?php
       for($i=0;$i<10;$i++) {
           sleep(6);
           echo " ";
       }
       ?>
   ```

10. 使用技术手段临时禁用ip

    你的站点将只能给在线用户使用，配置有点麻烦

    how to do: 创建一个假链接` <a href="killme.cgi"><nothing></a> `

    ```php
    <?php
    	// Add IP.
    	add_temp_firewall_rule($REMOTE_ADDR,"30s");
    ?>
    function add_temp_firewall_rule($addr) {
    	// The chain chhttp is flushed in a cron job to avoid ipchains overflow
        system("/usr/bin/sudo -u root /sbin/ipchains -I 1 chhttp -p tcp -s ".$addr." --dport 80 -j REJECT");
        syslog("user rejected due to too many copy attemps : ".$addr);
    }
    ```

    > iptables添加一个临时的链`ipchains`
    >
    > 一量访问这个链接就会把这个ip禁用，之后crontab 30s后就flush这个链



# 使用

https://www.httrack.com/html/fcguide.html

```bash
httrack "http://www.all.net/" -O "/tmp/www.all.net" "+*.all.net/*" -v
```

> 将www.all.net网站所有文件以及目录结构保存在`/tmp/www.all.net`目录下
>
> 除了`*.all.net`域的所有内容将不会抓取



# 简单脚本

https://www.httrack.com/html/scripting.html