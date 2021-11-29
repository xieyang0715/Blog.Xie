---
title: hexo优化网页打开速度
date: 2020-10-15 07:24:15
tags: hexo
---

网页打开比较慢, 从F12来看, 网页打开需要20s。优化后400ms打开。    

![image-20201015155659228](http://myapp.img.mykernel.cn/image-20201015155659228.png)

可以看到前面3个文件，加载时间特别长，进行如下优化 

<!--more-->

注意：参考[hexo博客安装中的服务器](http://liangcheng.mykernel.cn/2020/09/27/%E6%90%AD%E5%BB%BA%E8%87%AA%E5%8A%A8%E5%8F%91%E5%B8%83%E5%8D%9A%E5%AE%A2hexotypora%E4%B8%83%E7%89%9B%E4%BA%91%E5%AD%98%E5%82%A8/)安装步骤优化

# themes/landscape/layout/_partial/head.ejs文件

下载这个文件, 上传HI_SiYsKILxRpg3hIP6sJ7fM7PqlPevT.ttf至七牛, 并修改如下

```bash
@font-face {
  font-family: 'Source Code Pro';
  font-style: normal;
  font-weight: 400;
  src: local('Source Code Pro Regular'), local('SourceCodePro-Regular'), url(http://myapp.img.mykernel.cn/HI_SiYsKILxRpg3hIP6sJ7fM7PqlPevT.ttf) format('truetype');
}
```



先css文件上传至七牛，之后替换为此处

```js
<link href="//myapp.img.mykernel.cn/css" rel="stylesheet" type="text/css">
```

注意，需要重启nginx-pro。或者删除html目录，重建

![image-20201015162851788](http://myapp.img.mykernel.cn/image-20201015162851788.png)



# themes/landscape/layout/_partial/after-footer.ejs文件

使用百度的js

```js
<script src="http://apps.bdimg.com/libs/jquery/2.0.3/jquery.min.js"></script>
```

![image-20201015162057797](http://myapp.img.mykernel.cn/image-20201015162057797.png)







#  themes/landscape/source/css/_variables.styl文件

将banner文件上传至七牛存储，将URL替换banner的URL

注意：修改banner需要重新生成网页文件, docker就直接重启 

```stylus
banner-url = "http://myapp.img.mykernel.cn/banner.jpg"
```

![image-20201015161132925](http://myapp.img.mykernel.cn/image-20201015161132925.png)







网页已经优化到1.17s时，发现最长时长是字体文件

![image-20201015163231067](http://myapp.img.mykernel.cn/image-20201015163231067.png)



# themes/landscape/source/css/_variables.styl文件

将字体文件fontawesome-webfont.woff先下载下来，上传至七牛

```bash
font-icon-path = "http://myapp.img.mykernel.cn/fontawesome-webfont"
```





目前全部走七牛，已经下降至1.11s了

![image-20201015170137935](http://myapp.img.mykernel.cn/image-20201015170137935.png)





# 主页图片太多

在每个文章合适的位置加如下标记, 可以少加载图片

```bash
<!--more-->
```

目前打开网页600ms

![image-20201015172548976](http://myapp.img.mykernel.cn/image-20201015172548976.png)

将目录配置为主页

```bash
# vim hexo-config.yaml 
index_generator:
  path: '/indexes/'
  per_page: 10
  order_by: -date


# vim source/indexes/index.md 
---
title: 
date: 2020-10-14 17:37:16
toc: true
permalink: index.html 
---

# vim themes/landscape/_config.yml  # 修改目录和主页文字位置
menu:
  目录: /
  主页: /indexes/

```



现在网页打开352ms

![image-20201015175141078](http://myapp.img.mykernel.cn/image-20201015175141078.png)





