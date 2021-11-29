---
title: hexo添加注释插件
date: 2021-09-27 09:17:56
tags:
- hexo
---

# 前言

今天上班，对hexo有哪些插件比较感兴趣，搜索

![image-20210927091857059](http://myapp.img.mykernel.cn/image-20210927091857059.png)

进入页面后，有一个 [hexo-admonition](https://github.com/lxl80/hexo-admonition) 插件



<!--more-->
# 安装插件

<div class="admonition warning"><p>由于我的博客不支持此插件，会影响目录点击，所以此插件<b>没有安装</b></p>
</div>

<strong><s>npm install hexo-admonition --save</s></strong>

里面还说明了需要配置自定义css,  我们next主题有一个存放css的地方: ` themes/next/source/css/`, 如果放在这里，每次更新next版本，就需要添加文件，所以看看next配置文件是否支持加载外部的css.

编辑`_config.next.yml`

```yaml
# Define custom file paths.
# Create your custom files in site directory `source/_data` and uncomment needed files below.
custom_file_path:
  #head: source/_data/head.njk
  #header: source/_data/header.njk
  #sidebar: source/_data/sidebar.njk
  #postMeta: source/_data/post-meta.njk
  #postBodyEnd: source/_data/post-body-end.njk
  #footer: source/_data/footer.njk
  #bodyEnd: source/_data/body-end.njk
  #variable: source/_data/variables.styl
  #mixin: source/_data/mixins.styl
  style: source/_data/styles.styl        # 这里就是自定义样式的文件
```

> 如果没有此文件
>
> 由于hexo 5.0之后支持把next的配置文件存放为`_config.[theme].yaml`, 所以可以
>
> ```bash
> cp -i themes/next/_config.yml ./_config.next.yml 
> ```



现在添加这个文件

```bash
install -dv source/_data/
```

编辑 `source/_data/styles.styl`

```css
.admonition {
  margin: 1.5625em 0;
  padding: .6rem;
  overflow: hidden;
  font-size: .64rem;
  page-break-inside: avoid;
  border-left: .3rem solid #42b983;
  border-radius: .3rem;
  box-shadow: 0 0.1rem 0.4rem rgba(0,0,0,.05), 0 0 0.05rem rgba(0,0,0,.1);
  background-color: #fafafa;
}

p.admonition-title {
  position: relative;
  margin: -.6rem -.6rem .8em -.6rem !important;
  padding: .4rem .6rem .4rem 2.5rem;
  font-weight: 700;
  background-color:rgba(66, 185, 131, .1);
}

.admonition-title::before {
  position: absolute;
  top: .9rem;
  left: 1rem;
  width: 12px;
  height: 12px;
  background-color: #42b983;
  border-radius: 50%;
  content: ' ';
}

.info>.admonition-title, .todo>.admonition-title {
  background-color: rgba(0,184,212,.1);
}

.warning>.admonition-title, .attention>.admonition-title, .caution>.admonition-title {
  background-color: rgba(255,145,0,.1);
}

.failure>.admonition-title, .missing>.admonition-title, .fail>.admonition-title, .error>.admonition-title {
  background-color: rgba(255,82,82,.1);
}

.admonition.info, .admonition.todo {
  border-color: #00b8d4;
}

.admonition.warning, .admonition.attention, .admonition.caution {
  border-color: #ff9100;
}

.admonition.failure, .admonition.missing, .admonition.fail, .admonition.error {
  border-color: #ff5252;
}

.info>.admonition-title::before, .todo>.admonition-title::before {
  background-color: #00b8d4;
  border-radius: 50%;
}

.warning>.admonition-title::before, .attention>.admonition-title::before, .caution>.admonition-title::before {
  background-color: #ff9100;
  border-radius: 50%;
}

.failure>.admonition-title::before,.missing>.admonition-title::before,.fail>.admonition-title::before,.error>.admonition-title::before{
  background-color: #ff5252;;
  border-radius: 50%;
}

.admonition>:last-child {
  margin-bottom: 0 !important;
}
```

# markdown使用示例

提示类型 `type` 将用作 CSS 类名称，暂支持如下类型：

- `note`
- `info, todo`
- `warning, attention, caution`
- `error, failure, missing, fail`

## 带标题

markdown

```bash
!!! note Hexo-admonition 插件使用示例
    这是基于 hexo-admonition 插件渲染的一条提示信息。类型为 note，并设置了自定义标题。

    提示内容开头留 4 个空格，可以有多行，最后用空行结束此标记。

```

html格式，<strong>由于我的博客不能使用markdown语法，故采用原生html</strong>

```bash
<div class="admonition note">
	<p class="admonition-title">Hexo-admonition 插件使用示例</p>
	<p>这是基于 hexo-admonition 插件渲染的一条提示信息。类型为 note，并设置了自定义标题</p>
	<p>提示内容开头留 4 个空格，可以有多行，最后用空行结束此标记。</p>
</div>
```

<div class="admonition note"><p class="admonition-title">Hexo-admonition 插件使用示例
</p><p>这是基于 hexo-admonition 插件渲染的一条提示信息。类型为 note，并设置了自定义标题, 提示内容开头留 4 个空格，可以有多行，最后用空行结束此标记。</p>
</div>

## 带标题和引用

```css
!!! note "嵌套链接及引用块"
    欢迎访问我的博客链接：[悟尘纪](https://www.lixl.cn)
    > 这里嵌套一行引用信息。
```

html格式，<strong>由于我的博客不能使用markdown语法，故采用原生html</strong>

```bash
<div class="admonition note">
	<p class="admonition-title">嵌套链接及引用块</p>
	<p>欢迎访问我的博客链接：<a target="_blank" rel="noopener" href="https://www.lixl.cn">悟尘纪</a></p>
    <blockquote>
    <p>这里嵌套一行引用信息。</p>
    </blockquote>
</div>
```

<div class="admonition note"><p class="admonition-title">嵌套链接及引用块
</p><p>欢迎访问我的博客链接：<a target="_blank" rel="noopener" href="https://www.lixl.cn">悟尘纪</a></p>
<blockquote>
<p>这里嵌套一行引用信息。</p>
</blockquote>
</div>
### emmet语法补全

```emment
.admonition.note>p.admonition-title{注意}+p{你好}+blockquote>p{嵌套}
```



![image-20210930162720564](http://myapp.img.mykernel.cn/image-20210930162720564.png)

## 空标题

```css
!!! Warning ""
    这是一条不带标题的警告信息。
    注意书写此语法时，需要在原生markdown. 而不是typora中，会自动添加换行。
```

html格式，<strong>由于我的博客不能使用markdown语法，故采用原生html</strong>

```bash
<div class="admonition warning">
	<p>这是一条不带标题的警告信息。</p>
	<p>注意书写此语法时，需要在原生markdown. 而不是typora中，会自动添加换行。</p>
</div>
```

<div class="admonition warning"><p>这是一条不带标题的警告信息。
注意书写此语法时，需要在原生markdown. 而不是typora中，会自动添加换行。</p>
</div>

## 默认标题

```css
!!! warning
    这是一条采用默认标题的警告信息。
```

html格式，<strong>由于我的博客不能使用markdown语法，故采用原生html</strong>

```bash
<div class="admonition warning">
	<p class="admonition-title">warning</p>
	<p>这是一条采用默认标题的警告信息。</p>
</div>
```

<div class="admonition warning"><p class="admonition-title">warning</p><p>这是一条采用默认标题的警告信息。</p>
</div>


# 不需要额外使用插件

直接在这个文件中添加

`source/_data/styles.styl`

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
```



