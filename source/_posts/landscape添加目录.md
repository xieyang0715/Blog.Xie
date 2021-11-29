---
title: landscape主题添加目录
date: 2020-10-14 02:48:32
tags: hexo
toc: true
---

# 添加目录

themes/landscape/layout/_partial/article.ejs 的<%- post.content %>之前, 添加如下代码

```js
	<!-- Table of Contents -->
<% if (!index && post.toc){ %>
  <div id="toc" class="toc-article">
    <strong class="toc-title">文章目录</strong>
    <%- toc(post.content, { list_number: false }) %>
  </div>
<% } %>

```

# 添加css

themes/landscape/source/css/_partial/article.styl 结尾添加以下css代码

```css
/*toc*/
.toc-article
  background #eee
  border 1px solid #bbb
  border-radius 10px
  margin 1.5em 0 0.3em 1.5em
  padding 1.2em 1em 0 1em
  max-width 28%
.toc-title
  font-size 120%
#toc
  line-height 1em
  font-size 0.9em
  float right
  .toc
    padding 0
    margin 1em
    line-height 1.8em
    li
      list-style-type none
  .toc-child
    margin-left 1em
```

# toc: true启用目录

在写博客时，在markdown前面加 toc: true

```mark
title: landscape添加目录
date: 2020-10-14 02:48:32
tags:
toc: true 
```

