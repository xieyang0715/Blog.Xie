---
title: hexo添加子菜单submenu
date: 2021-11-09 13:35:43
tags:
- hexo
photos:
- http://myapp.img.mykernel.cn/001.jpg
---



# 前言

next主题提供了[子菜单的功能](https://theme-next.js.org/docs/theme-settings/#Configuring-Menu-Items), 看到自己的网站的菜单过多，所以精简一番。

```yaml
menu:
  home: / || fa fa-home
  archives: /archives/ || fa fa-archive
  Docs:
    default: /docs/ || fa fa-book
    Getting Started:
      default: /getting-started/ || fa fa-flag
      Installation: /installation.html || fa fa-download
      Configuration: /configuration.html || fa fa-wrench
    Third Party Services:
      default: /third-party-services/ || fa fa-puzzle-piece
      Math Equations: /math-equations.html || fa fa-square-root-alt
      Comment Systems: /comments.html || fa fa-comment-alt

```

> 1. menu下的1级`key: value`, 
>
>    - key就是菜单的名称。
>
>    - `value`就是`路径 || 此key对应的class值`
>
> 2. 如果有子菜单必须有default., 例如`Docs` 必须有default.
>
>    - 子菜单`Getting Started`
>      1. 必须有default
>      2. 子菜单下有2个分类`Installation` `Configuration`
>    - 子菜单`Third Party Services`
>      1. 必须有default
>      2. 子菜单下有2个分类 `Math Equations`  ` Comment Systems`
>
> 3. 从文档页面来看，这个docs这个菜单是对整个的简介。`Getting Started` 是对`Installation` `Configuration`的概述。`Third Party Services`是对 `Math Equations`  ` Comment Systems`的概述

<!--more-->

# 生成子菜单

## 准备配置文件

```yaml
menu:
  主页: /  ||  fa fa-home
  Docs:
    default: /docs/ || fa fa-book

    Working Skills:
      default: /skills/ || fa fa-light fa-thumbs-up
      Python: /python || fa fa-brands fa-python
      Linux:  /linux || fa fa-brands fa-linux
      Tools: /tools || fa fa-light fa-toolbox
    Favorite:
      default: /favorite || fa fa-light fa-heart
      Creative Cloud:  /cc || fa fa-light fa-camera
    About Me:
      default: /about ||  fa  fa-light fa-file-user
      简历: /bio ||  fa  fa-light fa-page 
      日志: /notes ||  fa-light fa-book
```

> 1. 子菜单: docs -> work/favorite/about 
> 2. url: /docs/skills/linux, /docs/skills/python, /docs/skills/tools, /docs/favorite/cc, /docs/about/bio,  /docs/about/notes
> 3. 类名，要做准备相应的图标

## 准备目录结构

1. 准备一个子菜单的页面

   ```bash
   hexo new page docs
   ```

2. 此页面下有多个子菜单

   ```bash
   $ ls source/docs/ -R
   source/docs/:
   about/  favorite/  index.md  skills/
   
   source/docs/about:
   bio/  index.md  notes/
   
   source/docs/about/bio:
   index.md
   
   source/docs/about/notes:
   index.md
   
   source/docs/favorite:
   cc/  index.md
   
   source/docs/favorite/cc:
   index.md
   
   source/docs/skills:
   index.md  linux/  python/  tools/
   
   source/docs/skills/linux:
   index.md
   
   source/docs/skills/python:
   index.md
   
   source/docs/skills/tools:
   index.md
   ```

​	每个页面生成一个示例index.md

```markdown
---
title: docs
date: 2021-11-09 10:25:38
---

hello world
```



# 准备css

## 字体

https://pro.fontawesome.com/releases/v6.0.0-beta2/webfonts/fa-thin-100.woff2

https://pro.fontawesome.com/releases/v6.0.0-beta2/webfonts/fa-solid-900.woff2

https://pro.fontawesome.com/releases/v6.0.0-beta2/webfonts/fa-regular-400.woff2

https://pro.fontawesome.com/releases/v6.0.0-beta2/webfonts/fa-light-300.woff2

https://pro.fontawesome.com/releases/v6.0.0-beta2/webfonts/fa-duotone-900.woff2

https://pro.fontawesome.com/releases/v6.0.0-beta2/webfonts/fa-brands-400.woff2

将字体放在`source/webfronts`目录下，渲染后，就在根目录下的/webfronts/目录下，供css相对加载

```bash
$ ls source/webfonts/
fa-brands-400.woff2  fa-duotone-900.woff2  fa-light-300.woff2  fa-regular-400.woff2  fa-solid-900.woff2  fa-thin-100.woff2
```



## 准备css

编辑source/_data/style.stl

```css

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

@font-face {
    font-family: "Font Awesome 6 Duotone";
    font-style: normal;
    font-weight: 900;
    font-display:block;src: url(/webfonts/fa-duotone-900.woff2) format("woff2"),url(/webfonts/fa-duotone-900.ttf) format("truetype")
}
@font-face {
    font-family: "Font Awesome 6 Brands";
    font-style: normal;
    font-weight: 400;
    font-display:block;src: url(/webfonts/fa-brands-400.woff2) format("woff2"),url(/webfonts/fa-brands-400.ttf) format("truetype")
}
@font-face {
    font-family: "Font Awesome 6 Pro";
    font-style: normal;
    font-weight: 300;
    font-display:block;src: url(/webfonts/fa-light-300.woff2) format("woff2"),url(/webfonts/fa-light-300.ttf) format("truetype")
}
@font-face {
    font-family: "Font Awesome 6 Pro";
    font-style: normal;
    font-weight: 400;
    font-display:block;src: url(/webfonts/fa-regular-400.woff2) format("woff2"),url(/webfonts/fa-regular-400.ttf) format("truetype")
}
@font-face {
    font-family: "Font Awesome 6 Pro";
    font-style: normal;
    font-weight: 900;
    font-display:block;src: url(/webfonts/fa-solid-900.woff2) format("woff2"),url(/webfonts/fa-solid-900.ttf) format("truetype")
}

.sub-menu i  {
  margin-right: 8px;   
}

.fa-light, .fa-regular, .fal, .far {
    font-family: "Font Awesome 6 Pro";
}

.fa-brands, .fab {
    font-family: "Font Awesome 6 Brands";
    font-weight: 400;
}

.fa-address-card:before {
    content: "\f2bb"
}
.fa-file:before {
    content: "\f15b"
}

.fa-thumbs-up:before {
  content: "\f164";
}
.fa-python:before {
  content: "\f3e2"; 
}

.fa-linux:before {
  content: "\f17c"; }

.fa-toolbox:before {
  content: "\f552"; }


.fa-heart:before {
  content: "\f004"; }

```

<strong style="color: red">前端开发告诉此方法过于原始，后又提供一种iconfont网站生成</strong>参考以下部分配置



# iconfont

>  https://www.iconfont.cn/
>
> 使用方法见 [iconfont使用方法]()

使用这个就非常简单了，仅需要引用css, 并在menu菜单中配置

## 引入css

` source/_data/styles.styl`

```diff
@font-face {
  font-family: "iconfont"; /* Project id 2926463 */
  /* Color fonts */
  src:
       url('https://at.alicdn.com/t/font_2926463_hnvhui14s6.woff2?t=1636504534236') format('woff2'),
       url('https://at.alicdn.com/t/font_2926463_hnvhui14s6.woff?t=1636504534236') format('woff'),
       url('https://at.alicdn.com/t/font_2926463_hnvhui14s6.ttf?t=1636504534236') format('truetype');
}

.iconfont {
  font-family: "iconfont" !important;
  font-size: 16px;
  font-style: normal;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
+  padding-right: 8px;
}

.icon-favorite:before {
  content: "\e6a0";
}

.icon-yonggong:before {
  content: "\e612";
}

.icon-about:before {
  content: "\e601";
}

.icon-profile-resume-biodata-paper-office-hr-human-research-feaffaf:before {
  content: "\e780";
}

.icon-Tools:before {
  content: "\e895";
}

.icon-CreativeCloud:before {
  content: "\e66b";
}

.icon-rizhi:before {
  content: "\e613";
}

.icon-linux:before {
  content: "\e80b";
}

.icon-Python:before {
  content: "\e891";
}

```

> 图标右边需要一点距离

## 引入 menu

```yaml
menu:
  主页: /  ||  fa fa-home
  Docs:
    default: /docs/ || fa fa-book

    Working Skills:
      default: /skills/ || iconfont icon-yonggong
      Python: /python || iconfont icon-Python
      Linux:  /linux || iconfont icon-linux
      Tools: /tools || iconfont icon-Tools
    Favorite:
      default: /favorite || iconfont icon-favorite
      Creative Cloud:  /cc || iconfont icon-CreativeCloud
    About Me:
      default: /about || iconfont  icon-about
      简历: /bio ||  iconfont icon-profile-resume-biodata-paper-office-hr-human-research-feaffaf
      日志: /notes || iconfont icon-rizhi
```

> 后面的类正好匹配上面的css

## 结果 

![image-20211110084143088](http://myapp.img.mykernel.cn/image-20211110084143088.png)
