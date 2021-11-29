---
title: hexo迁移hugo
date: 2021-10-26 13:19:30
tags:
- hugo
- hexo
---



# 前言

当我的博客文章达到200+时，hexo主题渲染非常慢，经常会出现写了文章发布之后，后台渲染失败的情况，我也不想升级主机的配置。

通过hexo的QQ群，问了大佬，向我推荐**hugo**

然后通过网上搜索时，自动补全，估计全是博客站点。通过搜索**对比**关键词，找到一篇文章：https://juejin.cn/post/6844903637353791496

![image-20211026132122179](http://myapp.img.mykernel.cn/image-20211026132122179.png)



<!--more-->
## SSG

Static Site Generator, 静态站点生成器, 会获取您的站点内容，将其应用于模板，并生成纯静态 HTML 文件，以便传递给访问者。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/7/12/1648d759b824b7f4~tplv-t2oaga2asx-watermark.awebp)

## 选择SSG

**需要开箱即用的大量动态功能和扩展**

> **Hugo** 提供了大量内置功能，其次 **Gatsby** 也挺适合于这个情况

**你在意构建和部署时间吗？**

>  **Hugo**。它以其超快的构建时间而闻名，可以在几毫秒内将内容和模板组合成一个简单站点，以这个速度几秒钟内可以完成数千页。

**博客或小型个人网站**

> JekyII, hexo, Hugo，Pelican，Gatsby。

**文档**

> **GitBook** 使编写和维护高质量的文档变得容易，现在已是最流行的文档工具。
>
> 也可以了解一下 Docusaurus，MkDocs。

**电子商务**

> 用户体验方面，例如速度和UI定制，搜索引擎优化也是必不可少的。
>
> 大型电商网站需要 CMS 进行产品管理，这个时候就要思考哪个 SSG 更适合你选择的无头 CMS。
>
> 根据我们的经验，我们推荐 **Gatsby** 和 **Nuxt** 这样的响应式框架。但如果你还是需要一切从简，你可以考虑 **Jekyll** 或 **Hugo**。

**营销网站**

