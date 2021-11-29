---
title: iconfont使用方法
date: 2021-11-10 08:42:32
tags:
- 黑马前端
photos:
- http://myapp.img.mykernel.cn/O1CN01Mn65HV1FfSEzR6DKv_!!6000000000514-55-tps-228-59.svg
---



# 前言

在生成博客时, menu的图标根据配置文件是参考一个网站，https://fontawesome.com/v5.15/icons?d=gallery&p=2 来引用图标，但是有一些图标是专业用户，需要付费生成。

请教前端开发之后，原来有一个专门的网站`iconfont`，可以非常方便生成图标字体、css类。 



以下将介绍上面2种方式生成图标的过程

<!--more-->



# 引用fontawecome.com

## 生成图标的过程

1. 获取字体文件。
2. 引用字体。
3. 生成unicode编码

> unicode编码，就是字体的key, 引用字体之后，就会在字体文件中查找key对应的值。

## 获取字体

![image-20211110085023378](http://myapp.img.mykernel.cn/image-20211110085023378.png)

> 此图标就是通过此字体渲染的
>
> 1. `.fab`就是此`i`标签的类
>
> 2. 右边就是css，其格式如下，作用就是通过css选择器选择html标签(`即图中的<i class="fab fa-500px" style="font-size: 48px;"></i>`), 选择后，这个标签的表现形式就是通过以下的`key:value`对来定义的。
>
>    ```css
>    css选择器 {
>        key1: value1;
>        key2: value2;
>        ...
>        keyn: valuen;
>    }
>    ```
>
> 3. 然后这个字体`"Font Awesome 5 Brands"`也需要定义，我们点右侧的`promin.css:4`
>
> 4. 以下就是点击后的界面
>
>    ![image-20211110085427461](http://myapp.img.mykernel.cn/image-20211110085427461.png)
>
>    首先先点左下角展开css
>
>    然后看@font-face这个css，就是定义字体的，定义名称、字是否斜体、粗细、字体显示格式，字体文件引用、...

## 引用字体

因为以上这个字体文件引用的是`../webfonts/pro-fa-brands-400-5.15.4.eot?#iefix`，所以需要找到此`prom.min.css`文件的位置

![image-20211110085719683](http://myapp.img.mykernel.cn/image-20211110085719683.png)

然后我们将看到左边会定位到这个css文件

![image-20211110085751262](http://myapp.img.mykernel.cn/image-20211110085751262.png)

> 可以发现css文件中引用的相对路径的webfonts目录
>
> 目录中有很多字体文件，我们只需要这一个@font-face中url引用的所有字体文件名，并下载

## 获取编码

从获取字体`2.3`节中，我们看到除了`fab`类还一个`fa-500px`类名，我们在`pro.min.css`文件中搜索(ctrl +f)这个类名

![image-20211110090117545](http://myapp.img.mykernel.cn/image-20211110090117545.png)

## 生成图标

参考: http://blog.mykernel.cn/2021/11/09/hexo%E6%B7%BB%E5%8A%A0%E5%AD%90%E8%8F%9C%E5%8D%95submenu/#%E5%87%86%E5%A4%87css



# 引用iconfont.com

## 简单示例

这个就比较简单了，使用示例

1. 注册iconfont.com

2. 进入这个页面, https://www.iconfont.cn/help/detail?spm=a313x.7781069.1998910419.15&helptype=about 

3. 将图标加入购物车后，可以批量下载素材和代码、批量添加至项目

   在首页搜索一个关键词，之后出现很多图标，每一个图标当鼠标悬停其上时，就会有3个图标，第1个就是购物车、第2个是收藏

   ![image-20211110090616277](http://myapp.img.mykernel.cn/image-20211110090616277.png)

   

4. 添加很多图标到购物车之后，选择右上角的购物车, 可以添加到项目

   ![image-20211110090733834](http://myapp.img.mykernel.cn/image-20211110090733834.png)

5. 添加到项目之后 

   https://www.iconfont.cn/help/detail?spm=a313x.7781069.1998910419.15&helptype=about 这个页面的**项目管理**



## 如何使用素材

> 官方教程: https://www.iconfont.cn/help/detail?spm=a313x.7781069.1998910419.17&helptype=code

### unicode

unicode是字体在网页端最原始的应用方式

- 兼容好
- 操作字体属性
- 单色图标

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		@font-face {
		  font-family: 'iconfont';  /* Project id 2926463 */
		  src: 
		       url('https://at.alicdn.com/t/font_2926463_b8alglmjkxe.woff2?t=1636462429700') format('woff2'),
		       url('https://at.alicdn.com/t/font_2926463_b8alglmjkxe.woff?t=1636462429700') format('woff'),
		       url('https://at.alicdn.com/t/font_2926463_b8alglmjkxe.ttf?t=1636462429700') format('truetype');
		}
		.iconfont{
		    font-family:"iconfont" !important;
		    font-size:16px;font-style:normal;
		    -webkit-font-smoothing: antialiased;
		    -webkit-text-stroke-width: 0.2px;
		    -moz-osx-font-smoothing: grayscale;
		}
	</style>
</head>
<body>
	<i class="iconfont">&#xe6a0;</i>
	<i class="iconfont">&#x33;</i>
</body>
</html>
```

### font class

font-class是unicode使用方式的一种变种，主要是解决unicode书写不直观，语意不明确的问题

- 兼容性良好，支持ie8+，及所有现代浏览器
- 相比于unicode语意明确，书写更直观。可以很容易分辨这个icon是什么
- 因为使用class来定义图标，所以当要替换图标时，只需要修改class里面的unicode引用。
- 不过因为本质上还是使用的字体，所以多色图标还是不支持的。

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
@font-face {
  font-family: "iconfont"; /* Project id 2926463 */
  /* Color fonts */
  src: 
       url('https://at.alicdn.com/t/font_2926463_b8alglmjkxe.woff2?t=1636462429700') format('woff2'),
       url('https://at.alicdn.com/t/font_2926463_b8alglmjkxe.woff?t=1636462429700') format('woff'),
       url('https://at.alicdn.com/t/font_2926463_b8alglmjkxe.ttf?t=1636462429700') format('truetype');
}

.iconfont {
  font-family: "iconfont" !important;
  font-size: 16px;
  font-style: normal;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
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

	</style>
</head>
<body>
	<i class="iconfont icon-favorite"></i>
</body>
</html>
```



### symbol

这是一种全新的使用方式，应该说这才是未来的主流，也是平台目前推荐的用法。相关介绍可以参考这篇[文章](https://www.iconfont.cn/help/detail?spm=a313x.7781069.1998910419.17&helptype=code) 这种用法其实是做了一个svg的集合，与上面两种相比具有如下特点：

- 支持多色图标了，不再受单色限制。
- 通过一些技巧，支持像字体那样，通过`font-size`,`color`来调整样式。
- **兼容性较差**，支持 ie9+,及现代浏览器。
- 浏览器渲染svg的**性能一般**，还不如png。

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<script src="https://at.alicdn.com/t/font_2926463_b8alglmjkxe.js?spm=a313x.7781069.1998910419.78&file=font_2926463_b8alglmjkxe.js"></script>
	<style>
		.icon {
	       width: 1em; height: 1em;
	       vertical-align: -0.15em;
	       fill: currentColor;
	       overflow: hidden;
    	}
	</style>
</head>
<body>
	<svg class="icon" aria-hidden="true">
    <use xlink:href="#icon-favorite"></use>
</svg>
</body>
</html>
```

