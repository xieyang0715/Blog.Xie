---
title: "hexo切换主题NexT；typora自动编号"
date: 2021-01-14 08:29:24
tags: hexo
---



将hexo主题切换, 进入nginx-pro容器里面进行更新主题，参考: https://theme-next.iissnan.com/getting-started.html

- [x] 主题 
- [x] 评论
- [x] toc配置 
- [x] 阅读量
- [x] 文章锚点 https://github.com/theme-next/hexo-theme-next-anchor

添加好插件之后，将nginx-pro中的代码推送至github

```bash
git add .
git commit -m '更换next主题'
git push origin master
```

   

配置后的备份， 参考:

http://blog.mykernel.cn/2020/09/27/%E6%90%AD%E5%BB%BA%E8%87%AA%E5%8A%A8%E5%8F%91%E5%B8%83%E5%8D%9A%E5%AE%A2hexotypora%E4%B8%83%E7%89%9B%E4%BA%91%E5%AD%98%E5%82%A8/#8-3-%E5%A4%87%E4%BB%BD



next默认会添加序号，由于是typora编辑，默认不加序号，每次手工添加序号左侧查看，在网页中会重复。

```bash
https://www.zhihu.com/question/355171943
```

复制代码,  编辑文件为`base.user.css`,里面粘贴如下内容保存后重启Typora，输入标

题时会自动出现序号

![image-20210114181536996](http://myapp.img.mykernel.cn/image-20210114181536996.png)

```css
/** initialize css counter */
#write {
    counter-reset: h1
}

h1 {
    counter-reset: h2
}

h2 {
    counter-reset: h3
}

h3 {
    counter-reset: h4
}

h4 {
    counter-reset: h5
}

h5 {
    counter-reset: h6
}

/** put counter result into headings */
#write h1:before {
    counter-increment: h1;
    content: counter(h1) ". "
}

#write h2:before {
    counter-increment: h2;
    content: counter(h1) "." counter(h2) ". "
}

#write h3:before,
h3.md-focus.md-heading:before /** override the default style for focused headings */ {
    counter-increment: h3;
    content: counter(h1) "." counter(h2) "." counter(h3) ". "
}

#write h4:before,
h4.md-focus.md-heading:before {
    counter-increment: h4;
    content: counter(h1) "." counter(h2) "." counter(h3) "." counter(h4) ". "
}

#write h5:before,
h5.md-focus.md-heading:before {
    counter-increment: h5;
    content: counter(h1) "." counter(h2) "." counter(h3) "." counter(h4) "." counter(h5) ". "
}

#write h6:before,
h6.md-focus.md-heading:before {
    counter-increment: h6;
    content: counter(h1) "." counter(h2) "." counter(h3) "." counter(h4) "." counter(h5) "." counter(h6) ". "
}

/** override the default style for focused headings */
#write>h3.md-focus:before,
#write>h4.md-focus:before,
#write>h5.md-focus:before,
#write>h6.md-focus:before,
h3.md-focus:before,
h4.md-focus:before,
h5.md-focus:before,
h6.md-focus:before {
    color: inherit;
    border: inherit;
    border-radius: inherit;
    position: inherit;
    left:initial;
    float: none;
    top:initial;
    font-size: inherit;
    padding-left: inherit;
    padding-right: inherit;
    vertical-align: inherit;
    font-weight: inherit;
    line-height: inherit;
}


.sidebar-content {
    counter-reset: h1
}

.outline-h1 {
    counter-reset: h2
}

.outline-h2 {
    counter-reset: h3
}

.outline-h3 {
    counter-reset: h4
}

.outline-h4 {
    counter-reset: h5
}

.outline-h5 {
    counter-reset: h6
}

.outline-h1>.outline-item>.outline-label:before {
    counter-increment: h1;
    content: counter(h1) ". "
}

.outline-h2>.outline-item>.outline-label:before {
    counter-increment: h2;
    content: counter(h1) "." counter(h2) ". "
}

.outline-h3>.outline-item>.outline-label:before {
    counter-increment: h3;
    content: counter(h1) "." counter(h2) "." counter(h3) ". "
}

.outline-h4>.outline-item>.outline-label:before {
    counter-increment: h4;
    content: counter(h1) "." counter(h2) "." counter(h3) "." counter(h4) ". "
}

.outline-h5>.outline-item>.outline-label:before {
    counter-increment: h5;
    content: counter(h1) "." counter(h2) "." counter(h3) "." counter(h4) "." counter(h5) ". "
}

.outline-h6>.outline-item>.outline-label:before {
    counter-increment: h6;
    content: counter(h1) "." counter(h2) "." counter(h3) "." counter(h4) "." counter(h5) "." counter(h6) ". "
}

blockquote {
  margin: 1.5625em 0;
  padding: 0.6rem;
  overflow: hidden;
  font-size: 1rem;
  page-break-inside: avoid;
  border-left: 0.3rem solid #42b983;
  border-radius: 0.3rem;
  box-shadow: 0 0.1rem 0.4rem rgba(0,0,0,0.05), 0 0 0.05rem rgba(0,0,0,0.1);
  background-color: #fafafa;
  border-color: #ff9100;  
}
```

