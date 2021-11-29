---
title: curl命令使用
date: 2021-09-24 11:01:23
tags:
- curl
---

# 查看解析时间

```bash
root@ip-10-0-0-245:/usr/local/istio#  curl  -w "\ntime_namelookup:  %{time_namelookup}s\ntime_connect:  %{time_connect}s\ntime_appconnect:  %{time_appconnect}s\ntime_pretransfer:  %{time_pretransfer}s\ntime_redirect:  %{time_redirect}s\ntime_starttransfer:  %{time_starttransfer}s\n----------\ntime_total:  %{time_total}s\n\n\n" www.baidu.com
<!DOCTYPE html>
<!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=http://s1.bdstatic.com/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=百度一下 class="bg s_btn"></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>新闻</a> <a href=http://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>地图</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>视频</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>贴吧</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>登录</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">登录</a>');</script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">更多产品</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>关于百度</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>使用百度前必读</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>意见反馈</a>&nbsp;京ICP证030173号&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>

time_namelookup:  0.004224s
time_connect:  0.027428s
time_appconnect:  0.000000s
time_pretransfer:  0.027475s
time_redirect:  0.000000s
time_starttransfer:  0.051504s
----------
time_total:  0.051560s
```

> https://stackoverflow.com/questions/18215389/how-do-i-measure-request-and-response-times-at-once-using-curl



# `-fsSL`

1. Treats non-2xx/3xx responses as errors (`-f`).
2. Disables the progress meter (`-sS`).
3. Handles HTTP redirects (`-L`).

> 转载： https://www.joyfulbikeshedding.com/blog/2020-05-11-best-practices-when-using-curl-in-shell-scripts.html

## curl脚本中判断非2xx/3xx响应为错误

```bash
root@ip-10-0-0-245:/usr/local/istio# curl -f https://httpstat.us/500
curl: (22) The requested URL returned error: 500 
root@ip-10-0-0-245:/usr/local/istio# echo $?
22
```

如果不加-f

```bash
root@ip-10-0-0-245:/usr/local/istio# curl  https://httpstat.us/500
500 Internal Server Errorroot@ip-10-0-0-245:/usr/local/istio# echo $?
0
```



## 获取请求地址的响应状态码

在脚本中，我们需要基于响应状态码，来做不同的事。例如：下载预编译的二进制程序，2xx就成功，404就不存在，就只能从源码编译了

```bash
CODE=$(curl -sSL -w '%{http_code}' -o binary.tar.gz https://myserver.com/binary.tar.gz)
if [[ "$CODE" =~ ^2 ]]; then
    # Server returned 2xx response
    do_something_with binary.tar.gz
elif [[ "$CODE" = 404 ]]; then
    # Server returned 404, so compiling from source
    compile_from_source
else
    echo "ERROR: server returned HTTP code $CODE"
    exit 1
fi
```

> - `-w '%{http_code}'` 此选项，用来控制curl命令，将http响应状态码写入标准输出
> - 将curl包装在`$()`目的是将响应状态码放在变量`CODE`中
> - `-o <filename>` 目的将HTTP响应body写入文件，而不是写到标准输出。这非常重要，因为我们只想在标准输出获取响应码
> - `-sSL`  -s是禁用进度条和错误信息，-S是启用错误信息，-L 跟踪302跳转