> 之前还没提过 [**Middleman**](https://link.juejin.cn?target=https%3A%2F%2Fmiddlemanapp.com%2F)。它的与众不同之处在于它可以灵活搭建任何类型的网站，而不是专注于博客引擎。这对于高级营销网站来说非常棒，MailChimp 和 Vox Media 等公司也将它用于自己的网站。
>
> 也可以了解一下 Gatsby，Hugo，Jekyll。

## 是否修改网站和生成器

以下是各个框架使用的语言：

- **JavaScript**：Next.js & Gatsby（适用于 React）、Nuxt & VuePress（适用于 Vue）、Hexo、GitBook、Metalsmith、Harp、Spike。
- **Python**：Pelican、MkDocs、Cactus.
- **Ruby**：Jekyll、Middleman、Nanoc、Octopress.
- **Go**：Hugo、InkPaper。
- **.NET**：Wyam、pretzel。


## **非技术用户是否需要管理此网站？**

这种情况应该将有无头 CMS 的 SSG 放在首位。CMS 的选择很重要，找到可以对接的 SSG 同样重要。

**Gatsby** 的新功能，[使用 GraphQL 实现](https://link.juejin.cn/?target=https%3A%2F%2Fwww.gatsbyjs.org%2Fdocs%2Fquerying-with-graphql%2F)。这里不解释 [GraphQL 是啥](https://link.juejin.cn/?target=https%3A%2F%2Fsnipcart.com%2Fblog%2Fgraphql-nodejs-express-tutorial)，简而言之，它可以实现更快更简洁的数据查询。

**首发地址** [*Snipcart blog*](https://link.juejin.cn/?target=https%3A%2F%2Fsnipcart.com%2Fblog%2Fchoose-best-static-site-generator) **本文地址（英语）** [*newsletter*](https://link.juejin.cn/?target=http%3A%2F%2Fsnipcart.us5.list-manage2.com%2Fsubscribe%3Fu%3Dc019ca88eb8179b7ffc41b12c%26id%3D3e16e05ea2)



# 部署

>  官网：https://gohugo.io/
>
> 特点：
>
> - 构建速度平均1个站点不到1s.
> - 支持多种类型的资源
> - 支持简洁的markdown语法
> - 一条命令完成预置模板的优化：seo, 评论，分析，...
> - 多种语言支持
> - markdown输出不仅仅只有html, 还有 json, amp, ...
>
> 模板：https://themes.gohugo.io/
>
> > 我更喜欢这个模板: https://themes.gohugo.io/themes/hugo-tranquilpeak-theme/
>
> 基于go的模板，只要提供了相应的逻辑，就可以完成从简单到复杂的网页: https://gohugo.io/templates/
>
> 网页展示：https://gohugo.io/showcase/
>
> 1s安装：https://gohugo.io/getting-started/installing/

## 安装

[安装](https://gohugo.io/getting-started/installing/)hugo, 由于我现在的站点和下一版本的站点，完全不兼容，所以需要新开一个域名，故命名为`student.mykernel.cn`, 与原来的站点完全独立。

### 准备环境

```bash
docker run --rm --name hugo -it ubuntu:18.04  bash
```

把执行的命令变成dockerfile

- https://golang.org/dl/go1.17.2.linux-amd64.tar.gz
- https://code.aliyun.com/1062670898/hugo

```dockerfile
FROM ubuntu:18.04

LABEL MANTAINER=2192383945@qq.com

RUN sed -i -e "s%http://archive.ubuntu.com/ubuntu/%http://mirrors.aliyun.com/ubuntu/%g" -e "s%http://security.ubuntu.com/ubuntu%http://mirrors.aliyun.com/ubuntu%g" /etc/apt/sources.list && \
	apt update && \
	apt install git wget -y --fix-missing

# 准备go程序, 可以直接安装 golang
ADD  https://golang.org/dl/go1.17.2.linux-amd64.tar.gz /usr/local/
RUN tar xf /usr/local/go1.17.2.linux-amd64.tar.gz -C /usr/local/ && rm -fr /usr/local/go1.17.2.linux-amd64.tar.gz
ENV PATH="$PATH:/usr/local/go/bin"

# 编译安装hugo, 确保hugo一直是最新版本
ENV GOPROXY=https://goproxy.io,direct
RUN git clone https://github.com/gohugoio/hugo.git && cd hugo && \
	mkdir -p src/github.com/gohugoio && ln -sf $(pwd) src/github.com/gohugoio/hugo && go get . && go build -o hugo main.go && \
	install hugo /usr/local/bin && rm -fr /root/go/ /hugo
	
# 清理
RUN apt install -y locales && rm -rf /var/lib/apt/lists/* \
    && localedef -i zh_CN -c -f UTF-8 -A /usr/share/locale/locale.alias zh_CN.UTF-8
ENV LANG zh_CN.utf8

EXPOSE 3000

# 运行
ENTRYPOINT ["hugo"]
CMD ["server", "--bind", "0.0.0.0", "--port", "3000"]
```

> 此文件在在code.aliyun.com上，由cr.console.aliyun.com完成构建

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/slcnx/hugo:0.88.2
```

### 一键启动

> https://gohugo.io/getting-started/quick-start/

```docker-compose
version: "3.3"
services:
  fastgithub:
    image: slcnx/fastgithub
    network_mode: host
    restart: always
    volumes:
    - cacert:/fastgithub/cacert/
  prepare:
    depends_on:
    - fastgithub
    restart: on-failure
    tty: true
    entrypoint: sh -c 'cp /tmp/cacert/fastgithub.cer /usr/local/share/ca-certificates/fastgithub.crt && update-ca-certificates && find /hugo_data ! -path /hugo_data \( -type f -o -type d \) -delete && hugo new site /hugo_data && cd /hugo_data && git init &&  git submodule add  https://github.com/kakawait/hugo-tranquilpeak-theme themes/hugo-tranquilpeak-theme  && cp -r /hugo_data/themes/hugo-tranquilpeak-theme/exampleSite/* .'
    environment:
      https_proxy: http://127.0.0.1:38457
      http_proxy: http://127.0.0.1:38457
    network_mode: host
    working_dir: /
    image: registry.cn-hangzhou.aliyuncs.com/slcnx/hugo:0.88.2
    volumes:
    - db_data:/hugo_data
    - cacert:/tmp/cacert
    command: ""
  render_blog:
    depends_on:
    - fastgithub
    - prepare
    image: registry.cn-hangzhou.aliyuncs.com/slcnx/hugo:0.88.2
    working_dir: /hugo_data
    volumes:
      - db_data:/hugo_data
    restart: always
    entrypoint: sh -c 'hugo'
  blog:
    depends_on:
    - fastgithub
    - prepare
    image: registry.cn-hangzhou.aliyuncs.com/slcnx/hugo:0.88.2
    working_dir: /hugo_data
    volumes:
      - db_data:/hugo_data
    ports:
     - "3002:3002"
    restart: always
    command: ["server", "--bind", "0.0.0.0", "--port", "3002", "-b", "http://119.3.139.239:3002/"]
  render:
    depends_on:
    - prepare
    - fastgithub
    image: registry.cn-hangzhou.aliyuncs.com/slcnx/hugo:0.88.2
    volumes:
      - db_data:/hugo_data
    working_dir: /hugo_data
    restart: always
    tty: true
    entrypoint:
    - cat
    command: ""
volumes:
  db_data: {}
  cacert: {}
```

> -  上面的fastgithub, 请参考：另一篇 github标签下的fastgithub加速制作过程。
> -  prepare 完成示例页面的准备
> - blog 提供 hugo server，会自动判断文件变化而渲染
> - render_blog 会一直渲染
> - render是一个测试程序
>
> 原网页：https://tranquilpeak.kakawait.com/
>
> 现网页：
>
> ![image-20211026172932394](http://myapp.img.mykernel.cn/image-20211026172932394.png)



### 生成第一个文章

默认情况下，通过`archetypes/default.md` 为模板，通过`hugo new posts/my-first-post.md`命令来生成我们的文件。



## 暂停

由于hexo带有百度分析、自带渲染的目录结构、暂时还没有取代的，先暂时停止。





