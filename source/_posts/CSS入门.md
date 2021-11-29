---
title: CSS入门
date: 2021-09-30 20:07:17
tags:
- 黑马前端
---

# CSS第1天

## 学习目标

- 理解
  - css目的，作用，3种引入方式

- 应用
  - 书写3种css方式 
  - 通过样式规则给标签添加简单的样式

## HTML局限性

html只关心内容和排版，但是太丑了。

- html不能满足设计者
- 如果修改行的高度，需要给每一个tr标签添加`height`属性，如果有1万行呢？在HTML里面添加样式非常臃肿和繁琐。

## CSS 网页的美容师

让网页更丰富多彩，布局更加灵活自如

CSS的最大的贡献，让HTML从样式脱离，**HTML专注用标签做结构呈现，样式交给CSS.**（结构和样式分离）

js是网页的魔法师

## CSS初识

CSS(cascading style sheets), 层叠样式表

作用:

1. 设置版面的**布局**和**外观**显示样式
2. 丰富功能：字体、颜色、背景的控制及整体排版等，还可以针对不同的浏览器设置不同的样式。

## CSS写在页面中哪个位置 （书写位置 ）

### 标签内联样式

语法 

```html
<标签名 style="属性1：属性值; 属性2：属性值;...."> 内容 </标签名>
```

- 任何html标签都拥有style属性，用来设置标签内联样式
- 每个属性值和属性之间空格分开，key: value之后要写分号

缺点：会和html写在一起，不经常使用



#### 没有颜色

![image-20210929223322926](http://myapp.img.mykernel.cn/image-20210929223322926.png)

#### 添加颜色 `color: pink;`

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
	<h1 style="color: pink;">世纪佳缘，我也在这里等你思密达</h1>
</body>
</html>
```

> 颜色发生了变化
>
> ![image-20210929223427042](http://myapp.img.mykernel.cn/image-20210929223427042.png)

#### 字体修改 `font-size: 18px`

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
	<h1 style="color: pink; font-size: 18px;">世纪佳缘，我也在这里等你思密达</h1>
</body>
</html>
```

> 18像素， **每个属性值和属性之间空格分开，key: value之后要写分号**
>
> ![image-20210929223701287](http://myapp.img.mykernel.cn/image-20210929223701287.png)

#### Input中提示语使用灰色 `color: gray;`

引用自：http://blog.mykernel.cn/2021/09/23/%E5%89%8D%E8%A8%80%E5%8F%8Aweb%E6%A0%87%E5%87%86/#%E8%87%AA%E5%B7%B1%E5%81%9A%E7%9A%84%E7%BB%93%E6%9E%9C

```html
		<tr>
			<td>所在地区</td>
			<td>
				<input type="text" name="region" value="北京" style="color: gray;">
			</td>
		</tr>
```

> ![image-20210929224049185](http://myapp.img.mykernel.cn/image-20210929224049185.png)



#### chrome快速添加标签内联样式

![image-20211118194523306](http://myapp.img.mykernel.cn/image-20211118194523306.png)

### 内嵌样式

写在head头部标签中

#### 位置在head标签中

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>忆江南</title>
	<style type="text/css">
		/*CSS文本*/
	</style>
</head>
<body>
<h1>忆江南(1)</h1>

唐.白居易

<p>江南好，风景旧曾谙。(2) 日出江花红胜火，春来江水绿如蓝，(3) 能不忆江南。</p>

<h4>作者介绍</h4>

<p>白居易(772－846) ，字乐天，太原(今属山西)人。唐德宗朝进士，元和三年(808)拜左拾遗，后贬江州(今属江西)司马，移忠州(今属四川)刺史，又为苏州(今属江苏)、同州(今属陕西大荔)刺史。晚居洛阳，自号醉吟先生、香山居士。其诗政治倾向鲜明，重讽喻，尚坦易，为中唐大家。也是早期词人中的佼佼者，所作对后世影响甚大。</p>
<h4>注释</h4>
  
 <p> (1)据《乐府杂录》，此词又名《谢秋娘》，系唐李德裕为亡姬谢秋娘作。又名《望江南》、《梦江南》等。分单调、双调两体。单调二十七字，双凋五十四字，皆平韵。(2)谙(音安)：熟悉。(3)蓝：蓝草，其叶可制青绿染料。</p>
<h4>品评</h4>
  
 <p> 此词写江南春色，首句“江南好”，以一个既浅切又圆活的“好”字，摄尽江南春色的种种佳处，而作者的赞颂之意与向往之情也尽寓其中。同时，唯因“好”之已甚，方能“忆”之不休，因此，此句又已暗逗结句“能不忆江南”，并与之相关阖。次句“风景旧曾谙”，点明江南风景之“好”，并非得之传闻，而是作者出牧杭州时的亲身体验与亲身感受。这就既落实了“好”字，又照应了“忆”字，不失为勾通一篇意脉的精彩笔墨。三、四两句对江南之“好”进 　行形象化的演绎，突出渲染江花、江水红绿相映的明艳色彩，给人以光彩夺目的强烈印象。其中，既有同色间的相互烘托，又有异色间的相互映衬，充分显示了作者善于著色的技巧。篇末，以“能不忆江南”收束全词，既托出身在洛阳的作者对江南春色的无限赞叹与怀念，又造成一种悠远而又深长的韵味，把读者带入余情摇漾的境界中。</p> 
</body>
</html>
```

> 没有样式
>
> ![image-20210929224855742](http://myapp.img.mykernel.cn/image-20210929224855742.png)

#### 写css样式

达到的效果

![image-20210929224906988](http://myapp.img.mykernel.cn/image-20210929224906988.png)

##### 将`<h1>忆江南(1)</h1>`变成绿色

在以上代码中的head中CSS文本处写以下代码

```html
<head>
	<meta charset="utf-8">
	<title>忆江南</title>
	<style type="text/css">
		/*CSS文本*/
		h1 {
			color: green;
		}
	</style>
</head>
```

> 1. h1表示 css选择了h1标签
> 2. {} 这是固定格式，其中就css属性和值
>
> 看页面
>
> ![image-20210929225302908](http://myapp.img.mykernel.cn/image-20210929225302908.png)

##### 将以上标题缩小一点

以上选择的h2中回车写下一个处理器即可

```css
		h1 {
			color: green;
			font-size: 20px;
		}
```

##### 将作者介绍更新为紫色

F12中查看`作者介绍`是 `h4`

![image-20210929225532846](http://myapp.img.mykernel.cn/image-20210929225532846.png)

所以更新css选择器和属性

```html
<head>
	<meta charset="utf-8">
	<title>忆江南</title>
	<style type="text/css">
		/*CSS文本*/
		h1 {
			color: green;
			font-size: 20px;
		}
		h4 {
			color: purple;
		}
	</style>
</head>
```

> ![image-20210929225649990](http://myapp.img.mykernel.cn/image-20210929225649990.png)
>
> - 现在可以发现内嵌样式的优点
>   1. 只需要选择所有标签
>   2. 而标签内联时，需要每个h4标签上写样式

##### 将文本改成蓝色

F12查看文本的标签`p`, 更新内嵌样式

```html
	<style type="text/css">
		/*CSS文本*/
		h1 {
			color: green;
			font-size: 20px;
		}
		h4 {
			color: purple;
		}
		p {
			color: blue;
		}
	</style>
```

> ![image-20210929225915785](http://myapp.img.mykernel.cn/image-20210929225915785.png)



### 外部样式 `link`

- 实现CSS样式共享

- 所有样式以`.css`结尾的文件放在`css`目录中

准备一个html

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>document</title>
	
</head>
<body>
	<h3>来呀！</h3>
</body>
</html>
```

> 浏览器中查看
>
> ![image-20210930095853550](http://myapp.img.mykernel.cn/image-20210930095853550.png)

现在把h3文字变成颜色

准备`css/style.css`

```css
h3 {
	color: pink;
}
```

然后在页面中引入这个css

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>document</title>
	<link rel="stylesheet" type="text/css" href="./css/style.css" />
</head>
<body>
	<h3>来呀！</h3>
</body>
</html>
```

> `<link rel="stylesheet" type="text/css" href="./css/style.css" />` 定义css来源
>
> | 属性 | 属性值     | 描述            |
> | ---- | ---------- | --------------- |
> | rel  | stylesheet | 引入css的样式表 |
> | type | text/css   | 文本类型的css   |
> | href | 引入url    | 文件路径、url   |

> 现在网页中查看
>
> ![image-20210930100139876](http://myapp.img.mykernel.cn/image-20210930100139876.png)

现在把这个css样式共享给其他页面使用

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>外部样式表</title>
	<link rel="stylesheet" type="text/css" href="./css/style.css">
</head>
<body>
	<h3>引入其他样式! </h3>
</body>
</html>
```

> 也引入了之前的样式，刷新也可以调用这个样式了
>
> ![image-20210930101203022](http://myapp.img.mykernel.cn/image-20210930101203022.png)

### css引入总结

<table border="0">
    <thead>
        <tr>
            <th>书写位置分类</th>
            <th>名称</th>
            <th>代码</th>
            <th>优点</th>
            <th>缺点</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td rowspan="2">内部样式表</td>
            <td>标签内联</td>
            <td>
            	<code>
                    &lt;body&gt; <br />
    &lt;h1 style="color: pink;"&gt;世纪佳缘，我也在这里等你思密达&lt;/h1&gt; <br />
&lt;/body&gt; <br />
                </code>
            </td>
        <td><ul>
            <li>临时使用</li>
            <li><strong style="color: purple;">控制1个标签</strong></li>
            </ul></td>
        <td>臃肿，重复代码多</td>
    </tr>
    <tr>
        <td>head内嵌，理论上可以放在html中任何位置，<span style="color: purple; font-size: 20px;">
                习惯放在
            </span><code>head</code>中
        </td>
        <td>
            <code>
                &lt;head&gt;<br />
                    &lt;style type="text/css"&gt;<br />
                        /*CSS文本*/<br />
                        h1 {<br />
                            color: green;<br />
                        }<br />
                    &lt;/style&gt;<br />
                &lt;/head&gt;<br />
            </code>
        </td>
    <td>
        <ul>
            <li>结构和样式分离， html5中<code>&lt;!DOCTYPE html&gt;</code> 这种文档格式，可以省略<code>&lt;style type="text/css"&gt;</code>中的<code>type="text/css"</code></li>
            <li><strong style="color: purple;">控制一个页面</strong></li>
        </ul>
        </td>
    <td>
        <p>
            只能管当前页面
        </p>
        <p>
            <strong>由于可以放随意位置，放在其他位置时，结构和样式就不分离了</strong>
        </p>
    </td>
</tr>
<tr>
  <td>外链样式表</td>
  <td>link外链</td>
    <td>
    <code>
       &lt;head&gt; <br />
    &lt;link rel="stylesheet" type="text/css" href="./css/style.css"&gt;<br />
&lt;/head&gt;<br />
     </code>
    </td>
  <td>
    <ul>
      <li>结构和样式完全分离</li>
      <li>可以共享css给页面使用 <strong style="color: purple;">控制整个站点</strong></li>
      <li>一般放在css目录中</li>
      <li>一般可以省略 type="text/css" </li>
    </ul>
  </td>
  <td></td>
</tr>

现在的这个例子，需要把所有行的高度调整高一点, 通过以上的内部样式表的 `head内嵌`

>  引用自：http://blog.mykernel.cn/2021/09/23/%E5%89%8D%E8%A8%80%E5%8F%8Aweb%E6%A0%87%E5%87%86/#%E8%87%AA%E5%B7%B1%E5%81%9A%E7%9A%84%E7%BB%93%E6%9E%9C

```html
	<style>
		tr {
			height: 40px;
		}
	</style>
```

<div class="admonition warning">
    <p>
        css中，所有单位一般为px
    </p>
</div>

团队风格

1. css紧凑，不提倡，阅读不方便。

   ```css
   h3 {
       color: deeppink;
       font-size: 20px;
       width: 50px;
   }
   ```

   > 这样方便和别人共享代码

   横着写，节约空间，当我们整体写完之后，我们可以把css压缩之后上传。

2. html和css统一小写

### css样式规则总结

```css
css选择器 {
    样式属性: 样式值;
    样式属性: 样式值;
    ...
}
```

- 属性: 大小、颜色、宽度、高度, ....
- 属性和值之间`: `和`空格`分隔，值结尾是`;`结尾

## css选择器

### 学习目标

- 理解
  - 作用
  - id选择器和类选择区别
- 应用
  - 能够使用基础选择器，给页面添加样式。

### 作用

<span style="color: purple;">css选择器</span>**分组html标签，按分组来赋予不同的样式**

把h3所有标签分成一组，赋予红色 

```css
h3 {
    color: red;
}
```

把h2所有标签分成一组，赋予紫色

```css
h2 {
    color: purple;
}
```

### 基础选择器

#### 标签选择器

html的标签名来选择

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>基础选择器</title>
</head>
<body>
	标签选择器:
	<div>标签选择器</div>
	<div>页面同选起</div>
	<div>直接写标签</div>
	<div>全部不放弃</div>
	<span>标签选择器</span>
	<span>页面同选起</span>
	<span>直接写标签</span>
	<span>全部不放弃</span>
</body>
</html>
```

> - 页面中会选择同类的所有标签
>
> ![image-20210930104408157](http://myapp.img.mykernel.cn/image-20210930104408157.png)



所有div改成红色, 所有span改成绿色

```css
<head>
	<meta charset="utf-8">
	<title>基础选择器</title>
	<style type="text/css">
		div {
			color: red;
		}
		span {
			color: green;
		}
	</style>
</head>
```

![image-20210930104547891](http://myapp.img.mykernel.cn/image-20210930104547891.png)



缺点：一改会修改相同标签的所有样式

#### 类选择器 (可以重复使用, 样式常用)

##### 单类名

优点：选择相同标签名所有标签中，任意单个或多个

1. `.`开始，后接类的名称，表示匹配类所属的标签
2. 类名不要纯数字或中文名称

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>基础选择器</title>
	<style type="text/css">

	</style>
</head>
<body>
	差异化选择器:
	<div>差异化选择</div>
	<div>一个或多个</div>
	<div>上面点定义</div>
	<div>类名别写错</div>
	<div>class来做</div>
	<div>工作类最多</div>
</body>
</html>
```

> - `.`开始，后接类的名称，表示匹配类所属的标签

现在想让最后一个div变色, 在这个div标签上，添加属性`class="red"`

```html
	<div class="red">工作类最多</div>
```

现在使用内嵌样式，给red类匹配的标签添加红色

```css
	<style type="text/css">
		.red {
			color: red;
		}
	</style>
```

> ![image-20210930105210550](http://myapp.img.mykernel.cn/image-20210930105210550.png)

使用类选择器实现以下图

![image-20210930105822607](http://myapp.img.mykernel.cn/image-20210930105822607.png)

```html
<!DOCTYPE html>
<html>
	<head>
	    <meta charset="utf-8">
	    <title>google</title>
	    <style type="text/css">
		    .blue {
		        color: blue;
		        font-size: 100px;
		    }

		    .red {
		        color: red;
		        font-size: 100px;
		    }

		    .orange {
		        color: orange;
		        font-size: 100px;
		    }

		    .green {
		        color: green;
		        font-size: 100px;
		    }
	    </style>
	</head>

	<body>
	    <span class="blue">G</span>
	    <span class="red">o</span>
	    <span class="orange">o</span>
	    <span class="blue">g</span>
	    <span class="green">l</span>
	    <span class="red">e</span>
	</body>
</html>
```

- 1行显示，`span`布局
- 控制颜色使用类

##### 多类名

- `class`属性值，有多个类名时，需要空格分开
- 一个值是多个单词时，使用`-`连接

达到更多的选择目的

如上的google案例，每个类名里面都有`font-size: 100px;` 相当于每个类名有相同的字体

```html
<!DOCTYPE html>
<html>
	<head>
	    <meta charset="utf-8">
	    <title>google</title>
	    <style type="text/css">
	    	.font100 {
	    		font-size: 100px;
	    	}
		    .blue {
		        color: blue;
		    }

		    .red {
		        color: red;
		    }

		    .orange {
		        color: orange;
		    }

		    .green {
		        color: green;
		    }
	    </style>
	</head>

	<body>
	    <span class="blue font100">G</span>
	    <span class="red font100">o</span>
	    <span class="orange font100">o</span>
	    <span class="blue font100">g</span>
	    <span class="green font100">l</span>
	    <span class="red font100">e</span>
	</body>
</html>
```

> 现在浏览器还是相同的结果

##### 类命名规范

```bash
　　头：header
　　内容：content/container
　　尾：footer
　　导航：nav
　　侧栏：sidebar
　　栏目：column
　　页面外围控制整体布局宽度：wrapper
　　左右中：left right center
　　登录条：loginbar
　　标志：logo
　　广告：banner
　　页面主体：main
　　热点：hot
　　新闻：news
　　下载：download
　　子导航：subnav
　　菜单：menu
　　子菜单：submenu
　　搜索：search
　　友情链接：friendlink
　　页脚：footer
　　版权：copyright
　　滚动：scroll
　　内容：content
　　标签页：tab
　　文章列表：list
　　提示信息：msg
　　小技巧：tips
　　栏目标题：title
　　加入：joinus
　　指南：guild
　　服务：service
　　注册：regsiter
　　状态：status
　　投票：vote
　　合作伙伴：partner
```



#### id选择器 (惟一的, js常用)

使用方式和单类名选择器一样的

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>document</title>
</head>
<body>
	<p>愿你走出半生</p>
	<p>归来仍是少年</p>
	<p>愿我洗尽铅华</p>
	<p>依旧笑靥如花</p>
</body>
</html>
```

现在把`依旧笑靥如花` 变成绿色

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>document</title>
	<style type="text/css">
		#pink {
			color: pink;
		}
	</style>
</head>
<body>
	<p>愿你走出半生</p>
	<p>归来仍是少年</p>
	<p>愿我洗尽铅华</p>
	<p id="pink">依旧笑靥如花</p>
</body>
</html>
```

> ![image-20210930111453817](http://myapp.img.mykernel.cn/image-20210930111453817.png)

id选择器和类选择器一样，区别是什么？

- 元素的id是惟一的， 例如：身份证号码
- 类名是可以重复使用的，例如：人名

##### id命名规范

```bash
（1）页面结构
　　容器： container
　　页头：header
　　内容：content/container
　　页面主体：main
　　页尾：footer
　　导航：nav
　　侧栏：sidebar
　　栏目：column
　　页面外围控制整体布局宽度：wrapper
　　左右中：left right center
　　（2）导航
　　导航：nav
　　主导航：mainbav
　　子导航：subnav
　　顶导航：topnav
　　边导航：sidebar
　　左导航：leftsidebar
　　右导航：rightsidebar
　　菜单：menu
　　子菜单：submenu
　　标题： title
　　摘要： summary
　　（3）功能
　　标志：logo
　　广告：banner
　　登陆：login
　　登录条：loginbar
　　注册：regsiter
　　搜索：search
　　功能区：shop
　　标题：title
　　加入：joinus
　　状态：status
　　按钮：btn
　　滚动：scroll
　　标签页：tab
　　文章列表：list
　　提示信息：msg
　　当前的： current
　　小技巧：tips
　　图标： icon
　　注释：note
　　指南：guild
　　服务：service
　　热点：hot
　　新闻：news
　　下载：download
　　投票：vote
　　合作伙伴：partner
　　友情链接：link
　　版权：copyright
```



#### 通配符选择器

使用`*`表示选择所有<strong>标签</strong>

缺点：会匹配页面所有元素，会降低页面的响应速度，不建议使用。

让所有标签变色

```css
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>document</title>
	<style type="text/css">
		#pink {
			color: pink;
		}
		* {
			color: red;
		}
	</style>
</head>
<body>
	<p>愿你走出半生</p>
	<p>归来仍是少年</p>
	<p>愿我洗尽铅华</p>
	<p id="pink">依旧笑靥如花</p>
</body>
</html>
```

> ![image-20210930112207515](http://myapp.img.mykernel.cn/image-20210930112207515.png)

常用于， 清理所有标签的内外边距

```html
* {
	margin: 0;           /* 定义外边距 */
	padding: 0;          /* 定义内边距 */
}
```

### 复合选择器



### css选择总结

| 类别       | 选择器       | 作用                                                      | 缺点                                                 | 用法                |
| ---------- | ------------ | --------------------------------------------------------- | ---------------------------------------------------- | ------------------- |
| 基础选择器 | 标签选择器   | 选择相同的标签名                                          | 不能选择一类标签中的其中1个或多标签                  | p {color: red; }    |
|            | 类选择器     | **可以选择1个或多个标签**，通过用来控制样式               |                                                      | .nav {color: red; } |
|            | id选择器     | 一次**只能选择一个标签**                                  | id通过和`js`搭配使用                                 | #nav {color: red; } |
|            | 通配符选择器 | 选择所有标签，通常结合margin, padding清除所有标签的边距。 | 选择太多，部分不需要这个样式。会影响浏览器加载速度。 | * {color: red; }    |
| 复合选择器 |              |                                                           |                                                      |                     |

**约定**

1. 不用`*`通配
2. 尽量少用`id`选择器来控制样式
3. 不使用布局标签(`div`, `span`, 没有具体语义)来作为标签选择器

## 常见样式

- 应用
  - 字体样式，字号、字体
  - 外观：颜色、高度
  - 使用常用的emment语法
  - 使用开发人员工具代码调试



### font字体

https://www.w3.org/TR/CSS21/fonts.html#q15.0

#### font-size 大小

```css
p {
    font-size: 20px
}
```

单位

| 相对长度单位 | 说明                       |
| ------------ | -------------------------- |
| em           | 相对于当前对象内的字体尺寸 |
| **px**       | 像素，常用                 |

| 绝对长度单位 | 说明 |
| ------------ | ---- |
| in           | 英寸 |
| cm           | 厘米 |
| mm           | 毫米 |
| pt           | 点   |

- 常用px

- google, firefox 默认 font-size 16px
- 不同浏览器的大小不一致，一般给body指定整个页面的大小

统一大小

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>font-size</title>
	<style type="text/css">
		body {
			font-size: 16px;
		}
	</style>
</head>
<body>
	<p>粉红色的回忆</p>
	<p>夏天夏天</p>
</body>
</html>
```

> ![image-20210930114735598](http://myapp.img.mykernel.cn/image-20210930114735598.png)

一般，这个第1个是标题，字号大小要给大一些

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>font-size</title>
	<style type="text/css">
		body {
			font-size: 16px;
		}
		.title {
			font-size: 20px;
		}
	</style>
</head>
<body>
	<p class="title">粉红色的回忆</p>
	<p>夏天夏天</p>
</body>
</html>
```

> 现在浏览器的这个标题就变大了
>
> ![image-20210930114844111](http://myapp.img.mykernel.cn/image-20210930114844111.png)

#### font-family 字体

- 字体值 英文没有空格时就可以不加引号，其他情况均需要加引号。

- 值有多个时，逗号分隔，浏览器以短路机制来获取字体。
- 英文不区分大小写
- 值支持unicode字体
- 一般常用宋体和微软雅黑，让大部分电脑正常显示字体

```css
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>font-size</title>
	<style type="text/css">
		body {
			font-size: 16px;
		}
		.title {
			font-size: 20px;
			font-family: "宋体";
		}
	</style>
</head>
<body>
	<p class="title">粉红色的回忆</p>
	<p>夏天夏天</p>
</body>
</html>
```

> ![image-20210930115111540](http://myapp.img.mykernel.cn/image-20210930115111540.png)

```css
			font-family: Arial, "Microsoft YaHei", "微软雅黑", "宋体";
```

> - 浏览器从前向右查找字体，第1个找到就立即显示，不向后查找。都没有查找到，就使用电脑的默认字体。

浏览器不支持中文时，使用英文或unicode编码

```css
{
    font-family: arial, "\5b8b\4f53"
}
```

|  字体名称   |    英文名称     |     Unicode 编码     |
| :---------: | :-------------: | :------------------: |
|    宋体     |     SimSun      |      \5B8B\4F53      |
|   新宋体    |     NSimSun     |   \65B0\5B8B\4F53    |
|    黑体     |     SimHei      |      \9ED1\4F53      |
|  微软雅黑   | Microsoft YaHei | \5FAE\8F6F\96C5\9ED1 |
| 楷体_GB2312 |  KaiTi_GB2312   |  \6977\4F53_GB2312   |
|    隶书     |      LiSu       |      \96B6\4E66      |
|    幼园     |     YouYuan     |      \5E7C\5706      |
|  华文细黑   |     STXihei     | \534E\6587\7EC6\9ED1 |
|   细明体    |     MingLiU     |   \7EC6\660E\4F53    |
|  新细明体   |    PMingLiU     | \65B0\7EC6\660E\4F53 |

> 来源: http://code.ciaoca.com/style/cssfont2unicode/

一般常用宋体和微软雅黑

#### font-weight 字体粗细

- 语义标签的 `hx`
- css的，没有语义

| 属性值  | 描述                            |
| ------- | ------------------------------- |
| normal  | 默认值(不加粗的),相当不加此属性 |
| bold    | 定义粗体（加粗）                |
| 100-900 | 400等于normal, 700等于bold.     |

现在上面的页面，粉红色的回忆，应该加粗

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>font-size</title>
	<style type="text/css">
		body {
			font-size: 16px;
		}
		.title {
			font-size: 20px;
			font-family: Arial, "Microsoft YaHei", "微软雅黑", "宋体";
			font-weight: 700;
		}
	</style>
</head>
<body>
	<p class="title">粉红色的回忆</p>
	<p>夏天夏天</p>
</body>
</html>
```

> ![image-20210930120124548](http://myapp.img.mykernel.cn/image-20210930120124548.png)



#### font-style 字体风格 斜

- 语义标签: i, em
- css, 没有语义

<div class="admonition warning" >
    <p>
        实际中，我们很少使用CSS给字体加斜体效果
    </p>
    <p>
        反而，我们经常把斜体修改成普通模式
    </p>
</div>



把`粉红色的回忆` 粉红

```html
		.title {
			font-size: 20px;
			font-family: Arial, "Microsoft YaHei", "微软雅黑", "宋体";
			font-weight: 700;
			font-style: italic;
		}
```

| 属性   | 作用 |
| ------ | ---- |
| normal | 默认 |
| italic | 斜体 |

普通斜体

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>font-size</title>
	<style type="text/css">
		body {
			font-size: 16px;
		}
		.title {
			font-size: 20px;
			font-family: Arial, "Microsoft YaHei", "微软雅黑", "宋体";
			font-weight: 700;
			font-style: italic;
		}
	</style>
</head>
<body>
	<p class="title">粉红色的回忆</p>
	<p>夏天夏天</p>
	<em>音乐专辑</em>
</body>
</html>
```

> ![image-20210930123717297](http://myapp.img.mykernel.cn/image-20210930123717297.png)

不让他斜体

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>font-size</title>
	<style type="text/css">
		body {
			font-size: 16px;
		}
		.title {
			font-size: 20px;
			font-family: Arial, "Microsoft YaHei", "微软雅黑", "宋体";
			font-weight: 700;
			font-style: italic;
		}
		em {
			font-style: normal;
		}
	</style>
</head>
<body>
	<p class="title">粉红色的回忆</p>
	<p>夏天夏天</p>
	<em>音乐专辑</em>
</body>
</html>
```

> 回正

#### font 综合设置字体样式(重点)

可以将前面的缩写成一句话

```css
选择器 {
    font: font-style font-weight font-size/line-height font-family;
}
```

<div class="admonition warnging">
    <p class="admonition-title">
        注意
    </p>
    <p>
        上面后2个的<span style="color: red; font: normal 700 20px Arial;">font-size, font-family</span>一定不能省
    </p>
    <p>
        其他可以省略
    </p>
</div>



可以可以简写之前的

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>font-size</title>
	<style type="text/css">
		body {
			font-size: 16px;
		}
		.title {
			font: italic 700 20px Arial, "Microsoft YaHei", "微软雅黑", "宋体";
		}
		em {
			font-style: normal;
		}
	</style>
</head>
<body>
	<p class="title">粉红色的回忆</p>
	<p>夏天夏天</p>
	<em>音乐专辑</em>
</body>
</html>
```

> 现在预览的结果，一模一样

#### font 总结

| 属性        | 表示         | 注意点                                                  |
| ----------- | ------------ | ------------------------------------------------------- |
| font-size   | 字大小或字号 | px                                                      |
| font-family | 字体         | 按团队约定，通常宋体和微软雅黑, 实际按公司约定          |
| font-weight | 字体粗细     | bold/normal                                             |
| font-style  | 字体倾斜     | italic/normal，实际中常用normal                         |
| font        | 字体连写     | <ul><li>有顺序</li><li>字体大小和字体必须存在</li></ul> |

### 文档

css 2

## 外观属性

### color颜色

```css
color: 
```

| 颜色分类 | 值                                                           |
| -------- | ------------------------------------------------------------ |
| 预定义   | red, green, blue, pink, ...                                  |
| 十六进制 | #FF0000, #FF600,  #29D794,#号开头<br/>#rrggbb 表示红占rr, 绿占gg, 蓝占bb. 每一项的范围是0-F。<br/>如果结果是#FF0000,<span type="color: #F00;"> **两两相同时，可以简写为2个字母为1个字母**</span>, #F00 |
| RGB代码  | rgb(255,0,0)或rgb(100%,0%,0%)                                |

`十六进制` 

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title></title>
	<style type="text/css">
		h1 {
			color: #000000;
		}
	</style>
</head>
<body>
	<h1>你好</h1>
</body>
</html>
```

> 黑色

```css
color: #FFFFFF; # 白色
color: #FF0000; # 红色
color: #00ff00; # 绿
color: #0000ff; # 蓝色
```

获取屏幕颜色工具`FSCapture`

https://www.faststonecapture.cn/download

打开程序后，最右方点开设置，里面有一个**屏幕拾色器**, 直接复制, 放在style的color中，并且<span style="color: #FF6D84;">在前面加#号</span>。

![image-20210930133423499](http://myapp.img.mykernel.cn/image-20210930133423499.png)

### line-height 行间距 `24px`

- 理解
  - 行高和高度3种关系
  - 为什么行高等于高度单行文字会垂直居中
- 应用
  - 使用行高实现单行文字垂直居中
  - 能会测量行高

#### 测量行高

顶线：

中线：

底线：

基线：

基线与基线的距离：英文

底线与底线的距离：中文

测量方法：1. 使用fastone capture截图，2. fire work切片得到高度像素。

直接F12定位到行，查看CSS的`line-height`

#### 单行文本垂直居中

行高 = 上距离  +  盒子高度 +  下距离

让上下距离一样即可

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 100px;
			height: 50px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div>文字垂直居中</div>
</body>
</html>
```

![image-20211008132953615](http://myapp.img.mykernel.cn/image-20211008132953615.png)

现在是16px(文字高度，默认)+34px=50px 盒子高度

调整行高为50px

```css
			line-height: 50px;
```

50 - 16 = 34, 

#### 多行文本垂直居中

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 300px;
			height: 50px;
			line-height: 50px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div>123<br>567</div>
</body>
</html>
```

> 这样，换行之后，里面的文本就不能垂直居中了
>
> ![image-20211110204629119](http://myapp.img.mykernel.cn/image-20211110204629119.png)

#### 文字偏下

行高大于高度

```css
			line-height: 60px;
```

60-16=44，则平分是22和22。盒子才高度50，所以 22 + 16 + ？ = 50 ，则下边距为12. 文字会偏下。

#### 默认行高16px

在顶部

16的行高-16的文字=0，则上边距为0



**段落中的行间距**

比字号大7-8像素左右即可

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title></title>
	<style type="text/css">
		h1 {
			color: #F00;
		}
	</style>
</head>
<body>
	<h1>你好</h1>
	<p>第1行111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111</p>
</body>
</html>
```

> ![image-20210930140800504](http://myapp.img.mykernel.cn/image-20210930140800504.png)

现在给间距

```css
		p {
			line-height: 24px;
		}
```

> ![image-20210930140942011](http://myapp.img.mykernel.cn/image-20210930140942011.png)

### text-align 文本水平对齐方式

让盒子中的文本、行内元素、行内块均可以居中对齐

| 属性值 | 解释                                                         |
| ------ | ------------------------------------------------------------ |
| left   |                                                              |
| right  |                                                              |
| center | 常用 <strong style="color: red">注意多行文字也是水平居中</strong> |

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title></title>
	<style type="text/css">
		h1 {
			color: #F00;
			text-align:  center;
		}
	</style>
</head>
<body>
	<h1>你好</h1>
</body>
</html>
```

> ![image-20210930140101036](http://myapp.img.mykernel.cn/image-20210930140101036.png)

### text-indent 首行缩进 `2em`

段落前面空2个空白距离, 之前有特殊字符`&nbsp;` 给2个即可。但是太多段落，怎么一下控制？

值是不同单位，常用`em`单位, 1em表示1个字符宽度，2em即2个字符宽度，可以写nem, n表示数字。

```css
		p {
			line-height: 24px;
			text-indent: 2em;
		}
```

> ![image-20210930142128702](http://myapp.img.mykernel.cn/image-20210930142128702.png)

### text-decoration 文本的装饰

通常用来给链接`<a`标签修改装饰效果

| 值           | 描述                             |
| ------------ | -------------------------------- |
| none         | 默认，定义标准的文本，没有下划线 |
| underline    | 下划线                           |
| overline     | 上划线                           |
| line-through | 贯穿线                           |

```html
	<a href="#">链接</a>
```

```css
		a {
			text-decoration: none;
		}
```

> ![image-20210930143912030](http://myapp.img.mykernel.cn/image-20210930143912030.png)

同样也可以修改颜色

### 外观总结

| 属性            | 表示     | 注意                                                         |
| --------------- | -------- | ------------------------------------------------------------ |
| color           | 颜色     | 通常十六进制，可以使用工具拾取                               |
| line-height     | 段内行高 | `<p></p>`标签内的每行之间的高度，比字体大小多7-8个px. 即可<br /> 当行的高度和高度一样时，文字会垂直居中。(行高-16px)/2就是上边距的高度。 |
| text-align      | 水平对齐 | 文本对齐 center/right/left                                   |
| text-indent     | 首行缩进 | 段落首行的缩进 单位为em, 通常2em, 缩进2个字符距离            |
| text-decoration | 文本修饰 | 文字添加下划线:underline, `<a`标签取消下划线：none,          |

## 综合案例

先有结构才能写样式

### 原始页面

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
中乙队赛前突然换帅仍胜毅腾 高原黑马欲阻击舜天
2014年07月16日20:11  新浪体育 我有话说(195人参与) 收藏本文
　　新浪体育讯　7月16日是燕京啤酒2014中国足协杯第三轮比赛，丽江嘉云昊队主场迎战哈尔滨毅腾队的比赛日。然而就在比赛日中午，丽江嘉云昊队主帅李虎和另外两名成员悄然向俱乐部提出了辞呈，并且收拾行囊准备离开。在这样的情况下，丽江嘉云昊队不得不由此前的教练员杨贵东代理指挥了本场比赛。

　　在昨日丽江嘉云昊队主帅李虎就缺席了赛前的新闻发布会，当时俱乐部给出的解释是李虎由于身体欠佳，去医院接受治疗。然而今日李虎出现在俱乐部时，向记者否认了这一说法，并且坦言已经向俱乐部提出了辞呈。


　　据记者多方了解的情况，李虎极其教练组近来在执教成绩上承受了不小的压力，在联赛间歇期期间，教练组曾向俱乐部提出能够多引进有实力的球员补强球队，然而由于和俱乐部在投入以及成绩指标上的分歧，李虎最终和教练组一起在比赛日辞职。

　　这样的情况并没有影响到丽江嘉云昊队的队员，在比赛中丽江队在主场拼的非常凶，在暴雨之中仍然发挥出了体能充沛的优势，最终凭借点球击败了中超球队哈尔滨毅腾，顺利晋级下一轮比赛。根据中国足协杯的赛程，丽江嘉云昊队将在本月23日迎战江苏舜天队。
</body>
</html>
```

> 浏览器打开是一堆
>
> ![image-20210930130103951](http://myapp.img.mykernel.cn/image-20210930130103951.png)

目标页面: http://sports.sina.com.cn/c/2014-07-16/20117255619.shtml

### 现在搭建结构

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
<h1>中乙队赛前突然换帅仍胜毅腾 高原黑马欲阻击舜天</h1>
<div>2014年07月16日20:11  新浪体育 我有话说(195人参与) 收藏本文</div>
<hr/>
　　<p>新浪体育讯　7月16日是燕京啤酒[微博]2014中国足协杯第三轮比赛，丽江嘉云昊队主场迎战哈尔滨毅腾队的比赛日。然而就在比赛日中午，丽江嘉云昊队[微博]主帅李虎和另外两名成员悄然向俱乐部提出了辞呈，并且收拾行囊准备离开。在这样的情况下，丽江嘉云昊队不得不由此前的教练员杨贵东代理指挥了本场比赛。</p>

　　<p>在昨日丽江嘉云昊队主帅李虎就缺席了赛前的新闻发布会，当时俱乐部给出的解释是李虎由于身体欠佳，去医院接受治疗。然而今日李虎出现在俱乐部时，向记者否认了这一说法，并且坦言已经向俱乐部提出了辞呈。</p>


　　<p>据记者多方了解的情况，李虎[微博]极其教练组近来在执教成绩上承受了不小的压力，在联赛间歇期期间，教练组曾向俱乐部提出能够多引进有实力的球员补强球队，然而由于和俱乐部在投入以及成绩指标上的分歧，李虎最终和教练组一起在比赛日辞职。</p>

　　<p>这样的情况并没有影响到丽江嘉云昊队的队员，在比赛中丽江队在主场拼的非常凶，在暴雨之中仍然发挥出了体能充沛的优势，最终凭借点球击败了中超球队哈尔滨毅腾，顺利晋级下一轮比赛。根据中国足协杯的赛程，丽江嘉云昊队将在本月23日迎战江苏舜天队。</p>
</body>
</html>
```

> - h1
>
> - div 第二行单行, 不要放p标签，因为下面会有p标签，风格不一样。
>
>   ![image-20210930130455717](http://myapp.img.mykernel.cn/image-20210930130455717.png)
>
> - hr 一条线
>
> - 其他p标签
>
> ![image-20210930130523932](http://myapp.img.mykernel.cn/image-20210930130523932.png)

### 配置样式

#### 样式统一字体大小

- 浏览器的字体大小风格统一 16px

  ```css
  <head>
  	<meta charset="utf-8">
  	<title>Document</title>
  	<style type="text/css">
  		body {
  			font-size: 16px;
  		}
  	</style>
  </head>
  ```

#### 样式控制标题

- 标题太粗了，也太大了

  在以上的style中添加以下行

  ```css
  		.title {
  			font-size: 25px;
  			font-weight: normal;
  		}
  ```

  ```html
  <h1 class="title">中乙队赛前突然换帅仍胜毅腾 高原黑马欲阻击舜天</h1>
  ```

  

- 最后一句话加粗

  ```html
  　　<p>这样的情况并没有影响到丽江嘉云昊队的队员，在比赛中丽江队在主场拼的非常凶，在暴雨之中仍然发挥出了体能充沛的优势，最终凭借点球击败了中超球队哈尔滨毅腾，顺利晋级下一轮比赛。<strong>根据中国足协杯的赛程，丽江嘉云昊队将在本月23日迎战江苏舜天队。</strong></p>
  ```

#### 样式控制突出文字

- `[微博] `专用样式，蓝色

比较重要的信息，在百度中搜索传知播客，F12可以查，鼠标放在红色的字上。

- chrome

![image-20210930131301546](http://myapp.img.mykernel.cn/image-20210930131301546.png)

- firefox

![image-20210930131401668](http://myapp.img.mykernel.cn/image-20210930131401668.png)

```html
<em>[微博]</em>
```

> ![image-20210930131728953](http://myapp.img.mykernel.cn/image-20210930131728953.png)

现在要求微博不倾斜

```css
		em {
			font-style: normal;
			color: skyblue;
		}
```

#### div中的颜色，单独控制

-  控制时间为灰色
   而且span标签在一行中，可以放多个，所以时间就使用span包裹
   
    ```html
    <div><span class="time">2014年07月16日20:11</span>  新浪体育 我有话说(195人参与) 收藏本文</div>
    ```
   
    然后在head中控制time类为灰色
   
    ```css
            .time {
                color: #ccc;
            }
    ```
   
    > 可用格式
    >
    > ```css
    > #cccccc
    > #ccc
    > gray
    > ```
   
    > 现在网页可以查看到
    >
    > ![image-20210930134248447](http://myapp.img.mykernel.cn/image-20210930134248447.png)

- 控制**新浪体育 我有话说(195人参与)**为红色, **收藏本文**添加链接

  这个颜色使用span

  ```css
  <div><span class="time">2014年07月16日20:11</span> <span class="purple">新浪体育 我有话说(195人参与)</span> <a href="#">收藏本文</a></div>
  ```

  ```css
  		.purple {
  			color: purple;
  		}
  ```

  > ![image-20210930134917875](http://myapp.img.mykernel.cn/image-20210930134917875.png)

- 在收藏本文之后有一个文本框和普通按钮

  ```html
  <div>
  	<span class="time">2014年07月16日20:11</span> 
  	<span class="purple">新浪体育 我有话说(195人参与)</span> 
  	<a href="#">收藏本文</a>
  	<input type="text" name="" value="请输入查询条件" />
  	<input type="button" name="" value="搜索" />
  </div>
  ```

  > ![image-20210930135155291](http://myapp.img.mykernel.cn/image-20210930135155291.png)

- 在收藏本文之后有一个文本框 红色字 和 普通按钮绿色字

  ```html
  	<input type="text" name="" value="请输入查询条件" class="search" />
  ```

  ```css
  		.search {
  			color: red;
  		}
  ```

  > ![image-20210930135322094](http://myapp.img.mykernel.cn/image-20210930135322094.png)

  ```html
  	<input type="button" name="" value="搜索" class="btn" />
  ```

  ```css
  		.btn {
  			color: green;
  			font-weight: bold;
  		}
  ```

  > ![image-20210930135520418](http://myapp.img.mykernel.cn/image-20210930135520418.png)

#### 配置对齐

由于html中, 以下2行，均需要居中对齐，所以引用一个css

```html
<h1>中乙队赛前突然换帅仍胜毅腾 高原黑马欲阻击舜天</h1>
<div>
	<span class="time">2014年07月16日20:11</span> 
	<span class="purple">新浪体育 我有话说(195人参与)</span> 
	<a href="#">收藏本文</a>
	<input type="text" name="" value="请输入查询条件" class="search" />
	<input type="button" name="" value="搜索" class="btn" />
</div>
```

添加一个水平居中对齐的css

```css
		.text-align-center {
			text-align: center;
		}
```

```html
<h1 class="title text-align-center">中乙队赛前突然换帅仍胜毅腾 高原黑马欲阻击舜天</h1>
<div class="text-align-center">
	<span class="time">2014年07月16日20:11</span> 
	<span class="purple">新浪体育 我有话说(195人参与)</span> 
	<a href="#">收藏本文</a>
	<input type="text" name="" value="请输入查询条件" class="search" />
	<input type="button" name="" value="搜索" class="btn" />
</div>
```

#### 配置行间距

```css
		p {
			line-height: 24px;
		}
```

#### 配置行首缩进2个字符

```css
		p {
			line-height: 24px;
			text-indent: 2em;
		}
```

> ![image-20210930142247245](http://myapp.img.mykernel.cn/image-20210930142247245.png)

#### 链接的线去掉

```css
		a {
			text-decoration: none;
		}
```

> ![image-20210930144058896](http://myapp.img.mykernel.cn/image-20210930144058896.png)
>
> 第2个框

#### 给重点文本加下划线

新浪体育 我有话说(195人参与) 

```css
		.purple {
			color: purple;
			text-decoration: underline;
		}
```

> ![image-20210930144058896](http://myapp.img.mykernel.cn/image-20210930144058896.png)

## 开发者工具 F12

### 开发者工具 窗口位置调整

右上角3个点，可以调整窗口位置 dock side. 上、下、左、右，分开。

### 开发者工具定位元素

- firefox

  没有下面详细，但是打印的控制台是对象

- chrome

  ![image-20210930145105511](http://myapp.img.mykernel.cn/image-20210930145105511.png)

左边是HTML, 右边是CSS

### 开发者工具的放大和缩小

ctrl 滚轮放在和缩小

### css在线调整外观或文字，并快速定位到具体行

- 在线调整右边的css, 定位到标题后，不要直接修改css的字大小，而是选中px, 按上下箭头来调整

  ![image-20210930150933082](http://myapp.img.mykernel.cn/image-20210930150933082.png)

  > 当我们调整的合适了，就可以把px值放回自己的代码中

- 在线调整颜色, 光标定位到我们加了颜色的第2个span上，CSS处会有我们定义的颜色，左边有一个圆圈，可以点一下，在调色板中选我们满意的颜色。

  满意之后，就把右侧的十六进制数字复制下来

  ![image-20210930151148275](http://myapp.img.mykernel.cn/image-20210930151148275.png)



- css调整满意后，在哪里修改css呢？

  右边在21行，所以我们直接找到这个文件的21行

  - firfox，显示的不是具体文件行

  ![image-20210930151508451](http://myapp.img.mykernel.cn/image-20210930151508451.png)

  - chrome, 显示的是具体哪个文件什么行

    ![image-20210930151657899](http://myapp.img.mykernel.cn/image-20210930151657899.png)

## sublime快捷操作

### 前言

`emmet`语法，是zen coding语法 

- 补全骨架: !, html, 
- 补全标签 ：标签名 按tab键
- 生成多个相同标签: 标签名*个数
- 父子级标签: ul > li, ul > li*3
- 兄弟标签:     dl > dt+dd
- 带有类或id的标签:   div.demo或.demo,  div.nav或.nav;    div#footer或#footer
- 自增类名: .demo$*3

更多文档: https://www.w3cplus.com/tools/emmet-cheat-sheet.html



### 安装emmet

https://github.com/sergeche/emmet-sublime#how-to-install

### 骨架

```html
!
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
</head>
<body>
	
</body>
</html>
```

```html
html:5
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
</head>
<body>
	
</body>
</html>
```

### 标签

````html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <base href="">
</head>

<body>
	<!-- h1 -->
    <h1></h1>
    <!-- h2 -->
    <h2></h2>
    <!-- h3 -->
    <h3></h3>
    <!-- h4 -->
    <h4></h4>
    <!-- h5 -->
    <h5></h5>
    <!-- h6 -->
    <h6></h6>
    <!-- p -->
    <p></p>
    <!-- hr -->
    <hr />
    <!-- br -->
    <br />
    <!-- div -->
    <div></div>
    <!-- span -->
    <span></span>
    <!-- img -->
    <img src="" alt="" />
    <!-- a -->
    <a href=""></a>
    <!-- pre -->
    <pre></pre>
    <!-- &nbsp; -->
    <!-- table>caption+tr>th*3 -->
    <table>
    	<caption></caption>
    	<tr>
    		<th></th>
    		<th></th>
    		<th></th>
    	</tr>
    	<!-- tr>td*3 -->
    	<tr>
    		<td></td>
    		<td></td>
    		<td></td>
    	</tr>
    	<!-- tr>td*3 -->
    	<tr>
    		<td></td>
    		<td></td>
    		<td></td>
    	</tr>
    </table>

    <!-- ul>li*3 -->
    <ul>
    	<li></li>
    	<li></li>
    	<li></li>
    </ul>
    
    <!-- ol>li*3 -->
    <ol>
    	<li></li>
    	<li></li>
    	<li></li>
    </ol>

    <!-- dl>dt+dd*3 -->
    <dl>
    	<dt></dt>
    	<dd></dd>
    	<dd></dd>
    	<dd></dd>
    </dl>

    <!-- input -->
    <input type="text">
    <!-- label -->
    <label for=""></label>
    <!-- form -->
    <form action=""></form>
    
</body>

</html>
````

### 带有类的标签

类名为demo的`div`标签

```html
	<!-- .demo -->
	<div class="demo"></div>
	<!-- div.demo -->
	<div class="demo"></div>
```

id为nav的`div`标签

```html
    <!-- #nav -->
    <div id="nav"></div>

    <!-- div#nav -->
    <div id="nav"></div>
```

### 自增类名

```html
    <!-- .demo$*3 -->
    <div class="demo1"></div>
    <div class="demo2"></div>
    <div class="demo3"></div>
```

### 类属性补全

#### 宽度200

```css
/* w200*/
width: 200px
```

#### 高度

```css
/* h200 */
height: 200px
```



## 今日总结

1. html只有结构，需要css
2. css可以美化网页
3. css引入位置
   - 内联: `<h1 style="color: red; "`
   - 内嵌: `<head> <style> h1 { color: red; } </style></head>`
   - 外链: `<head><link /></head>`
4. css基础选择
   - 标签
   - 类：单类、多类，可以重复。类命名规范 `.`
   - id 惟一，用一次。id命名规范  `#`
   - 通配 `*` 
5. css样式
   - 字体: size, family, weight, style(倾斜)，综合font：` [style] [weight] <size>/[line-height] <family>`
   - 外观: color, line-height(文字大小+8), text-align(center), text-indent(2em), text-decoration(none, underline)
   - 链接去掉下划线，重点文本突出em
6. chrome调试，而firefox用来写css selector，方便查看document.queryselector的对象。
   - 定位元素，获取HTML/CSS
   - 在线调整字体大小(上下箭头)，颜色(调色板)，chrome才可以定位到源码的具体位置.
7. 快捷键
   - emmet语法, tab补全
   - `!`骨架，标签名补全
   - `table>tr`下一级，`li*3`, 3个li标签。
   - `.demo` 带有demo类的div标签，`#avn`带有avn的div
   - .demo$*3 自增类



# CSS第2天

## 学习目标

导航栏的案例

css复合选择器 -> 标签的显示模式(链接不能设置大小) -> 文本垂直居中(行高) -> 盒子添加背景 -> css三大特性

-  复合选择器
  - 后代选择器
  - 并集选择器
- 标签显示模式
- css背景
  - 背景位置
- css三大特性
  - 优先级

## css复合选择器

### 学习目标

- 理解
  - 复合选择器应用场景
- 应用
  - 后代选择器给元素添加样式
  - 并集选择
  - 伪类选择器

### 为什么要使用css复合选择器？

更精确的定位到标签

由两个或多个基础选择器组合而成

### 所有后代选择器

```js
// 常用写法
类别选择器 标签选择器
.class   h3

外层标签  内层标签
tr       li
```



下图，左边和上边都是a标签，如何定位具体左边还是上边的链接呢？

![image-20210930160257378](http://myapp.img.mykernel.cn/image-20210930160257378.png)

多个a标签肯定是盒子套出来的

模拟出来

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
</head>
<body>
	<!-- .nav>a*3{内部链接} -->
	<div class="nav">
		<a href="">内部链接</a>
		<a href="">内部链接</a>
		<a href="">内部链接</a>
	</div>
	<!-- a*3{外部链接} -->
	<a href="">外部链接</a>
	<a href="">外部链接</a>
	<a href="">外部链接</a>
</body>
</html>
```

> 浏览器显示
>
> ![image-20210930160553324](http://myapp.img.mykernel.cn/image-20210930160553324.png)

现在需要把内部链接改成`pink`色

```css
	<style>
		.nav a {
			color: pink;
		}
	</style>
```

a是nav的儿子



示例

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.nav a {
			color: pink;
		}
	</style>
</head>
<body>
	<!-- .nav>a*3{内部链接} -->
	<div class="nav">
		<a href="">内部链接</a>
		<a href="">内部链接</a>
		<a href="">内部链接</a>
	</div>
	<!-- a*3{外部链接} -->
	<a href="">外部链接</a>
	<a href="">外部链接</a>
	<a href="">外部链接</a>

	<!-- ul>li*3{一条狗} -->
	<ul>
		<li>一条狗</li>
		<li>一条狗</li>
		<li>一条狗</li>
	</ul>

	<!-- .wangsicong>ul>li*3{王可可是一条狗} -->
	<div class="wangsicong">
		<ul>
			<li>王可可是一条狗</li>
			<li>王可可是一条狗</li>
			<li>王可可是一条狗</li>
		</ul>
	</div>
	
</body>
</html>
```

> ![image-20210930161506018](http://myapp.img.mykernel.cn/image-20210930161506018.png)

让`王可可是一条狗`全部变成红色

```css
		.wangsicong ul li {
			color: red;
		}
```

或

```css
		.wangsicong  li {
			color: red;
		}
```

![image-20210930161627718](http://myapp.img.mykernel.cn/image-20210930161627718.png)

### 亲儿子元素选择器

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
	</style>
</head>
<body>
	<div>
		<strong>儿子</strong>
		<strong>儿子</strong>
		<strong>儿子</strong>
	</div>

	<div>
		<p>
			<strong>孙子</strong>
			<strong>孙子</strong>
			<strong>孙子</strong>
		</p>
	</div>
</body>
</html>
```

配置div内的strong的颜色

```css
	<style>
		div strong {
			color: red;
		}
	</style>
```

> ![image-20210930162116390](http://myapp.img.mykernel.cn/image-20210930162116390.png)
>
> 现在后代选择器，会选择所有后代

```html
	<style>
		div > strong {
			color: pink;
		}
	</style>
```

> ![image-20210930162215124](http://myapp.img.mykernel.cn/image-20210930162215124.png)
>
> 现在可以看出只有一级子元素被选择

### 交集选择器

既满足第1个特点，也满足第二个特点

```css
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
	<p class="red">红色</p>
	<p class="red">红色</p>
	<p class="red">红色</p>
	<div class="red">红色</div>
	<div class="red">红色</div>
	<div class="red">红色</div>
	<p>蓝色</p>
	<p>蓝色</p>
	<p>蓝色</p>
</body>
</html>
```

现在如果只匹配标签p，的结果

```css
<head>
	<style type="text/css">
		p {
			color: red;
		}
	</style>
</head>
```

> ![image-20211002113426503](http://myapp.img.mykernel.cn/image-20211002113426503.png)

蓝色的地方也变成红色，这个不是我们期望的。

想p标签 并且 类名为red的变成红色

```css
	<style type="text/css">
		p.red {
			color: red;
		}
	</style>
```

> ![image-20211002113524743](http://myapp.img.mykernel.cn/image-20211002113524743.png)

### 并集选择器

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
	<p>我和你</p>
	<p>我和你</p>
	<p>我和你</p>
	<span>我和你</span>
	<span>我和你</span>
	<span>我和你</span>
	<div>我和你</div>
	<div>我和你</div>
	<h1>我和你</h1>
	<h1>我和你</h1>
</body>
</html>
```

需求：p和span，红色。

> 可以给p和span添加一个红色的类名，非常麻烦，所以选择使用标签选择器

```css
	<style type="text/css">
		p {
			color: red;
		}
		span {
			color: red;
		}
	</style>
```

简写

```css
	<style type="text/css">
		p,span {
			color: red;
		}
	</style>
```

省了一行代码



需求：第1个div也变成红色

```html
	<div class="red">我和你</div>
```

```css
		.red {
			color: red;
		}
```

简写

```css
	<style type="text/css">
		p,span,.red {
			color: red;
		}
	</style>
```

### 测试题

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
	<!-- 主导航栏 -->
	<div class="nav">
		<ul>
			<li><a href="#">公司首页</a></li>
			<li><a href="#">公司简介</a></li>
			<li><a href="#">公司产品</a></li>
			<li><a href="#">联系我们</a></li>
		</ul>
	</div>

	<!-- 侧导航栏 -->
	<div class="sitenav">
		<div class="site-1">左侧导航栏</div>
		<div class="site-r"><a href="#">登录</a></div>
	</div>
</body>
</html>
```

> ![image-20211002114957794](http://myapp.img.mykernel.cn/image-20211002114957794.png)

不修改html结构代码，完成以下任务

> 结构：主导航栏，无序列表包含链接 。 侧导航：div包含2个div(其中1个div包含1个链接)

1. 链接 登录 的颜色为红色
2. 主导航栏里面的所有链接改为橙色
3. 主导航栏和侧导航里面的文字都是14像素并且微软雅黑

```css
	<style type="text/css">
		/*链接 登录 的颜色为红色*/
		.site-r>a {
			color: red;
			/*text-decoration: none;*/
		}

		/*主导航栏里面的所有链接改为橙色*/
		.nav a {
			color: orange;
		}
		
		/*主导航栏和侧导航里面的文字都是14像素并且微软雅黑*/
		.nav,.sitenav {
			font-size: 14px;
			font-family: "微软雅黑";
		}
	</style>
```

更新后的

```css
		.nav, .sitenav {
			font:  14px "微软雅黑";
		}
```

### 链接伪类选择器

#### 与类选择器的区别

伪 类选择器，与类选择器区别

```css
.demo  # 类选择，1个点
:link  # 伪类选择, 2个点；通过添加特殊效果
```

#### 伪类选择器不同行为的样式 

| 类别     |    名称     | 作用                       |
| -------- | :---------: | -------------------------- |
| 链接伪类 |  `a:link`   | 未访问的链接应用此样式     |
|          | `a:visited` | 已访问的链接应用此样式     |
|          |  `a:hover`  | 鼠标移动到链接上应用此样式 |
|          | `a:active`  | 待定的链接应用此样式       |
| 结构伪类 |             |                            |

<div class="admonition warning">
    <p class="admonition-title">
        注意
    </p>
    <p>
        定义一个链接这4个样式时，必须按照<strong>lvha</strong>首字母顺序定义。 
    </p>
    <p>
        记忆: <strong>LV</strong>包包 非常<strong>H</strong>ao
    </p>
</div>

示例

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
	<a href="http://www.xiaomi.com">小米手机</a>
</body>
</html>
```

未访问的链接，黑色，不要下划线

```css
	<style type="text/css">
		/*未访问的链接，黑色，不要下划线*/
		a:link {
			color: #333; 
			text-decoration: none;
		}
	</style>
```

访问过的链接

```css
		/*已经访问过的链接*/
		a:visited {
			color: orange;
		}
```

鼠标悬念时的样式

```css
		/*鼠标悬念时的样式*/
		a:hover {
			color: red;
		}
```

按下鼠标不松开时的颜色

```css
		a:active {
			color: green;
		}
```

现在我们更新css时

```css
	<style type="text/css">
		/*鼠标悬念时的样式*/
		a:hover {
			color: red;
		}
		/*未访问的链接，黑色，不要下划线*/
		a:link {
			color: #333; 
			text-decoration: none;
		}
		/*已经访问过的链接*/
		a:visited {
			color: orange;
		}

		a:active {
			color: green;
		}
	</style>
```

> 把悬停变红放在第1个时，浏览器中悬停将不会变红



#### 单独给链接定义样式

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
	<a href="">手机</a>
</body>
</html>
```

> ![image-20211002122718256](http://myapp.img.mykernel.cn/image-20211002122718256.png)

如果链接外套一个导航

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
	<div class="nav">
		<a href="">手机</a>
	</div>
</body>
</html>
```

现在给父亲给颜色, 

```css
	<style type="text/css">
		.nav {
			color: red;
		}
	</style>
```

> 浏览器中不会把链接文字![image-20211002122847193](http://myapp.img.mykernel.cn/image-20211002122847193.png)变成红色, 而是默认样式
>
> 

需要给链接单独指定样式

```css
	<style type="text/css">
		.nav a {
			color: red;
		}
	</style>
```

#### 实际开发如何给链接定义样式

首先给一个全局的所有 链接样式，统一风格

```css
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
	<style type="text/css">
		.nav a {
			font-weight: 700;
			font-size: 16px;
			color: gray;
			text-decoration: none;
		}

	</style>
</head>
<body>
	<div class="nav">
		<a href="">手机</a>
	</div>
</body>
</html>
```

> ![image-20211002123216633](http://myapp.img.mykernel.cn/image-20211002123216633.png)

然后给指定链接定义颜色, 悬停时的颜色

```css
		.nav a:hover {
			color: orange;
		}
```

现在在这个导航之外的链接将不会应用这些样式

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
	<style type="text/css">
		.nav a {
			font-weight: 700;
			font-size: 16px;
			color: gray;
			text-decoration: none;
		}
		.nav a:hover {
			color: orange;
		}
	</style>
</head>
<body>
	<div class="nav">
		<a href="">手机</a>
	</div>
	<a href="#">没有妈妈的孩子像棵䓍</a>
</body>
</html>
```

> ![image-20211002123650808](http://myapp.img.mykernel.cn/image-20211002123650808.png)

### 复合选择器总结

| 分类       | css选择器                                               | 描述                                      |
| ---------- | ------------------------------------------------------- | ----------------------------------------- |
| 所有后代   | `.nav a`                                                | a在.nav内的任何层级                       |
| 一级子类   | `.nav>a`                                                | a在.nav内的1级子级                        |
| 交集选择   | `p.nav`                                                 | 不仅是p标签，而且类名是nav                |
| 并集选择器 | `p,div,.red,...`                                        | p和div标签                                |
| 链接伪类   | `a:link a:visited  a:hover a:active` 这个顺序不可以修改 | lv包包非常好，通常使用通用样式和`a:hover` |

## 标签显示模式

### div/span显示不同的原因？

显示模式有关

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
	<div>div</div>
	<div>div</div>
	<div>div</div>
	<span>span</span>
	<span>span</span>
	<span>span</span>
</body>
</html>
```

> div 独占一行，span 一行可以多个，这个是 为什么呢？
>
> ![image-20211002124658323](http://myapp.img.mykernel.cn/image-20211002124658323.png)

### 目标

- 理解
  - 3种显示模式
  - 3种显示模式的特点及区别
  - 3种显示模式的相互转化
- 应用
  - 3种显示模式的相互转化的代码怎么写

### 显示模式是 什么？

div独有的，1行1个div.  也叫块标签。

span独有的，1行可以有很多个span. 也叫行标签。

标签可以分类 ：**块标签/块元素**和**行内标签/行内元素**。

- 块标签：   1行只能放1个。
- 行内标签：1行可以放多个。

实际开发中，需要看这个样式需要哪种标签，显示1行还是1行多个？



### 块级元素

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
	<div>我是块级元素</div>
	<div>我是块级元素</div>

</body>
</html>
```

> ![image-20211002125457548](http://myapp.img.mykernel.cn/image-20211002125457548.png)

- 1行1个div
- 高度和宽度、内外边距都可以直接控制。
- 宽度默认容器（父级宽度一样）的100%
- 是一个容器盒子，里面可以放**行级元素**或**块级元素**。

####  高度可控制

```htm\
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
	<style type="text/css">
		div {
			height: 200px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div>我是块级元素</div>
	<div>我是块级元素</div>

</body>
</html>
```

> ![image-20211002125731252](http://myapp.img.mykernel.cn/image-20211002125731252.png)
>
> 块元素占的高度

#### 宽度可控制，默认继承自父级元素

由以上的代码结果，可以知道，宽度默认和父级`body`一样宽。

给一个宽度

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
	<style type="text/css">
		div {
			width: 100px;
			height: 200px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div>我是块级元素</div>
	<div>我是块级元素</div>

</body>
</html>
```

> ![image-20211002125935442](http://myapp.img.mykernel.cn/image-20211002125935442.png)
>
> 现在的宽度只有1半宽

#### 块级元素容器内放其他行级元素或块级元素

```html
	<div>
		<!-- 行级 -->
		<strong>文字</strong>
		<!-- 块级 -->
		<h1>标题1</h1>
	</div>
```

> ![image-20211002130247114](http://myapp.img.mykernel.cn/image-20211002130247114.png)

#### 注意

- 文字才能组成段落，因此`p`内不能放块级元素，特别p内不能方向div.
- 同理, hx, dt是 文字类块级元素，里面不能放其他块级元素。

```html
	<p>
		<div>123</div>
	</p>
```

> 解析成2个P和div
>
> ![image-20211002130533634](http://myapp.img.mykernel.cn/image-20211002130533634.png)

### 行内元素

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
	<span>我是行内元素</span>
	<span>我是行内元素</span>
	<span>我是行内元素</span>
	<span>我是行内元素</span>
</body>
</html>
```

#### 高/宽设定, 背景相关

```html
	<style type="text/css">
		span {
			width: 200px;
			height: 200px;
			background-color: pink;
		}
	</style>
```

> ![image-20211002131634834](http://myapp.img.mykernel.cn/image-20211002131634834.png)

#### 行内元素只能容纳文本和其他行内元素

可以放行内元素或文本

```html
	<span><strong>123</strong></span>
```

不能放块级元素

```html
	<span><div>123456</div></span>
```



### 行内块元素

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
</head>
<body>
	<img src="./images/3.jpg">
	<img src="./images/3.jpg">
</body>
</html>
```

> ![image-20211002132723863](http://myapp.img.mykernel.cn/image-20211002132723863.png)
>
> 1. 2个图，在1行中显示。有行内的特点
> 2. 2个图间有空格
> 3. 行内元素不能设置宽高，但是这个可以设置

#### 宽高

默认图片大小 

```css
	<style type="text/css">
		img {
			width: 200px;
		}
	</style>
```

> ![image-20211002132916788](http://myapp.img.mykernel.cn/image-20211002132916788.png)

### 显示模式总结 

| 显示模式   | 排列    | 设置样式             | 默认宽度                 | 包含           | 特殊                                            | 常用标签                            |
| ---------- | ------- | -------------------- | ------------------------ | -------------- | ----------------------------------------------- | ----------------------------------- |
| 块内元素   | 1行1个  | 可设置高、宽         | 父级元素的宽度，默认100% | 任何元素       | 文字块级元素(p,hx,dt)中不能块级元素。           | hx, p, div, ul, ol, li              |
| 行内元素   | 1行多个 | 不可以直接设置高、宽 | 本身内容宽度             | 文本或行内元素 | a中不能a标签，a可以放块元素，转换成块元素最安全 | a, strong, b, em,i,del,s,ins,u,span |
| 行内块元素 | 1行1个  | 可设置高、宽         | 本身内容宽度             |                |                                                 | img, input, td                      |

### 显示模式转换

a标签，转换成可以设置宽度和高度

块元素转行内: `display: inline;`

行内元素转块级: `display: block;`

块、行内元素转换为行内块：`display: inline-block`, 链接转换成行内块方便用户点击 ，大小变大了。整个范围可以点击

> 手册搜索: display

#### 行转块

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
	<style type="text/css">
		span {
			width: 100px;
			height: 100px;
			background-color: red;
		}
	</style>
</head>
<body>
	<span>行内</span>
</body>
</html>
```

> ![image-20211002134231253](http://myapp.img.mykernel.cn/image-20211002134231253.png)

没有大小

转换

```css
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Document</title>
	<style type="text/css">
		span {
			display: block;
			/*转换后生效*/
			width: 100px;
			height: 100px;
			background-color: red;
		}
	</style>
</head>
<body>
	<span>行内</span>
	<span>行内</span>
</body>
</html>
```

> ![image-20211002134349128](http://myapp.img.mykernel.cn/image-20211002134349128.png)

现在span就1行一个了



a标签，只让其可以设定宽和高

```html
	<a href="#">百度</a>
	<a href="#">sina</a>
```

```css
		a {
			display: inline-block;
			width: 200px;
			height: 200px;
			background-color: orange;
		}
```

> ![image-20211002134826417](http://myapp.img.mykernel.cn/image-20211002134826417.png)

#### 块转行

```html
		div {
			display: inline;
			/*转换后宽高不生效*/
			width: 100px; 
			height: 100px;
			background-color: purple;
		}
```

## css背景

### 目标

- 理解
  - 背景作用
  - css背景图片和插入图片的区别
- 应用
  - css背景属性，给页面元素添加背景样式
  - 能设置不同的背景图片位置

### 背景颜色(color)

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.bg {
			width: 600px;
			height: 500px;
		}
	</style>
</head>
<body>
	<!-- div.bg -->
	<div class="bg">
		
	</div>
</body>
</html>
```

页面显示空白，但是加一个颜色，就可以看到这个盒子的整体范围

```css
background-color: pink;
```

![image-20211008135932573](http://myapp.img.mykernel.cn/image-20211008135932573.png)

https://www.w3school.com.cn/cssref/pr_background-color.asp

### 背景图片(image)

| 值           | 描述                             |
| :----------- | :------------------------------- |
| url('*URL*') | 指向图像的路径。**不提倡加引号** |
| none         | 默认值。不显示背景图像。         |

```css
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.bg {
			width: 600px;
			height: 500px;
			background-color: pink;
			background-image: url(./images/l.png);	
		}
	</style>
</head>
<body>
	<!-- div.bg -->
	<div class="bg">
		123
	</div>
</body>
</html>
```

![image-20211008141225517](http://myapp.img.mykernel.cn/image-20211008141225517.png)

插入图片在盒子里只放图片，背景图片，是在盒子底下放图片。

### 背景平铺（repeat)

默认背景是平铺的，虽然只有1个图片，尺寸小的时候，会铺满整个盒子。

| 值        | 描述                                                |
| :-------- | :-------------------------------------------------- |
| repeat    | **默认**。背景图像将在垂直方向和水平方向重复。      |
| repeat-x  | 背景图像将在水平方向重复。                          |
| repeat-y  | 背景图像将在垂直方向重复。                          |
| no-repeat | 背景图像将仅显示一次。                              |
| inherit   | 规定应该从父元素继承 background-repeat 属性的设置。 |

> https://www.w3school.com.cn/cssref/pr_background-repeat.asp

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.bg {
			width: 600px;
			height: 500px;
			background-color: pink;
			background-image: url(./images/l.png);	
+			background-repeat: no-repeat;
		}
	</style>
</head>
<body>
	<!-- div.bg -->
	<div class="bg">
		123
	</div>
</body>
</html>
```

![image-20211008141415641](http://myapp.img.mykernel.cn/image-20211008141415641.png)

因为不平铺了，就显示出来背景色了



水平平铺

```css
background-repeat: repeat-x;
```

纵向平铺

```css
background-repeat: repeat-y;
```

### 背景位置(position)

现在不平铺时，`no-repeat`时，如何将图片挪动位置呢？

| 值                                       | 描述                                                         |
| :--------------------------------------- | :----------------------------------------------------------- |
| top\|center\|bottom\|left\|center\|right | 如果您仅规定了一个关键词，那么第二个值将是"center"。默认值：0% 0%。 |
| x% y%                                    | 第一个值是水平位置，第二个值是垂直位置。左上角是 0% 0%。右下角是 100% 100%。如果您仅规定了一个值，另一个值将是 50%。 |
| xpos ypos                                | 第一个值是水平位置，第二个值是垂直位置。左上角是 0 0。单位是像素 (0px 0px) 或任何其他的 CSS 单位。如果您仅规定了一个值，另一个值将是50%。您可以混合使用 % 和 position 值。 |

> https://www.w3school.com.cn/cssref/pr_background-position.asp

注意

1. 必须有background-image属性
2. position后面可以接x/y坐标或方位名词。

3. 使用方位名词时，顺序无所谓 left center或center left 效果一样的。
4. 只指定一个方位名词时，另一个词时居中对齐。
5. 当方位名词和坐标和像素时，第1个一定是X轴，另一个一定是Y轴。

```css
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.bg {
			width: 600px;
			height: 500px;
			background-color: pink;
			background-image: url(./images/l.png);	
			background-repeat: no-repeat;
		}
	</style>
</head>
<body>
	<!-- div.bg -->
	<div class="bg">
		123
	</div>
</body>
</html>
```

右上角

```css
			/*background-position: 100% 0%;*/
			background-position: right top;
```

左下角

```css
			/*background-position: 0% 100%;*/
			background-position: left bottom;
```

居中

```css
			/*background-position: 50% 50%;*/
			background-position: center center;
```

左边居中

```css
			background-position: 0% 50%;
			/*background-position: left center;*/
```

使用方位名词时，顺序无所谓

```css
			background-position: center left;
```

只指定一个方位名词时，另一个词时是居中对齐（center)

```css
			background-position: top;
```

![image-20211008142528957](http://myapp.img.mykernel.cn/image-20211008142528957.png)

混合使用: x在中间，y在顶部

```css
			background-position: center 0%;
```

![image-20211008142528957](http://myapp.img.mykernel.cn/image-20211008142528957.png)

### 背景附着 attachment

背景是滚动的，还是固定的

| 值      | 描述                                                    |
| :------ | :------------------------------------------------------ |
| scroll  | 默认值。背景图像会随着页面其余部分的滚动而移动。        |
| fixed   | 当页面的其余部分滚动时，背景图像不会移动。              |
| inherit | 规定应该从父元素继承 background-attachment 属性的设置。 |

> https://www.w3school.com.cn/cssref/pr_background-attachment.asp

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		body {
			background-image: url(./images/1.jpg);
			background-position: top ;
			height: 3000px;
			background-repeat: no-repeat;
		}
		p {
			font-size: 30px;
			color: #fff;
		}
	</style>
</head>
<body>
	<p>天王盖地虎，小鸡炖蘑菇</p>
	<p>天王盖地虎，小鸡炖蘑菇</p>
	<p>天王盖地虎，小鸡炖蘑菇</p>
	<p>天王盖地虎，小鸡炖蘑菇</p>
	<p>天王盖地虎，小鸡炖蘑菇</p>
	<p>天王盖地虎，小鸡炖蘑菇</p>
	<p>天王盖地虎，小鸡炖蘑菇</p>
	<p>天王盖地虎，小鸡炖蘑菇</p>
	<p>天王盖地虎，小鸡炖蘑菇</p>
	<p>天王盖地虎，小鸡炖蘑菇</p>
</body>
</html>
```

> 由于高度是1090，一般会出现滚动条，1页显示不完。

现在托动滚动条时，背景不会随着滚动而滚动。默认滚动的

```css
			background-attachment: fixed;
```

现在无论滚动滚动条，背景不会动。

### 背景简写 background

官方并不强制顺序，一般可以这样写。

```css
			/*背景颜色 背景图片地址 背景平铺 背景滚动 背景位置*/
			background: transparent url(./images/1.jpg) no-repeat scroll center top;
```



### 背景透明

鼠标放上去后，图片的颜色会加深

```css
background-color: rgba(0,0,0,0.3);
```

- rgba, 最后一个参数，0-1之间，控制透明度
- 习惯把最后一个小数写成, 0.3 为 .3
- 背景半透明是指盒子背景半透明，盒子内容不受影响
- CSS3, IE9版本之下不支持。



```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.alpha {
			width: 300px;
			height: 300px;
			background-color: black;
		}
	</style>
</head>
<body>
	<div class="alpha"></div>
</body>
</html>
```

> ![image-20211008161857655](http://myapp.img.mykernel.cn/image-20211008161857655.png)

想要半透明, 30%的透明度，范围在0-1之间

```diff
		.alpha {
			width: 300px;
			height: 300px;
+			background-color: rgba(0,0,0,.3);
		}
```

```html
	<div class="alpha">
		哒哒
	</div>
```

> ![image-20211008162424297](http://myapp.img.mykernel.cn/image-20211008162424297.png)



### 背景总结

| 属性                  | 作用         | 值                                                           |
| --------------------- | ------------ | ------------------------------------------------------------ |
| background-color      | 背景颜色     | 预定义颜色值/十六进制/RGB代码                                |
| background-image      | 背景图片     | url(图片路径), none                                          |
| background-repeat     | 是否平铺     | repeat/no-repeat/repeat-x/repeat-y                           |
| background-position   | 背景位置     | 位置名(left/bottom/center/right/top), x/y坐标, 准确数值(px). 坐标和数值或结合位置名混写时，必须先x后y轴，当省略时，一般另一个是居中 |
| background-attachment | 背景附着     | scroll(滚动):默认，背景会滚动； fixed(固定)：背景会始终在当前屏幕上； |
| 背景简写              | 更简单       | 背景颜色 背景图片地址 背景平铺 背景滚动 背景位置 ；没有顺序，一般常按前面的顺序写 |
| 背景透明              | 让盒子半透明 | background-color: rgba(0,0,0,.3); 后面必须是4个值。          |



## 案例

### 简单导航栏

#### 简单实现

![image-20211008110710685](http://myapp.img.mykernel.cn/image-20211008110710685.png)

1. 这行中有多个行内的元素，使用行内元素: a
2. 鼠标放上去会变黄色, 前背和背景：是链接伪类的hover
3. 链接没有下划线
4. 有宽度，由于行内元素不可以加宽度 ，会转换成行内的块。
5. 文本居中，继承父级

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		a:link {
			display: inline-block;
			color: white;
			text-decoration: none;
			text-align: center;
			background-color: pink;
			width: 100px;
			height: 30px;
		}
		a:hover {
			color: yellow;
			background-color: orange;
		}
	</style>
</head>
<body>
	<a href="#">体育</a>
	<a href="#">娱乐</a>
	<a href="#">汽车</a>
</body>
</html>
```

​          

#### 文字垂直居中

```css
line-height: 30px;
```

![image-20211008135213832](http://myapp.img.mykernel.cn/image-20211008135213832.png)

### 魔兽世界

https://wow.blizzard.cn/landing

下载里面的图片，图片越大越好，分辨率在每个电脑不一样，如果给一个大图，每个电脑均能显示完

首先放页面的大图

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		body {
			background-image: url(./images/1.jpg);
		}
	</style>
</head>
<body>
	
</body>
</html>
```

![image-20211008143858502](http://myapp.img.mykernel.cn/image-20211008143858502.png)

现在图片是一样的了, 但是可以注意到右下角没有显示完整。 说明分辨率不一样，要水平居中对齐

```css
			background-position: top ;
```

右边有滚动条，我们没有，给一个高度

```html
			height: 3000px;
```

现在有重复

```css
			background-repeat: no-repeat;
```

### 新闻的小点，使用图片

不使用列表的点，而是使用图片，让页面显示一样的。

![image-20211008145630315](http://myapp.img.mykernel.cn/image-20211008145630315.png)



```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.icon {
			/*w150px*/
			width: 150px;
			/*h35px*/
			height: 35px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div class="icon"></div>
</body>
</html>
```

> 盒子
>
> ![image-20211008145955519](http://myapp.img.mykernel.cn/image-20211008145955519.png)

把小图片，放在盒子的左侧中间

```css
			background-image: url(https://i1.sinaimg.cn/dy/main/icon/photoNewsLeft2.gif);
			background-position: left;
			background-repeat: no-repeat;
```

> ![image-20211008150127916](http://myapp.img.mykernel.cn/image-20211008150127916.png)

现在不想顶格，留10px的距离 

```css
			background-position: 10px center;
```

> ![image-20211008150254621](http://myapp.img.mykernel.cn/image-20211008150254621.png)

### 链接导航栏

#### 结构

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.nav {
			text-align: center;
		}
	</style>
</head>
<body>
	<div class="nav">
		<a href="#">网站首页</a>
		<a href="#">网站首页</a>
		<a href="#">网站首页</a>
		<a href="#">网站首页</a>
		<a href="#">网站首页</a>
		<a href="#">网站首页</a>
	</div>

	<a href="#">123</a>
</body>
</html>
```

![image-20211008151637586](http://myapp.img.mykernel.cn/image-20211008151637586.png)

#### 样式

每个a有大小，并且和背景图片一模一样大

图片是120x50的分辨率

![image-20211008161349073](http://myapp.img.mykernel.cn/image-20211008161349073.png)

```css
		.nav a {
            /* 行内，不能直接给宽度和高度，需要转换为行内块 */
			display: inline-block;
			width: 120px;
			height: 50px;
            /*背景色：仅测试使用*/
			background-color: pink;
			/* 下划线去掉 */
            text-decoration: none;
			/* 行高==盒子高，文字会垂直居中 */
            line-height: 50px;
			/* 前景色 白色 */
            color: #fff;
		}
```

> ![image-20211008160445421](http://myapp.img.mykernel.cn/image-20211008160445421.png)

添加背景图片![bg](http://myapp.img.mykernel.cn/bg.png)

```diff
		.nav a {
			display: inline-block;
			width: 120px;
			height: 50px;
+			background: transparent url(./images/bg.png) no-repeat;
			text-decoration: none;
			line-height: 50px;
			color: #fff;
		}
```

鼠标经过链接，背景图片更新一下![bgc](http://myapp.img.mykernel.cn/bgc.png)

```css
		.nav a:hover {
			background-image: url(./images/bgc.png);
		}
```

## css三大特性

### 目标

- 理解
  - css样式 冲突采取的原则
  - 哪些常见的样式会有继承
- 应用
  - 写出css优先级的算法
  - 计算常见的选择器的叠加值

### 层叠性

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
</head>
<body>
	<div>
		长江后浪推前浪，前浪死在沙滩上。
	</div>
</body>
</html>
```

样式

```css
	<style>
		div {
			color: red;
		}
	</style>
```

> ![image-20211008163700166](http://myapp.img.mykernel.cn/image-20211008163700166.png)

重复写

```html
	<style>
		div {
			color: red;
		}
		div {
			color: pink;
		}
	</style>
```

> ![image-20211008163820376](http://myapp.img.mykernel.cn/image-20211008163820376.png)

后写的样式，会覆盖前面的样式定义

现在看，在第1个div中添加字体

```diff
	<style>
		div {
			color: red;
+			font-size: 30px;
		}
		div {
			color: pink;
		}
	</style>
```

> ![image-20211008164032875](http://myapp.img.mykernel.cn/image-20211008164032875.png)
>
> 后者的div中的color会覆盖前面的定义，而前面的font-size没有覆盖，依然生效



### 继承性

**子元素可以继承父亲的样式**: text-, font-, line-, color

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			color: red;
		}
	</style>
</head>
<body>
	<div>
		<p>王思聪</p>
	</div>
</body>
</html>
```

虽然定义颜色在div, 没有给p定义颜色，同样会显示红色。

css有继承性，子标签会继承父标签的样式。

### 优先级

1）谁选择出当前标签。未选择的，继承就权重为0

2）选中标签后，谁大听谁的。

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			color: red;
		}
		div {
			color: pink;
		}
	</style>
</head>
<body>
	<div>权重</div>
</body>
</html>
```

> 选择器相同，层叠性，后面定义会覆盖前面的定义

```html

	<style>
		.one {
			color: red;
		}
		div {
			color: pink;
		}
	</style>

```

>  现在的类名选择时，就类选择优先



```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		#two {
			color: blue;
		}
		.one {
			color: red;
		}
		div {
			color: pink;
		}
	</style>
</head>
<body>
	<div class="one" id="two">权重</div>
</body>
</html>
```

>  id选择器优先

```css
	<div class="one" id="two" style="color: yellow;">权重</div>
```

> 行内优先最大

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		#two {
			color: blue;
		}
		.one {
			color: red;
		}
		div {
+			color: pink!important;
		}
	</style>
</head>
<body>
	<div class="one" id="two" style="color: yellow;">权重</div>
</body>
</html>
```

> 样式属性后添加，最高优先级

#### 权重计算 

| 标签选择器                                 | 计算权重公式 | 描述                       |
| ------------------------------------------ | ------------ | -------------------------- |
| 继承(从父亲继承过来的样式)或通配(*)        | 0,0,0,0      | 没有权重                   |
| 每个标签选择器                             | 0,0,0,1      |                            |
| 每个类、伪类                               | 0,0,1,0      | 权重大于标签选择器         |
| 每个id                                     | 0,1,0,0      | id大于类                   |
| 每个标签内样式                             | 1,0,0,0      | 标签内联样式优先级比以上大 |
| 每个!important，样式属性后添加，最高优先级 | 无穷大       | 最高优先级                 |



#### 权重叠加 

div ul li --> 0,0,0,3

.nav ul li --> 0,0,1,2

a:hover --> 0,0,1,0

.nav    a  --> 0,0,1,1 

注意

1. 选择器是否能匹配到标签
2. 0,0,0,3 + 0,0,0,7 = 0,0,0,10 , **不会进位**为 0,0,1,0。 工作中不存在很多div。
3. 继承的权重是0，只要选择器没有直接选择到指定标签 
4. 权重相同，会有层叠性，就近原则。



#####  权重优先级：

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
</head>
<body>
	<div>人生4大悲</div>
	<div class="nav">
		<a href="#">1</a>
		<a href="#">2</a>
		<a href="#">3</a>
		<a href="#">4</a>
	</div>
</body>
</html>
```

将.nav>a变成红色

```css
	<style>
		.nav > a {
			color: red;
		}
	</style>
```

现在将第1个a变成pink色

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.nav > a {
			color: red;
		}

+		.first {
+			color: pink;
+		}
	</style>
</head>
<body>
	<div>人生4大悲</div>
	<div class="nav">
+		<a href="#" class="first">1</a>
		<a href="#">2</a>
		<a href="#">3</a>
		<a href="#">4</a>
	</div>
</body>
</html>
```

> 发现修改不了，原因是上面的优先级更高 1，1. 下面是1，0.
>
> 检查时，发现，权重不够被层叠掉了
>
> ![image-20211008171406263](http://myapp.img.mykernel.cn/image-20211008171406263.png)

加权重

```css
		.nav > .first {
			color: pink;
		}
```

> ![image-20211008171508231](http://myapp.img.mykernel.cn/image-20211008171508231.png)

##### 不会进位

##### 继承权重为0

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			color: red;
		}
	</style>
</head>
<body>
	<div>
		<p>继承的权重为0</p>
	</div>
</body>
</html>
```

> ![image-20211008171908630](http://myapp.img.mykernel.cn/image-20211008171908630.png)
>
> p标签继承自父亲的样式为color

```css
	<style>
		div {
			color: red;
		}
		p {
			color: green;
		}
	</style>
```

> ![image-20211008172007555](http://myapp.img.mykernel.cn/image-20211008172007555.png)

```css
		.demo {
			color: blue;
		}
		#test {
			color: pink;
		}
```

```html
	<div class="demo" id="test">
		<p>继承的权重为0</p>
	</div>
```

> 结果仍然是绿色，因为继承的权重为0
>
> ![image-20211008172007555](http://myapp.img.mykernel.cn/image-20211008172007555.png)

#### 权重6题

##### 第1题

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "
http://www.w3.org/TR/html4/loose.dtd">
<html>
	<head>
		<meta http-equiv="content-type" content="text/html;charset=utf-8" />
		<meta name="keywords" content="关键词1,关键词2,关键词3" />
		<meta name="description" content="对网站的描述" />
		<title>第1题</title>
		<style type="text/css">
			div div{ 
				color:blue;
			}
			div{
				color:red;
			}
		</style>
	</head>
	<body>
		<div>
			<div>
				<div>
					试问这行字体是什么颜色的？
				</div>
			</div>
		</div>
	</body>
</html>

```

> **均可以匹配到div,** 上面权重2，下面1.则蓝色
>
> ![image-20211008172818301](http://myapp.img.mykernel.cn/image-20211008172818301.png)

##### 第2题

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "
http://www.w3.org/TR/html4/loose.dtd">
<html>
	<head>
		<meta http-equiv="content-type" content="text/html;charset=utf-8" />
		<meta name="keywords" content="关键词1,关键词2,关键词3" />
		<meta name="description" content="对网站的描述" />
		<title>第2题</title>
		<style type="text/css">
			#father{
				color:red;
			}
			p {
				color:blue;  
			}
		</style>
	</head>
	<body>
		<div id="father">
			<p>试问这行字体是什么颜色的？</p>
		</div>
	</body>
</html>

```

> 上面匹配父，**下面匹配真正的标签**，所以继承的权重为0，当前定义的权重为1是最大的。
>
> ![image-20211008172818301](http://myapp.img.mykernel.cn/image-20211008172818301.png)

##### 第3题

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
<head>
	<meta http-equiv="Content-Type" content="text/html;charset=UTF-8">
	<title>Document</title>
	<style type="text/css">
		div div div div div div div div div div div div{    
			color:red;
		}
		.me {  
			color:blue;
		}
	</style>
</head>
<body>
	<div>
		<div>
			<div>
				<div>
					<div>
						<div>
							<div>
								<div>
									<div>
										<div>
											<div>
												<div class="me">试问这行文字是什么颜色的</div>
											</div>
										</div>
									</div>
								</div>
							</div>
						</div>
					</div>
				</div>
			</div>
		</div>
	</div>
</body>
</html>

```

> **均可以匹配最后一个**，
>
> 第1个是0,0,0,15
>
> 第2个是0,0,1,0 
>
> 始终为类大，因为不会进位。
>
> ![image-20211008172818301](http://myapp.img.mykernel.cn/image-20211008172818301.png)

##### 第4题

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "
http://www.w3.org/TR/html4/loose.dtd">
<html>
	<head>
		<meta http-equiv="content-type" content="text/html;charset=utf-8" />
		<meta name="keywords" content="关键词1,关键词2,关键词3" />
		<meta name="description" content="对网站的描述" />
		<title>第4题</title>
		<style type="text/css">
			div p {   
				color:red;
			}
			#father {
				color:red;
			}
			p.c2{    
				color:blue;
			}
		</style>
	</head>
	<body>
		<div id="father" class="c1">
			<p class="c2">
				试问这行字体是什么颜色的？
			</p>
		</div>
	</body>
</html>

```

> 求p标签的颜色，能匹配p的只有1和最后一个。
>
> 第1个权限为0，0，0，2
>
> 最后一个权重为0,0,1,1
>
> 蓝色
>
> ![image-20211008172818301](http://myapp.img.mykernel.cn/image-20211008172818301.png)

##### 第5题

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
<head>
	<meta http-equiv="Content-Type" content="text/html;charset=UTF-8">
	<title>Document</title>
	<style type="text/css">
		.c1 .c2 div{  
			color: blue;
		}
		div #box3 {  
			color:green;
		}
		#box1 div { 
			color:yellow;
		}
	</style>
</head>
<body>
	<div id="box1" class="c1">
		<div id="box2" class="c2">
			<div id="box3" class="c3">
				文字
			</div>
		</div>
	</div>
</body>
</html>

```

> 1）均选择出div
>
> 2）谁大听谁的
>
> 黄色，id最大，并且2和3一样，层叠性，3会覆盖2.
>
> ![image-20211008173425644](http://myapp.img.mykernel.cn/image-20211008173425644.png)

##### 第6题

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "
http://www.w3.org/TR/html4/loose.dtd">
<html>
	<head>
		<meta http-equiv="content-type" content="text/html;charset=utf-8" />
		<meta name="keywords" content="关键词1,关键词2,关键词3" />
		<meta name="description" content="对网站的描述" />
		<title>第6题</title>
		<style type="text/css">
			#father #son{ 
				color:blue;
			}
			#father p.c2{ 
				color:black;
			}
			div.c1 p.c2{  
				color:red;
			}
			#father{
				color:green !important;
			}
		</style>
	</head>
	<body>
		<div id="father" class="c1">
			<p id="son" class="c2">
				试问这行字体是什么颜色的？
			</p>
		</div>
	</body>
	
</html>

```

> 蓝色
>
> 1. 继承不考虑
> 2. 其他能匹配p，
> 3. 看id选择器，只有1和2. 由于1中id选择器2个，肯定比2大，故蓝色

## 今日总结

| 分类       | css选择器                                               | 描述                                      |
| ---------- | ------------------------------------------------------- | ----------------------------------------- |
| 所有后代   | `.nav a`                                                | a在.nav内的任何层级                       |
| 一级子类   | `.nav>a`                                                | a在.nav内的1级子级                        |
| 交集选择   | `p.nav`                                                 | 不仅是p标签，而且类名是nav                |
| 并集选择器 | `p,div,.red,...`                                        | p和div标签                                |
| 链接伪类   | `a:link a:visited  a:hover a:active` 这个顺序不可以修改 | lv包包非常好，通常使用通用样式和`a:hover` |

| 显示模式   | 排列    | 设置样式             | 默认宽度                 | 包含           | 特殊                                            | 常用标签                            |
| ---------- | ------- | -------------------- | ------------------------ | -------------- | ----------------------------------------------- | ----------------------------------- |
| 块内元素   | 1行1个  | 可设置高、宽         | 父级元素的宽度，默认100% | 任何元素       | 文字块级元素(p,hx,dt)中不能块级元素。           | hx, p, div, ul, ol, li              |
| 行内元素   | 1行多个 | 不可以直接设置高、宽 | 本身内容宽度             | 文本或行内元素 | a中不能a标签，a可以放块元素，转换成块元素最安全 | a, strong, b, em,i,del,s,ins,u,span |
| 行内块元素 | 1行1个  | 可设置高、宽         | 本身内容宽度             |                |                                                 | img, input, td                      |

转换: display， inline, block, inline-block

| 属性                  | 作用         | 值                                                           |
| --------------------- | ------------ | ------------------------------------------------------------ |
| background-color      | 背景颜色     | 预定义颜色值/十六进制/RGB代码                                |
| background-image      | 背景图片     | url(图片路径), none                                          |
| background-repeat     | 是否平铺     | repeat/no-repeat/repeat-x/repeat-y                           |
| background-position   | 背景位置     | 位置名(left/bottom/center/right/top), x/y坐标, 准确数值(px). 坐标和数值或结合位置名混写时，必须先x后y轴，当省略时，一般另一个是居中 |
| background-attachment | 背景附着     | scroll(滚动):默认，背景会滚动； fixed(固定)：背景会始终在当前屏幕上； |
| 背景简写              | 更简单       | 背景颜色 背景图片地址 背景平铺 背景滚动 背景位置 ；没有顺序，一般常按前面的顺序写 |
| 背景透明              | 让盒子半透明 | background-color: rgba(0,0,0,.3); 后面必须是4个值。          |

| 特性   | 描述                                                         |
| ------ | ------------------------------------------------------------ |
| 层叠性 | 相同标签选择器，靠后的生效，chrome中不生效的会删除。         |
| 继承性 | 子标签一般会继承父样式: text-, font-, line-, color           |
| 优先级 | 标签(个位1) < 类、伪类(十位1)< id(百位1) < 标签内联(千位1) < !important(无穷)。<br/> 判断样式优先级: 1) 选择器是否定位到标签？<br /> 1. 未定位到，可能是父类，继承的样式，权重为0. 当没有直接定位到此标签的样式时，才生效。<br /> 2. 定位到了，就看权重大小，只要优先级高的位存在，优先级低的位无论怎么加，不会进位，而且始终小。0,0,0,15 始终小于 0,0,1,0. <br/> 3. 如果同样 是 0,0,2,0 == 0,0,2,0  就会有层叠性 |



# CSS第3天

## 学习

CSS学习重点：盒子模型、浮动、定位 

边框：border

内边距：padding

外边距：margin

photoshop基本操作

## 盒子模型

### 学习目标

- 理解
  - 盒子模型4部分组成
  
    > F12中青色是 padding
    >
    > 蓝色是内容
  
  - 内边距作用，对盒子的影响
  
    > 文字到边框距离
    >
    > 盒子变大
  
  - padding不同值代表的意思
  
    > padding 会同时控制上下左右4个边距
  
  - 块级的盒子居中对齐
  
  - 外边距合并的解决方法
- 应用
  - 利用边框复合写法给元素添加边框
  - 计算盒子实际大小
  - 利用盒子模型完成案例

### 网页布局的本质

排放盒子位置，丢图片、文字进去

![image-20211009164908213](http://myapp.img.mykernel.cn/image-20211009164908213.png)

> 这些都是排放盒子，盒子**非常重要**, 那么盒子有什么样的特点呢？

### 盒子模型(box model)

盒子是关键点，所以需要研究一下盒子。

装手机的盒子的厚度，就是盒子的高度，即**边框**(border)

手机就是内容区域, content

为了防止手机在运输过程中会动，四周会填充泡沫，**内边距**

![image-20211009170740493](http://myapp.img.mykernel.cn/image-20211009170740493.png)

标准盒子模型

![](https://imgs.developpaper.com/imgs/2786317785-5f6d90d69a04e_articlex.png)

>  https://developpaper.com/css-standard-box-model-and-ie-box-model-weird-box-model/

content 区域的size就是：盒子的大小及高度

内边距：就是padding的宽和高

盒子厚度：就是border的宽和高

外边距：就是margin的宽和高

### 盒子边框(border)

| 属性                                                         | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [border-style](https://www.runoob.com/cssref/pr-border-style.html) | 用于设置元素所有边框的样式，或者单独地为各边设置边框样式。**dotted\|dashed\|solid** |
| [border-width](https://www.runoob.com/cssref/pr-border-width.html) | 简写属性，用于为元素的所有边框设置宽度，或者单独地为各边边框设置宽度。 **直接给像素** |
| [border-color](https://www.runoob.com/cssref/pr-border-color.html) | 简写属性，设置元素的所有边框中可见部分的颜色，或为 4 个边分别设置颜色。 |

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 200px;
			height: 200px;
		}
	</style>
</head>
<body>
	<div></div>
</body>
</html>
```

> 浏览器中不显示，因为没有背景色

#### 添加边框 `width`

```css
border-width: 1px;
```

> 刷新后，还是没有效果

####  添加边框的样式 `style`

##### 实线 `solid`

```css
			border-style: solid;
```

> 实线
>
> ![image-20211009172419275](http://myapp.img.mykernel.cn/image-20211009172419275.png)

##### 虚线 `dashed`

```css
			border-width: 10px;
			border-style: dashed;
```

> ![image-20211009172704187](http://myapp.img.mykernel.cn/image-20211009172704187.png)

##### 点 `dotted`

```css
			border-width: 10px;
			border-style: dotted;
```

> ![image-20211009172901478](http://myapp.img.mykernel.cn/image-20211009172901478.png)

#### 边框的颜色 `color`

```css
			border-width: 10px;
			border-style: dotted;
			border-color: pink;
```

> ![image-20211009173010149](http://myapp.img.mykernel.cn/image-20211009173010149.png)

####  border边框简写 `border`

没有顺序，一边宽、样式、颜色

```css
			border: 10px dotted pink;
```

#### 边框总结

某些盒子只要部分边框

| 属性                                                         | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [border](https://www.runoob.com/cssref/pr-border.html)       | 简写属性，用于把针对四个边的属性设置在一个声明。                      style/width/color **不分顺序** |
| [border-style](https://www.runoob.com/cssref/pr-border-style.html) | 用于设置元素所有边框的样式，或者单独地为各边设置边框样式。           **solid\|dotted\|dashed** |
| [border-width](https://www.runoob.com/cssref/pr-border-width.html) | 简写属性，用于为元素的所有边框设置宽度，或者单独地为各边边框设置宽度。 像素 |
| [border-color](https://www.runoob.com/cssref/pr-border-color.html) | 简写属性，设置元素的所有边框中可见部分的颜色，或为 4 个边分别设置颜色。     颜色 |
|                                                              |                                                              |
| [border-bottom](https://www.runoob.com/cssref/pr-border-bottom.html) | 简写属性，用于把下边框的所有属性设置到一个声明中。style/width/color **不分顺序** |
| [border-bottom-color](https://www.runoob.com/cssref/pr-border-bottom-color.html) | 设置元素的下边框的颜色。                                     |
| [border-bottom-style](https://www.runoob.com/cssref/pr-border-bottom-style.html) | 设置元素的下边框的样式。                                     |
| [border-bottom-width](https://www.runoob.com/cssref/pr-border-bottom-width.html) | 设置元素的下边框的宽度。                                     |
|                                                              |                                                              |
| [border-left](https://www.runoob.com/cssref/pr-border-left.html) | 简写属性，用于把左边框的所有属性设置到一个声明中。style/width/color **不分顺序** |
| [border-left-color](https://www.runoob.com/cssref/pr-border-left-color.html) | 设置元素的左边框的颜色。                                     |
| [border-left-style](https://www.runoob.com/cssref/pr-border-left-style.html) | 设置元素的左边框的样式。                                     |
| [border-left-width](https://www.runoob.com/cssref/pr-border-left-width.html) | 设置元素的左边框的宽度。                                     |
|                                                              |                                                              |
| [border-right](https://www.runoob.com/cssref/pr-border-right.html) | 简写属性，用于把右边框的所有属性设置到一个声明中。style/width/color **不分顺序** |
| [border-right-color](https://www.runoob.com/cssref/pr-border-right-color.html) | 设置元素的右边框的颜色。                                     |
| [border-right-style](https://www.runoob.com/cssref/pr-border-right-style.html) | 设置元素的右边框的样式。                                     |
| [border-right-width](https://www.runoob.com/cssref/pr-border-right-width.html) | 设置元素的右边框的宽度。                                     |
|                                                              |                                                              |
| [border-top](https://www.runoob.com/cssref/pr-border-top.html) | 简写属性，用于把上边框的所有属性设置到一个声明中。style/width/color **不分顺序** |
| [border-top-color](https://www.runoob.com/cssref/pr-border-top-color.html) | 设置元素的上边框的颜色。                                     |
| [border-top-style](https://www.runoob.com/cssref/pr-border-top-style.html) | 设置元素的上边框的样式。                                     |
| [border-top-width](https://www.runoob.com/cssref/pr-border-top-width.html) | 设置元素的上边框的宽度。                                     |

> 来源：https://www.runoob.com/css/css-border.html

##### 上边框

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
            /* 盒子大小 */
			width: 200px;
			height: 400px;

			/*上边框 宽度 实线 红色*/
			border-top: 2px solid red;
			
		}
	</style>
</head>
<body>
	<div></div>
</body>
</html>
```

> ![image-20211009173701206](http://myapp.img.mykernel.cn/image-20211009173701206.png)

##### 左边框

```css
			border-left: 1px solid green;
```

> ![image-20211009173735199](http://myapp.img.mykernel.cn/image-20211009173735199.png)

##### 右边框

```css
			border-right: 1px solid blue;
```

> ![image-20211009173829648](http://myapp.img.mykernel.cn/image-20211009173829648.png)

##### 下边框

```css
			border-bottom: 11px dotted pink;
```

> ![image-20211009173935671](http://myapp.img.mykernel.cn/image-20211009173935671.png)

#### 表单边框

表单仅显示下边框

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>document</title>
	<style type="text/css">
		div {
			width: 400px;
			height: 200px;

			/*border: 2px solid pink;*/
			border-top: 2px solid red;
			border-left: 2px solid green;
			border-right: 2px dotted orange;
			border-bottom: 2px dashed blue;
		}

		input {
			/*四个边框都不要样式*/
			border-style: none;
			/*仅下边框有样式, 写着靠后会覆盖前面的*/
			border-bottom: 1px dashed blue;
		}
	</style>
</head>
<body>
	<div></div>
	用户名： <input type="text" name=""> <br />
	密码： <input type="text" name="">
</body>
</html>
```

> ![image-20211010085441998](http://myapp.img.mykernel.cn/image-20211010085441998.png)

#### 表格细线边框 `border-collapse`

之前写过一个案例，小说排行榜

> http://blog.mykernel.cn/2021/09/23/%E5%89%8D%E8%A8%80%E5%8F%8Aweb%E6%A0%87%E5%87%86/#%E5%B0%8F%E8%AF%B4%E6%8E%92%E8%A1%8C%E6%A6%9C

![](http://myapp.img.mykernel.cn/pa.png)

边框很粗，如何变细？

现在的边框不在html中定义，而是在css中统一写

```css
	<style type="text/css">
		table {
			border:  1px solid pink;
		}
	</style>
```

> ![image-20211010090415322](http://myapp.img.mykernel.cn/image-20211010090415322.png)
>
> 边框有线了，但是中间却没有了
>
> 表格的格式: ![](http://myapp.img.mykernel.cn/image-20210927220347959.png)
>
> 行的格式
>
> ![](http://myapp.img.mykernel.cn/image-20210927220428556.png)
>
> > 给行定义格式
> >
> > ```css
> > 		table, tr {
> > 			border:  1px solid pink;
> > 		}
> > ```
> >
> > > 可以发现还是![](http://myapp.img.mykernel.cn/image-20211010090415322.png)
>
> 单元格子: ![](http://myapp.img.mykernel.cn/image-20210927220526145.png)
>
> > ```css
> > 	<style type="text/css">
> > 		table, tr, td {
> > 			border:  1px solid pink;
> > 		}
> > 	</style>
> > ```
> >
> > > ![image-20211010092615499](http://myapp.img.mykernel.cn/image-20211010092615499.png)
> > >
> > > 现在没有th
>
> 添加th
>
> ```css
> 	<style type="text/css">
> 		table,  td, th {
> 			border:  1px solid pink;
> 		}
> 	</style>
> ```
>
> > ![image-20211010092700373](http://myapp.img.mykernel.cn/image-20211010092700373.png)
>
> 现在还是粗了, 原因是表格的边框和单元格边框重叠，并且单元格和单元格边框重叠了

单元格合并

```css
		table, tr, td, th {
			border:  1px solid pink;
			border-collapse: collapse;
		}
```

> ![image-20211010093006211](http://myapp.img.mykernel.cn/image-20211010093006211.png)

让第1行颜色加深

```css
		.hotpink {
			background-color: hotpink;
		}
```

```html
		<tr class="hotpink">
			<th>排名</th>
			<th>关键词</th>
			<th>趋势</th>
			<th>今日搜索</th>
			<th>最近七日</th>
			<th>相关链接</th>
		</tr>
```

> ![image-20211010093208110](http://myapp.img.mykernel.cn/image-20211010093208110.png)

让奇数行变浅色

```css
		.pink {
			background-color: pink;
		}
```

> 让奇数行添加此`class=pink`
>
> ![image-20211010093344447](http://myapp.img.mykernel.cn/image-20211010093344447.png)

### 盒子内边距(padding)

![](https://imgs.developpaper.com/imgs/2786317785-5f6d90d69a04e_articlex.png)

**文字到边框的距离**

| 属性                                                         | 说明                                       |
| :----------------------------------------------------------- | :----------------------------------------- |
| [padding](https://www.runoob.com/cssref/pr-padding.html)     | 使用简写属性设置在一个声明中的所有填充属性 |
| [padding-bottom](https://www.runoob.com/cssref/pr-padding-bottom.html) | 设置元素的底部填充                         |
| [padding-left](https://www.runoob.com/cssref/pr-padding-left.html) | 设置元素的左部填充                         |
| [padding-right](https://www.runoob.com/cssref/pr-padding-right.html) | 设置元素的右部填充                         |
| [padding-top](https://www.runoob.com/cssref/pr-padding-top.html) | 设置元素的顶部填充                         |

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 200px;
			height: 200px;
			/*边框*/
			border: 1px solid red;
		}
	</style>
</head>
<body>
	<div></div>
</body>
</html>
```

> 边框: ![image-20211011203118746](http://myapp.img.mykernel.cn/image-20211011203118746.png)

盒子的边框中是装内容的，放文本: **王者荣耀**

```html
	<div>王者农药</div>
```

> 浏览器中已经出现了文字:
>
> ![image-20211011203237963](http://myapp.img.mykernel.cn/image-20211011203237963.png)
>
> 默认的文字在左上角

内边距定义, 左边的距离

```css
		div {
			width: 200px;
			height: 200px;
			/*边框*/
			border: 1px solid red;

			
			/*pl10*/
			padding-left: 10px;
		}
```

> ![image-20211011203427619](http://myapp.img.mykernel.cn/image-20211011203427619.png)

上边的距离

```css
			/*pt30*/
			padding-top: 30px;
```

> ![image-20211011203527136](http://myapp.img.mykernel.cn/image-20211011203527136.png)

#### padding会把盒子变大

现在可以发现，我们的高度和宽度中间的200x200，放在F12的computed中的内容200X200上，网页上变成蓝色。

![image-20211011203749554](http://myapp.img.mykernel.cn/image-20211011203749554.png)

> **浏览器F12定位到方框中，才看右边的computed**
>
> 右边`-`表示没有定义

#### 不同padding值

##### 第1个值

```css
		div {
			width: 200px;
			height: 200px;
			/*边框*/
			border: 1px solid red;

			padding: 20px;
		}
```

> ![image-20211011204251485](http://myapp.img.mykernel.cn/image-20211011204251485.png)
>
> 全部是20，而且右边会标注出蓝色是 20的px

##### 前2个值`上下边距 左右边距`

```css

			padding: 10px 20px;
```

> ![image-20211011204421962](http://myapp.img.mykernel.cn/image-20211011204421962.png)

##### 3个值 `上 左右 下`

```css
padding: 10px 20px 30px;
```



##### 4个值 `下 右 下 左` 顺时针

```css
padding: 10px 20px 30px 40px;
```

> ![image-20211011204753472](http://myapp.img.mykernel.cn/image-20211011204753472.png)

#### 练习

##### 练习padding

1. 要求盒子有1个左边内边距 5像素

   ```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
   	<meta charset="UTF-8">
   	<meta name="viewport" content="width=device-width, initial-scale=1.0">
   	<title>Document</title>
   	<style>
   		div {
   			width: 200px;
   			height: 200px;
   			border: 1px solid red;
   			
   			padding-left: 5px;
   		}
   	</style>
   </head>
   <body>
   	<div></div>
   </body>
   </html>
   ```

   > 浏览器F12定位到方框中，才看右边的computed
   >
   > ![image-20211011205604267](http://myapp.img.mykernel.cn/image-20211011205604267.png)

2. 要求简写：上下25px, 左右15px

   ```css
   <!DOCTYPE html>
   <html lang="en">
   <head>
   	<meta charset="UTF-8">
   	<meta name="viewport" content="width=device-width, initial-scale=1.0">
   	<title>Document</title>
   	<style>
   		div {
   			width: 200px;
   			height: 200px;
   			border: 1px solid red;
   
   			/*padding-left: 5px;*/
   			padding: 25px 15px;
   		}
   	</style>
   </head>
   <body>
   	<div></div>
   </body>
   </html>
   ```

   > 浏览器F12定位到方框中，才看右边的computed
   >
   > ![image-20211011205732632](http://myapp.img.mykernel.cn/image-20211011205732632.png)

3. 简写，黑子 上内边距12px, 下内0，左内25，右内10

   ```css
   		div {
   			width: 200px;
   			height: 200px;
   			border: 1px solid red;
   
   			/*padding-left: 5px;*/
   			padding: 12px 10px 0px 25px
   		}
   ```

   > 同上题源码， 下边是 0px 正好显示`-`
   >
   > ![image-20211011205916583](http://myapp.img.mykernel.cn/image-20211011205916583.png)

##### 实现新浪导航 `字数不一样多的处理`

![image-20211011210348487](http://myapp.img.mykernel.cn/image-20211011210348487.png)



- 左边导航，右边导航
- 新浪**导航栏的核心就是字数不一样多，不方便我们给宽度`width`，还是 给`padding`**

1. 结构

   变色，说明都是链接

   在一行，大盒子是div

   文本垂直居中

   ```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
   	<meta charset="UTF-8">
   	<meta name="viewport" content="width=device-width, initial-scale=1.0">
   	<title>Document</title>
   </head>
   <body>
   	<div class="nav">
   		<a href="#">设为首页</a>
   		<a href="#">手机新浪网</a>
   		<a href="#">移动客户端</a>
   		<a href="#">登录</a>
   		<a href="#">微博</a>
   		<a href="#">博客</a>
   		<a href="#">邮箱</a>
   		<a href="#">网站导航</a>
   		<a href="#">关闭</a>
   	</div>
   </body>
   </html>
   ```

   

2. 样式盒子

   剪切下来，如图

   ![sina](http://myapp.img.mykernel.cn/sina.png)

   打开PS, 加载图片，使用space + 鼠标左键移动图片。使用ctrl  +/- 来放大图片

   选中左侧的切片

   ![image-20211011211440352](http://myapp.img.mykernel.cn/image-20211011211440352.png)

   在图片上，选中上边框下面到底部的距离，右键看切片选项, 可以看出`41Height`

   ![image-20211011211834055](http://myapp.img.mykernel.cn/image-20211011211834055.png)

   由于浏览器中，宽度同浏览器的高度，所以css如下

   ```css
   	<style>
   		.nav {
   			/*宽度和浏览器一样*/
   			height: 41px;
   		}
   	</style>
   ```

   现在使用吸管工具，选取白色区域

   ![image-20211011211958623](http://myapp.img.mykernel.cn/image-20211011211958623.png)

   在前景色处点之后，就可以看到当前的颜色，可以看到红色框中是`#FCFCFC`

   ![image-20211011212320544](http://myapp.img.mykernel.cn/image-20211011212320544.png)

   ```css
   		.nav {
   			/*宽度和浏览器一样*/
   			height: 41px;
   			background-color: #fcfcfc;
   		}
   ```

   现在上边框，如法泡制，量边框，取颜色

   ![image-20211011213043582](http://myapp.img.mykernel.cn/image-20211011213043582.png)

   ```css
   		.nav {
   			/*宽度和浏览器一样*/
   			height: 41px;
   			background-color: #fcfcfc;
   
   			border-top: 3px solid #ff8500;
   		}
   ```

   > ![image-20211011213247103](http://myapp.img.mykernel.cn/image-20211011213247103.png)

   现在下边框，如法泡制，量边框，取颜色

   ![image-20211011213138816](http://myapp.img.mykernel.cn/image-20211011213138816.png)

   ```css
   		.nav {
   			/*宽度和浏览器一样*/
   			height: 41px;
   			background-color: #fcfcfc;
   
   			border-top: 3px solid #ff8500;
   			border-bottom: 1px solid #edeef0;
   		}
   ```

   > ![image-20211011213312789](http://myapp.img.mykernel.cn/image-20211011213312789.png)

3. 装内容

   ```html
   	<div class="nav">
   		<a href="#">设为首页</a>
   		<a href="#">手机新浪网</a>
   		<a href="#">移动客户端</a>
   		<a href="#">登录</a>
   		<a href="#">微博</a>
   		<a href="#">博客</a>
   		<a href="#">邮箱</a>
   		<a href="#">网站导航</a>
   		<a href="#">关闭</a>
   	</div>
   ```

   ```css
   		.nav a {
   			/*高度和父亲一样*/
   			height: 41px;
   			background-color: pink;
   		}
   ```

   > ![image-20211011213751173](http://myapp.img.mykernel.cn/image-20211011213751173.png)

   a是行内没有大小，需要转换

   ```css
   		.nav a {
   			display: inline-block;
   ```

   > ![image-20211011213841047](http://myapp.img.mykernel.cn/image-20211011213841047.png)
   >
   > 现在的高度有了

   现在给padding时，会变大盒子，由于上下有了，不能加大，而是左右添加像素

   ```diff
   		.nav a {
   			display: inline-block;
   			/*高度和父亲一样*/
   			height: 41px;
   			background-color: pink;
   
   
   +			padding: 0px 20px;
   +			background-color: pink;
   		}
   ```

   > ![image-20211011214043209](http://myapp.img.mykernel.cn/image-20211011214043209.png)

   文本垂直居中, 取消下划线

   ```diff
   		.nav a {
   			display: inline-block;
   			/*高度和父亲一样*/
   			height: 41px;
   			background-color: pink;
   
   
   			padding: 0px 20px;
   			background-color: pink;
   
   +			line-height: 41px;
   +			text-decoration: none;
   		}
   		
   
   ```

   > ![image-20211011214515386](http://myapp.img.mykernel.cn/image-20211011214515386.png)

   PS获取新浪的文字颜色

   ![image-20211011214639271](http://myapp.img.mykernel.cn/image-20211011214639271.png)

   ```css
   			color: #4c4c4c;
   ```

   > ![image-20211011214653622](http://myapp.img.mykernel.cn/image-20211011214653622.png)

   字体大小 

   ```css
   font-size: 12px;
   ```

   > ![image-20211011215436476](http://myapp.img.mykernel.cn/image-20211011215436476.png)

   新浪的字，鼠标放上去字会变黄，背景变灰

   ```css
   		.nav a:hover {
   			background-color: #ccc;
   			color: orange;
   		}
   ```

   > ![image-20211011215624503](http://myapp.img.mykernel.cn/image-20211011215624503.png)

#### 内盒尺寸计算（元素实际大小 ）

padding会撑开盒子，一个网页如果2个盒子组成，其中一个变大 ，另一个盒子就挤下来了。

- 盒子实际宽  = 内容宽 + 内边距左右 + 边框左右
- 盒子实际高 = 内容高  + 内边距上下 + 边框上下

> 盒子的200px 200px
>
> ```html
> <!DOCTYPE html>
> <html lang="en">
> <head>
> 	<meta charset="UTF-8">
> 	<meta name="viewport" content="width=device-width, initial-scale=1.0">
> 	<title>Document</title>
> 	<style>
> 		div {
> 			/*内容的宽度和高度*/
> 			width: 200px;
> 			height: 200px;
> 
> 			background-color: pink;
> 
> 		}
> 	</style>
> </head>
> <body>
> 	<div></div>
> </body>
> </html>
> ```
>
> 
>
> ![image-20211011215954495](http://myapp.img.mykernel.cn/image-20211011215954495.png)
>
> 现在给一个padding值，现在横220px, 纵220px.
>
> ```CSS
> 			/*添加内边距*/
> 			padding: 10px;
> ```
>
> ![image-20211011220159885](http://myapp.img.mykernel.cn/image-20211011220159885.png)

解决方法：

内边距要给，真正的内容宽度 = 测量的内容宽度(**量**是的内容+padding，作为内容宽度) - padding左右

> 由于测量时，直接给的高度，是内容的高度。在未定义padding前，是正常的。
>
> 定义padding后，就必须是之前测量高度-添加的padding值。
>
> 所以测量的高度是最终  height + padding 的值。某个值为0时，另一个值就是整个高度

所以最终的新浪，由于定义的padding第1个值是上下，为0表示，内容就是测量的高度。

##### 练习

1. 盒子实际大小 

   ```css
   width: 100;
   padding: 10px;
   border: 5px;
   ```

   100+20 + 10

2. 求盒子实际宽高

   ```css
   width: 200px
   heigth: 200px
   border: 1px solid #000000;
   border-top: 5px solid blue;
   padding: 50px;
   padding-left: 100px;
   ```

   宽：200+2+100+50=352

   高：200+1+5+50+50=306

   > 下面会覆盖上面的部分属性

#### padding不影响盒子大小 

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			/*内容的宽度和高度*/
			width: 200px;
			height: 200px;

			background-color: pink;
		}

		p {
			/*块内元素的宽度和父亲一样宽*/
			height: 30px;
			background-color: purple;
		}
	</style>
</head>
<body>
	<div><p></p></div>
</body>
</html>
```

> ![image-20211011222233266](http://myapp.img.mykernel.cn/image-20211011222233266.png)



现在给p标签30像素的左内边距

```diff
		p {
			/*块内元素的宽度和父亲一样宽*/
			height: 30px;
			background-color: purple;

+			padding-left: 30px;
		}
	</style>
</head>
<body>
+	<div><p>dadada</p></div>
```

> 如果没有宽度就就不会撑开盒子, 会自动把文本的宽度从原始宽度中减去padding
>
> ![image-20211011222509662](http://myapp.img.mykernel.cn/image-20211011222509662.png)

现在不给宽度会继承自父亲的200px的宽度，但是如果我们给一个宽度200px

```diff
		p {
			/*块内元素的宽度和父亲一样宽*/
+			width: 200px;
			height: 30px;
			background-color: purple;

			padding-left: 30px;
		}
```

> 会超出30px
>
> ![image-20211011222809803](http://myapp.img.mykernel.cn/image-20211011222809803.png)





### 盒子外边距(margin)

盒子到盒子，盒子周围的距离 

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 200px;
			height: 200px;

			background-color: pink;
		}

	</style>
</head>
<body>
	<div></div>
</body>
</html>
```

> ![image-20211011215954495](http://myapp.img.mykernel.cn/image-20211011215954495.png)

现在使用外边距

```html
		div {
			width: 200px;
			height: 200px;

			background-color: pink;
			margin-left: 100px;
			margin-top: 50px;
			margin-right: 10px;
			

		}
```

> ![image-20211011223358421](http://myapp.img.mykernel.cn/image-20211011223358421.png)

简写

```css
1: 四周
2：上下，左右

```



#### 察言观色

```css
		div {
			width: 200px;
			height: 200px;

			background-color: pink;
			margin: 100px;
			padding: 20px 10px;
		}

```

> ![image-20211011223610130](http://myapp.img.mykernel.cn/image-20211011223610130.png)
>
> 蓝色是内容，青色是padding, 橙色是margin, 浏览器中margin显示右边很多，但是实际上的右下方只有100.

通过www.mi.com

- 青色是padding. 蓝色是 文本。
- 右下方详细写了padding/content大小 

![image-20211011223816740](http://myapp.img.mykernel.cn/image-20211011223816740.png)



#### 块级盒子水平居中

把小米官网缩小时(ctrl + 鼠标向下滚)，https://www.mi.com/， 页面中是水平居中的，除了最上面的。

水平居中的好处，无论屏幕什么分辨率，都可以在中间显示。

![image-20211011224216749](http://myapp.img.mykernel.cn/image-20211011224216749.png)

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 600px;
			height: 400px;
			background-color: pink;
		}

	</style>
</head>
<body>
	<div></div>
</body>
</html>
```

> ![image-20211011224428379](http://myapp.img.mykernel.cn/image-20211011224428379.png)s

让块级元素居中对齐 

1. 必须有宽度
   - 没有宽度，是拉满的，就不能左右加外边距了。
2. 左右外边距设置为auto
   - 只有`margin-left: auto;` 只会左边充满。
   - 只有`margin-right: auto;` 只会右边充满。

```css
		div {
			width: 600px;
			height: 400px;
			background-color: pink;

			margin-left: auto;
			margin-right: auto;
		}
```

> ![image-20211011224552941](http://myapp.img.mykernel.cn/image-20211011224552941.png)

> 小米商城
>
> ```css
> .container {
>     width: 1226px;
>     margin-right: auto;
>     margin-left: auto;
> }
> ```

简写

```css
写法1
margin: auto;

写法2
margin: 0 auto;

写法3
			margin-left: auto;
			margin-right: auto;
```

#### 盒子中的文字居中和块级盒子居中

```css
text-align: center /* 文字  行内元素 行内块元素 居中*/
margin: 10px auto; /* 块级居中 */
```



```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 600px;
			height: 300px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div></div>
</body>
</html>
```

> ![image-20211011225030017](http://myapp.img.mykernel.cn/image-20211011225030017.png)

让块级盒子水平居中

```css
			margin: 0 auto;
```

> ![image-20211011224552941](http://myapp.img.mykernel.cn/image-20211011224552941.png)

先给一个文字

```html
<body>
	<div>稳住 </div>
```

> ![image-20211011225212649](http://myapp.img.mykernel.cn/image-20211011225212649.png)
>
> 文字在左上角

让盒子中的文字居中

```css
			text-align: center;
```

> ![image-20211011225318692](http://myapp.img.mykernel.cn/image-20211011225318692.png)

行内元素，也受`text-align`居中控制

```html
<div>稳住 <strong>我们能赢</strong></div>
```

> ![image-20211011225506584](http://myapp.img.mykernel.cn/image-20211011225506584.png)

行内块元素，input/img/td, 也受text-align影响

```css
<div>稳住 <strong>我们能赢</strong> <input type="text" name=""></div>
```

> ![image-20211011225613629](http://myapp.img.mykernel.cn/image-20211011225613629.png)

上面给一定的距离

```css
		div {
			width: 600px;
			height: 300px;
			background-color: pink;

			margin: 20px auto 0px;

			text-align: center;
		}
```

> ![image-20211012205000739](http://myapp.img.mykernel.cn/image-20211012205000739.png)

#### 插入图片和背景图片区别

- 应用场景
  - 插入图片：重要的信息，产品展示内，`<img />`, 移动位置padding/margin
  - 背景图片：log, 小图标或超大背景图片, 移动位置 background-position

```css
		img {
			width: 500px;
			height: 500px;
			border: 1px solid red;
            padding: 30px; /*会影响图片位置 */
			margin: 30px; /*会影响图片位置 */
		}



		.bg {
			width: 500px;
			height: 500px;
			border: 1px solid red;
            
			background: url(images/1.png) no-repeat;
			padding: 300px; /*不会影响背景位置 */
            margin: 20px; /*会影响背景位置 */
			background-position: top; /*背景修改位置 */
		}
```

##### 插入图片

准备一个大盒子

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.pic {
			width: 500px;
			height: 500px;
			border: 1px solid red;
		}
	</style>
</head>
<body>
	<div class="pic">
		
	</div>
</body>
</html>
```

> ![image-20211012205409609](http://myapp.img.mykernel.cn/image-20211012205409609.png)

在盒子中，插入一个图片

```html
<body>
	<div class="pic">
		<img src="./images/1.png" />
	</div>
</body>
</html>
```

> ![image-20211012205736180](http://myapp.img.mykernel.cn/image-20211012205736180.png)

图片要移动位置, padding/margin

- 大盒子：移动，盒子会变大

  ```css
  		.pic {
  			width: 500px;
  			height: 500px;
  			border: 1px solid red;
  
  			padding: 30px;
  		}
  ```

  > 1. 盒子变大了
  >
  >    ![image-20211012210027133](http://myapp.img.mykernel.cn/image-20211012210027133.png)
  >
  > 2. 图片位置变了
  >
  >    ![image-20211012210057787](http://myapp.img.mykernel.cn/image-20211012210057787.png)

- 图片的标签盒子

  ```css
  	<style>
  		.pic {
  			width: 500px;
  			height: 500px;
  			border: 1px solid red;
  
  			/*padding: 30px;*/
  		}
  
  		.pic img {
  			margin: 30px;
  		}
  ```

  > 图片的外边距有了，图片就下来了
  >
  > ![image-20211012210248163](http://myapp.img.mykernel.cn/image-20211012210248163.png)

##### 背景图片

```html
	<div class="bg">
		
	</div>
```

```css
		.bg {
			background:  url(./images/1.png);
		}
```

> 显示不出来，需要加宽度和高度
>
> ```css
> 		.pic, .bg {
> 			width: 500px;
> 			height: 500px;
> 			border: 1px solid red;
> 		}
> ```
>
> > ![image-20211013222431717](http://myapp.img.mykernel.cn/image-20211013222431717.png)

现在给盒子添加内边距

```css
		.bg {
			background: url(images/1.png) no-repeat;
			padding: 300px;
		}
```

> 蓝色部分的内边距虽然加大了，但是背景还在是原处
>
> ![image-20211013222545842](http://myapp.img.mykernel.cn/image-20211013222545842.png)

使用外边距 margin

```diff
		.bg {
			background: url(images/1.png) no-repeat;
			padding: 300px;
+			margin: 20px;
		}
```

> 可以发现外边距上是不能放背景的, 所以margin可以控制背景的位置 
>
> ![image-20211013222712994](http://myapp.img.mykernel.cn/image-20211013222712994.png)

使用背景的位置  

```diff
		.bg {
			background: url(images/1.png) no-repeat;
			padding: 300px;
+			background-position: top;
		}
```

> 在顶部中间
>
> ![image-20211013222837503](http://myapp.img.mykernel.cn/image-20211013222837503.png)

#### 清理元素默认的内外边距

哪里来的空隙？

```css
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
</head>
<body>
	一个问题
</body>
</html>
```

> 红色和绿色之间有一个空隙
>
> ![image-20211013223440922](http://myapp.img.mykernel.cn/image-20211013223440922.png)

现在放在盒子中的

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>Document</title>
	<style>
		.bg {
			border: 1px solid pink;
			width: 700px;
			height: 700px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div class="bg">
		123
	</div>
</body>
</html>
```

> 注意盒子中没有空隙，但是盒子上面有一个空隙
>
> ![image-20211013223713963](http://myapp.img.mykernel.cn/image-20211013223713963.png)

当我们鼠标放在body上时, 可以观察到body的默认样式块元素、外边距8px

![image-20211013223913438](http://myapp.img.mykernel.cn/image-20211013223913438.png)

同样加一个p标签，同样也会有黄色外边距

![image-20211013224256342](http://myapp.img.mykernel.cn/image-20211013224256342.png)

现在去掉大量的元素内外边距, CSS第1句话， 之后自己的内外边距才是准的

```css
* {
    padding: 0;
    margin: 0;
}
```

注意：行内的元素，尽量只设置左右内外边距，不要设置上下内外边距，为了照顾浏览器的兼容性

> ```html
> <!DOCTYPE html>
> <html lang="en">
> <head>
> 	<meta charset="UTF-8">
> 	<title>Document</title>
> 	<style>
> 		* {
> 		    padding: 0;
> 		    margin: 0;
> 		}
> 	</style>
> </head>
> <body>
> 	<span>
> 		尽量只设置左右内外边距，不要设置上下内外边距，为了照顾浏览器的兼容性
> 	</span>
> </body>
> </html>
> ```
>
> ![image-20211013224804376](http://myapp.img.mykernel.cn/image-20211013224804376.png)
>
> 现在如果加一个外边距
>
> ```css
> 		span {
> 			margin: 30px;
> 		}
> ```
>
> 可以发现，给行内元素就算设置了上下的内外边距，但是只会有左右内外边距生效。IE8不支持，所以直接以后只给行内元素上下内外边距。
>
> ![image-20211013224934943](http://myapp.img.mykernel.cn/image-20211013224934943.png)

#### 外边距合并

##### 垂直外边距会自动合并

准备2个盒子

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.top, .bottom {
			width: 200px;
			height: 200px;
			background-color: pink;
		}
		.bottom {
			background-color: purple;
		}

	</style>
</head>
<body>
	
	<div class="top">
		
	</div>

	<div class="bottom">
		
	</div>
</body>
</html>
```

> ![image-20211013225338776](http://myapp.img.mykernel.cn/image-20211013225338776.png)

给上方盒子下外边距100px

```css
		.top {
			margin-bottom: 100px;
		}
```

> ![image-20211013225521438](http://myapp.img.mykernel.cn/image-20211013225521438.png)

给下方盒子上外边距50px

```css
		.bottom {
			background-color: purple;
			margin-top: 50px;
		}
```

> ![image-20211013225639395](http://myapp.img.mykernel.cn/image-20211013225639395.png)

实际上，这个长度才100px

![image-20211013230333761](http://myapp.img.mykernel.cn/image-20211013230333761.png)

> 获取屏幕颜色工具`FSCapture`
>
> https://www.faststonecapture.cn/download

垂直2个盒子，仅量只给其中一个盒子的外边距

##### 兄弟关系的盒子不存在此现象

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.top, .bottom {
			display: inline-block;
			width: 200px;
			height: 200px;
			background-color: pink;
		}
		.top {
			margin: 100px;
		}
		.bottom {
			background-color: purple;
			margin: 50px;
		}

	</style>
</head>
<body>
	
	<span class="top">
		
	</span>

	<span class="bottom">
		
	</span>
</body>
</html>
```

> ![image-20211013230856273](http://myapp.img.mykernel.cn/image-20211013230856273.png)

##### 父子外边距

###### 现象

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		* {
			margin: 0px;
			padding: 0px;
		}
		.father {
			width: 500px;
			height: 500px;
			background-color: pink;
		}
		.son {
			width: 200px;
			height: 200px;
			background-color: purple;
		}
	</style>
</head>
<body>
	<div class="father">
		<div class="son"></div>
	</div>
</body>
</html>
```

> ![image-20211013231212425](http://myapp.img.mykernel.cn/image-20211013231212425.png)

想让内盒子向下走， 如果给里面盒子外边距时

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		* {
			margin: 0px;
			padding: 0px;
		}
		.father {
			width: 500px;
			height: 500px;
			background-color: pink;
			/*padding-top: 50px;*/ /* 外盒子会变大
			/*margin-top: 50px; /*盒子会移动*/*/
		}
		.son {
			width: 200px;
			height: 200px;
			background-color: purple;
			margin-top: 50px;
		}
	</style>
</head>
<body>
	<div class="father">
		<div class="son"></div>
	</div>
</body>
</html>
```

> 父级盒子也会向下走
>
> ![image-20211013231620319](http://myapp.img.mykernel.cn/image-20211013231620319.png)

###### 解决方案

1. 为父元素定义**上边框**。

   ```diff
   		.father {
   			width: 500px;
   			height: 500px;
   			background-color: pink;
   			/*padding-top: 50px;*/ /* 外盒子会变大*/
   			/*margin-top: 50px; /*盒子会移动*/
   +			border-top: 1px solid transparent;
   		}
   		.son {
   			width: 200px;
   			height: 200px;
   			background-color: purple;
   			margin-top: 50px;
   		}
   ```

   > 孩子的上边距在父亲上了
   >
   > ![image-20211014225745754](http://myapp.img.mykernel.cn/image-20211014225745754.png)

2. 为父元素定义**上内边距**。

   ```diff
   		.father {
   			width: 500px;
   			height: 500px;
   			background-color: pink;
   			/*padding-top: 50px;*/ /* 外盒子会变大*/
   			/*margin-top: 50px; /*盒子会移动*/
   			/*border-top: 1px solid transparent;*/
   +			padding-top: 1px;
   		}
   		.son {
   			width: 200px;
   			height: 200px;
   			background-color: purple;
   			margin-top: 50px;
   		}
   ```

   > ![image-20211014225745754](http://myapp.img.mykernel.cn/image-20211014225745754.png)

3. 为父元素添加overflow:hidden

   ```diff
   		.father {
   			width: 500px;
   			height: 500px;
   			background-color: pink;
   			/*padding-top: 50px;*/ /* 外盒子会变大*/
   			/*margin-top: 50px; /*盒子会移动*/
   			/*border-top: 1px solid transparent;*/
   			/*padding-top: 1px;*/
   +			overflow: hidden;
   		}
   		.son {
   			width: 200px;
   			height: 200px;
   			background-color: purple;
   			margin-top: 50px;
   		}
   ```

   > ![image-20211014225745754](http://myapp.img.mykernel.cn/image-20211014225745754.png)

还有其它方法：浮动、固定、绝对定位的盒子不会有问题，后面会总结

### 盒子模型布局稳定性

让中间的盒子移动，父使用padding或子使用margin

按稳定

```
width > padding > margin
```

- width 没有问题，经常使用宽度剩余法 高度剩余法。
- padding属于盒子大小一部分
- margin 有外边距合并的问题。

### PS基础操作

大小 颜色

通常网页美工大部分效果由前端工程师在photoshop中制作，所以一般我们就打开图片，进行测量

- 测量
  - 视图 - 标尺，像素调整
  - 放大或缩小，测量更精确：ctrl +/-
  - 空格可以托动视图

<img src="http://myapp.img.mykernel.cn/ps2.png" />





#### 案例图

<img src="http://myapp.img.mykernel.cn/lieb.png" />

ps加载此图

#### 获取盒子宽度和高度

![image-20211014231727289](http://myapp.img.mykernel.cn/image-20211014231727289.png)

#### 获取颜色

参考：[链接](#3.2.5.3.2. 实现新浪导航 字数不一样多的处理)

### 新闻案例1

<img src="http://myapp.img.mykernel.cn/lieb.png" />

1. 左侧的图片是，案例的图片 arr.jpg

   ![arr](http://myapp.img.mykernel.cn/arr.jpg)

2. 盒子颜色上深下浅，是一个背景图片 `1px`X`226px`

   ![line](http://myapp.img.mykernel.cn/line.jpg)

- 步骤

  - 先弄盒子
  - 再给数据

#### 案例步骤

先上后下，先外后内



##### 从外到内

上面这个页面是在一个大盒子中

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>新闻列表综合案例</title>
	<style>
		* {
			margin: 0px;
			padding: 0px;
		}
		.box {
			width: 328px;
			height: 226px;
			border: 1px solid black;
			overflow: hidden;
		}
	</style>
</head>
<body>
	<div class="box">
	</div>
</body>
</html>
```

> - 通过ps测量宽度和高度 `330` * `228`
> - **测量的时候，不要量边框，如果量了就减2个像素，因为边框最低占1个像素**
>
> **结果**
>
> ![image-20211016220610617](http://myapp.img.mykernel.cn/image-20211016220610617.png)
>
> 让这个盒子水平居中，上面100px
>
> ```diff
> 		.box {
> 			width: 328px;
> 			height: 226px;
> 			border: 1px solid black;
> 			overflow: hidden;
> +			margin: 100px auto;
> 		}
> ```
>
> 添加渐变色
>
> ```diff
> 		.box {
> 			width: 328px;
> 			height: 226px;
> 			border: 1px solid black;
> 			overflow: hidden;
> 			margin: 100px auto;
> +			background: url(images/line.jpg);
> 		}
> ```
>
> ![image-20211016222745470](http://myapp.img.mykernel.cn/image-20211016222745470.png)

 >
   > 现在查看原图
 >
   > <img src="http://myapp.img.mykernel.cn/lieb.png" />
   >
   > 可以发现大盒子内，上边和下边四周有宽度。定义方法有2种：
   >
   > 1. 给大盒子定义内边距，最简单。
   > 2. 给上下2个盒子定义外边距
   >
   > 通过PS测量内边距，![image-20211018224619241](http://myapp.img.mykernel.cn/image-20211018224619241.png)
   >
   > 由于给了内边距会撑大盒子，因为有宽度和高度。所以当存在内边距和边框时，真正的宽度是内容的。
   >
   > ```css
   > 		.box {
   > 			/*width: 328px; 328-30 左右的内边距*/
   > 			width: 298px;
   > 			/*height: 226px; 226 - 30 上下的内边距*/
   > 			height: 196px;
   > 			border: 1px solid black;
   > 			overflow: hidden;
   > 			margin: 100px auto;
   > 			padding: 15px;
   > 			background: url(images/line.jpg);
   > 		}
   > ```
   >
   > > 网页通过F12定位也可以看到大小 
   > >
   > > ![image-20211018225148800](http://myapp.img.mykernel.cn/image-20211018225148800.png)
 >
   > 现在测试给盒子添加一个内容
   >
   > ```html
 > <body>
   > 	<div class="box">
 > 		123
   > 	</div>
   > </body>
   > ```
   >
   > > 可以看到是边框内
   > >
   > > ![image-20211018225402313](http://myapp.img.mykernel.cn/image-20211018225402313.png)

##### 内部从上到下

###### 标题盒子

先给一个标题

```html
<body>
	<div class="box">
		<h2>最新文章/New Articles</h2>
	</div>
</body>
```

> ![image-20211019152116078](http://myapp.img.mykernel.cn/image-20211019152116078.png)
>
> 文件有点大
>
> ```css
> 		.box>h2 {
> 			font-size: 18px;
> 		}
> ```
>
> > 注意：不是直接写h2, 因为可能有很多h2
>
> 添加底边框
>
> ```diff
> 		.box>h2 {
> 			font-size: 18px;
> +			border-bottom: 1px solid #ccc;
> 		}
> ```
>
> ![image-20211019153415183](http://myapp.img.mykernel.cn/image-20211019153415183.png)
>
> 现在这个线有点离这个标题点近，给h2 上下添加padding即可
>
> ```diff
> 		.box>h2 {
> 			font-size: 18px;
> 			border-bottom: 1px solid #ccc;
> +			padding: 5px 0px;
> 		}
> ```
>
> ![image-20211019154816136](http://myapp.img.mykernel.cn/image-20211019154816136.png)

###### 列表盒子

```html
<body>
	<div class="box">
		<h2>最新文章/New Articles</h2>

		<ul>
			<li><a href="#">北京招聘网页设计，平面设计，php</a></li>
			<li><a href="#">体验javascript的魅力</a></li>
			<li><a href="#">jquery世界来临</a></li>
			<li><a href="#">网页设计师的梦想</a></li>
			<li><a href="#">jquery中的链式编程是什么</a></li>
		</ul>
	</div>
</body>
</html>
```

> ![image-20211019155135204](http://myapp.img.mykernel.cn/image-20211019155135204.png)

上面的图片是小点，而我们的案例是箭头。使用小点的危害，上面是firefox, 而在edge中, 展示不一样

> ![image-20211019171302242](http://myapp.img.mykernel.cn/image-20211019171302242.png)

###### 去掉列表默认样式

```css
		li {
			list-style: none;
		}
```

> ![image-20211019171425080](http://myapp.img.mykernel.cn/image-20211019171425080.png)

案例图，把每个li下面有虚线, li是块元素，可以指定大小

![image-20211019171647546](http://myapp.img.mykernel.cn/image-20211019171647546.png)

```css
		.box > ul > li {
			border-bottom: 1px dotted gray;
			height: 28px;
		}
```

> 通过ps测量是32px高度, 测量带了上下border, 所以-2px, 就30px
>
> **不要直接写在li中，页面中还有许多的li, 会影响其他的li**
>
> ![image-20211019172201605](http://myapp.img.mykernel.cn/image-20211019172201605.png)
>
> > 内容28+border 1px
> >
> > 没有定义宽度就会继承父的内容宽度，所以在padding中

文字没有下划线

```css
		.box > ul > li > a {
			text-decoration: none;
			font: 16px "宋体";
			color: #333;
		}
```

> 1. 不能直接a, 带上父
> 2. 颜色可以通过ps吸取

文字需要垂直居中

```diff
		.box > ul > li > a {
			text-decoration: none;
			font: 16px "宋体";
			color: #333;
+			line-height: 28px;
		}
```

> ![image-20211019172841798](http://myapp.img.mykernel.cn/image-20211019172841798.png)

鼠标放上去，a加下划线

```css
		.box > ul > li > a:hover {
			text-decoration: underline;
		}
```

> ![image-20211019172941978](http://myapp.img.mykernel.cn/image-20211019172941978.png)

###### 小图片使用背景图片

```diff
		.box > ul > li {
			border-bottom: 1px dotted gray;
			height: 28px;
+			background: no-repeat  5px center url(http://myapp.img.mykernel.cn/arr.jpg);
		}
```

> 1. 测量案例图的内边距起始位置到箭头，长度，5px
>
> ![image-20211019174006607](http://myapp.img.mykernel.cn/image-20211019174006607.png)
>
> 2. 结果
>
>    ![image-20211019174918845](http://myapp.img.mykernel.cn/image-20211019174918845.png)

###### 文字压了图片

1. 字的角度，链接包含文本，链接加margin. padding.

   ```diff
   		.box > ul > li > a {
   			text-decoration: none;
   			font: 16px "宋体";
   			color: #333;
   			line-height: 28px;
   
   +			margin-left: 20px;
   		}
   ```

   > 大概20px
   >
   > ![image-20211019174242236](http://myapp.img.mykernel.cn/image-20211019174242236.png)

   ```diff
   		.box > ul > li > a {
   			text-decoration: none;
   			font: 16px "宋体";
   			color: #333;
   			line-height: 28px;
   
   +			padding-left: 20px;
   		}
   ```

   > ![image-20211019174242236](http://myapp.img.mykernel.cn/image-20211019174242236.png)

2. **很多情况下使用**<strong style="color: orange">li</strong>的角度，padding-left, 会将字弄过去，但是**背景不受padding影响**。**margin-left时，背景会受影响。**

   > 以上粗体，参考：http://blog.mykernel.cn/2021/09/30/CSS%E5%85%A5%E9%97%A8/#%E8%83%8C%E6%99%AF%E5%9B%BE%E7%89%87

   ```diff
   /* 将以上的a链接中的padding-left 去掉*/ 
   ```

   > ```diff
   > 		.box > ul > li {
   > 			border-bottom: 1px dotted gray;
   > 			height: 28px;
   > 			background: no-repeat  5px center url(http://myapp.img.mykernel.cn/arr.jpg);
   > +			padding-left: 20px;
   > 		}
   > ```
   >
   > ![image-20211019175100483](http://myapp.img.mykernel.cn/image-20211019175100483.png)
   >
   > 
   >
   > > 注意：padding会把整个盒子变大，是**内容宽度外额外加的。**
   > >
   > > 但是这里没有，原因li没有给宽度，继承了父的宽度，所以给li的padding, 内容会自动计算，由图可以知278是自动计算的 。
   > >
   > > http://blog.mykernel.cn/2021/09/30/CSS%E5%85%A5%E9%97%A8/#padding%E4%BC%9A%E6%8A%8A%E7%9B%92%E5%AD%90%E5%8F%98%E5%A4%A7
   > >
   > > 盒子宽度 
   > >
   > > ```css
   > > 		.box {
   > > 			width: 298px;
   > > 			height: 196px;
   > > 			border: 1px solid black;
   > > 			margin: 100px auto;
   > > 			background-image: url(http://myapp.img.mykernel.cn/line.jpg);
   > > 			padding: 15px;
   > > 		}
   > > ```

   ###### h2和ul盒子距离太近

   - h2角度解决
     - margin bottom
   - ul角度
     - margin top

   > 由于ul没有这个标签定义，有现成的h2标签选择器，所以使用现成的
   >
   > ```diff
   > 		.box>h2 {
   > 			font-size: 18px;
   > 			border-bottom: 1px solid #ccc;
   > 			padding: 5px 0px;
   > 
   > +			margin-bottom: 10px;
   > 		}
   > ```
   >
   > > 这黄色的部分就是margin-bottom
   > >
   > > ![image-20211019175846310](http://myapp.img.mykernel.cn/image-20211019175846310.png)
   >
   > 

   

## 今日总结

<table>
	<tr>
		<th>名称</th>
		<th>属性</th>
		<th>描述</th>
	</tr>
	<tr>
		<td rowspan="9">边框</td>
		<td>border-width</td>
		<td>宽度</td>
	</tr>
	<tr>
		<td>border-style</td>
		<td>样式: solid实线|dotted 点|dashed 虚线</td>
	</tr>
	<tr>
		<td>border-color</td>
		<td>颜色</td>
	</tr>
	<tr>
		<td>border: 四个边简写; border-{bottom|left|right|top}: 每个边简写</td>
		<td>混写 border: 1px solid pink;</td>
	</tr>
	<tr>
		<td>border-{bottom|left|right|top}-color</td>
		<td>颜色</td>
	</tr>
	<tr>
		<td>border-{bottom|left|right|top}-style</td>
		<td>样式: solid|dotted|dashed</td>
	</tr>
	<tr>
		<td>border-{bottom|left|right|top}-width</td>
		<td>宽度</td>
	</tr>
	<tr>
		<td>border-collapse</td>
		<td>
			<ul>
				<li>细线边框</li>
				<li>同时给table, td, th定义边框时,边框会重叠，然后通过此属性只显示1条 </li>
			</ul>
		</td>
	</tr>
    <tr>
    	<td>border-radius</td>
        <td>圆角边框；指定半径长度; 圆: 先正方形, 50%; 圆角: 先矩形, 高度的一半的精确像素.</td>
    </tr>
	<tr>
		<td rowspan="5">内边距(宽度)</td>
		<td>padding: 15px</td>
		<td>上下左右边距 15像素</td>
	</tr>
	<tr>
		<td>padding: 10px 20px</td>
		<td>上下 10px 左右 20px</td>
	</tr>
	<tr>
		<td>padding: 10px 20px 30px</td>
		<td>上 10px 左右 20px 下30px</td>
	</tr>
	<tr>
		<td>padding: 10px 20px 30px 40px</td>
		<td>上 10px 右 20px 下30px 左40px, 顺时针</td>
	</tr>
	<tr>
		<td>padding; padding-{bottom|left|right|top}</td>
		<td>上下左右</td>
	</tr>
	<tr>
		<td rowspan="2">盒子外边距 (宽度)</td>
		<td>margin; margin-{bottom|left|right|top}</td>
		<td>上下左右</td>
	</tr>
	<tr>
		<td>margin: 4个值</td>
		<td>同padding4个值</td>
	</tr>
	<tr>
		<td rowspan="8">内盒尺寸计算</td>
		<td>只有height/width</td>
		<td>
			内容宽度,盒子整体都是h/w(是height/width缩写)
		</td>
	</tr>
	<tr>
		<td>height/width + border</td>
		<td>
			内容宽度: h/w,盒子整体: h/w+border(上下/左右)
		</td>
	</tr>
	<tr>
		<td>height/width + border + padding</td>
		<td>
			内容宽度: h/w,盒子整体: h/w+border(上下/左右)+padding(上下/左右)
		</td>
	</tr>
	<tr>
		<td>height/width + border + padding + margin</td>
		<td>
			内容宽度: h/w,盒子整体: h/w+border(上下/左右)+padding(上下/左右) + margin(上下/左右)
		</td>
	</tr>
	<tr>
		<td>height/继承的宽度 + border</td>
		<td>
			内容宽度: h/(继承的宽度-border(上下/左右)),盒子整体: h/继承的宽度
		</td>
	</tr>
	<tr>
		<td>height/继承的宽度 + border + padding</td>
		<td>
			内容宽度: h/(继承的宽度-border(上下/左右)-padding(上下/左右)),盒子整体: h/继承的宽度
		</td>
	</tr>
	<tr>
		<td>height/继承的宽度 + border + margin</td>
		<td>
			内容宽度: h/(继承的宽度-border(上下/左右)-margin(上下/左右)),盒子整体: h/继承的宽度
		</td>
	</tr>
	<tr>
		<td>height/继承的宽度 + border + padding + margin</td>
		<td>
			内容宽度: h/(继承的宽度-border(上下/左右)-padding(上下/左右)-margin(上下/左右)),盒子整体: h/继承的宽度
		</td>
	</tr>
	<tr>
		<td rowspan="3">盒子颜色</td>
		<td style="color: #A0B0D6;">content</td>
		<td style="color: #A0B0D6;">蓝色</td>
	</tr>
	<tr>
		<td style="color: #C3C2A0;">padding</td>
		<td style="color: #C3C2A0;">青色</td>
	</tr>
	<tr>
		<td style="color: #F9CC9D;">margin</td>
		<td style="color: #F9CC9D;">黄色</td>
	</tr>
	<tr>
		<td rowspan="2">位置调整</td>
		<td>盒子</td>
		<td>垂直居中: margin: 0 auto; 或margin: auto; 上边给一点距离: margin: 10px auto;</td>
	</tr>
	<tr>
		<td>文字及行内元素</td>
		<td>text-align: center;</td>
	</tr>
	<tr>
		<td rowspan="2">图片位置</td>
		<td>img</td>
		<td>插入图片, margin调整位置 或包装图片外层盒子的margin/padding调整位置</td>
	</tr>
	<tr>
		<td>background</td>
		<td>背景图片, 依赖有高度和(继承的)宽度. background-position调整位置  或包装图片外层盒子的margin可以调整位置, padding不会影响背景的位置</td>
	</tr>
	<tr>
		<td>清理内外边距</td>
		<td> <code>
			* {
				padding: 0px;
				margin: 0px;
			}
		</code></td>
		<td>清理网页多余的空隙</td>
	</tr>
	<tr>
		<td rowspan="3">外边距合并</td>
		<td>垂直外边距</td>
		<td>自动合并</td>
	</tr>
	<tr>
		<td>兄弟外边距</td>
		<td>不会合并</td>
	</tr>
	<tr>
		<td>父子外边距</td>
		<td>
			<dl>
				<dt>给子元素定义margin， 父元素也会向下走, 让父元素不向下走的方法，有3种: </dt>
				<dd>给父元素定义上边框 border-top</dd>
				<dd>给父元素定义上内边距 padding-top</dd>
				<dd>给父元素添加overflow:hidden</dd>
			</dl>
		</td>
	</tr>
	<tr>
		<td>盒子稳定性</td>
		<td>
			<code>优先级递减: width > padding > margin</code>
		</td>
		<td>
			使用这3个元素时，哪个优先采用? d
			<ol>
				<li>width最优选择, 宽度剩余法 高度剩余法. 或直接继承外部宽度。</li>
				<li>padding: 属于盒子大小一部分</li>
				<li>margin: 有外边距合并的问题</li>
			</ol>
		</td>
	</tr>
	<tr>
		<td>PS基础操作</td>
		<td>
			放大或缩小，测量更精确：ctrl +/-
			空格可以托动视图
			吸管取颜色
			矩形取像素
		</td>
		<td></td>
	</tr>
    <tr>
    	<td>去掉默认样式</td>
        <td><code>list-style: none;</code></td>
        <td>一般列表有默认的样式，可以这样去掉</td>
    </tr>
</table>


## CSS3的

不影响布局，只是更好看

### 圆角

border-radius: length; 半径

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 200px;
			height: 200px;
			border: 1px solid pink;
		}
	</style>
</head>
<body>
	<div></div>
</body>
</html>
```

> ![image-20211019221917217](http://myapp.img.mykernel.cn/image-20211019221917217.png)

指定半径

```css
		div {
			border-radius: 50%;
		}
```

> ![image-20211019222044090](http://myapp.img.mykernel.cn/image-20211019222044090.png)

```php+HTML
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		p {
			width: 100px;
			height: 20px;
			background-color: red;
			font-size: 12px;
			text-align: center;
			color: #fff;
		}
	</style>
</head>
<body>
	<p>今日特价 免费送</p>
</body>
</html>
```

> ![image-20211019222446159](http://myapp.img.mykernel.cn/image-20211019222446159.png)

使用高度的一半

```css
			border-radius: 10px;
```

> ![image-20211019222544625](http://myapp.img.mykernel.cn/image-20211019222544625.png)

### 盒子阴影

```css
box-shadow: 水平阴影 垂直阴影 模糊距离(虚实) 阴影尺寸(影子大小) 阴影颜色 内/外阴影
```

| 值       | 描述                                            |
| -------- | ----------------------------------------------- |
| h-shadow | **必需**。水平阴影的位置。允许负值              |
| x-shadow | **必需**。垂直阴影的位置。允许负值              |
| blur     | 可选。模糊距离 允许负值                         |
| spread   | 可选。阴影的尺寸 允许负值                       |
| color    | 可选。阴影的景色 允许负值                       |
| inset    | 可选。将外部阴影(outset)改为内部阴影。 允许负值 |

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 200px;
			height: 200px;
			background-color: pink;
			margin: 50px auto;
		}
	</style>
</head>
<body>
	<div></div>
</body>
</html>
```

> ![image-20211020085850068](http://myapp.img.mykernel.cn/image-20211020085850068.png)

```diff
		div {
			width: 200px;
			height: 200px;
			background-color: pink;
			margin: 50px auto;
+			/*box-shadow: 水平阴影 垂直阴影 模糊距离(虚实) 阴影尺寸(影子大小) 阴影颜色 内/外阴影*/
+			box-shadow: 2px 2px 2px 2px black; /*默认 外部阴影*/
		}
```



> ![image-20211020090239184](http://myapp.img.mykernel.cn/image-20211020090239184.png)

#### 水平阴影

鼠标选中浏览器的F12中的box-shadow, 第1个2px，按方向的上箭头，观察变化

> 阴影在水平线上移动

当为39px

![image-20211020090356453](http://myapp.img.mykernel.cn/image-20211020090356453.png)

当为115px

![image-20211020090416181](http://myapp.img.mykernel.cn/image-20211020090416181.png)

#### 垂直阴影

鼠标选中浏览器的F12中的box-shadow, 第2个2px，按方向的上箭头，观察变化

> 后面的阴影盒子会向下移动

28px

![image-20211020090629117](http://myapp.img.mykernel.cn/image-20211020090629117.png)

228px

![image-20211020090702346](http://myapp.img.mykernel.cn/image-20211020090702346.png)

#### 阴影虚实

鼠标选中浏览器的F12中的box-shadow, 第3个2px，按方向的上箭头，观察变化

> 后面的阴影数值小越实，越大越模糊

1px

![image-20211020090805882](http://myapp.img.mykernel.cn/image-20211020090805882.png)

12px

![image-20211020090750526](http://myapp.img.mykernel.cn/image-20211020090750526.png)

#### 阴影尺寸

鼠标选中浏览器的F12中的box-shadow, 第4个2px，按方向的上箭头，观察变化

> 后面的尺寸会放大或缩小

-37px

![image-20211020090934757](http://myapp.img.mykernel.cn/image-20211020090934757.png)

100px

![image-20211020091002479](http://myapp.img.mykernel.cn/image-20211020091002479.png)

#### 阴影颜色

鼠标选中浏览器的F12中的box-shadow, 第5个值，通过调色板修改阴影颜色 

#### 内外阴影

默认在外阴影，不可以写。想要内阴影。

```diff
		div {
			width: 200px;
			height: 200px;
			background-color: pink;
			margin: 50px auto;
			/*box-shadow: 水平阴影 垂直阴影 模糊距离(虚实) 阴影尺寸(影子大小) 阴影颜色 内/外阴影*/
+			box-shadow: 2px 2px 2px 2px black inset; /*默认 外部阴影*/
		}
```

> ![image-20211020092212362](http://myapp.img.mykernel.cn/image-20211020092212362.png)

#### 半透明的影子

一般使用半透明的影子, 参考小米官网

 ![image-20211020092346024](http://myapp.img.mykernel.cn/image-20211020092346024.png)

![image-20211020093607054](http://myapp.img.mykernel.cn/image-20211020093607054.png)

> 请教前端大佬，说F12并不会显示所有样式，需要在F12样式上边点:hover, 才会显示hover样式
>
> 当点了hover之后，相当于当前定位的元素，被鼠标悬浮。

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 200px;
			height: 200px;
			background-color: pink;
			margin: 50px auto;
			/*box-shadow: 水平阴影 垂直阴影 模糊距离(虚实) 阴影尺寸(影子大小) 阴影颜色 内/外阴影*/
		}
		div:hover {
			width: 200px;
			height: 200px;
			background-color: pink;
			margin: 50px auto;
			/*box-shadow: 水平阴影 垂直阴影 模糊距离(虚实) 阴影尺寸(影子大小) 阴影颜色 内/外阴影*/
+			box-shadow: 0px 15px 30px  rgba(0, 0, 0, .1); /*默认 外部阴影*/
		}
	</style>
</head>
<body>
	<div>
	</div>
</body>
</html>
```

## css书写的规范

### 空格规范

选择器和`{`之间必须有空格, 第二个花括号必须与选择器对齐。

属性名与`:`不能有空格。`:`与值必须有空格

```css
.selector {
    font-size: 16px;
}
```

### 选择器规范

并集选择器，每个选择器必须独占一行

```css
table,
td,
th {
	border: 1px solid black;
    overflow: hidden;
}
```

选择器嵌套不大于3级，位置靠后的限定条件尽可能精确

```css
/* good */
#username input {}
.comment .avatar {}

/* bad */
.page .header .login input {}
.comment div * {}
```

### 属性规则

嵌入式的css: 所有属性必须另起一行, 不用节约空间，后面有压缩方法。

属性定义必须有`;`结尾

# CSS第4天

## 学习目标

- 记忆
  - 说出css的布局的三种机制
- 理解
  - 普通流在布局中的特点
  - 能够说出我们为什么用浮动
  - 我们为什么要清除浮动
- 应用
  - 利用浮动完成导航栏案例
  - 能够清除浮动
  - 能够使用PS切图工具

> 学习完，可以完整的布局页面

## 浮动(float)

### css布局的三种机制

> 前面学习了盒子
>
> 网页**布局**的核心--- 用CSS来摆放盒子

#### 普通流(标准流)

- 块元素，独占一行，自上而下。
- 行内元素，多个元素摆放成一行，遇到父边缘会自动换行。

> http://blog.mykernel.cn/2021/09/30/CSS%E5%85%A5%E9%97%A8/#%E6%98%BE%E7%A4%BA%E6%A8%A1%E5%BC%8F%E6%80%BB%E7%BB%93
>
> > 行块特点
> >
> > 行块转换

#### 浮动流

让普通流浮动起来，让多个块级元素，单行显示。

#### 定位

将盒子定在浏览器的某个位置，css里不定位，特别是后面的js特效。

### 为什么使用浮动？

> 普通标准流，已经不能满足块级元素单行显示或 排列时，只能通过浮动。

思考？

1. 多个块元素放在一行？

   ```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
   	<meta charset="UTF-8">
   	<meta name="viewport" content="width=device-width, initial-scale=1.0">
   	<title>Document</title>
   	<style>
   		div {
   			display: inline-block;
   			width: 300px;
   			height: 300px;
   			background-color: pink;
   		}
   	</style>
   </head>
   <body>
   	<div>1</div>
   	<div>2</div>
   	<div>3</div>
   </body>
   </html>
   ```

   > ![image-20211020102226522](http://myapp.img.mykernel.cn/image-20211020102226522.png)
   >
   > 行内块默认有空白缝隙，空白缝隙非常难以去掉，就算去掉了兼容性有问题。
   >
   > 京东中的一行多个块，没有空隙
   >
   > ![image-20211020102409957](http://myapp.img.mykernel.cn/image-20211020102409957.png)
   >
   > 即便别人网页有空隙，也是计算出来的，如果通过行内块，还需要计算这个默认缝隙，非常麻烦 

2. 2个块元素在单 行中左右对齐？

   不行

### 什么是浮动？

浮动属性设置后，就是浮动流, **会漂浮在普通流之上**，移动到指定的位置。

作用

1. 多个块div 排列成一行，使得浮动成为布局重要的手段
2. 可以实现盒子左右对齐
3. 浮动最早用来控制图片，实现文字(行内元素)环绕图片(行内块)的效果。

![image-20211020103506815](http://myapp.img.mykernel.cn/image-20211020103506815.png)



```css
选择器 { float: 属性值; }
```

| 属性值 | 描述                 |      |
| ------ | -------------------- | ---- |
| none   | 元素不浮动（默认值） |      |
| left   | 元素向左浮动         |      |
| right  | 元素向右浮动         |      |

示例

图片![img](http://myapp.img.mykernel.cn/img.jpg)



```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>Document</title>
</head>
<body>
<p>1993年，在古装片《战神传说》中扮演一个武功超群的渔民；同年，主演动作喜剧片《至尊三十六计之偷天换日》，在片中饰演赌术高明的千门高手钱文迪；此外，他还主演了爱情片《天长地久》，在片中塑造了一个风流不羁的江湖浪子形象。</p>

<p>1994年，刘德华投资并主演了剧情片《天与地》，在片中饰演面对恶势力却毫不退缩的禁毒专员张一鹏。1995年，主演赛车励志片《烈火战车》，在片中饰演叛逆、倔强的阿祖，并凭借该片获得第15届香港电影金像奖最佳男主角提名；同年在动作片《大冒险家》中演绎了立仁从小时候父母双亡到长大后进入泰国空军的故事。</p>
<img src="http://myapp.img.mykernel.cn/img.jpg" alt="" />

<p>1996年，主演黑帮题材的电影《新上海滩》，在片中饰演对冯程程痴情一片的丁力。1997年，担任剧情片《香港制造》的制作人；同年，主演爱情片《天若有情之烽火佳人》，在片中饰演家世显赫的空军少尉刘天伟；12月，与梁家辉联袂主演警匪动作片《黑金》，在片中饰演精明干练、嫉恶如仇的调查局机动组组长方国辉。1998年，主演动作片《龙在江湖》，饰演重义气的黑帮成员韦吉祥；同年，出演喜剧片《赌侠1999》；此外，他还担任剧情片《去年烟花特别多》的制作人。</p>
<p>1993年，在古装片《战神传说》中扮演一个武功超群的渔民；同年，主演动作喜剧片《至尊三十六计之偷天换日》，在片中饰演赌术高明的千门高手钱文迪；此外，他还主演了爱情片《天长地久》，在片中塑造了一个风流不羁的江湖浪子形象。</p>
<p>1994年，刘德华投资并主演了剧情片《天与地》，在片中饰演面对恶势力却毫不退缩的禁毒专员张一鹏。1995年，主演赛车励志片《烈火战车》，在片中饰演叛逆、倔强的阿祖，并凭借该片获得第15届香港电影金像奖最佳男主角提名；同年在动作片《大冒险家》中演绎了立仁从小时候父母双亡到长大后进入泰国空军的故事。</p>
<p>1996年，主演黑帮题材的电影《新上海滩》，在片中饰演对冯程程痴情一片的丁力。1997年，担任剧情片《香港制造》的制作人；同年，主演爱情片《天若有情之烽火佳人》，在片中饰演家世显赫的空军少尉刘天伟；12月，与梁家辉联袂主演警匪动作片《黑金》，在片中饰演精明干练、嫉恶如仇的调查局机动组组长方国辉。1998年，主演动作片《龙在江湖》，饰演重义气的黑帮成员韦吉祥；同年，出演喜剧片《赌侠1999》；此外，他还担任剧情片《去年烟花特别多》的制作人。</p>
</body>
</html>
```

> ![image-20211020103938157](http://myapp.img.mykernel.cn/image-20211020103938157.png)

要把图片放在左边

```css
	<style>
		img {
			float: left;
			margin: 10px;
		}
	</style>
```

> ![image-20211020104543250](http://myapp.img.mykernel.cn/image-20211020104543250.png)

右浮动

```diff
	<style>
		img {
			float: right;
			margin: 10px;
		}
	</style>
```

> ![image-20211020104616757](http://myapp.img.mykernel.cn/image-20211020104616757.png)

#### 浮动

>  不是标准流，漂浮在普通流之上

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.one {
			width: 200px;
			height: 200px;
			background-color: pink;
		}
		.two {
			width: 300px;
			height: 300px;
			background-color: purple;
		}
	</style>
</head>
<body>
	<div class="one"></div>
	<div class="two"></div>
</body>
</html>
```

> 普通流
>
> ![image-20211020104924257](http://myapp.img.mykernel.cn/image-20211020104924257.png)

现在给粉红色的盒子添加浮动

```diff
		.one {
			width: 200px;
			height: 200px;
			background-color: pink;
+			float: left;
		}
```

> 现在的浮动盒子已经脱离标准流了
>
> ![image-20211020105027406](http://myapp.img.mykernel.cn/image-20211020105027406.png)

#### 漏

浮动的盒子，脱离了标准流，自己原来的位置，就给下面的标准流使用了。



#### 特

float属性，会改变display属性。

> 任何元素均可以浮动，浮动元素会生成一个块级框，无论本身啥元素，**生成的块级框和行内块的行为**极其相似。
>
> float添加之后，相当于，而**不是真正的display 值转换为 `inline-block`**，只是行为相似。

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			height: 100px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div></div>
</body>
</html>
```

> ![image-20211020105632738](http://myapp.img.mykernel.cn/image-20211020105632738.png)

给盒子添加浮动

```diff
		div {
			height: 100px;
			background-color: pink;
+			float: left;	
		}
```

> 盒子消失了

当给盒子中添加字

```diff
<body>
	<div>123456</div>
</body>
```

> ![image-20211020105755008](http://myapp.img.mykernel.cn/image-20211020105755008.png)
>
> 可以发现现在不是块级元素了，因为不给宽度 ，默认和父亲一样宽。
>
> 也不是行内元素，因为行内元素没有高度。
>
> **最符合行内块元素。有高度，宽度取决于内容宽度**。
>
> ```diff
> <body>
> 	<div>123456 77777777777</div>
> </body>
> ```
>
> > ![image-20211020105942499](http://myapp.img.mykernel.cn/image-20211020105942499.png)

#### 多个div单行显示

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.one,
		.two {
			width: 200px;
			height: 100px;
			background-color: pink;
		}

		.one {

		}

		.two {
			background-color: purple;
		}
	</style>
</head>
<body>
	<div class="one"></div>
	<div class="two"></div>
</body>
</html>
```

> ![image-20211020110444229](http://myapp.img.mykernel.cn/image-20211020110444229.png)

给one加left

```diff
		.one {
			float: left;
		}
```

> .one浮动起来，two占用one原来的位置

给two添加left

```diff
		.two {
			background-color: purple;
+			float: left;
		}
```

> 现在.one和.two**都有行内块的行为**，所以2个块转换成行内块元素，就会在1行显示。显示的宽度由内容的宽度或实际的宽度决定。
>
> ![image-20211020110748327](http://myapp.img.mykernel.cn/image-20211020110748327.png)
>
> 而且中间没有缝隙, 和真正的行内块有区别，**真正的行内块会有缝隙**。

如果父亲的宽度装不下行内块时，就会自动折行。

### 浮动小结

> 浮动是常见的布局
>
> - 早期，文本围绕图片
> - 现在，块级盒子行内显示

| 特点 | 描述                                                         |
| ---- | ------------------------------------------------------------ |
| 浮   | 加了float属性的盒子，就漂浮在普通块/行内元素的上方           |
| 漏   | 加了float属性的盒子，它原来的位置会漏给标准流的盒子。块盒子会在其下，行内元素及行内块元素不会， |
| 特   | 加了float属性的盒子，会在上层拥有行内块的行为，除了多个浮动元素间隙不存在。 |

> 浮动一定要写在某个选择器选择中，作为第一个属性, 例如
>
> ```css
> /* good */
> .selector {
>     float: left;
> 	width: 200px;
>     height: 200px;
> }
> 
> /* bad */
> .selector {
> 	width: 200px;
>     height: 200px;
>     float: left;
> }
> ```

### 浮动小细节

这种效果怎么制作 ？

![image-20211020112256927](http://myapp.img.mykernel.cn/image-20211020112256927.png)

>  如果上面4个是浮动流，下面的长条下次标准流，那下面的标准流就会占用上面浮动流漏出来的位置。
>
> <strong style="color: purple">上面4个包装在一个标准流div，下面是一个标准流。</strong>

同理，上面也一样，标准流包装 浮动流。

![image-20211020112541870](http://myapp.img.mykernel.cn/image-20211020112541870.png)

![image-20211020112619551](http://myapp.img.mykernel.cn/image-20211020112619551.png)

> 一个完整的网页，由<strong style="color: purple">标准流</strong> + <strong style="color: purple">内嵌浮动流</strong> + <strong style="color: purple">定位</strong>

### 案例

#### 制作小米官网

##### 案例分析

要求外形，而不要内容，图片....

![image-20211020112904236](http://myapp.img.mykernel.cn/image-20211020112904236.png)

> 1. 父亲是标准流
>
> 2. 子是浮动流
>
> 3. 由于内部左右结构不一样，所以分成左右结构
>
>    ![image-20211020113044705](http://myapp.img.mykernel.cn/image-20211020113044705.png)
>
> 4. 右边每个小li，有一个小距离。给每个盒子margin-right, 但是最后一个没有margin-right. 给margin-left合适。

##### 准备盒子

###### 标准流的父盒子，水平居中

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		/*清除样式*/
		* {
			margin: 0px;
			padding: 0px;
		}

		.box {
			width: 1226px;
			height: 615px;
			background-color: pink;
			margin: auto;
		}
	</style>
</head>
<body>
	<div class="box"></div>
</body>
</html>
```

> 宽度？
>
> - 要精细，标尺拉出来，贴在左边。贴在右边。贴在上边、下边。
>
>   ![image-20211020113748570](http://myapp.img.mykernel.cn/image-20211020113748570.png)
>
>   然后通过矩形测量，宽度1226px, 高度615px.
>
>   > 由于此时没有ps, 所以暂时不加ps操作。不过测量的位置如图
>
> - 结果
>
>   ![image-20211020113944559](http://myapp.img.mykernel.cn/image-20211020113944559.png)

 ###### 子浮动盒子

 由于网页是一排一排非常整齐，一般使用无序列表来做。

```html
	<div class="box">
		<div class="left">1</div>
		<ul class="right">
		</ul>
	</div>
```

> - left测量
>
>   由于这个框占整个，所以父亲多高就多高，但是宽度得测量。ps 测量为234px
>
>   ![image-20211020114621372](http://myapp.img.mykernel.cn/image-20211020114621372.png)
>
>   ```css
>   		.left {
>   			width: 234px;
>   			height: 615px;
>   			background-color: purple;
>   			float: left;
>   		}
>   ```
>
>   无论是否浮动，都在div块内，显示大小。

###### 子非浮动盒子

```css
		.right {
			width: 992px; /*1226-234, 原因里面2个盒子是连接紧密的*/
			height: 615px;
			background-color: skyblue;
		}
```

> 1. 可以不测量 原因里面2个盒子是连接紧密的
>
>    > ul块是普通块，所以会占用上面的left浮动位置，所以右边就没有显示完整
>
>    ![image-20211020120504283](http://myapp.img.mykernel.cn/image-20211020120504283.png)
>
> 由于不浮动，添加浮动
>
> ```diff
> 		.right {
> 			width: 992px; /*1226-234, 原因里面2个盒子是连接紧密的*/
> 			height: 615px;
> 			background-color: skyblue;
> 			float: right;
> 		}
> ```
>
> > <strong style="color: red">如果右侧不是计算的，也要确保左浮盒子和右浮盒子的宽度之和为父亲的宽度，否则行内块元素行为：当元素宽度大于剩余下的父元素宽度时，就会下一行显示</strong>
> >
> > 无论是左浮，就依次向下堆。右浮，本来就那么大的位置。放在右边刚刚占满
> >
> > ![image-20211020120639041](http://myapp.img.mykernel.cn/image-20211020120639041.png)

##### 准备基本外形

###### 写li结构

往里面放小`li`, 里面有8个小`li`

```html
		<ul class="right">
			<li></li>
		</ul>
```

> 没有必要写8个，先搞定1个。其它复制即可。

###### 测量li的大小

通过PS的标尺, 精确的测量单个大小 `234px x 300px`

![image-20211020143528284](http://myapp.img.mykernel.cn/image-20211020143528284.png)

```css
		.right > li {
			width: 234px;
			height: 300px;
			background-color: pink;
		}
```

> 可以看到单个li有了
>
> ![image-20211020143718825](http://myapp.img.mykernel.cn/image-20211020143718825.png)
>
> 但是li有小圆点, 是自己的列表样式，可以去掉, 注意，我们的作用域是整个网页。
>
> ```css
> 		li {
> 			list-style: none;
> 		}
> ```

###### 浮动li

先加结构

```html
			<li>1</li>
			<li>1</li>
			<li>1</li>
			<li>1</li>
			<li>1</li>
			<li>1</li>
			<li>1</li>
			<li>1</li>
```

> 网页会竖着显示，原因，li是块，所以会每行一个。
>
> ![image-20211020144214544](http://myapp.img.mykernel.cn/image-20211020144214544.png)

 加浮动，由于在盒子中的行内元素，遇到边界会自动换行

```diff
		.right > li {
+			float: left;
			width: 234px;
			height: 300px;
			background-color: pink;
		}
```

> ![image-20211020144313234](http://myapp.img.mykernel.cn/image-20211020144313234.png)

###### li左侧的宽度

![image-20211020144630901](http://myapp.img.mykernel.cn/image-20211020144630901.png)

测量出来是14px

```diff
		.right > li {
			float: left;
			width: 234px;
			height: 300px;
			background-color: pink;

+			margin-left: 14px;
		}
```

> ![image-20211020144739560](http://myapp.img.mykernel.cn/image-20211020144739560.png)
>
> 注意：不能大1px, F12调整margin-left为15px时, 右边放不下，就会掉下来
>
> ![image-20211022131235525](http://myapp.img.mykernel.cn/image-20211022131235525.png)

###### li底侧的边框

```diff
		.right > li {
			float: left;
			width: 234px;
			height: 300px;
			background-color: pink;

			margin-left: 14px;
+			margin-bottom: 14px;
		}
```

> PS测量后，同样是 14px
>
> ![image-20211022131613403](http://myapp.img.mykernel.cn/image-20211022131613403.png)

##### 准备内容

1. 图片
2. 盒子，名称
3. 盒子，描述
4. 盒子，价格

![image-20211022131753555](http://myapp.img.mykernel.cn/image-20211022131753555.png)



复制网页图片

```html
		<div class="left">
			<img src="https://cdn.cnbj1.fds.api.mi-img.com/mi-mall/c583f2edc613f1be20fa415910b13ce3.jpg?thumb=1&w=234&h=614" alt="">
		</div>
```

```css

		.left>img {
			width: 100%;
		}
```

> 无论网页真实的图片是多大，让图片大小和盒子宽度一样即可。图片一般不操作高度。
>
> - width: 234px 和父亲一样宽
> - width: 100% 和父亲一样宽
>
> ![image-20211022172541904](http://myapp.img.mykernel.cn/image-20211022172541904.png)
>
> 下面这条线，不管。

##### 注意

- 外层：普通流；内层：浮动流
- 浮动流的累加宽度不能大于普通流的宽度，如果大于就会自动换行。

### 案例扩展



#### 导航栏案例

![image-20211025144726106](http://myapp.img.mykernel.cn/image-20211025144726106.png)

##### 案例分析

是上下结构, 下面是浮动的小空隙

![image-20211025144901502](http://myapp.img.mykernel.cn/image-20211025144901502.png)

- 上面是banner 广告栏

  小米:![image-20211025145309504](http://myapp.img.mykernel.cn/image-20211025145309504.png)

  ![image-20211025145339273](http://myapp.img.mykernel.cn/image-20211025145339273.png)

- 下面是导航栏 nav

##### 准备盒子

###### 准备上面的盒子

`760X150`

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		*  {
			margin: 0px;
			padding: 0px;
		}
		.banner {
			width: 760px;
			height: 150px;
			background-color: pink;

			margin: auto;
		}
	</style>
</head>
<body>
	<!-- banner是广告 -->
	<div class="banner">
		
	</div>

</body>
</html>
```

> ![image-20211025145521769](http://myapp.img.mykernel.cn/image-20211025145521769.png)

丢图片

![banner](http://myapp.img.mykernel.cn/banner.jpg)

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		*  {
			margin: 0px;
			padding: 0px;
		}
		.banner {
			width: 760px;
			height: 150px;
			background-color: pink;

			margin: auto;
		}
	</style>
</head>
<body>
	<!-- banner是广告 -->
	<div class="banner">
		<img src="http://myapp.img.mykernel.cn/banner.jpg" alt="">
	</div>

</body>
</html>
```

###### 准备下面的盒子

和上面一样的宽度，只是高度`32`

![nav_bg](http://myapp.img.mykernel.cn/nav_bg.jpg)

```diff
<body>
	<!-- banner是广告 -->
	<div class="banner">
		<img src="http://myapp.img.mykernel.cn/banner.jpg" alt="">
	</div>
+	<!-- nav 导航栏 -->
+	<div class="nav">
		
+	</div>
</body>
</html>
```

css

```css
		.nav {
			width: 760px;
			height: 32px;
			background-color: pink;

			margin: 0 auto;
		}
```

添加背景图片

```diff
		.nav {
			width: 760px;
			height: 32px;
			background-color: pink;

			margin: 0 auto;
+			background: url(http://myapp.img.mykernel.cn/nav_bg.jpg) repeat-x;
		}
```

##### 准备外形

###### 结构

导航，实际中采用li+a的做法。

1. li + a 更清晰，排列的更整齐，一看就是列表。
2. 如果放一堆a, 搜索引擎就认为堆砌关键字，影响网站排名。li+a这样,a就隔开了。

```html
	<!-- nav 导航栏 以后重要的导航均使用li+a -->
	<div class="nav">
		<ul>
			<li><a href="#">网站首页</a></li>
			<li><a href="#">网站首页</a></li>
			<li><a href="#">网站首页</a></li>
			<li><a href="#">网站首页</a></li>
			<li><a href="#">网站首页</a></li>
			<li><a href="#">网站首页</a></li>
		</ul>
	</div>
```

下面把全网页的li的默认样式去掉, 所以直接定义li

```css
		li {
			list-style: none;
		}
```

> ![image-20211025160433054](http://myapp.img.mykernel.cn/image-20211025160433054.png)

###### 样子 li+a. 浮动给li, 大小给a.

<strong style="color: red" >下面要把字浮动起来, 是给li加浮动，还是给a加浮动？</strong>

> 因为a是行内元素，本身在1行之上。
>
> 数字是竖着显示，因为li是块元素，所以每行显示。

```css
		.nav li {
			float: left;
		}
```

> ![image-20211025160714221](http://myapp.img.mykernel.cn/image-20211025160714221.png)

鼠标经过（只能a实现）就整个盒子变色, 换了背景图片。而这个盒子肯定有宽度和高度，这个宽度和高度，是给a还是li？

> 给a的，经过a时，才有效果。

盒子大小，给a。 盒子大小不用测量了，因为有一张图片

![button2](http://myapp.img.mykernel.cn/button2.jpg)

`80x32`

```css
		.nav li a {
			width: 80px;
			height: 32px;
			background-color: pink;
		}
```

> a是行内元素，不能有大小
>
> ![image-20211025161315053](http://myapp.img.mykernel.cn/image-20211025161315053.png)
>
> 转换模式
>
> ```diff
> 		.nav li a {
> +			display: block;
> 			width: 80px;
> 			height: 32px;
> 			background-color: pink;
> 		}
> ```
>
> ![image-20211025161402476](http://myapp.img.mykernel.cn/image-20211025161402476.png)

加背景图片

![button1](http://myapp.img.mykernel.cn/button1.jpg)

```diff
		.nav li a {
			display: block;
			width: 80px;
			height: 32px;
+			background: url(http://myapp.img.mykernel.cn/button1.jpg);
		}
```

文字垂直居中, 黑色，没有下划线, 文字水平居中

```diff
		.nav li a {
			display: block;
			width: 80px;
			height: 32px;
			background: url(http://myapp.img.mykernel.cn/button1.jpg);
+			line-height: 32px;
+			text-align: center;
+			text-decoration: none;
+			color: rgb(64,81,10);
		}
```

> 注意：颜色是测量的 

鼠标经过a, 换背景

```css
		.nav li a:hover {
			background-image: url(http://myapp.img.mykernel.cn/button2.jpg);
		}
```

### 浮动的扩展

#### 浮动与父盒子关系

里面浮动的盒子，只能在父盒子的内容区域中。不会压住父盒子的padding和margin.

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.father {
			width: 500px;
			height: 500px;
			background-color: pink;
		}

		.son {
			width: 200px;
			height: 200px;
			background-color: purple;
		}
	</style>
</head>
<body>
	<div class="father">
		<div class="son"></div>
	</div>
	
</body>
</html>
```

> ![image-20211025162421489](http://myapp.img.mykernel.cn/image-20211025162421489.png)

给父亲添加border

```diff
		.father {
			width: 500px;
			height: 500px;
			background-color: pink;

+			border: 20px solid red;
		}
```

> ![image-20211025162506287](http://myapp.img.mykernel.cn/image-20211025162506287.png)
>
> 孩子在边框内
>
> 给孩子添加浮动, 一样在父盒子内容区域中
>
> ```diff
> 		.son {
> 			width: 200px;
> 			height: 200px;
> 			background-color: purple;
> +			float: left;
> 		}
> ```
>
> 浮动到右侧
>
> ```diff
> 		.son {
> 			width: 200px;
> 			height: 200px;
> 			background-color: purple;
> +			float: right;
> 		}
> ```
>
> ![image-20211025162648619](http://myapp.img.mykernel.cn/image-20211025162648619.png)

父亲有padding值

```diff
		.father {
			width: 500px;
			height: 500px;
			background-color: pink;

			border: 20px solid red;
+			padding: 20px;
		}
```

> ![image-20211025162741400](http://myapp.img.mykernel.cn/image-20211025162741400.png)
>
> 可以观察到儿子盒子浮动在padding之内，去掉儿子浮动, 一样在内边距内。
>
> ![image-20211025162823107](http://myapp.img.mykernel.cn/image-20211025162823107.png)

所以浮动元素需要考虑内边距和边框

#### 浮动与兄弟盒子关系

| damao  | ermao  | 结果                     |
| ------ | ------ | ------------------------ |
| 不浮动 | 不浮动 | 上下排列                 |
| 浮动   | 不浮动 | 浮动之前的位置被后者占据 |
| 浮动   | 浮动   | 行内排列块               |
|        |        |                          |
|        |        |                          |

##### 不浮动+不浮动 -> 上下排列

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.father {
			width: 500px;
			height: 500px;
			background-color: pink;
		}

		.damao {
			width: 150px;
			height: 150px;
			background-color: purple;
		}

		.ermao {
			width: 200px;
			height: 200px;
			background-color: skyblue;
		}
	</style>
</head>
<body>
	<div class="father">
		<div class="damao"></div>
		<div class="ermao"></div>
	</div>
</body>
</html>
```

> 父盒子中，有2个未加浮动的盒子
> ![image-20211025163148664](http://myapp.img.mykernel.cn/image-20211025163148664.png)

##### 浮动+不浮动 -> 浮动之前的位置被后者占据

```diff
		.damao {
			width: 150px;
			height: 150px;
			background-color: purple;
+			float: left;
		}

```

> ![image-20211025163629787](http://myapp.img.mykernel.cn/image-20211025163629787.png)

##### 浮动 + 浮动 -> 行内排列块

```diff
		.damao {
			width: 150px;
			height: 150px;
			background-color: purple;
			float: left;
		}

		.ermao {
			width: 200px;
			height: 200px;
			background-color: skyblue;
+			float:  left;
		}
```

> ![image-20211025163746451](http://myapp.img.mykernel.cn/image-20211025163746451.png)

##### 不浮动 + 浮动 -> 上下排列

```html
		.damao {
			width: 150px;
			height: 150px;
			background-color: purple;
+			/*float: left;*/
		}

		.ermao {
			width: 200px;
			height: 200px;
			background-color: skyblue;
			float:  left;
		}
```

> ![image-20211025163859719](http://myapp.img.mykernel.cn/image-20211025163859719.png)

##### 结论 单行排列技巧

浮动的元素，只会影响后面的元素。前面的元素标准流不受影响。

一个盒子内多个盒子：

- 想竖着排内部多个盒子，直接全部标准流。
- 想横着排内部多个盒子，直接全部浮动。

## 清除浮动

### 标准流`子`盒子会撑高父亲盒子

浮动核心：数据展示给用户，表格。 数据收集，表单。多个块元素单行显示，浮动。

很多情况下，父盒子不方便给高度

浮动盒子没有大小。

> 新闻页面中，子盒子中，有的页面文字少，有的页面文字多。所以不好给高度。

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		/*one父盒子不给高度，子盒子有多少内容就显示多少内容, 子盒子撑开父盒子比较合理*/
		.one {
			width: 500px;
			background-color: pink;
		}

		.damao {
			width: 200px;
			height: 200px;
			background-color: purple;
			/*float: left;*/
		}

		.ermao {
			width: 250px;
			height: 250px;
			background-color: skyblue;
			/*float:  left;*/
		}
		.two {
			width: 700px;
			height: 150px;
			background-color: #000;
		}
	</style>
</head>
<body>
	<div class="one">
		<div class="damao"></div>
		<div class="ermao"></div>
	</div>
	<div class="two">
		
	</div>
</body>
</html>
```

> 父亲不定义高度时，标准流的孩子会撑高父亲。
>
> ![image-20211025165235784](http://myapp.img.mykernel.cn/image-20211025165235784.png)
>
> 通过F12调整damao的高度时， 父亲的高度会随之变高
>
> ![image-20211025165330169](http://myapp.img.mykernel.cn/image-20211025165330169.png)

### 浮动引起的问题

#### 浮动盒子的父级没有高度, 与父亲同级的下面的盒子会跑到浮动上面去

当damao和ermao浮动时，不占位置 ，而父级没有高度

```diff
		/*one父盒子不给高度，子盒子有多少内容就显示多少内容, 子盒子撑高父盒子比较合理*/
		.one {
			width: 500px;
			background-color: pink;
		}

		.damao {
			width: 200px;
			height: 200px;
			background-color: purple;
+			float: left;
		}

		.ermao {
			width: 250px;
			height: 250px;
			background-color: skyblue;
+			float:  left;
		}
		.two {
			width: 700px;
			height: 150px;
			background-color: #000;
		}
```

> ![image-20211025165645825](http://myapp.img.mykernel.cn/image-20211025165645825.png)
>
> two盒子跑到下面去了，如果给了父亲高度，那就不能实现标准流子盒子自动撑高父盒子。

#### 多个浮动盒子，最后一个浮动盒子宽度超出父亲会自动转到一下行排列

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.father {
			background-color: pink;
			width: 300px;
			height: 700px;
		}

		.son1 {
			float: left;
			background-color: purple;
			width: 100px;
			height: 100px;
		}

		.son2 {
			float: left;
			background-color: orange;
			width: 100px;
			height: 100px;
		}

		.son3 {
			float: left;
			background-color: green;
			width: 700px;
			height: 100px;
		}
	</style>
</head>
<body>
	<div class="father">
		<div class="son1">1</div>
		<div class="son2">2</div>
		<div class="son3">3</div>
	</div>
</body>
</html>
```

![image-20211116202758380](http://myapp.img.mykernel.cn/image-20211116202758380.png)

son1, son2, son3浮动，会在一行，但是第1行剩下的位置的宽度装不下son3，浮动就会换下行。

而在标准流中，子盒子比父亲大，不会换行

### 父盒子中子盒子浮动时，清除浮动让父盒子自动撑高

浮动盒子没有高度，其父盒子也没有高度。

下面的块元素会占据上面的空间，如`4.3.2`就会出现问题。

所以使用清除浮动解决, 就根据孩子高度，父亲会自动撑高。

```css
选择器 { clear: 属性值; }
```

| 值    | 描述                             |
| ----- | -------------------------------- |
| left  | 清除左侧浮动影响                 |
| right | 清除右侧浮动影响                 |
| both  | 清除所有浮动影响，工作中经常使用 |

#### 额外标签法

劣势：添加很多空标签。w3c推荐的方法。

在浮动元素末尾添加空标签`<div style="clear:both"></div>`

```diff
		.one {
			width: 500px;
			background-color: pink;
+			border: 10px solid red;
		}
+		.clear {
+			clear: both;
+		}
	<div class="one">
		<div class="damao"></div>
		<div class="ermao"></div>
+		<div class="clear"></div>
	</div>
```

> ![image-20211025170905913](http://myapp.img.mykernel.cn/image-20211025170905913.png)
>
> 改变一个盒子高度时，父盒子也会自动撑高
>
> ![image-20211025170948526](http://myapp.img.mykernel.cn/image-20211025170948526.png)

#### 父亲添加overflow方法

给父级添加`overflow: none|scroll|hidden` 其中之一都可以实现

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		/*one父盒子不给高度，子盒子有多少内容就显示多少内容, 子盒子撑开父盒子比较合理*/
		.one {
			width: 500px;
			background-color: pink;
			border: 10px solid red;
+			overflow: hidden;
		}
		.damao {
			width: 200px;
			height: 200px;
			background-color: purple;
			float: left;
		}

		.ermao {
			width: 250px;
			height: 250px;
			background-color: skyblue;
			float:  left;

		}
		.two {
			width: 700px;
			height: 150px;
			background-color: #000;
		}

	</style>
</head>
<body>
	<div class="one">
		<div class="damao"></div>
		<div class="ermao"></div>
	</div>
	<div class="two">
		
	</div>
</body>
</html>
```

`overflow: hidden` 的局限性

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			border: 10px solid red;
			width: 100px;
			height: 100px;
		}
	</style>
</head>
<body>
	<div>
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
	</div>
</body>
</html>
```

现在加overflow

```diff
		div {
			border: 10px solid red;
			width: 100px;
			height: 100px;
+			overflow: hidden;
		}
```

> 多出来的部分剪切走了
>
> ![image-20211025172338886](http://myapp.img.mykernel.cn/image-20211025172338886.png)

现在auto

```diff
		div {
			border: 10px solid red;
			width: 100px;
			height: 100px;
+			overflow: auto;
		}
```

> 滚动条：垂直
>
> ![image-20211025172637049](http://myapp.img.mykernel.cn/image-20211025172637049.png)

现在scroll

```diff
		div {
			border: 10px solid red;
			width: 100px;
			height: 100px;
+			overflow: scroll;
		}
```

> 滚动条, 水平、垂直
>
> ![image-20211025172534891](http://myapp.img.mykernel.cn/image-20211025172534891.png)

无法显示溢出父元素的部分。

#### 使用after伪元素清除浮动

额外标签法的升级版

```css
		/*.随便写的字符:after 伪元素*/
		.clearfix:after {
			content: "";
			display: block;
			height: 0;
			visibility: hidden;
			clear: both;
		}
```

> 高版本浏览器支持
>
> `:after`伪元素，就会在此类匹配的盒子内，最后一个添加一个元素。

```css
		.clearfix {
			*zoom: 1;
		}
```

> 低版本浏览器. ie6/ie7清除浮动

父亲调用

```diff
+	<div class="clearfix">
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
		天王盖地虎，导师一米五。
	</div>
```

> ![image-20211025173125856](http://myapp.img.mykernel.cn/image-20211025173125856.png)
>
> 这个不行

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		/*one父盒子不给高度，子盒子有多少内容就显示多少内容, 子盒子撑开父盒子比较合理*/
+		.clearfix:after {
+			content: "";
+			display: block;
+			height: 0;
+			visibility: hidden;
+			clear: both;
+		}

+		.clearfix {
+			*zoom: 1;
+		}
		.one {
			width: 500px;
			background-color: pink;
			border: 10px solid red;
		}
		.damao {
			width: 200px;
			height: 200px;
			background-color: purple;
			float: left;
		}

		.ermao {
			width: 250px;
			height: 250px;
			background-color: skyblue;
			float:  left;

		}
		.two {
			width: 700px;
			height: 150px;
			background-color: #000;
		}

	</style>
</head>
<body>
+	<div class="one clearfix">
		<div class="damao"></div>
		<div class="ermao"></div>
	</div>
	<div class="two">
		
	</div>
</body>
</html>
```

> 现在浮动元素也能撑高父盒子了
>
> ![image-20211025173256023](http://myapp.img.mykernel.cn/image-20211025173256023.png)

方法比前2种更好

缺点：代码多、IE6/IE7额外写。

#### 使用双伪元素清除浮动

会使用就可以，代码复制即可

```css
		.clearfix:before, 
		.clearfix:after {
			content: "";
			display: table;
		}

		.clearfix:after {
			clear: both;
		}

		.clearfix {
			*zoom: 1;
		}
```

> 其它浏览器均不认识最后一句话

#### 清除浮动总结

什么时候清除浮动？

- <strong style="color: red;">父级没有高度，子级标准流，正常撑高父级。</strong>
- <strong style="color: red;">父级没有高度，子级是浮动，而且影响了下面的布局。</strong>

| 清除浮动         | 优点           | 缺点              |
| ---------------- | -------------- | ----------------- |
| 额外标签         | 简单           | 冗余标签          |
| overflow: hidden | 书写简单       | 文字会溢出        |
| after伪元素      | 结构语义化正确 | ie6-7不兼容:after |
| 驭伪元素         | 结构语义化正常 | ie6-7不兼容:after |



## ps切图

### 图片格式

- gif              256色，透明背景和动画效果

- jpeg(.jpg) 高清，产品图

- png   有gif, jpeg特点

- psd  ps专用格式, 拥有图层。 网页美工一般会给我们psd文件，方便我们切图。

  > 透明背景
  >
  > ![image-20211102170440296](http://myapp.img.mykernel.cn/image-20211102170440296.png)
  >
  > 切图工具
  >
  > ![image-20211102170844294](http://myapp.img.mykernel.cn/image-20211102170844294.png)

### 切图方法

- PS切片工具

- PS插件cutterman

### 切片工具

#### 普通切片

##### 选中图

###### 方式一

切片工具将图片划一个矩形

![image-20211102171116775](http://myapp.img.mykernel.cn/image-20211102171116775.png)

> 注意选中后，图片四周是虚线
>
> 切的图可以移动、调整大小。

###### 方式二 基于新建图层的切片  

 如果基于方式，我们怎么切出logo?

> 我们只能通过切片来选择logo, 通过手工划，最后的大小可能不一样
>
> ![image-20211102172510210](http://myapp.img.mykernel.cn/image-20211102172510210.png)

![image-20211102172633402](http://myapp.img.mykernel.cn/image-20211102172633402.png)

> 通过选择工具，选自动选择、基于图层来自动定位，
>
> > 自动选择即，鼠标点了哪里就自动选中哪个图层。也可以基于组
>
> 右下角就会自动出现相应的图层

现在选中图层

![image-20211102172759960](http://myapp.img.mykernel.cn/image-20211102172759960.png)

> 注意：图层被 选中背景是浅灰色

![image-20211102172844971](http://myapp.img.mykernel.cn/image-20211102172844971.png)

> 通过图层菜单，基于图层完成自动切片，现在logo就被 选择了
>
> ![image-20211102172942579](http://myapp.img.mykernel.cn/image-20211102172942579.png)

现在再通过4。4。3。1。2来导出GIF格式

![image-20211102222534792](http://myapp.img.mykernel.cn/image-20211102222534792.png)

> 注意：gif应该是透明背景，这里不是。

现在不想要背景，先隐藏背景

![image-20211102173248953](http://myapp.img.mykernel.cn/image-20211102173248953.png)

然后导出**jpeg, png， gif **3个图

![image-20211102173439638](http://myapp.img.mykernel.cn/image-20211102173439638.png)

> 可以发现gif和 png是透明背景的。
>
> 而jpeg背景不透明。



###### 方式三 辅助线切图

通过视图 - 标尺，从上方或左方拉辅助线，通过放大/缩小，手托的方式精确定位上下边

![image-20211102215542594](http://myapp.img.mykernel.cn/image-20211102215542594.png)

现在选择切片工具的属性，基于参考线的切片

![image-20211102220124107](http://myapp.img.mykernel.cn/image-20211102220124107.png)

> 注意，现在得到的是9个切片，现在想使用哪个切片，就使用切片选择工具，来点击哪个切片即可
>
> 现在通过切片选择工具，选中间的切片, 可以发现中间的框框变成黄色

现在通过4.4.3.1.2导出

![image-20211102220435621](http://myapp.img.mykernel.cn/image-20211102220435621.png)



##### 导出图

由于我们写的web, 所以就存储为web格式，专用给网页准备的格式

![image-20211102171519339](http://myapp.img.mykernel.cn/image-20211102171519339.png)



透明背景：gif, png

产品：   jpeg

![image-20211102171742716](http://myapp.img.mykernel.cn/image-20211102171742716.png)

选中的切片

![image-20211102171911570](http://myapp.img.mykernel.cn/image-20211102171911570.png)

##### 清理切片及辅助线

上面切片后，使用切片选择工具, 选中刚刚的切片，然后删除即可

![image-20211102172313753](http://myapp.img.mykernel.cn/image-20211102172313753.png)

切片太多了，通过视图 - 清除切片即可

> 辅助线：视图，清除辅助线

##### 切背景图

现在想要切一个文字的背景

![image-20211102220618963](http://myapp.img.mykernel.cn/image-20211102220618963.png)

当我们直接切片时

![image-20211102220744174](http://myapp.img.mykernel.cn/image-20211102220744174.png)

![image-20211102220821347](http://myapp.img.mykernel.cn/image-20211102220821347.png)

现在先找到文字，再隐藏，通过4.4.3.1.1.2方法定位到哪个图层，通过眼睛看是否隐藏来确定

![image-20211102220926281](http://myapp.img.mykernel.cn/image-20211102220926281.png)

再切图 

![image-20211102221024480](http://myapp.img.mykernel.cn/image-20211102221024480.png)

> 就算基于图层的切片，也需要先将文字隐藏

### ps插件 cutterman

> 官网：http://www.cutterman.cn/zh/cutterman
>
> 要求PS完整版，只要窗口的扩展功能，可以选中，表示可以安装插件。
>
> 首次需要登陆，之后没有网也不需要登陆了。



#### 基础使用

![image-20211102221715187](http://myapp.img.mykernel.cn/image-20211102221715187.png)

> 格式：web
>
> 输出位置

#### 基于图层切片 导出

传统需要图层下面的非透明背景隐藏，而使用cutterman则不需要。

选中logo, 不需要隐藏下方

![image-20211102222127131](http://myapp.img.mykernel.cn/image-20211102222127131.png)

> 1. 选择图层，定位到图层
> 2. 插件选web格式，同时选4个格式
> 3. 输出位置
> 4. 导出选中图层

![image-20211102222243241](http://myapp.img.mykernel.cn/image-20211102222243241.png)

> 传统的背景不隐藏的情况gif不会隐藏背景，而插件，可以不需要隐藏背景，切下来的图gif, png一定是透明背景。jpeg就是非透明背景。

#### 基于图层 导出

上面还需要基于图层新建切片，不需要切片一样可以导出

![image-20211102222938881](http://myapp.img.mykernel.cn/image-20211102222938881.png)

> 选中图层图片，直接导出即可

#### 获取文字后的背景图片

![image-20211102223310487](http://myapp.img.mykernel.cn/image-20211102223310487.png)

> 1. 只需要获取这个文字对应的背景框的图层，通过隐藏和显示和确定
> 2. 一键导出即可
>
> > 而在上面中，选中这个图层，直接导出是不可以的，传统只能导切片。
> >
> > 如果选中图层，生成切片，同样会把字也导出。

#### 导出选区图

![image-20211102223937458](http://myapp.img.mykernel.cn/image-20211102223937458.png)

## 今日总结

| 三种布局 | 特点                             |      |
| -------- | -------------------------------- | ---- |
| 标准流   | 行内：1行多个； 块元素：1行1个； |      |
| 浮动流   | 多个块单行展示                   |      |
| 定位     | 盒子定在浏览器某个位置           |      |

浮动: float表示，写在css属性第1个, 标准流中的浮动子盒子会有行内块特性，但是没有间隙，子盒子超界时，会自动换行。

> 早期：文字围绕图片
>
> 现在：块级盒子行内显示

| 特点 | 描述                                                         |
| ---- | ------------------------------------------------------------ |
| 浮   | 加了float属性的盒子，就漂浮在普通块/行内元素的上方           |
| 漏   | 加了float属性的盒子，它原来的位置会漏给标准流的盒子。块盒子会在其下，行内元素及行内块元素不会， |
| 特   | 加了float属性的盒子，会在上层拥有行内块的行为，除了多个浮动元素间隙不存在。 |

清除浮动：不指定父的高度，子盒子是标准流时，父级高度会自动撑高。但是一般子盒子是浮动流，所以需要以下几种方式来清除浮动，也达到父例子自动撑高的目的。

| 清除浮动         | 优点           | 缺点              |
| ---------------- | -------------- | ----------------- |
| 额外标签         | 简单           | 冗余标签          |
| overflow: hidden | 书写简单       | 文字会溢出        |
| after伪元素      | 结构语义化正确 | ie6-7不兼容:after |
| 驭伪元素         | 结构语义化正常 | ie6-7不兼容:after |

PS切图：传统有切片/辅助线切片/基于图层新建切片，切空背景时，需要隐藏背景。切导航需要隐藏文字。使用cutterman时，可以直接选中图层不需要基于图层新建切片/可以直接选中带文字的背景图层 / 可以基于选区切片。

## 作业

### 搜索趣图

![image-20211102225730556](http://myapp.img.mykernel.cn/image-20211102225730556.png)

#### 外层框

##### 内容大小、边框大小及border大小

测量，通过辅助线，测量到边框之外

![image-20211102231025458](http://myapp.img.mykernel.cn/image-20211102231025458.png)

> ![image-20211102231439685](http://myapp.img.mykernel.cn/image-20211102231439685.png)
>
> - 参考线选择好之后，切片属性的基于参考线切片
> - 切片选择工具，选择中间的切片，右键 切片选项
> - 得到的w240 h298
> - 放大图片之后 ，上面边框3像素, 左右下边框1像素
> - 所以w240-3, h298-3

实现网页

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.one {
			width: 237px;
			height: 295px;
			background: pink;
			border-top: 7px solid black;
			border-left: 1px solid black;
			border-right: 1px solid black;
			border-bottom: 1px solid black;
		}
	</style>
</head>
<body>
	<div class="one"></div>
</body>
</html>
```

> ![image-20211103140225302](http://myapp.img.mykernel.cn/image-20211103140225302.png)

##### 边框颜色 

获取底边框颜色: `d9e0ee`

![image-20211103140429758](http://myapp.img.mykernel.cn/image-20211103140429758.png)

> 1. 选择吸管工具
> 2. 吸管工具点底边框
> 3. 再点左侧前景色
> 4. 可以在弹出的拾色器窗口中复制下面的16进制的色

与以上方法类似，取得上边框的颜色 `ff8400`

> 1. space + 鼠标 点着图到上边框
> 2. 吸管取色

```html
		.one {
			width: 238px;
			height: 295px;
			/*background: pink;*/
			border-top: 2px solid #ff8400;
			border-left: 1px solid #d9e0ee;
			border-right: 1px solid #d9e0ee;
			border-bottom: 1px solid #d9e0ee;
		}
```

##### 去掉默认的边距，且放在中间

```css
		* {
			margin: 0px;
			padding: 0px;
		}
```

在.one中添加

```css
		.one {
			margin: 100px;
		}
```



#### 内层框

##### 预估框

通过粗略的查看

![image-20211103140949314](http://myapp.img.mykernel.cn/image-20211103140949314.png)

##### 内层框排列 在多行

可以引用标准流套标准流

```html
		.two {
			height: 79px;
			background-color: pink;
		}
		.three {
			height: 79px;
			background-color: purple;
		}
		.four {
			height: 79px;
			background-color: orange;
		}
```

> ![image-20211103141438159](http://myapp.img.mykernel.cn/image-20211103141438159.png)
>
> 我们可以看到里面3个盒子是上下排列，说明已经达到我们预期，现在只需要测试各个盒子的大小
>
> 里面3个盒子可以不需要宽度，将继承父亲的宽度

##### 最上边盒子的大小

视图 - 参考 线 ，像素。画好参考线

> 画的时候，要借助放大镜，准确的包括在边框之外
>
> ![image-20211103141937035](http://myapp.img.mykernel.cn/image-20211103141937035.png)

切片  - 基于参考线，切片选择工具，选中上面的切片

右键，查看选项

> `36px` 高度是包含下边框的，所以需要-1
>
> ![image-20211103142128274](http://myapp.img.mykernel.cn/image-20211103142128274.png)

```diff
		.two {
			height: 35px;
			border-bottom: 1px solid #d9e0ee;
		}
```

##### 中间图片的大小，先内容后内边框

图片，可以通过图片大小获取 `218x160`

> ![image-20211103142708759](http://myapp.img.mykernel.cn/image-20211103142708759.png)

```html
		.three {
			width: 218px;
			height: 160px;
			background-color: pink;
		}
```

测量图片到上边框的距离

> `7`像素，说明图片padding/margin可以是`7`像素
>
> ![image-20211103142946949](http://myapp.img.mykernel.cn/image-20211103142946949.png)

测量图片到左边框的距离

> `8`像素
>
> ![image-20211103143512025](http://myapp.img.mykernel.cn/image-20211103143512025.png)

根据稳定性，优先选择`width > padding > margin`

```html
		.three {
			width: 218px;
			height: 160px;
			background-color: pink;
			padding: 7px 8px;
		}
```

> 可以发现F12看到正常的，蹭的内容正好是图片的大小
>
> ![image-20211103144019745](http://myapp.img.mykernel.cn/image-20211103144019745.png)

现在引入图片

```diff
</head>
<body>
	<div class="one">
		<div class="two"></div>
		<div class="three">
+			<img src="./images/happy.gif" alt="">
		</div>
		<div class="four"></div>
	</div>
</body>
</html>
```

> ![image-20211103144202209](http://myapp.img.mykernel.cn/image-20211103144202209.png)

注意，我们只测试了左边，和上边，就只给左和上边距

```diff
	.three {
		width: 218px;
		height: 160px;
		/*background-color: pink;*/
+		padding: 7px 0px 0px 8px;
	}
```
##### 下面盒子大小

先加一个列表

```diff
<body>
	<div class="one">
		<div class="two"></div>
		<div class="three">
			<img src="./images/happy.gif" alt="">
		</div>
+		<ul class="four">
+			<li><a href="#">GIF: 小胖墩游泳被卡 被救后一脸无辜</a></li>
+			<li><a href="#">GIF: 小胖墩游泳被卡 被救后一脸无辜</a></li>
+			<li><a href="#">GIF: 小胖墩游泳被卡 被救后一脸无辜</a></li>
+		</ul>

	</div>
</body>
```

> ![image-20211103171321339](http://myapp.img.mykernel.cn/image-20211103171321339.png)

去掉样式

```css
		li {
			list-style: none;
		}
```

> ![image-20211103171521458](http://myapp.img.mykernel.cn/image-20211103171521458.png)

字跑出来了，需要将字完全放在框中。

> ps写相同的字，和原始字比较，发现12-13px比较合适
>
> F12调整，发现14px像素刚刚在内
>
> ![image-20211103171708079](http://myapp.img.mykernel.cn/image-20211103171708079.png)
>
> 现在选择13px

```html
		.four a {
			font-size: 13px;
		}
```

> ![image-20211103171956813](http://myapp.img.mykernel.cn/image-20211103171956813.png)

都是文字，测量行高，基线到基线

测量行高：基线到基线的距离

![image-20211103170655854](http://myapp.img.mykernel.cn/image-20211103170655854.png)

```html
		.four li {
			padding: 0px 18px 15px;
			line-height: 26px;
		}
```

> ![image-20211103172204283](http://myapp.img.mykernel.cn/image-20211103172204283.png)



测量文字到左边的

> ![image-20211103162054510](http://myapp.img.mykernel.cn/image-20211103162054510.png)

测量的左边18px

```diff
		.four a {
			font-size: 12px;
			line-height: 26px;
+			padding-left: 18px;
		}
```

现在文字没有下划线，颜色`666666` 加背景

```css
		.four a {
			font-size: 12px;
			line-height: 26px;
			padding-left: 18px;
+			text-decoration: none;
+			color: #666666;
+			background: url(./images/square.png) no-repeat 0 center;
		}
```

> ![image-20211103172929341](http://myapp.img.mykernel.cn/image-20211103172929341.png)

#### 填文字

##### 加标题

标题, PS测量是18px

```html
		<div class="two">
			<h2>搜索趣图</h2>
		</div>
```

```css
		.two h2 {
			font-size: 18px;
			font-weight: normal;
		}
```

> ![image-20211103173640305](http://myapp.img.mykernel.cn/image-20211103173640305.png)
>
> 可以发现文字在框框的上边，需要在中间，line-height=height
>
> ![image-20211103173826218](http://myapp.img.mykernel.cn/image-20211103173826218.png)

定义行高就是框框内容的高度，由于框的下面的边框1px

```diff
		.two h2 {
			font-size: 18px;
			font-weight: normal;
+			height: 35px;
+			line-height: 35px;
		}
```

> ![image-20211103174155060](http://myapp.img.mykernel.cn/image-20211103174155060.png)

ps测量左侧为12px

> ![image-20211103174135195](http://myapp.img.mykernel.cn/image-20211103174135195.png)

```diff
		.two h2 {
			font-size: 18px;
			font-weight: normal;
			height: 35px;
			line-height: 35px;
+			padding-left: 12px;
		}
```

> ![image-20211103174318538](http://myapp.img.mykernel.cn/image-20211103174318538.png)

##### 下边的背景需要右移

背景右移要么外层加padding, 内层加margin

![image-20211103174512867](http://myapp.img.mykernel.cn/image-20211103174512867.png)

为了方便，直接在内层加8个margin

```diff
		.four a {
			font-size: 12px;
			line-height: 26px;
			padding-left: 18px;
			text-decoration: none;
			color: #666666;
			background: url(./images/square.png) no-repeat 0 center;
+			margin: 0px 0px 0px 8px;
		}
```

> ![image-20211103174637478](http://myapp.img.mykernel.cn/image-20211103174637478.png)

##### 字需要向背景靠

现在明显a的padding多了，上面量的padding是字到左侧边框，而上面加了margin是背景边框，所以字到背景就是 18-8

```diff
		.four a {
			font-size: 12px;
			line-height: 26px;
+			padding-left: 10px;
			text-decoration: none;
			color: #666666;
			background: url(./images/square.png) no-repeat 0 center;
			margin: 0px 0px 0px 8px;
		}
```

> 背景左边是margin
>
> 背景右边到字是padding
>
> 字只需要line-height
>
> ![image-20211103175602395](http://myapp.img.mykernel.cn/image-20211103175602395.png)

##### 链接放上去需要显示颜色

![image-20211103175651415](http://myapp.img.mykernel.cn/image-20211103175651415.png)

![image-20211103175754374](http://myapp.img.mykernel.cn/image-20211103175754374.png)

现在可以写hover了

```css
		.four a:hover {
			text-decoration: underline;
			color: #ff8400;
		}
```

#### 最终代码

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		* {
			margin: 0px;
			padding: 0px;
		}
		.one {
			width: 238px;
			height: 295px;
			/*background: pink;*/
			border-top: 2px solid #ff8400;
			border-left: 1px solid #d9e0ee;
			border-right: 1px solid #d9e0ee;
			border-bottom: 1px solid #d9e0ee;
			margin: 100px;
		}

		.two {
			height: 35px;
			border-bottom: 1px solid #d9e0ee;
		}
		.three {
			width: 218px;
			height: 160px;
			/*background-color: pink;*/
			padding: 7px 0px 0px 8px;
		}

		li {
			list-style: none;
		}
		.four a {
			font-size: 12px;
			line-height: 26px;
			padding-left: 10px;
			text-decoration: none;
			color: #666666;
			background: url(./images/square.png) no-repeat 0 center;
			margin: 0px 0px 0px 8px;
		}
		.four a:hover {
			text-decoration: underline;
			color: #ff8400;
		}
		.two h2 {
			font-size: 18px;
			font-weight: normal;
			height: 35px;
			line-height: 35px;
			padding-left: 12px;
		}
	</style>
</head>
<body>
	<div class="one">
		<div class="two">
			<h2>搜索趣图</h2>
		</div>
		<div class="three">
			<img src="./images/happy.gif" alt="">
		</div>
		<ul class="four">
			<li><a href="#">GIF: 小胖墩游泳被卡 被救后一脸无辜</a></li>
			<li><a href="#">GIF: 小胖墩游泳被卡 被救后一脸无辜</a></li>
			<li><a href="#">GIF: 小胖墩游泳被卡 被救后一脸无辜</a></li>
		</ul>

	</div>
</body>
</html>
```

#### 优化

由于h2是块, 去div, 将所有two类修改成.one中的h2

```diff
	<div class="one">

+		<h2>搜索趣图</h2>

		<div class="three">
```

```diff
		.one h2 {
			height: 35px;
			border-bottom: 1px solid #d9e0ee;
			font-size: 18px;
			font-weight: normal;
			height: 35px;
			line-height: 35px;
			padding-left: 12px;
		}
```

由于img也是块，去div, 将three类修改为.one中的img

```diff
	<div class="one">

		<h2>搜索趣图</h2>

+		<img src="./images/happy.gif" alt="">
		<ul class="four">
			<li><a href="#">GIF: 小胖墩游泳被卡 被救后一脸无辜</a></li>
			<li><a href="#">GIF: 小胖墩游泳被卡 被救后一脸无辜</a></li>
			<li><a href="#">GIF: 小胖墩游泳被卡 被救后一脸无辜</a></li>
		</ul>

	</div>
```

```css
		.one img {
			width: 218px;
			height: 160px;
			/*background-color: pink;*/
			padding: 7px 0px 0px 8px;
		}
```

> ![image-20211103180820891](http://myapp.img.mykernel.cn/image-20211103180820891.png)
>
> > <strong style="color: red">注意：当图片单独成行，下面会自动加一个空隙</strong>

现在的h2字过大了

通过F12调整,h12大小为16px

#### 修复问题

本来PS的上边框是3px, 而我只写了2px, 所以小了1像素。

```css
		.one {
			width: 237px;
			height: 295px;
			/*background: pink;*/
			border-top: 3px solid #ff8400;
			border-left: 1px solid #d9e0ee;
			border-right: 1px solid #d9e0ee;
			border-bottom: 1px solid #d9e0ee;
			margin: 100px;
		}
```

> ![image-20211103181300253](http://myapp.img.mykernel.cn/image-20211103181300253.png)

原始案例

![image-20211103181325594](http://myapp.img.mykernel.cn/image-20211103181325594.png)

几乎一样了

#### 与老师代码对比

需要将内层的选择器加上外部的类定位 

```css
		.one .four a {
			font-size: 12px;
			line-height: 26px;
			padding-left: 10px;
			text-decoration: none;
			color: #666666;
			background: url(./images/square.png) no-repeat 0 center;
			margin: 0px 0px 0px 8px;
		}
		.one .four a:hover {
			text-decoration: underline;
			color: #ff8400;
		}
```

> .one打头

而老师写的

```diff
		.searchPic ul {
			margin-left: 8px;
		}
		.searchPic ul li{
			padding-left: 10px;
			height: 26px;
			line-height: 26px;
			background: url(images/square.png) no-repeat left center; /* 背景设置 */
		}
		.searchPic ul li a:hover {
			text-decoration: underline;  /* 添加下划线 */
			color: #ff8400;
		}
		.searchPic ul li a {
			font-size: 12px;
			color: #666;
			text-decoration: none; /* 取消下划线 */
		}

```

> ul 加的左外边框
>
> li上加的高度、行高、背景、左内边框
>
> a上边的。
>
> 相同颜色 可以缩写

### 新闻列表

![image-20211103205347316](http://myapp.img.mykernel.cn/image-20211103205347316.png)

> 1. 水平居中，上面有一点距离
>
> 2. 盒子有背景
>
> 3. 里面上下盒子
>
>    - 上盒子是标题，左侧有border , border左边还有一段距离
>
>    - 下盒子是列表，下盒子左边也有间距
>
>      > 说明外层盒子有padding

#### 准备父盒子

1. 保存图片，ps加载

2. 辅助线测量，宽度和高度，生成盒子。

   > ```css
   > border: 1px solid #009900
   > w262px
   > h329px
   > ```
   >
   > 基于参考线切图，切图选择工具，中间的图 `262x329`
   >
   > ![image-20211103210252478](http://myapp.img.mykernel.cn/image-20211103210252478.png)
   >
   > ![image-20211103210537783](http://myapp.img.mykernel.cn/image-20211103210537783.png)

3. 测量内边距，生成内边距。

   > ```css
   > padding: 10px
   > ```
   >
   > ![image-20211103210447301](http://myapp.img.mykernel.cn/image-20211103210447301.png)

4. 添加背景

5. 居中

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		* {
			margin: 0px;
			padding: 0px;
		}
		.out {
			border: 1px solid #009900;
			width: 240px;
			height: 307px;
			padding: 10px;
			background-color: pink;

			margin: 10px auto;
			background: url(./images/bg.gif);
		}
	</style>	
</head>
<body>
	<div class="out">
	</div>
</body>
</html>
```

> <strong style="color: red">注意：测量时，不要测量边框，避免最后忘记减！！！</strong>
>
> ![image-20211103213744361](http://myapp.img.mykernel.cn/image-20211103213744361.png)

#### 准备上面的标题盒子

1. 标题本身是块，不需要加盒子

   ```html
   	<div class="out">
   		<h1>爱宠知识</h1>
   	</div>
   ```

2. 爱宠知识, 颜色

   ```css
   #ffffff
   ```

3. 加左边框

   ```css
   border-left: 4px solid  #c9e143;
   ```

   ![image-20211103211758173](http://myapp.img.mykernel.cn/image-20211103211758173.png)

   ![image-20211103211843770](http://myapp.img.mykernel.cn/image-20211103211843770.png)

4. 测量文字左padding

   ```css
   padding-left: 6px;
   ```

   ![image-20211103212015535](http://myapp.img.mykernel.cn/image-20211103212015535.png)

5. 测量行高，让内容居中

   ```css
   line-height: 23px
   height: 23px
   ```

   ![image-20211103212253884](http://myapp.img.mykernel.cn/image-20211103212253884.png)

6. 测量文字大小

   ```css
   font-size: 20px
   ```

   ![image-20211103212444806](http://myapp.img.mykernel.cn/image-20211103212444806.png)

7. 写选择器，以外层为第一个选择器

```css
		.out h1 {
			color: #fff;
			border-left: 4px solid  #c9e143;
			padding-left: 6px;
			line-height: 23px; 
			height: 23px;
			font-size: 20px;
		}
```

> F12去掉背景看
>
> ![image-20211103214521190](http://myapp.img.mykernel.cn/image-20211103214521190.png)



#### 准备下面的列表盒子

1. 列表

   ```html
   <body>
   	<div class="out">
   		<h1>爱宠知识</h1>
   		<ul>
   			<li><a href="#">养狗比养猫对健康更有利</a></li>
   			<li><a href="#">日本正宗柴犬亮相，你怎么看柴犬</a></li>
   			<li><a href="#">狗狗歌曲《新年旺旺》</a></li>
   			<li><a href="#">带宠兜风，开车带宠需要注意什么</a></li>
   			<li><a href="#">【爆笑】这狗狗太不给力了</a></li>
   			<li><a href="#">狗狗与男童相同着装拍有爱造型照</a></li>
   			<li><a href="#">狗狗各个阶段健康大事件</a></li>
   			<li><a href="#">调皮宠物狗陷在沙发里的搞笑瞬间</a></li>
   			<li><a href="#">为什么每次大小便后，会有脚踢土？</a></li>
   		</ul>
   	</div>
   </body>
   </html>
   ```

   ![image-20211103215232816](http://myapp.img.mykernel.cn/image-20211103215232816.png)

2. 去掉li样式

   ```css
   		li {
   			list-style: none;
   		}
   ```

   ![image-20211103215324710](http://myapp.img.mykernel.cn/image-20211103215324710.png)

3. 字大小及颜色

   - 吸管
   - 写字比较

   ```css
   		.out .iner a {
   			text-decoration: none;
   			color: #0066cc;
   			font-size: 14px;
   		}
   ```

   ![image-20211103215716962](http://myapp.img.mykernel.cn/image-20211103215716962.png)

4. 加li背景

   ```css
   		.out .iner li {
   			background: url(./images/tb.gif) no-repeat 0 center;
   		}
   ```

   ![image-20211103215915555](http://myapp.img.mykernel.cn/image-20211103215915555.png)

5. 背景左边是margin

   ![image-20211103215950640](http://myapp.img.mykernel.cn/image-20211103215950640.png)

   

6. 背景右边到字是padding

   ![image-20211103220510824](http://myapp.img.mykernel.cn/image-20211103220510824.png)

7. 行高line-height

   ![image-20211103220221089](http://myapp.img.mykernel.cn/image-20211103220221089.png)

   ![image-20211103223103610](http://myapp.img.mykernel.cn/image-20211103223103610.png)

   8. li的下边框, 灰色
   9. 背景白色

```css
		.out .iner {
			background-color: white;
		}
		.out .iner li {
			background: url(./images/tb.gif) no-repeat 0 center;
			margin-left: 11px;
			padding-left: 15px;
			line-height: 31px;
			height: 31px;
			border-bottom: 1px dashed #666;

		}
```

![image-20211103221633147](http://myapp.img.mykernel.cn/image-20211103221633147.png)

#### 修复问题

字体太大，浏览器调整　 12px比较与案例一样

案例的下面框有上外边框，如果是内边框，ul的背景也会上去

![image-20211103222135609](http://myapp.img.mykernel.cn/image-20211103222135609.png)

```css
		.out .iner {
			background-color: white;
			margin-top: 5px;
		}
```

![image-20211103222534948](http://myapp.img.mykernel.cn/image-20211103222534948.png)



右侧下边框不对，说明右边还有一个margin

```css
			padding-right: 10px;
```

> ![image-20211103222633504](http://myapp.img.mykernel.cn/image-20211103222633504.png)

![image-20211103223125990](http://myapp.img.mykernel.cn/image-20211103223125990.png)

#### 对比老师的代码

行高30？字体？自己不能精确定位 

h1不需要行高

```css
/* me */
		.out .iner li {
			background: url(./images/tb.gif) no-repeat 0 center;
			margin-left: 11px;
			padding-left: 15px;
			line-height: 30px;
			height: 30px;
			border-bottom: 1px dashed #666;
			margin-right: 10px;
		}


/* pink */
.news ul{
	background:#FFF;
	margin-top:5px;
	padding:0 10px;
	}
.news li{
	border-bottom:#666 dashed 1px;
	list-style:none;
	background:url(images/tb.gif) no-repeat left;
	text-indent:1em; /* 16px */
}

```

a中

```css
/* me */
	.out .iner a {
			text-decoration: none;
			color: #0066cc;
			font-size: 12px;
		}


/* pink */
.news a{
	color:#06C;
	font-size:12px;
	text-decoration:none;
	line-height:30px;
	}

```

最后添加a悬浮时

```css
		.out a:hover{
			text-decoration:underline;
			color:#F00;
			}
```



### 鲁能

![QQ截图20211104191805](D:\黑马前端\QQ截图20211104191805.png)

#### 外框

##### 整体

![image-20211104195145684](http://myapp.img.mykernel.cn/image-20211104195145684.png)

左边`w20`

中间`w600`

右边`w20`

高度`h1103`

640x1103

##### 左边

测量边框内的原因：避免测量的结果还需要减边框

![image-20211104195231594](http://myapp.img.mykernel.cn/image-20211104195231594.png)

测量：这个下边框和左边的距离，因为下边框一定在内框的margin内。外框的padding内。

![image-20211104195316738](http://myapp.img.mykernel.cn/image-20211104195316738.png)

上边框颜色 `094683`

![image-20211104195609366](http://myapp.img.mykernel.cn/image-20211104195609366.png)

左边框颜色`bbbbbb`

![image-20211104195906923](http://myapp.img.mykernel.cn/image-20211104195906923.png)

##### 右边

同左边

##### 下边

边框/颜色同上

padding是35px

![image-20211104201910107](http://myapp.img.mykernel.cn/image-20211104201910107.png)

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		* {
			margin: 0px;
			padding: 0px;
		}
		.out {
			width: 640px;
			height: 1102px;
			/*四边统一1px*/
			border: 1px solid #bbbbbb; 
			/*上边重写*/
			border-top: 3px solid #094683;
			padding: 0px 20px 35px;

			background-color: pink;
			margin: 100px auto;
		}
	</style>
</head>
<body>
	<div class="out"></div>
</body>
</html>
```

> ![image-20211104202131209](http://myapp.img.mykernel.cn/image-20211104202131209.png)

#### 内框

##### 填充字

![image-20211105083657167](http://myapp.img.mykernel.cn/image-20211105083657167.png)

##### 布局上面

高度 82px, 外框的内边距是35px, 所以这个框：47px

![image-20211105084254604](http://myapp.img.mykernel.cn/image-20211105084254604.png)

##### 布局图片

图左边40, 离上边框 20

```css
		.out  img {
			margin: 20px 0px 0px 40px;
		}
```

![image-20211105094020640](http://myapp.img.mykernel.cn/image-20211105094020640.png)

##### 布局图片下面的文字

1. 块
2. 居中
3. 下面到文本的宽度

```css
		.out .three {
			text-align: center;
			margin-bottom: 21px;
		}
```

##### 下边整体

```css
		.out .bottom {
			padding-left: 20px;
		}
```



##### 布局段落

```css
		p {
			text-indent: 28px;
		}
```

![image-20211105114229025](http://myapp.img.mykernel.cn/image-20211105114229025.png)

到下内边距的距离

##### 布局框框

1. 边框、大小、颜色
2. 内边框：左、上、下、右
3. 到文本的外边距

```css
		.out .four {
			height: 89px;
			border: 1px dashed #bbb;
			margin: 15px 0px 29px;
			padding: 16px 24px 17px 10px;
		}
```



![image-20211105093841604](http://myapp.img.mykernel.cn/image-20211105093841604.png)

##### 布局下面的特殊的p

添加类`te`

```css
		.out .te {
			margin-top: 34px;
		}
```

##### 布局文章关键词

不能是段落，会缩进，所以就div

1. 到文本
2. 左边到外框的内边距

```css
		.out .five {
			padding-top: 21px;
		}
```



##### 布局最后一个框框

1. 上、左
2. 居中
3. 边框

```css
		.out .six {
			border: 1px solid #094683;
			text-align: center;
			margin-left: 21px;
			margin-top: 15px;
		}
```

![image-20211105114458294](http://myapp.img.mykernel.cn/image-20211105114458294.png)

##### 结果 

几乎一样了，就是字体大小不能确定

![image-20211105114850013](http://myapp.img.mykernel.cn/image-20211105114850013.png)

# CSS第5天

## 学成在线页面制作

### 目标

理解：

- 单页面开发流程
- css初始化语句
- css属性书写顺序

应用

- ps切图
- 引入外部样式表
- 把psd文件转换为html页面

目的：串联之前所有知识

### 准备素材

1. 学成在线PSD源文件，网页美工给的

> ![image-20211105124250054](http://myapp.img.mykernel.cn/image-20211105124250054.png)

2. 开发工具 = PS(切图、测量) + sublime(emmet代码) + chrome (测试)

### 准备工作

本次将结构与样式分离

1. 创建study目录。存放页面相关内容 图片，css, html
2. study/images 图片
3. study/index.html 主页
4. study/style.css 外链css
5. 样式引入html
6. 清除内外边框，看看是否成功。

```bash
install -dv study/images
touch study/{index.html,style.css}

```

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
+	<link rel="stylesheet" href="./style.css">
</head>
<body>
	123
</body>
</html>
```

![image-20211105124906550](http://myapp.img.mykernel.cn/image-20211105124906550.png)

> 有一个默认边距

`style.css`

```css
* {
	margin: 0px;
	padding: 0px;
}
```

> 现在的默认样式就没有了
>
> ![image-20211105125018914](http://myapp.img.mykernel.cn/image-20211105125018914.png)

### css属性书写顺序

遵循以下顺序

1. 布局定位: display/position/float/clear/visibility/overflow
2. 自身属性：width/height/margin/padding/border/background
3. 文本属性：color/font/text-decoration/text-align/vertical-align/white-space/break-word
4. 其他属性(css3): content/cursor/border-radius/box-shadow/text-shadow/background:linear-gradient ...

```css
.jdc {
    /* 1. 布局属性 */
	/*关系到盒子的模式：块、行、行内块*/
	display: block;
	/*定位 + 浮动 */
	position: relative;
	float: left;

	/* 2. 盒子模型自己*/
	width: 100px;
	height: 100px;
	margin: 0 10px;
	padding: 20px 0;

	/* 3. 文本属性*/
	font-family: Arial, 'Helvetica Neua', Helvetica, sans-serif;
	color: #333;
	
	/* 4. 其他属性 CSS3 */
	background-color: rgba(0, 0, 0, 0.5);
	border-radius: 10px; /* 圆角 */

}
```

### 布局流程

#### 版型

1. 确定版型，即可见的整个网页的宽度，左右对的非常整齐的部分

   ![image-20211107064454938](http://myapp.img.mykernel.cn/image-20211107064454938.png)

   2. 特殊的通览部分，即不需要宽度只需要高度

      ![image-20211107064728589](http://myapp.img.mykernel.cn/image-20211107064728589.png)

   3. 通览的版型，即便是通览区，里面的版型和第1个版型的宽度是一样的

      ![image-20211107064844180](http://myapp.img.mykernel.cn/image-20211107064844180.png)

	#### 行模块

页面布局是1行1行的，所以先确定行

![image-20211107065125593](http://myapp.img.mykernel.cn/image-20211107065125593.png)



#### 先结构后样式

##### 制作结构

##### css+div 完成布局



### 页面制作

#### 版心

![image-20211107065416858](http://myapp.img.mykernel.cn/image-20211107065416858.png)

```css
.w {
	width: 1200px;
	margin: auto;
}
```

#### 制作行

行模型：大概就是很多行，1行1行的做即可

##### 第1行 盒子

水平居中，上下外边距

![image-20211107070014615](http://myapp.img.mykernel.cn/image-20211107070014615.png)

先结构

###### 结构注释

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<link rel="stylesheet" href="./style.css">
</head>
<body>
	<!-- header头部模块开始 -->
	<div class="header">
		
	</div>
	<!-- header头部模块结束 -->
</body>
</html>
```

###### header高度

可以看到logo的高度，即可

![image-20211107070727063](http://myapp.img.mykernel.cn/image-20211107070727063.png)

![image-20211107070652550](http://myapp.img.mykernel.cn/image-20211107070652550.png)

所以宽度是版心的1200px, 高度是42px

###### 测量外边距

上下的距离 

![image-20211107070904805](http://myapp.img.mykernel.cn/image-20211107070904805.png)

![image-20211107070927838](http://myapp.img.mykernel.cn/image-20211107070927838.png)

上下30px

<strong style="color: red">先不管里面, 只管行的宽度和高度</strong>

###### 生成行的html和css (不管行内)

先加一个背景

```css
.header {
	width: 1200px;
	height: 42px;
	background-color: pink;
	margin: auto;
}
```

> ![image-20211107071240778](http://myapp.img.mykernel.cn/image-20211107071240778.png)
>
> 现在好了，接下来优化css

因为，我们定义了版心，宽度和margin, 所以可以直接引用

```diff
</head>
<body>
	<!-- header头部模块开始 -->
+	<div class="header w">
		
	</div>
	<!-- header头部模块结束 -->
</body>
</html>
```

```diff
.header {
+	/*width: 1200px;*/
	height: 42px;
	background-color: pink;
+	/*margin: auto;*/
}
```

> 现在刷新还是一样的结果
>
> ![image-20211107071240778](http://myapp.img.mykernel.cn/image-20211107071240778.png)

现在需要加外边距：`上下30px, 左右auto`

```diff
.header {
	/*width: 1200px;*/
	height: 42px;
	background-color: pink;
	/*margin: auto;*/
+	margin: 30px auto;
}
```

> 由于继承的auto, 现在需要重写，就是上下30px, 左右auto
>
> ![image-20211107071629547](http://myapp.img.mykernel.cn/image-20211107071629547.png)
>
> > 黄色条就是外边框

##### 第1行 内容

###### 分析盒子

分为5个盒子

1. 外层盒子
2. logo 标志
3. nav 导航: 无序列表
4. search 搜索框
5. user 用户盒子

> 第1个盒子是父盒子，2-5是子盒子，子盒子是并列关系

![image-20211107072011758](http://myapp.img.mykernel.cn/image-20211107072011758.png)

4个块同行：浮动

###### 切片切出logo + 背景

通过点眼睛确定此组为logo

![image-20211107072437713](http://myapp.img.mykernel.cn/image-20211107072437713.png)

切片尽量高度和测量的高度一样

![image-20211107072857118](http://myapp.img.mykernel.cn/image-20211107072857118.png)

文件 - 导出 - 导出web格式 png-24(png是没有背景), 导出选中的切片

![image-20211107073208477](http://myapp.img.mykernel.cn/image-20211107073208477.png)

![image-20211107073301224](http://myapp.img.mykernel.cn/image-20211107073301224.png)



修改图片名`logo.png`

放图片

```diff
	<!-- header头部模块开始 -->
	<div class="header w">
		<!-- .logo -->
		<div class="logo">
+			<img src="./images/logo.png" alt="">
		</div>
	</div>
	<!-- header头部模块结束 -->
```

> img本身是块，如果不加div包装，img默认会在下方生成一个高度。
>
> ![image-20211107073606097](http://myapp.img.mykernel.cn/image-20211107073606097.png)
>
> > 现在发现这个图的底色不是白色，需要修改背景

通过观察可以发现，整个网页的所有背景是一种颜色

![image-20211107073939887](http://myapp.img.mykernel.cn/image-20211107073939887.png)

通过吸管获取网页主色

![image-20211107074050301](http://myapp.img.mykernel.cn/image-20211107074050301.png)

```css
body {
	background-color: #f3f5f7;
}
```

> 现在logo与整个网页的背景一样了
>
> ![image-20211107074205689](http://myapp.img.mykernel.cn/image-20211107074205689.png)

###### 导航nav的无序列表 + 浮动 + 子盒子间隙 + 导航li间隙 + 链接高度/不给宽度?/内边距 + 鼠标经过链接的下边框 + 文字大小

- 结构

```diff
	<!-- header头部模块开始 -->
	<div class="header w">
		<!-- .logo -->
		<div class="logo">
			<img src="./images/logo.png" alt="">
		</div>

+		<!-- .nav -->
+		<div class="nav">
+			<!-- ul>li*3>a -->
+			<ul>
+				<li><a href="#">首页</a></li>
+				<li><a href="#">课程</a></li>
+				<li><a href="#">职业规划</a></li>
+			</ul>
+		</div>
	</div>
	<!-- header头部模块结束 -->
```

> 注意：上面的注释就是emmet语法快速生成的
>
> ![image-20211107074740456](http://myapp.img.mykernel.cn/image-20211107074740456.png)

- 样式

去掉列表样式

```css
* {
	margin: 0;
	padding: 0;
}
li {
	list-style: none;
}
```

> 这种代码每个页面都需要写一遍，就叫初始化
>
> ![image-20211107074934227](http://myapp.img.mykernel.cn/image-20211107074934227.png)

现在的logo和列表不在同一行，因为都是标准流，需要浮动

```css
.logo {
	float: left;
}
.nav {
	float: left;
}
```

> ![image-20211107075335996](http://myapp.img.mykernel.cn/image-20211107075335996.png)
>
> li本身是块元素，需要自上而下显示，li也需要同行显示 
>
> ```css
> .nav ul li {
> 	float: left;
> }
> ```
>
> ![image-20211107075548105](http://myapp.img.mykernel.cn/image-20211107075548105.png)

- 现在2个浮动：logo与导航之间有距离
  - logo: margin-right
  - nav: margin-left

![image-20211107075756385](http://myapp.img.mykernel.cn/image-20211107075756385.png)

> 测量的是到下边框的距离

```diff
.logo {
	float: left;
+	/*老师不知所以给的60px*/
+	margin-right: 70px;
}
```

> ![image-20211107075935105](http://myapp.img.mykernel.cn/image-20211107075935105.png)

- 导航中的a处理

只要导航栏的字不一样多，就不需要给宽度，通过padding值挤开即可

现在发现a有大小

![image-20211107080429415](http://myapp.img.mykernel.cn/image-20211107080429415.png)

- 测量高度

![image-20211107080528003](http://myapp.img.mykernel.cn/image-20211107080528003.png)

高40，加2px边框，正好高42

- a的内边距

![image-20211107080644052](http://myapp.img.mykernel.cn/image-20211107080644052.png)



![image-20211107081214650](http://myapp.img.mykernel.cn/image-20211107081214650.png)

- css

  ```css
  .nav ul li a {
  	display: block;
  	height: 40px;
  	line-height: 40px;
  	padding-left: 9px;
  	padding-right: 8px;
  }
  ```

- 鼠标经过a

取颜色

![image-20211107081453556](http://myapp.img.mykernel.cn/image-20211107081453556.png)

```css
.nav ul li a:hover {
	display: block;
	border-bottom: 2px solid #00a4ff;
}
```

> 现在鼠标经过就有下边框了
>
> ![image-20211107081626627](http://myapp.img.mykernel.cn/image-20211107081626627.png)
>
> ![image-20211107081725984](http://myapp.img.mykernel.cn/image-20211107081725984.png)
>
> > <strong style="color: red">注意：不同的字，下边框的长度不一样，因为我们没有给宽度，字有多长就多长，下边框也一样</strong>

- 文字大小

  ![image-20211107081934380](http://myapp.img.mykernel.cn/image-20211107081934380.png)

  > 1. 左侧的T
  > 2. 选中文字
  > 3. 上面写着18点，即18px

  颜色

  ![image-20211107082055736](http://myapp.img.mykernel.cn/image-20211107082055736.png)

  点开

  ![image-20211107084953055](http://myapp.img.mykernel.cn/image-20211107084953055.png)

  ```diff
  .nav ul li a {
  	display: block;
  	height: 40px;
  	line-height: 40px;
  	padding-left: 9px;
  	padding-right: 8px;
  	
  +	font-size: 18px;
  +	color: #050505;
  +	text-decoration: none;
  }
  
  ```

- 文字和文字的外边距

  ![image-20211107085324542](http://myapp.img.mykernel.cn/image-20211107085324542.png)

  44-8-9=27

  ```diff
  .nav ul li a {
  	display: block;
  	height: 40px;
  	line-height: 40px;
  	padding-left: 9px;
  	padding-right: 8px;
  +	margin-right: 27px;
  	font-size: 18px;
  	color: #050505;
  	text-decoration: none;
  }
  
  ```

  > ![image-20211107085707710](http://myapp.img.mykernel.cn/image-20211107085707710.png)

###### 搜索search -> 结构-外框样式-外框位置->内左样式(大小/颜色/内边距)->内右样式(背景, 平铺)->行内块有缝隙

![image-20211107085845328](http://myapp.img.mykernel.cn/image-20211107085845328.png)

- 结构

  ```html
  		<!-- .search -->
  		<div class="search">
  			<input type="text">
  			<!-- input:button -->
  			<!-- <input type="button" value=""> -->
  			<!-- input:button与button一样的 -->
  			<button>按钮</button>
  		</div>
  ```

  > ![image-20211107090054936](http://myapp.img.mykernel.cn/image-20211107090054936.png)
  >
  > > <strong style="color: red">为什么前2个浮动了，这个搜索框却不会在前面？</strong>
  > >
  > > ![image-20211107090242812](http://myapp.img.mykernel.cn/image-20211107090242812.png)
  > >
  > > 可以看出来，搜索框，其实也在下面，只是字不会被压住，所以在后面

- 外框样式

  先浮动搜索框

  ```diff
  .search {
  	float: left;
  }
  ```

  > 现在加浮动之后，浏览器看到的区域，就不一样了, 只在右边了
  >
  > ![image-20211107090452790](http://myapp.img.mykernel.cn/image-20211107090452790.png)

- 外框位置

  ![image-20211107090652309](http://myapp.img.mykernel.cn/image-20211107090652309.png)

  宽度95，而上面的字的右侧：内边距+外边距是<strong style="color: red">35px</strong>, 所以框子左边是60px

  ![image-20211107090800370](http://myapp.img.mykernel.cn/image-20211107090800370.png)

```diff
.search {
	float: left;
+	margin-left: 60px;
}
```

> ![image-20211107091006697](http://myapp.img.mykernel.cn/image-20211107091006697.png)

- 内部第1个文本Input的大小

  ![image-20211107091235542](http://myapp.img.mykernel.cn/image-20211107091235542.png)

  ![image-20211107091256059](http://myapp.img.mykernel.cn/image-20211107091256059.png)

  360x38

  ```css
  .search input {
  	width: 360px;
  	height: 38px;
  	border: 1px solid #00a4ff;
  }
  ```

  > ![image-20211107091640360](http://myapp.img.mykernel.cn/image-20211107091640360.png)
  >
  > > 由于，整体高度42，如果想下边框也能在下面，就简单调整高度为40px
  > >
  > > ```diff
  > > .search input {
  > > 	width: 360px;
  > > +	height: 40px;
  > > 	border: 1px solid #00a4ff;
  > > }
  > > ```
  > >
  > > ![image-20211107091824864](http://myapp.img.mykernel.cn/image-20211107091824864.png)

输入框里面有 提示 输入关键词, 通过input调整

![image-20211107092012972](http://myapp.img.mykernel.cn/image-20211107092012972.png)

```diff
		<!-- .search -->
		<div class="search">
+			<input type="text" value="输入关键词">
			<!-- input:button -->
			<!-- <input type="button" value=""> -->
			<!-- input:button与button一样的 -->
			<button>按钮</button>
		</div>
```

> ![image-20211107092113439](http://myapp.img.mykernel.cn/image-20211107092113439.png)

字的颜色浅

字的位置

![image-20211107092208207](http://myapp.img.mykernel.cn/image-20211107092208207.png)

```css
.search input {
	color: #ccc;
	/*padding-left在有宽度的盒子中会让整个盒子变宽，所以需要调整盒子宽度*/
	padding-left: 19px;
	width: calc(360px - 19px)
	height: 40px;
	border: 1px solid #00a4ff;
}
```

![image-20211107092923090](http://myapp.img.mykernel.cn/image-20211107092923090.png)

- 右侧样式

  1. 背景切图

     由于切的时候考虑左边的input有一个1px的边框，所以需要去掉这个边框

     ```diff
     .search input {
     	color: rgba(0, 0, 0, 0.5);
     	/*padding-left在有宽度的盒子中会让整个盒子变宽，所以需要调整盒子宽度*/
     	padding-left: 19px;
     	width: calc(360px - 19px);
     	height: 40px;
     	border: 1px solid #00a4ff;
     +	border-right: 0;
     }
     ```

     > ![image-20211107093448889](http://myapp.img.mykernel.cn/image-20211107093448889.png)

     ![image-20211107093635716](http://myapp.img.mykernel.cn/image-20211107093635716.png)

     > 注意：当我们选中这个切片，给的40px, 而我们将Input框给的40px+2px边框就是42的高度了。
     >
     > ![image-20211107093806816](http://myapp.img.mykernel.cn/image-20211107093806816.png)

  2. 给样式

     ```css
     .search button {
     	width: 50px;
     	height: 42px;
     	background-color: purple;
     }
     ```

     > 可以发现按钮自带边框
     >
     > ![image-20211107094036598](http://myapp.img.mykernel.cn/image-20211107094036598.png)
     >
     > 全局去掉button自带的边框
     >
     > ```css
     > button {
     > 	border: none;
     > }
     > ```
     >
     > ![image-20211107094614245](http://myapp.img.mykernel.cn/image-20211107094614245.png)

  现在加背景

  ```diff
  .search button {
  	width: 50px;
  	height: 42px;
  	background-color: purple;
  +	background: url(./images/btn.png) no-repeat;
  }
  ```

  > 因为图片是40像素，而我们的button高度是42，所以少了2px, 就平铺即可
  >
  > ```diff
  > .search button {
  > 	width: 50px;
  > 	height: 42px;
  > 	background-color: purple;
  > +		background: url(./images/btn.png) repeat-y;
  > }
  > ```
  >
  > ![image-20211107095115279](C:\Users\21923\AppData\Roaming\Typora\typora-user-images\image-20211107095115279.png)

现在有一个问题，行内块：图片/表单/单元格td,只要是行内块中间肯定有缝隙，浮动起来即可

```diff
.search input {
+	float: left;
	color: rgba(0, 0, 0, 0.5);
	/*padding-left在有宽度的盒子中会让整个盒子变宽，所以需要调整盒子宽度*/
	padding-left: 19px;
	width: calc(360px - 19px);
	height: 40px;
	border: 1px solid #00a4ff;
	border-right: 0;
}

.search button {
+	float: left;
	width: 50px;
	height: 42px;
	background-color: purple;
	background: url(./images/btn.png) repeat-y;
}
```

> ![image-20211107095639497](http://myapp.img.mykernel.cn/image-20211107095639497.png)

去掉按钮上的字

```diff
		<div class="search">
			<input type="text" value="输入关键词">
			<!-- input:button -->
			<!-- <input type="button" value=""> -->
			<!-- input:button与button一样的 -->
+			<button></button>
		</div>
```

> ![image-20211107095718275](http://myapp.img.mykernel.cn/image-20211107095718275.png)

###### user用户

![image-20211107100016506](http://myapp.img.mykernel.cn/image-20211107100016506.png)

1. 结构

   ```diff
   		<!-- .search -->
   		<div class="search">
   			<input type="text" value="输入关键词">
   			<!-- input:button -->
   			<!-- <input type="button" value=""> -->
   			<!-- input:button与button一样的 -->
   			<button></button>
   		</div>
   
   +		<!-- .user -->
   +		<div class="user">
   			
   +		</div>
   	</div>
   ```



2. 准备图片

   切出图片，导出web, jpeg高清，非常重要。导出仅当前选中切片

   ![image-20211107100418797](http://myapp.img.mykernel.cn/image-20211107100418797.png)



3. 完善结构

   ```diff
   		<!-- .user -->
   		<div class="user">
   +			<img src="./images/user.jpg" alt="">
   +			lilei-hanmm
   		</div>
   ```

   ![image-20211107100542210](http://myapp.img.mykernel.cn/image-20211107100542210.png)

   ![image-20211107100640445](http://myapp.img.mykernel.cn/image-20211107100640445.png)

4. 样式加浮动

   ```css
   
   .user {
   	float: left;
   }
   ```

5. 盒子属性

   ![image-20211107100844135](http://myapp.img.mykernel.cn/image-20211107100844135.png)

   ```diff
   .user {
   	float: left;
   +	margin-left: 30px;
   }
   ```

   ![image-20211107100927429](http://myapp.img.mykernel.cn/image-20211107100927429.png)

6. 文字属性

   字大小/颜色![image-20211107101042220](http://myapp.img.mykernel.cn/image-20211107101042220.png)

   ```diff
   .user {
   	float: left;
   	margin-left: 30px;
   +	font-size: 14px;
   +	color: #666;
   }
   ```

   文字垂直居中

   ```diff
   .user {
   	float: left;
   	margin-left: 30px;
   	font-size: 14px;
   	color: #666;
   +	height: 42px;
   +	line-height: 42px;
   }
   ```

   > height==line-height, 只是单行文本可以垂直居中，而这里还有图片，所以不能垂直居中，需要更高级的方法
   >
   > ![image-20211107101412943](http://myapp.img.mykernel.cn/image-20211107101412943.png)
   >
   > 对比案例
   >
   > ![image-20211107101450734](http://myapp.img.mykernel.cn/image-20211107101450734.png)

###### 测试

去掉header的背景

![image-20211107101633221](http://myapp.img.mykernel.cn/image-20211107101633221.png)

##### 第2行盒子

###### 分析盒子  注释



1. 通览：banner不给宽度，蓝色背景
2. 版心：水平居中
3. 3号框子是侧导航：subnav
4. 4号是课程 course

![image-20211107132014895](http://myapp.img.mykernel.cn/image-20211107132014895.png)

![image-20211107132241383](http://myapp.img.mykernel.cn/image-20211107132241383.png)

```html
	<!-- banner部分开发 -->
	<div class="banner"></div>
	<!-- banner部分结束 -->

</body>
```

![image-20211107132902821](http://myapp.img.mykernel.cn/image-20211107132902821.png)

```css
/*banner 开始*/

.banner {
	height: 421px;
}
```

###### 添加背景 + cutterman导出背景图

- 背景

现在通过PS分析

![image-20211107133541199](http://myapp.img.mykernel.cn/image-20211107133541199.png)



![image-20211107133600128](http://myapp.img.mykernel.cn/image-20211107133600128.png)

> 隐藏banner时，可以发现图片消失了，说明后面是背景，不是一张大图

吸取颜色

![image-20211107133239861](http://myapp.img.mykernel.cn/image-20211107133239861.png)

```diff
.banner {
	height: 421px;
+	background-color: #1c036c;
}
```

> ![image-20211107133352432](http://myapp.img.mykernel.cn/image-20211107133352432.png)

- cutterman导出背景图

  因为，我们如果选背景图，生成的切片，导出是这个样子，只要背景banner, 就需要隐藏上层才切片，麻烦

  ![image-20211107133843305](http://myapp.img.mykernel.cn/image-20211107133843305.png)

  选中背景使用cutterman导出，使用JPEG，banner更清晰 

  ![image-20211107134141873](http://myapp.img.mykernel.cn/image-20211107134141873.png)

  ![image-20211107134156834](http://myapp.img.mykernel.cn/image-20211107134156834.png)

  ```diff
  /*banner 开始*/
  
  .banner {
  	height: 421px;
  +	background: #1c036c url(./images/banner2.jpg);
  }
  ```

  > ![image-20211107134319112](http://myapp.img.mykernel.cn/image-20211107134319112.png)
  >
  > 平铺了

  取消平铺

  ```diff
  .banner {
  	height: 421px;
  +	background: #1c036c url(./images/banner2.jpg) no-repeat;
  }
  ```

  > ![image-20211107134409236](http://myapp.img.mykernel.cn/image-20211107134409236.png)
  >
  > 背景图片需要居中

  ```diff
  .banner {
  	height: 421px;
  +	background: #1c036c url(./images/banner2.jpg) no-repeat center center;
  }
  ```

  > ![image-20211107134520880](http://myapp.img.mykernel.cn/image-20211107134520880.png)

##### 第2行左侧盒子

######  版心 

```html
	<!-- banner部分开发 -->
	<div class="banner">
		123
	</div>
	<!-- banner部分结束 -->
```

> ![image-20211107134732660](http://myapp.img.mykernel.cn/image-20211107134732660.png)
>
> 123的位置在第2行的左上角了，应该在中间啊，这个怎么从中间开始？
>
> 版心就是在中间

```diff
	<!-- banner部分开发 -->
	<div class="banner">
+		<div class="w">
+			123
+		</div>
	</div>
	<!-- banner部分结束 -->

```

> 现在123就在中间了
>
> ![image-20211107135013462](http://myapp.img.mykernel.cn/image-20211107135013462.png)

###### 左侧盒子

高度同banner高度，只需要知道宽度

背景同父亲的背景

![image-20211107135245392](http://myapp.img.mykernel.cn/image-20211107135245392.png)

```diff
	<!-- banner部分开发 -->
	<div class="banner">
		<div class="w">
+			<!-- 左侧 subnav -->
+			<div class="subnav">
				
+			</div>
		</div>
	</div>
	<!-- banner部分结束 -->
```

```css
.subnav {
	width: 190px;
	height: 421px;
	background-color:  pink;
}
```

> ![image-20211107135629194](http://myapp.img.mykernel.cn/image-20211107135629194.png)
>
> 现在有了，但是这个球被遮住了，需要透明
>
> ![image-20211107135706026](http://myapp.img.mykernel.cn/image-20211107135706026.png)

```diff
.subnav {
	width: 190px;
	height: 421px;
+	/*黑色半透明*/
+	background-color:  rgba(0, 0, 0, 0.5);
}
```

> ![image-20211107135908998](http://myapp.img.mykernel.cn/image-20211107135908998.png)

现在导航的样子是竖着的，而li标签也是竖着的，默认样式即可

###### 导航的文字

现在就是这个文字的位置

1. 文字的起始位置，外盒子加padding, 内元素加margin
2. 行与行的文字高度，行高

![image-20211107140218450](http://myapp.img.mykernel.cn/image-20211107140218450.png)

![image-20211107140325479](http://myapp.img.mykernel.cn/image-20211107140325479.png)



```diff
.subnav {
	width: 190px;
	height: 421px;
	/*黑色半透明*/
	background-color:  rgba(0, 0, 0, 0.5);
+	padding-left: 20px;
}

```

```diff
	<!-- banner部分开发 -->
	<div class="banner">
		<div class="w">
			<!-- 左侧 subnav -->
			<div class="subnav">
+				123
			</div>
		</div>
	</div>
	<!-- banner部分结束 -->
```

> ![image-20211107140617288](http://myapp.img.mykernel.cn/image-20211107140617288.png)
>
> 注意：123的位置过来了，但是框框变大了，原因是padding
>
> ```diff
> .subnav {
> +	width: calc(190px - 20px);
> 	height: 421px;
> 	/*黑色半透明*/
> 	background-color:  rgba(0, 0, 0, 0.5);
> 	padding-left: 20px;
> 	line-height: 44px;
> }
> ```
>
> > 注意：这个宽度写calc时，需要在里面`-`左右加空格

##### 第2行里面左侧内容

###### 分析盒子

导航中，一般是li包装a, 所以结构是

```diff
	<!-- banner部分开发 -->
	<div class="banner">
		<div class="w">
			<!-- 左侧 subnav -->
			<div class="subnav">
+				<ul>
					<li><a href="#">前端开发</a></li>
					<li><a href="#">前端开发</a></li>
					<li><a href="#">前端开发</a></li>
					<li><a href="#">前端开发</a></li>
					<li><a href="#">前端开发</a></li>
					<li><a href="#">前端开发</a></li>
					<li><a href="#">前端开发</a></li>
					<li><a href="#">前端开发</a></li>
					<li><a href="#">前端开发</a></li>
+				</ul>
			</div>
		</div>
	</div>
	<!-- banner部分结束 -->
```

> ![image-20211107141253344](http://myapp.img.mykernel.cn/image-20211107141253344.png)

###### 加>标签

通过让>向右浮动

```html
				<ul>
					<li><a href="#">前端开发 <span>></span></a></li>
					<li><a href="#">前端开发 <span>></span></a></li>
					<li><a href="#">前端开发 <span>></span></a></li>
					<li><a href="#">前端开发 <span>></span></a></li>
					<li><a href="#">前端开发 <span>></span></a></li>
					<li><a href="#">前端开发 <span>></span></a></li>
					<li><a href="#">前端开发 <span>></span></a></li>
					<li><a href="#">前端开发 <span>></span></a></li>
					<li><a href="#">前端开发 <span>></span></a></li>
				</ul>
```

```css
.subnav span {
	float: right;
}
```

> ![image-20211107141556185](http://myapp.img.mykernel.cn/image-20211107141556185.png)
>
> 注意：>跑到最右边了，所以外层盒子，需要加一个右边的内边距
>
> ```diff
> .subnav {
> +	width: calc(190px - 40px);
> 	height: 421px;
> 	/*黑色半透明*/
> 	background-color:  rgba(0, 0, 0, 0.5);
> +	/*padding-left: 20px;*/
> +	padding: 0px 20px;
> 	line-height: 44px;
> }
> 
> ```
>
> > ![image-20211107141738290](http://myapp.img.mykernel.cn/image-20211107141738290.png)
>
> 

###### 导航行高

行与行的高度，就是li标签就是行

![image-20211107140348032](http://myapp.img.mykernel.cn/image-20211107140348032.png)

```css
.subnav li {
	height: 44px;
	line-height: 44px;
}
```

![image-20211107142058944](http://myapp.img.mykernel.cn/image-20211107142058944.png)

###### 文字的样式

大小/颜色

![image-20211107142209249](http://myapp.img.mykernel.cn/image-20211107142209249.png)

![image-20211107142227862](http://myapp.img.mykernel.cn/image-20211107142227862.png)

```css
.subnav a {
	font-size: 14px;
	color: #fff;
	text-decoration: none;
}

.subnav a:hover {
	color:  #00b4ff;
}
```

> ![image-20211107142504023](http://myapp.img.mykernel.cn/image-20211107142504023.png)

###### 修正文字

```diff
				<ul>
					<li><a href="#">前端开发 <span>></span></a></li>
					<li><a href="#">后端开发 <span>></span></a></li>
					<li><a href="#">移动开发 <span>></span></a></li>
					<li><a href="#">人工智能 <span>></span></a></li>
					<li><a href="#">商业预测 <span>></span></a></li>
					<li><a href="#">云计算&大数据 <span>></span></a></li>
					<li><a href="#">运维&测试 <span>></span></a></li>
					<li><a href="#">UI设计 <span>></span></a></li>
					<li><a href="#">产品 <span>></span></a></li>
				</ul>
```

> ![image-20211107142725201](http://myapp.img.mykernel.cn/image-20211107142725201.png)

##### 第2行里面右侧盒子

###### 分析盒子

1. 外层盒子：宽/高/颜色
2. 头部盒子
3. 下面的盒子

![image-20211107144750367](http://myapp.img.mykernel.cn/image-20211107144750367.png)

###### 外层盒子

宽度/高度/背景

```diff
		<div class="w">
			<!-- 左侧 subnav -->
			<div class="subnav">
				<ul>
					<li><a href="#">前端开发 <span>></span></a></li>
					<li><a href="#">后端开发 <span>></span></a></li>
					<li><a href="#">移动开发 <span>></span></a></li>
					<li><a href="#">人工智能 <span>></span></a></li>
					<li><a href="#">商业预测 <span>></span></a></li>
					<li><a href="#">云计算&大数据 <span>></span></a></li>
					<li><a href="#">运维&测试 <span>></span></a></li>
					<li><a href="#">UI设计 <span>></span></a></li>
					<li><a href="#">产品 <span>></span></a></li>
				</ul>
			</div>

+			<!-- 右侧 course -->
+			<div class="course">
+				123
+			</div>
		</div>
```

> 注意：层次不能错

给宽/高，通过PS测量

![image-20211110200325820](http://myapp.img.mykernel.cn/image-20211110200325820.png)

```css
.course {
	width: 228px;
	height: 300px;

	background-color: pink;
}
```

> ![image-20211110200301890](http://myapp.img.mykernel.cn/image-20211110200301890.png)

首先给宽度和高度和背景, 看看位置对不对。

现在这个位置应该在banner里面的右边，而目前在下面，原因是里面左边的subnav是标准流，所以兄弟标准流自然在下面了。

要想也上去，subnav应该是浮动流，当前的course也是浮动流

```diff
.subnav {
+	float: left;
	width: calc(190px - 40px);
	height: 421px;
	/*黑色半透明*/
	background-color:  rgba(0, 0, 0, 0.5);
	/*padding-left: 20px;*/
	padding: 0px 20px;
}
```

```diff
.course {
+	float: right;
	width: 228px;
	height: 300px;

	background-color: pink;
}
```

![image-20211110200656007](http://myapp.img.mykernel.cn/image-20211110200656007.png)

现在加上面的高度

![image-20211107143439609](http://myapp.img.mykernel.cn/image-20211107143439609.png)

```diff
.course {
	float: right;
	width: 228px;
	height: 300px;
+	/*本应该 .w 父盒子中的标准流子盒子定义上面的外边距时，父盒子 .w 会向下掉。*/
+	/*但是子盒子 .course 浮动了，就不存在这样的情况了; 定位也不存在掉落*/
+	margin-top: 50px;

	background-color: pink;
}
```

> ![image-20211107144317872](http://myapp.img.mykernel.cn/image-20211107144317872.png)

##### 第2行右侧内容

###### 分析盒子

![image-20211110201654526](http://myapp.img.mykernel.cn/image-20211110201654526.png)

- 上盒子 course-hd(head 是头部的)
- 下盒子 course-bd(body 是主体的简写)

里面的2个子盒子上下结构，块级元素。

块元素不给宽度时，默认的宽度同父亲，所以只需要给高度

###### 头部盒子

![image-20211110201926437](http://myapp.img.mykernel.cn/image-20211110201926437.png)

吸色: `#9bceea`

结构

```diff
	<div class="banner">
		<div class="w">
			<!-- 左侧 subnav -->
			<div class="subnav">
				<ul>
					<li><a href="#">前端开发 <span>></span></a></li>
					<li><a href="#">后端开发 <span>></span></a></li>
					<li><a href="#">移动开发 <span>></span></a></li>
					<li><a href="#">人工智能 <span>></span></a></li>
					<li><a href="#">商业预测 <span>></span></a></li>
					<li><a href="#">云计算&大数据 <span>></span></a></li>
					<li><a href="#">运维&测试 <span>></span></a></li>
					<li><a href="#">UI设计 <span>></span></a></li>
					<li><a href="#">产品 <span>></span></a></li>
				</ul>
			</div>

			<!-- 右侧 course -->
			<div class="course">
+				<div class="course-hd"></div>
			</div>
		</div>
	</div>
```

样式

1. 不给宽度，默认同父亲
2. 给高度
3. 给背景

```css
.course-hd {
	height: 48px;
	background-color: #9bceea;
}
```

![image-20211110202152199](http://myapp.img.mykernel.cn/image-20211110202152199.png)

给内容·

```diff
			<div class="course">
+				<div class="course-hd">我的课程表</div>
			</div>
```

> ![image-20211110202244823](http://myapp.img.mykernel.cn/image-20211110202244823.png)
>
> 垂直居中/水平居中
>
> 字号变大
>
> 颜色
>
> 加粗
>
> ![image-20211110202345829](http://myapp.img.mykernel.cn/image-20211110202345829.png)

```css
.course-hd {
	height: 48px;

	line-height: 48px;
	text-align: center;
	font-weight: 700;
	/*bold相当于400*/
	/*font-weight: bold;*/
	font-size: 18px;
	color: #fff;

	background-color: #9bceea;
}
```

###### 下面盒子分析及结构

多个块级盒子，先不写块，先写结构，分析

```diff
			<!-- 右侧 course -->
			<div class="course">
				<div class="course-hd">我的课程表</div>
+				<div class="course-bd">123</div>
			</div>
```

> ![image-20211110202939836](http://myapp.img.mykernel.cn/image-20211110202939836.png)

![image-20211110203104976](http://myapp.img.mykernel.cn/image-20211110203104976.png)



1. 左右有一个宽度
2. 使用course-bd一个padding值，一般左右是相等的
3. 不需要给内部元素margin值
4. course-bd给了padding值之后，不需要单心course会被撑开，因为course-bd这个块不会指定宽度，会自动根据padding/margin/border计算自己的内容宽度

测量左侧的宽度

![image-20211110203326876](http://myapp.img.mykernel.cn/image-20211110203326876.png) 

```css
.course-bd {
	padding: 0px 15px;
}
```

> ![image-20211110203434989](http://myapp.img.mykernel.cn/image-20211110203434989.png)
>
> 123已经右移了，但是不会撑开盒子

###### 下面盒子中的结构

![image-20211110203825555](http://myapp.img.mykernel.cn/image-20211110203825555.png)

1. 上面非常整齐，说明是ul, li

2. 而下面就是一个a标签

3. 3个li中的2行字不一样，放不同盒子

4. 3个li是2行，就不能(`text-align: center`, 仅适用于单行文本)垂直居中。

   因为文字默认在左上角，所以我们直接调整li的大小，刚刚好让文字在左上角即可，上面使用margin-top

   ![image-20211110204842118](http://myapp.img.mykernel.cn/image-20211110204842118.png)

###### 写结构

```diff
			<div class="course">
				<div class="course-hd">我的课程表</div>
				<div class="course-bd">
+					<ul>
+						<li><h4>继续学习 程序语言设计</h4><p>正在学习-使用对象</p></li>
+						<li><h4>继续学习 程序语言设计</h4><p>正在学习-使用对象</p></li>
+						<li><h4>继续学习 程序语言设计</h4><p>正在学习-使用对象</p></li>
+					</ul>
				</div>
			</div>
```

> ![image-20211110205355210](http://myapp.img.mykernel.cn/image-20211110205355210.png)

###### 测量 li

![image-20211110205549601](http://myapp.img.mykernel.cn/image-20211110205549601.png)

192X47

```css
.course-bd li {
	width: 192px;
	height: 47px;
	border-bottom: 1px solid #ccc;
}
```

> ![image-20211110205845013](http://myapp.img.mykernel.cn/image-20211110205845013.png)

###### 测量li与li之间

![image-20211110205934295](http://myapp.img.mykernel.cn/image-20211110205934295.png)

```diff
.course-bd li {
    /* 行高等于高度 只能让单行文本垂直居中 */
	width: 192px;
	height: 47px;
	border-bottom: 1px solid #ccc;
+	margin-top: 13px;	
}
```

![image-20211110210024265](http://myapp.img.mykernel.cn/image-20211110210024265.png)

###### 测量h4文字

![image-20211110210237085](http://myapp.img.mykernel.cn/image-20211110210237085.png)

```css
.course-bd li h4 {
	font-size: 16px;
	color: #4e4e4e;
}
```

![image-20211110210612955](http://myapp.img.mykernel.cn/image-20211110210612955.png)

###### 测量p文字

![image-20211110210455125](http://myapp.img.mykernel.cn/image-20211110210455125.png)

```css
.course-bd li p {
	font-size: 12px;
	color: #a5a5a5;
}
```

![image-20211110210856352](http://myapp.img.mykernel.cn/image-20211110210856352.png)

###### 解决最上面的li与上面的宽度不够

1. ul上面加10px， **外边距塌陷**

   ul和li都是标准流，li的margin-top和ul重合了

   ![image-20211110211256303](http://myapp.img.mykernel.cn/image-20211110211256303.png)

2. ul指定padding, 不会塌陷

   ```css
   .course-bd ul {
       padding-top: 10px;
   }
   ```

    ![image-20211110211736200](http://myapp.img.mykernel.cn/image-20211110211736200.png)

###### 写链接

因为`.course-bd`有内边距，所以这个链接的位置刚刚好

1. 就不给宽度，只需要高度
2. border 1

![image-20211111194555099](http://myapp.img.mykernel.cn/image-20211111194555099.png)

```diff
	<div class="banner">
		<div class="w">
			<!-- 左侧 subnav -->
			<div class="subnav">
				<ul>
					<li><a href="#">前端开发 <span>></span></a></li>
					<li><a href="#">后端开发 <span>></span></a></li>
					<li><a href="#">移动开发 <span>></span></a></li>
					<li><a href="#">人工智能 <span>></span></a></li>
					<li><a href="#">商业预测 <span>></span></a></li>
					<li><a href="#">云计算&大数据 <span>></span></a></li>
					<li><a href="#">运维&测试 <span>></span></a></li>
					<li><a href="#">UI设计 <span>></span></a></li>
					<li><a href="#">产品 <span>></span></a></li>
				</ul>
			</div>

			<!-- 右侧 course -->
			<div class="course">
				<div class="course-hd">我的课程表</div>
				<div class="course-bd">
					<ul>
						<li><h4>继续学习 程序语言设计</h4><p>正在学习-使用对象</p></li>
						<li><h4>继续学习 程序语言设计</h4><p>正在学习-使用对象</p></li>
						<li><h4>继续学习 程序语言设计</h4><p>正在学习-使用对象</p></li>
					</ul>

+					<a href="#" class="all">全部课程</a>
				</div>
			</div>
		</div>
	</div>
```

添加css

```css
.all {
	display: block;
	height: 38px;
	border: 1px solid #00a4ff;
}
```

> ![image-20211111194911688](http://myapp.img.mykernel.cn/image-20211111194911688.png)

现在的链接上面和li的下边框重叠了，所以加外边距

```diff
.all {
	display: block;
	
	height: 38px;
	border: 1px solid #00a4ff;
+	margin-top: 10px;
}
```

> ![image-20211111195048687](http://myapp.img.mykernel.cn/image-20211111195048687.png)

单行文字垂直居中`line-height`

文字水平居中

```diff
.all {
	display: block;
	
	height: 38px;
	border: 1px solid #00a4ff;
	margin-top: 10px;

+	text-align: center;
+	line-height: 38px;
}
```

> ![image-20211111195422380](http://myapp.img.mykernel.cn/image-20211111195422380.png)

现在文字像素/颜色

![image-20211111195512296](http://myapp.img.mykernel.cn/image-20211111195512296.png)

```diff
.all {
	display: block;
	
	height: 38px;
	border: 1px solid #00a4ff;
	margin-top: 10px;

	text-align: center;
	line-height: 38px;
+	font-size: 16px;
+	color: #00a4ff;
}
```

> ![image-20211111195557716](http://myapp.img.mykernel.cn/image-20211111195557716.png)

现在可以发现所有的a都没有下划线，所以在css初始化部分添加

```css
* {
	margin: 0;
	padding: 0;
}
li {
	list-style: none;
}
button {
	border: none;
}
a {
	text-decoration: none;
}
```

> ![image-20211111195703196](http://myapp.img.mykernel.cn/image-20211111195703196.png)

###### 去掉盒子的背景

```diff
.course {
	float: right;
	width: 228px;
	height: 300px;
	/*本应该父盒子中的标准流子盒子定义上面的外边距时，父盒子会向下掉。*/
	/*但是子盒子浮动了，就不存在这样的情况了*/
	margin-top: 50px;

+	background-color: #fff;
}
```

> ![image-20211111200208814](http://myapp.img.mykernel.cn/image-20211111200208814.png)

###### 鼠标经过a,背景变蓝。字体(前景)变白

```css
.all:hover {
	background-color: #00a4ff;
}
```

> ![image-20211111200325319](http://myapp.img.mykernel.cn/image-20211111200325319.png)

```css
.all:hover {
	background-color: #00a4ff;
	color: #fff;
}
```

> ![image-20211111200422213](http://myapp.img.mykernel.cn/image-20211111200422213.png)

##### 第3行精品模块 大盒子

- 元素继承
- 行内元素: a img, td 只有左右内外边距，没有上下的内外边距。

###### 分析盒子

![image-20211111201243479](http://myapp.img.mykernel.cn/image-20211111201243479.png)

> 1. 版心盒子
> 2. 3个内盒子
> 3. 左边第1个盒子，大小/颜色/水平/垂直剧中

###### 写结构

![image-20211111201551717](http://myapp.img.mykernel.cn/image-20211111201551717.png)

> 一定要确保3个盒子的注释在缩之后，明晰的结构

这里先写一个goods的盒子

![image-20211111201644273](http://myapp.img.mykernel.cn/image-20211111201644273.png)

> 123在页面左侧，应该在版心

###### 添加版心

```diff
	<!-- 精品部分开始 -->
+	<div class="goods w">
		123
	</div>
	<!-- 精品部分结束 -->
```

> ![image-20211111201800694](http://myapp.img.mykernel.cn/image-20211111201800694.png)

现在正常了, 但是现在写的页面太靠下了，不方便看

###### 临时添加页面高度，方便查看调试

```diff
body {
	background-color: #f3f5f7;
+	/*临时添加，写完后删除*/
+	height: 3000px;
}

```

> ![image-20211111202024433](http://myapp.img.mykernel.cn/image-20211111202024433.png)
>
> 现在右侧有滚动条了，可以让123在中间，而且刷新页面后，滚动条的位置还是这个位置

###### 测量高度

![image-20211111202146296](http://myapp.img.mykernel.cn/image-20211111202146296.png)

60px

```css
/*goods start*/
.goods {
	height: 60px;
	background-color: pink;
}

/*goods stop*/
```

> ![image-20211111202556491](http://myapp.img.mykernel.cn/image-20211111202556491.png)

注意：这个盒子有阴影

![image-20211111202229117](http://myapp.img.mykernel.cn/image-20211111202229117.png)



###### 添加盒子阴影

```diff
/*goods start*/
.goods {
	height: 60px;
	background-color: pink;


+	box-shadow: 2px 2px 2px rgba(0, 0, 0, .2);
}

```

> ![image-20211111202751838](http://myapp.img.mykernel.cn/image-20211111202751838.png)

###### 盒子位置

与上面有一定高度

![image-20211111202845101](http://myapp.img.mykernel.cn/image-20211111202845101.png)

```diff
/*goods start*/
.goods {
	height: 60px;
+	margin-top: 10px;
	
	background-color: pink;


	box-shadow: 2px 2px 2px rgba(0, 0, 0, .2);
}
```

> ![image-20211111202926224](http://myapp.img.mykernel.cn/image-20211111202926224.png)

##### 第3行精品模块 里面的盒子

- 可以划3个部分，精品推荐，中间盒子，右边盒子

  ![image-20211115185448013](http://myapp.img.mykernel.cn/image-20211115185448013.png)

- 前2个左对齐，后面右对齐

- 3个盒子均要垂直居中，给外层盒子垂直，里面的子盒子会继承特性。

  > 可以继承的特性: 
  >
  > - font- 
  > - line- ： `注意lin-height`继承之后，里面占用的高度看实际文字的高度
  > - text- 
  > - color

```diff
/*goods start*/
.goods {
	height: 60px;
+	/*行高继承*/
+	line-height: 60px;
	margin-top: 10px;

	background-color: pink;


	box-shadow: 2px 2px 2px rgba(0, 0, 0, .2);
}
```

第1个小标题h3

中间不重要，不需要使用li了，直接给a



###### 左侧格1

```html
	<!-- 精品部分开始 -->
	<div class="goods w">
		<h3>精品推荐</h3>
	</div>
	<!-- 精品部分结束 -->
```

> ![image-20211115185829189](http://myapp.img.mykernel.cn/image-20211115185829189.png)

修饰文字大小/颜色

```css
.goods h3 {
	font-size: 16px;
	color: #00a4ff;
}
```

> ![image-20211115190046927](http://myapp.img.mykernel.cn/image-20211115190046927.png)
>
> 精品推荐有一个小距离，只管左右，不管上下

左右距离

![image-20211115190135544](http://myapp.img.mykernel.cn/image-20211115190135544.png)

![image-20211115190149327](http://myapp.img.mykernel.cn/image-20211115190149327.png)

```diff
.goods h3 {
+	padding: 0 34px;
	font-size: 16px;
	color: #00a4ff;
}
```

> <strong style="color: red;">老师这里写的是<code>margin-left: 34px 因为只写左边</code></strong>
>
> ![image-20211115190335757](http://myapp.img.mykernel.cn/image-20211115190335757.png)

###### 处理小竖线的问题

![image-20211115190531745](http://myapp.img.mykernel.cn/image-20211115190531745.png)

右边的盒子不重要就`div>a*6`, 在a前加竖线即可

```diff
	<!-- 精品部分开始 -->
	<div class="goods w">
		<h3>精品推荐</h3>
		<!-- .goods-item -->
		<div class="goods-item">
			| <a href="">jQuery</a>
			| <a href="">Spark</a>
			| <a href="">jQuery</a>
			| <a href="">Spark</a>
			| <a href="">jQuery</a>
			| <a href="">Spark</a>
		</div>
	</div>
	<!-- 精品部分结束 -->
```

> ![image-20211115190926976](http://myapp.img.mykernel.cn/image-20211115190926976.png)

文字掉下来了，浮动即可解决。块级子元素浮动可以横排

```diff
.goods h3 {
+	float: left;
	padding: 0 34px;
	font-size: 16px;
	color: #00a4ff;
}

.goods-item {
+	float: left;
}
```

> ![image-20211115191110437](http://myapp.img.mykernel.cn/image-20211115191110437.png)

###### 中间盒子字符

```css
.goods-item a {
	color: #050505;
	font-size: 16px;
}
```

> ![image-20211115191407162](http://myapp.img.mykernel.cn/image-20211115191407162.png)

###### 竖线和a的距离

```diff
.goods-item a {
	color: #050505;
	font-size: 16px;
+	padding: 0 34px;
}
```

> ![image-20211115191826364](http://myapp.img.mykernel.cn/image-20211115191826364.png)

###### 竖线的颜色

调整这个颜色, 首先|属于`.goods-item`, 而a里面的文字虽然也属于，但是a的优先级更高

```html
		<div class="goods-item">
			| <a href="">jQuery</a>
			| <a href="">Spark</a>
			| <a href="">jQuery</a>
			| <a href="">Spark</a>
			| <a href="">jQuery</a>
			| <a href="">Spark</a>
		</div>
```

```diff
.goods-item {
	float: left;
+	color: #bfbfbf;
}
```

![image-20211115195702747](http://myapp.img.mykernel.cn/image-20211115195702747.png)

###### 右边盒子

虽然右边的盒子，可以出来是蓝色的，但是不是`a`标签，因为网页上可以点击的未必是a。可能是jquery.

```html
		<div class="mod">
			修改兴趣
		</div>
	</div>
```

> ![image-20211115200756745](http://myapp.img.mykernel.cn/image-20211115200756745.png)

右浮

```css
.mod {
	float: right;
}
```

![image-20211115204318763](http://myapp.img.mykernel.cn/image-20211115204318763.png)

字的大小及颜色

```diff
.mod {
	float: right;

+	color: #00a4ff;
+	font-size: 14px;
}
```

![image-20211115214604437](http://myapp.img.mykernel.cn/image-20211115214604437.png)

###### 右边位置

![image-20211115214814866](http://myapp.img.mykernel.cn/image-20211115214814866.png)

```diff
.mod {
	float: right;

	color: #00a4ff;
	font-size: 14px;
+	margin-right: 30px;	
}
```

> ![image-20211115214853163](http://myapp.img.mykernel.cn/image-20211115214853163.png)

###### pink色去掉，变成白色

```diff
.goods {
	height: 60px;
	/*行高继承*/
	line-height: 60px;
	margin-top: 10px;

+	background-color: #fff;

	box-shadow: 2px 2px 2px rgba(0, 0, 0, .2);
}

```

![image-20211115215021986](http://myapp.img.mykernel.cn/image-20211115215021986.png)



##### 第4行精品推荐

![image-20211116190559369](http://myapp.img.mykernel.cn/image-20211116190559369.png)

大盒子，上下2个盒子，box,box-hd, box-bd

结构

![image-20211116190824066](http://myapp.img.mykernel.cn/image-20211116190824066.png)

```html
	<!-- box start -->
	<div class="box">
		
	</div>
	<!-- box stop -->
```

###### 外层盒子大小

不给高度？

> 看里面的下面的box-bd具体占用的高度，心情好就如上面2排。心情不好，就放一排
>
> <strong style="color: red;">复习：有些父亲盒子不方便给高度，看儿子盒子占用大小</strong>

不给宽度？

让它自动撑开

###### 上面盒子

![image-20211116191257925](http://myapp.img.mykernel.cn/image-20211116191257925.png)

从上图可以看出文字到上面的精品块有一段小距离

- 让上面的盒子占用整个位置，不给背景，给个上内边距

- 让上面的盒子，占一部分，并让文字垂直居中。

  先测量文字下面的高度，计算和上面一样的高度即可，最终

  ![image-20211116191610743](http://myapp.img.mykernel.cn/image-20211116191610743.png)

  ![image-20211116191638640](http://myapp.img.mykernel.cn/image-20211116191638640.png)

这样这个内层盒子的有个上外边距，会让外层盒子也有这个上外边距，所以直接定义在外层盒子上

###### 写结构

```html
	<!-- box start -->
	<div class="box">
		123
	</div>
	<!-- box stop -->
</body>
</html>
```

![image-20211116191849915](http://myapp.img.mykernel.cn/image-20211116191849915.png)

###### 版心

```diff
	<!-- box start -->
+	<div class="box w">
		123
	</div>
	<!-- box stop -->
</body>
</html>
```

![image-20211116191935677](http://myapp.img.mykernel.cn/image-20211116191935677.png)

###### 盒子上面有外边距

```css
/*box start*/
.box {
	margin-top: 15px;
}
/*box stop*/

```

![image-20211116192031961](http://myapp.img.mykernel.cn/image-20211116192031961.png)

###### 上面的盒子

```html
	<!-- box start -->
	<div class="box w">
		<div class="box-hd">
			123
		</div>
	</div>
	<!-- box stop -->
```

由`5.1.6.2.10.2`中的上面盒子高59

![image-20211116191610743](http://myapp.img.mykernel.cn/image-20211116191610743.png)

```css
.box-hd {
	height: 59px;
	line-height: 59px;
}
```

![image-20211116192452396](http://myapp.img.mykernel.cn/image-20211116192452396.png)

###### 上面盒子浮动

浮动

- 左右对齐
- 一行多个块

![image-20211116192945911](http://myapp.img.mykernel.cn/image-20211116192945911.png)

精品推荐是大标题，给h3

右边查看全部，一点击肯定有新网页，使用a

###### 上面盒子的结构

```diff
	<!-- box start -->
	<div class="box w">
		<div class="box-hd">
+			<h3>精品推荐</h3>
+			<a href="#">查看全部</a>
		</div>
	</div>
	<!-- box stop -->
```

![image-20211116193358827](http://myapp.img.mykernel.cn/image-20211116193358827.png)

###### 上面盒子浮动

```css
.box-hd h3 {
	float: left;
}
.box-hd a {
	float: right;
}
/*box stop*/

```

![image-20211116193518717](http://myapp.img.mykernel.cn/image-20211116193518717.png)

###### 上面盒子字体颜色

```diff
.box-hd h3 {
	float: left;
+	color: #494949;
+	font-size: 20px;
}
.box-hd a {
	float: right;
+	font-size: 12px;
+	color: #a5a5a5;
}
/*box stop*/

```

![image-20211116193647371](http://myapp.img.mykernel.cn/image-20211116193647371.png)

###### 标题变细

![image-20211116194229480](http://myapp.img.mykernel.cn/image-20211116194229480.png)

效果图比网页细

```diff
.box-hd h3 {
	float: left;
	color: #494949;
	font-size: 20px;
+	font-weight: 400; /* 400等价normal */
}
```

###### 右边链接的右边有小距离

![image-20211116194507960](http://myapp.img.mykernel.cn/image-20211116194507960.png)

![image-20211116194528515](http://myapp.img.mykernel.cn/image-20211116194528515.png)

```diff
.box-hd a {
	float: right;

+	margin-right: 31px;
	font-size: 12px;
	color: #a5a5a5;
}
```

盒子模型的，所以比字体优先级高

![image-20211116194609156](http://myapp.img.mykernel.cn/image-20211116194609156.png)

###### 下面盒子

现在可以做下面这个模块了

![image-20211116194749136](http://myapp.img.mykernel.cn/image-20211116194749136.png)

下面这个，通用`ul>li`

###### 下面例子结构

```diff
	<!-- box start -->
	<div class="box w">
		<div class="box-hd">
			<h3>精品推荐</h3>
			<a href="#">查看全部</a>
		</div>

+		<div class="box-bd">
+			123
+		</div>
	</div>
	<!-- box stop -->
```

![image-20211116195136144](http://myapp.img.mykernel.cn/image-20211116195136144.png)

###### bd不给高度，写结构

可能放1行，也可以是2行，也可以是n行。

宽度呢，同父亲

ul中放小li

一共放10个盒子，不用放10个，先写1个

外层盒子不需要大小，但是每一个`li`是需要大写的，li测量的时候一定要精细。差了就会掉

> 而这些位置，右边有大空隙，不需要精细。
>
> ![image-20211116195554622](http://myapp.img.mykernel.cn/image-20211116195554622.png)

![image-20211116195325989](http://myapp.img.mykernel.cn/image-20211116195325989.png)

```html
		<div class="box-bd">
			<ul>
				<li></li>
			</ul>
		</div>
```



###### bd中的小li测量

![image-20211116195839231](http://myapp.img.mykernel.cn/image-20211116195839231.png)

右边的空隙宽`15px`, 下间隙高`15px`

```css
.box-bd {

}

.box-bd li {
	width: 228px;
	height: 270px;
	background-color: pink;
}
```

![image-20211116200057501](http://myapp.img.mykernel.cn/image-20211116200057501.png)

现在把li放10个

```diff
		<div class="box-bd">
			<ul>
+				<!-- li*10{$} -->
+				<li>1</li>
+				<li>2</li>
+				<li>3</li>
+				<li>4</li>
+				<li>5</li>
				<li>6</li>
				<li>7</li>
				<li>8</li>
				<li>9</li>
				<li>10</li>
			</ul>
		</div>
	</div>
```

![image-20211116200213573](http://myapp.img.mykernel.cn/image-20211116200213573.png)

###### bd中的小li浮动

```diff
.box-bd li {
+	float: left;
	width: 228px;
	height: 270px;
	background-color: pink;
}
```

![image-20211116200253319](http://myapp.img.mykernel.cn/image-20211116200253319.png)

###### bd中的小li间隙 小技巧

```diff
.box-bd li {
	float: left;

+	margin-right: 15px;
	width: 228px;
	height: 270px;
	background-color: pink;
}
```

![image-20211116200604800](http://myapp.img.mykernel.cn/image-20211116200604800.png)

第5个盒子的margin-right不能要，万一有n个盒子怎么调整？

1. 之前做一个`class: nomargin`, 在第5个和第10个，这样麻烦, 不知道有多少个li
2. 直接修改父盒子的宽度, 因为父盒子兄弟间都是块，如果超了宽度，也不会换行的。

```css
.box-bd {
	/*比版心宽15px, 就是下面li的margin-right多出来的15px*/
	width: 1215px;
}
```

li高之间的间隙

```diff
.box-bd li {
	float: left;

	margin-right: 15px;
+	margin-bottom: 15px;
	width: 228px;
	height: 270px;
	background-color: pink;
}
```

![image-20211116204426391](http://myapp.img.mykernel.cn/image-20211116204426391.png)

###### bd中的li中装内容分析结构

![image-20211116204630896](http://myapp.img.mykernel.cn/image-20211116204630896.png)

###### bd中的li中 cutterman切上面背景

导出Jpeg

![image-20211116205652796](http://myapp.img.mykernel.cn/image-20211116205652796.png)

```html
		<div class="box-bd">
			<ul>
				<!-- li*10{$} -->
				<li>
					<img src="images/pic.jpg" alt="">
				</li>
			</ul>
		</div>
```

> 当前只写一个，后续直接复制即可

![image-20211116205734910](http://myapp.img.mykernel.cn/image-20211116205734910.png)

###### img调整与父亲一样大

```css
.box-bd img {
	width: 100%;
}
```

![image-20211116210329273](http://myapp.img.mykernel.cn/image-20211116210329273.png)

###### bd中的li中的小标题

```html
				<li>
					<img src="images/pic.jpg" alt="">
					<h4>Think PHP 5.0 博客系统实战项目演练</h4>
				</li>
```

![image-20211116210629677](http://myapp.img.mykernel.cn/image-20211116210629677.png)

```css
.box-bd h4 {
	font-size: 14px;
	color: #050505;
}
```

![image-20211116210715866](http://myapp.img.mykernel.cn/image-20211116210715866.png)

###### bd中的li中的小标题左右距离

![image-20211116211050228](http://myapp.img.mykernel.cn/image-20211116211050228.png)

![image-20211116211104097](http://myapp.img.mykernel.cn/image-20211116211104097.png)

```diff
.box-bd h4 {
+	margin-left: 24px;
+	margin-right: 20px;
	font-size: 14px;
	color: #050505;
}
```

![image-20211116211206770](http://myapp.img.mykernel.cn/image-20211116211206770.png)

###### bd中的li中的小标题上下距离

![image-20211116211305135](http://myapp.img.mykernel.cn/image-20211116211305135.png)

![image-20211116211319819](http://myapp.img.mykernel.cn/image-20211116211319819.png)

```diff
.box-bd h4 {
+	margin-top: 25px;
+	margin-bottom: 20px;
	margin-left: 24px;
	margin-right: 20px;

	font-size: 14px;
	color: #050505;
}
```

![image-20211116211406062](http://myapp.img.mykernel.cn/image-20211116211406062.png)

###### bd中的li中的p结构

```diff
				<li>
					<img src="images/pic.jpg" alt="">
					<h4>Think PHP 5.0 博客系统实战项目演练</h4>
+					<p>高级  •  1125人在学习</p>
				</li>
```

![image-20211116211540349](http://myapp.img.mykernel.cn/image-20211116211540349.png)

```css
.box-bd p {
	margin-left: 24px;
	margin-right: 20px;
}
```

![image-20211116211638189](http://myapp.img.mykernel.cn/image-20211116211638189.png)

###### bd中的li中的p的字大小和颜色

```diff
.box-bd p {
	margin-left: 24px;
	margin-right: 20px;

+	color: #999999;
+	font-size: 12px;
}
```

![image-20211116211827056](http://myapp.img.mykernel.cn/image-20211116211827056.png)



###### bd中的li中的p的字 “高级"变颜色

```html
					<p><span>高级</span>  •  1125人在学习</p>
```

```css
.box-bd span {
	color: #ff7c2d;
}
```

![image-20211116211953542](http://myapp.img.mykernel.cn/image-20211116211953542.png)

###### bd中的li 纯白，而且有阴影

```diff
.box-bd li {
	float: left;

	margin-right: 15px;
	margin-bottom: 15px;
	width: 228px;
	height: 270px;
+	background-color: #fff;
}

```

![image-20211116212106095](http://myapp.img.mykernel.cn/image-20211116212106095.png)

```diff
.box-bd li {
	float: left;

	margin-right: 15px;
	margin-bottom: 15px;
	width: 228px;
	height: 270px;
	background-color: #fff;
+	box-shadow: 2px 2px 2px rgba(0, 0, 0, .2);
}
```

![image-20211116212201743](http://myapp.img.mykernel.cn/image-20211116212201743.png)

###### bd中的li写10个

```html
				<li>
					<img src="images/pic.jpg" alt="">
					<h4>Think PHP 5.0 博客系统实战项目演练</h4>
					<p><span>高级</span>  •  1125人在学习</p>
				</li>
```

现在把这个复制10份

![image-20211116212312273](http://myapp.img.mykernel.cn/image-20211116212312273.png)

###### 现在第4行盒子没有高度，里面上面块有高度，下面块没有高度，会影响后面的布局

```html
	<!-- box stop -->

	<div class="test">

	</div>
```

![image-20211116213022544](http://myapp.img.mykernel.cn/image-20211116213022544.png)

```css
.test {
	height: 500px;
	background-color: pink;
}
```

在这个位置下面

![image-20211116213126550](http://myapp.img.mykernel.cn/image-20211116213126550.png)

所以需要清除浮动, 在初始化部分

```css
.clearfix:before, 
.clearfix:after {
    content: "";
    display: table;
}

.clearfix:after {
    clear: both;
}

.clearfix {
    *zoom: 1;
}
```

添加清理类

```diff
+<div class="box-bd clearfix">
	<ul>
		...
	</ul>
</div>
```

![image-20211116213453450](http://myapp.img.mykernel.cn/image-20211116213453450.png)

现在清除浮动后的样子, 父级的高度就自动撑高为子元素的高度

![image-20211116213715779](http://myapp.img.mykernel.cn/image-20211116213715779.png)

##### 与此精品推荐同类的一些

这2个

![image-20211116213826813](http://myapp.img.mykernel.cn/image-20211116213826813.png)

我们只需要复制精品推荐，先使用sublime text折叠如下图，然后直接复制所有，到随后的html

![image-20211116213904372](http://myapp.img.mykernel.cn/image-20211116213904372.png)

然后如同下面，把下面的li行数修改成5个，多余的5个就删除

![image-20211117114428113](http://myapp.img.mykernel.cn/image-20211117114428113.png)

![image-20211117114500415](http://myapp.img.mykernel.cn/image-20211117114500415.png)

##### 页面底部处理

###### 页面底部结构分析

页面底部是纯白色, 而背景是灰色

![image-20211117150438440](http://myapp.img.mykernel.cn/image-20211117150438440.png)

所以底部是通临览的白底

![image-20211117150604078](http://myapp.img.mykernel.cn/image-20211117150604078.png)

###### 页面底部结构

```html
	<!-- footer start -->
	<div class="footer">
		123
	</div>
	<!-- footer stop -->
```

###### 页面底部测量高度写css

![image-20211117150858887](http://myapp.img.mykernel.cn/image-20211117150858887.png)

```css
/*footer*/
.footer {
	height: 417px;
	background-color: #fff;
}

/*footer*/
```

![image-20211117151105653](http://myapp.img.mykernel.cn/image-20211117151105653.png)

注意：上面白色部分就是footer的白色，只是和当前页面一个颜色，看不出来 

###### 页面底部内部盒子结构分析

![image-20211117151235893](http://myapp.img.mykernel.cn/image-20211117151235893.png)

###### 页面底部内部盒子结构

```html
	<!-- footer start -->
	<div class="footer">
		<div class="w">
			123
		</div>
	</div>
	<!-- footer stop -->
```

![image-20211117151321237](http://myapp.img.mykernel.cn/image-20211117151321237.png)

现在到了版心了

###### 页面底部内部盒子内部上外边距

![image-20211117151416126](http://myapp.img.mykernel.cn/image-20211117151416126.png)

- 给footer的padding
- 给版心也可以，但是引用版心过多，影响大

```diff
/*footer*/
.footer {
+	height: calc(417px - 34px);
+	padding-top: 34px;
	background-color: #fff;
}

```

![image-20211117151559171](http://myapp.img.mykernel.cn/image-20211117151559171.png)

###### 页面底部内部盒子内部分析

![image-20211118185340497](http://myapp.img.mykernel.cn/image-20211118185340497.png)

内部左右对齐

######  页面底部内部盒子内部左边盒子 copyright 版权

```html
	<div class="footer">
		<div class="w">
			<!-- 左侧 -->
			<div class="copyright">
				123
			</div>
		</div>
	</div>
```

![image-20211118185502229](http://myapp.img.mykernel.cn/image-20211118185502229.png)

- img, p, a

###### 页面底部内部盒子内部左边盒子 copyright的logo

```html
			<!-- 左侧 -->
			<div class="copyright">
				<img src="./images/logo.png" alt="">
			</div>
```

![image-20211118185643385](http://myapp.img.mykernel.cn/image-20211118185643385.png)

###### 页面底部内部盒子内部左边盒子 copyright的p标签

```html
			<div class="copyright">
				<img src="./images/logo.png" alt="">
				<p>学成在线致力于普及中国最好的教育它与中国一流大学和机构合作提供在线课程。© 2017年XTCG Inc.保留所有权利。-沪ICP备15025210号</p>
			</div>
```

![image-20211118185801786](http://myapp.img.mykernel.cn/image-20211118185801786.png)

太长了, 肯定是到了右边另起一行，给一个换行

```html
			<div class="copyright">
				<img src="./images/logo.png" alt="">
				<p>学成在线致力于普及中国最好的教育它与中国一流大学和机构合作提供在线课程。<br>© 2017年XTCG Inc.保留所有权利。-沪ICP备15025210号</p>
			</div>
```

![image-20211118185953912](http://myapp.img.mykernel.cn/image-20211118185953912.png)

###### 页面底部内部盒子内部左边盒子 copyright 文字大小和颜色

```css
.footer p {
	font-size: 12px;
	color: #666666;
}
```

![image-20211118190041979](http://myapp.img.mykernel.cn/image-20211118190041979.png)

###### 页面底部内部盒子内部左边盒子 copyright 文字 上下间隙

```diff
.footer p {
+	margin-top: 25px;
+	margin-bottom: 17px;
	font-size: 12px;
	color: #666666;
}
```

![image-20211118190434532](http://myapp.img.mykernel.cn/image-20211118190434532.png)

###### 页面底部内部盒子内部左边盒子 copyright 链接

```diff
			<div class="copyright">
				<img src="./images/logo.png" alt="">
				<p>学成在线致力于普及中国最好的教育它与中国一流大学和机构合作提供在线课程。<br>© 2017年XTCG Inc.保留所有权利。-沪ICP备15025210号</p>
+				<a href="#" class="app">下载APP</a>
			</div>
		</div>
```

![image-20211118190744952](http://myapp.img.mykernel.cn/image-20211118190744952.png)

###### 页面底部内部盒子内部左边盒子 copyright链接 大小/颜色/边框/垂直居中/水平居中

```css
.footer app {
	display: block;
	border: 1px solid #00a4ff;
	width: 118px;
	height: 34px;
	line-height: 34px;

	text-align: center;
	font-size: 16px;
	color: #00a4ff;
}
```

![image-20211118191050159](http://myapp.img.mykernel.cn/image-20211118191050159.png)

###### 页面底部内部盒子内部右边盒子 links 分析

![image-20211118191224101](http://myapp.img.mykernel.cn/image-20211118191224101.png)

这个模块，自定义列表 `dl > dt+dd`

1. 右对齐

![image-20211118191344379](http://myapp.img.mykernel.cn/image-20211118191344379.png)

###### 页面底部内部盒子内部右边盒子 links 结构

```html
			<!-- 右侧 -->
			<div class="links">
				<dl>
					<dt>关于学生网</dt>
					<dd>关于</dd>
					<dd>管理团队</dd>
					<dd>工作机会</dd>
					<dd>客户服务</dd>
					<dd>帮助</dd>
				</dl>
			</div>
```

![image-20211118191529816](http://myapp.img.mykernel.cn/image-20211118191529816.png)

###### 页面底部内部盒子内部右边盒子 links 右对齐

```css
.footer .copyright {
	float: left;
}
.footer .links {
	float: right;
}
```

![image-20211118191736492](http://myapp.img.mykernel.cn/image-20211118191736492.png)

###### 页面底部内部盒子内部右边盒子 links 内部分析

![image-20211118191800851](http://myapp.img.mykernel.cn/image-20211118191800851.png)

需要将dd添加链接

```html
			<!-- 右侧 -->
			<div class="links">
				<dl>
					<dt>关于学成网</dt>
					<dd><a href="#">关于</a></dd>
					<dd><a href="#">管理团队</a></dd>
					<dd><a href="#">工作机会</a></dd>
					<dd><a href="#">客户服务</a></dd>
					<dd><a href="#">帮助</a></dd>
				</dl>
			</div>
```

dt没有a, 而dd有a

###### 页面底部内部盒子内部右边盒子 links dd 文字

```css
.links dd a {
	font-size: 12px;
	color: #333333;
}
```

![image-20211118192351524](http://myapp.img.mykernel.cn/image-20211118192351524.png)

###### 页面底部内部盒子内部右边盒子 links dt 文字及间隙

![image-20211118192244710](http://myapp.img.mykernel.cn/image-20211118192244710.png)

给可以给dt margin, 太麻烦了，直接给dt一个高度，自然文字在上面

![image-20211118192536215](http://myapp.img.mykernel.cn/image-20211118192536215.png)

```css
.links dt {
	height: 33px;
	font-size: 16px;
	color: #333333;
}
```

![image-20211118192605027](http://myapp.img.mykernel.cn/image-20211118192605027.png)

###### 页面底部内部盒子内部右边盒子 links 再准备2组 dl>dt+dd

![image-20211118192719218](http://myapp.img.mykernel.cn/image-20211118192719218.png)

![image-20211118192732970](http://myapp.img.mykernel.cn/image-20211118192732970.png)

###### 页面底部内部盒子内部右边盒子 links  单行

```css
.links dl {
	float: left;
}
```

![image-20211118192821236](http://myapp.img.mykernel.cn/image-20211118192821236.png)

![image-20211118193055097](http://myapp.img.mykernel.cn/image-20211118193055097.png)

左边的宽度，才向左边走，需要left的间隙

###### 页面底部内部盒子内部右边盒子 links 间隙

给所有margin-left

```CSS
.links dl {
	float: left;
	margin-left: 100px;
}
```

![image-20211118193140396](http://myapp.img.mykernel.cn/image-20211118193140396.png)

##### 去掉body的高

```diff
body {
	background-color: #f3f5f7;
	/*临时添加，写完后删除*/
	/*height: 3000px;*/
}
```

### chrome调试工具

打开

1. F12
2. 右键-检查

界面：左侧html, 右侧css

css中可以调试数值, css 属性右边会对应你源码文件的具体哪一行

![image-20211118194052715](http://myapp.img.mykernel.cn/image-20211118194052715.png)



#### chrome提示单词问题

![image-20211118194251672](http://myapp.img.mykernel.cn/image-20211118194251672.png)



#### css不显示

类名与css选择器不匹配

#### 标签不匹配

当html这样时

```html
	<div class="header">
		<div class="logo">
		<div class="search"></div>
	</div>
```

> 第2行不结束

页面将会显示

```html
<div class="header">
		<div class="logo">
			<div class="search"></div>
		</div>
</div>
```

logo嵌套search

#### 常见

后面的value变颜色 

宽度后面少`;`, 则下一级不生效

多一个或少一个`}`, 后面的不起作用



# CSS第6天

## 定位 position

`background-position` 也使用过这个单词

### 目标

- 理解
  - 为什么用定位？
  - 哪4种类型的定位？各自的特点？
  - 为什么常用<strong style="color: red;">子绝父相</strong>布局
- 应用
  - 写出淘宝轮播图布局

### CSS布局的三种机制

- 标准流：多个div自上而下显示
- 浮动：多个div自左而右显示
- 定位：将盒子定在某个位置上，自由漂浮在其他盒子上。------ CSS离不开定位，特别是JS特效。

### 为什么使用定位 

同样和浮动一样，这种场景，使用之前的知识容易实现不？

盒子定位在某个位置，自由的漂浮在其他盒子的上面

#### 小黄色块在图片上移动

#### 浮动不能压住文字

#### 盒子固定在网页位置

就算网页上下滚动，位置还在这个位置

![image-20211122100042928](http://myapp.img.mykernel.cn/image-20211122100042928.png)

### 定位详解

定位  = 定位模式 + 边偏移

#### 边偏移

通过`top`, `bottom`, `left`, `right` 4个方位名词， 即距离原来的位置偏移多少位置。

例如: `top: 80px` 相对原来的位置向上移动80px

#### 定位模式(position)

看滚动屏幕时的表现

```css
css-selector { position: 定位模式; }
```

| 值       | 含义                                                         |
| -------- | ------------------------------------------------------------ |
| static   | 静态定位                                                     |
| relative | 相对定位: 相对于标准流原来的位置移动, 定位后原来的位置依然保留。 |
| absolute | 绝对定位                                                     |
| fixed    | 固定定位                                                     |

##### 静态定位

相当于你的`border: none` 就是没有边框，而定位模式使用`position: static`就表示不定位，也不存在边偏移。

布局时我们几乎不用。标准流、浮动流，定位默认就是静态定位。

##### 相对定位 relative 会占用位置, 未脱标

###### 相对于标准流原来的位置移动

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 200px;
			height: 200px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div></div>
</body>
</html>
```

> 一个200px, 200px, pink色的盒子
>
> ![image-20211122163619992](http://myapp.img.mykernel.cn/image-20211122163619992.png)

添加定位

```diff
		div {
			width: 200px;
			height: 200px;
			background-color: pink;

+			position: relative;
+			top: 100px;
+			left: 100px;
		}
```

> 盒子动了, 距离上面100px, 左边100px
>
> ![image-20211122163804039](http://myapp.img.mykernel.cn/image-20211122163804039.png)

相对定位：相对原来自己标准流的位置。



现在去掉定位给一个margin值

```css
	<style>
		div {
			width: 200px;
			height: 200px;
			background-color: pink;
			margin: 100px;
		}
	</style>
```

> ![image-20211122163937998](http://myapp.img.mykernel.cn/image-20211122163937998.png)

现在给一个相对定位

```css
		div {
			width: 200px;
			height: 200px;
			background-color: pink;
			margin: 100px;

			position: relative;
			top: 100px;
			left: 100px;
		}
```

> 现在是相对于之前margin位置，向下移动100px, 向右移动100px.

###### 标准流的区域继续占用， <strong style="color: red;">保留位置。 未脱标</strong>

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 200px;
			height: 200px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div>1</div>
	<div>2</div>
	<div>3</div>
</body>
</html>
```

> ![image-20211122164349720](http://myapp.img.mykernel.cn/image-20211122164349720.png)

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 200px;
			height: 200px;
			background-color: pink;
		}

		.two {
			background-color: purple;
		}
	</style>
</head>
<body>
	<div>1</div>
	<div class="two">2</div>
	<div>3</div>
</body>
</html>
```

> 给第2个盒子修改颜色
>
> ![image-20211122164444903](http://myapp.img.mykernel.cn/image-20211122164444903.png)

给第2个盒子添加相对定位

```diff
		.two {
			background-color: purple;

+			position: relative;
+			top: 100px;
+			left: 100px;
		}
```

> - 移动了，正常
>
>   ![image-20211122164549881](http://myapp.img.mykernel.cn/image-20211122164549881.png)
>
> - 原来的位置还被占用。

##### 绝对定位 absolute 不会占用位置,  脱标

绝对定位的当前盒子会相对外层 已经(绝对、相对、固定)定位的盒子，来相对移动。如果外层盒子均无定位则相对document移动。

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.father {
			width: 350px;
			height: 350px;
			background-color: pink;
		}

		.son {
			width: 200px;
			height: 200px;
			background-color: purple;
		}
	</style>
</head>
<body>
	<!-- .father>.son -->
	<div class="father">
		<div class="son"></div>
	</div>
</body>
</html>
```

> ![image-20211122165630258](http://myapp.img.mykernel.cn/image-20211122165630258.png)

现在给父盒子加margin

```diff
		.father {
			width: 350px;
			height: 350px;
			background-color: pink;

+			margin: 100px;
		}
```

> ![image-20211122165753269](http://myapp.img.mykernel.cn/image-20211122165753269.png)
>
> 标准流的子盒子，位置相对于父亲的内容位置。

现在给儿子加一个margin

```diff
		.son {
			width: 200px;
			height: 200px;
			background-color: purple;
+			margin-left: 50px;			
		}
```

> ![image-20211122165935518](http://myapp.img.mykernel.cn/image-20211122165935518.png)

现在儿子左边相对于父亲左侧内容区。<strong style="color: red;">注意：不能给margin-top, 因为上面会让父亲也向下跑</strong>

###### 父级无定位, 相对浏览器`document`位置来移动

现在使用绝对定位

```diff
		.son {
			width: 200px;
			height: 200px;
			background-color: purple;
			/*margin-left: 50px;*/
+			position: absolute;
+			top: 50px;
+			left: 50px;
		}
```

> ![image-20211122170131997](http://myapp.img.mykernel.cn/image-20211122170131997.png)

###### 父级有定位，相对父亲移动

```diff
		.father {
			width: 350px;
			height: 350px;
			background-color: pink;

			margin: 100px;
+			position: relative;
		}
```

> 父亲一旦有定位，儿子相对父亲移动
>
> ![image-20211122170415874](http://myapp.img.mykernel.cn/image-20211122170415874.png)

###### 爷爷有定位，父亲无定位，儿子相对爷爷

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.yeye {
			width: 500px;
			height: 500px;
			background-color: skyblue;

		}
		.father {
			width: 350px;
			height: 350px;
			background-color: pink;

			margin: 100px;
			position: relative;

		}
		.son {
			width: 200px;
			height: 200px;
			background-color: purple;
			/*margin-left: 50px;*/
			position: absolute;
			top: 50px;
			left: 50px;
		}
	</style>
</head>
<body>
	<div class="yeye">
		<!-- .father>.son -->
		<div class="father">
			<div class="son"></div>
		</div>
	</div>
</body>
</html>
```

> 父亲让爷爷塌陷, 目前father有定位，相对爸爸移动
>
> ![image-20211122170917107](http://myapp.img.mykernel.cn/image-20211122170917107.png)

如果父亲无定位，爷爷有定位

```diff
		.yeye {
			width: 500px;
			height: 500px;
			background-color: skyblue;
+			position: relative;

		}
		.father {
			width: 350px;
			height: 350px;
			background-color: pink;

			margin: 100px;
+/*			position: relative;
+*/
		}
```

> ![image-20211122171022029](http://myapp.img.mykernel.cn/image-20211122171022029.png)

###### 绝对定位的盒子 <strong style="color: red;">不保留原来的位置，脱标</strong>

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.father {
			width: 400px;
			height: 400px;
			background-color: pink;
		}

		.one {
			width: 200px;
			height: 200px;
			background-color: purple;
		}

		.two {
			width: 250px;
			height: 250px;
			background-color: skyblue;
		}
	</style>
</head>
<body>
	<div class="father">
		<div class="one"></div>
		<div class="two"></div>
	</div>
</body>
</html>
```

> ![image-20211122171814744](http://myapp.img.mykernel.cn/image-20211122171814744.png)

现在给one加一个绝对定位，看是否会保留位置

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.father {
			width: 400px;
			height: 400px;
			background-color: pink;
			position: relative;
		}

		.one {
+			position: absolute;
+			top: 0;
+			left: 0;

			width: 200px;
			height: 200px;
			background-color: purple;
		}

		.two {
			width: 250px;
			height: 250px;
			background-color: skyblue;
		}
	</style>
</head>
<body>
	<div class="father">
		<div class="one"></div>
		<div class="two"></div>
	</div>
</body>
</html>
```

> ![image-20211122172044724](http://myapp.img.mykernel.cn/image-20211122172044724.png)

###### 口决， <strong style="color: red;">子绝父相</strong>

儿子有定位时，如果父亲没有定位，那就相对浏览器了。

儿子一定在父盒子内，则表明父盒子一定要有定位。

父盒子使用绝对定位，就不会保留自己的标准流位置, 然后与父亲同级的标准流会占用父亲的位置。类似浮动，但是比浮动更高的位置。

父盒子使用相对定位，父亲同级的下面盒子就在父亲之后了。孩子也会可以相对定位的父亲了。

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.upper {
			width: 1000px;
			height: 90px;
			background-color: pink;
		}

		.down {
			width: 1000px;
			height: 150px;
			background-color: purple;
		}
	</style>
</head>
<body>
	<div class="upper"></div>
	<div class="down"></div>
</body>
</html>
```

> 上下两个盒子
>
> ![image-20211123184713696](http://myapp.img.mykernel.cn/image-20211123184713696.png)

现在给上面的盒子添加一个方向箭头

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.upper {
			width: 1000px;
			height: 90px;
			background-color: pink;
		}

		.down {
			width: 1000px;
			height: 150px;
			background-color: purple;
		}


+		.arr-l	{
			float: left;
			height: 50px;
			width: 50px;
			background-color: black;
		}
	</style>
</head>
<body>
	<div class="upper">
+		<div class="arr-l"></div>
	</div>
	<div class="down"></div>
</body>
</html>
```

> 这个箭头是在上面
>
> ![image-20211123185402080](http://myapp.img.mykernel.cn/image-20211123185402080.png)

一般情况下，是箭头下面是图片, 所以引入图片![im123g](http://myapp.img.mykernel.cn/im123g.jpg)

```diff
	<div class="upper">
+		<img src="http://myapp.img.mykernel.cn/im123g.jpg" alt="">
		<div class="arr-l"></div>
	</div>
	<div class="down"></div>
```

> 因为这个图片和父盒子一样大小，所以后面的子盒子会掉出父盒子
>
> ![image-20211123185646508](http://myapp.img.mykernel.cn/image-20211123185646508.png)

现在想让子盒子到父盒子上, 添加浮动

```diff
		.upper img {
			float: left;
		}
		.arr-l	{
			float: left;
			height: 50px;
			width: 50px;
			background-color: black;
		}
```

> 这样的结果还是和上面一样

现在使用定位

```diff
		.upper img {
			float: left;
		}
		.arr-l	{
+			position: absolute;
+			top: 0px;
+			left: 0px;
			float: left;
			height: 50px;
			width: 50px;
			background-color: black;
		}
```

> 使用相对定位时，相对于父盒子外的位置
>
> 使用绝对定位时，相对于document
>
> ![image-20211123190012276](http://myapp.img.mykernel.cn/image-20211123190012276.png)

现在让这个子盒子在大盒子内显示，所以子相对于父级有定位

- 给父亲绝对定位

  ```diff
  		.upper {
  +			position: absolute;
  			width: 1000px;
  			height: 90px;
  			background-color: pink;
  		}
  ```

  > 子盒子可以跑到父盒子中，但是与父亲同级的盒子占用了父亲的高度
  >
  > ![image-20211123190230038](http://myapp.img.mykernel.cn/image-20211123190230038.png)

- 给父亲相对定位 <strong style="color: red;">子绝父相, 子盒子一定在父盒子中</strong>

  ```diff
  		.upper {
  +			position: relative;
  			width: 1000px;
  			height: 90px;
  			background-color: pink;
  		}
  ```

  > 这个最好
  >
  > ![image-20211123190306446](http://myapp.img.mykernel.cn/image-20211123190306446.png)

现在F12让子盒子左边居中, 方向上箭头调整数值

> ![image-20211123190442477](http://myapp.img.mykernel.cn/image-20211123190442477.png)

右侧盒子

```diff
	<div class="upper">
		<img src="http://myapp.img.mykernel.cn/im123g.jpg" alt="">
		<div class="arr-l"></div>
+		<div class="arr-r"></div>
	</div>
```

```diff

		.arr-r {
			position: absolute;
			top: 20px;
+			right: 0px;
			float: left;
			height: 50px;
			width: 50px;
			background-color: black;
		}
```

> ![image-20211123190616695](http://myapp.img.mykernel.cn/image-20211123190616695.png)

##### 哈根达斯案例

![image-20211123190949295](http://myapp.img.mykernel.cn/image-20211123190949295.png)

这个图片与产品相关，就使用插入图片，不使用背景图片。

今日新单：在图上上面，需要定位

![adv00000](http://myapp.img.mykernel.cn/adv00000.jpg)

> 这个图片:  `310x190`

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.box {
			width: 310px;
			height: 190px;
			border: 1px solid #ccc;
			margin: 100px auto;
		} 
	</style>
</head>
<body>
	<div class="box"></div>
</body>
</html>
```

> ![image-20211123191301334](http://myapp.img.mykernel.cn/image-20211123191301334.png)

引入图片

```diff
	<div class="box">
+		<img src="http://myapp.img.mykernel.cn/adv00000.jpg" alt="">
	</div>
```

> ![image-20211123191342060](http://myapp.img.mykernel.cn/image-20211123191342060.png)

而示例的图片有一个小距离，给padding值，可以撑开盒子, 子元素占父的内容区域

```diff
		.box {
			width: 310px;
			height: 190px;
			border: 1px solid #ccc;
			margin: 100px auto;
+			padding: 10px;
		} 
```

> ![image-20211123191513161](http://myapp.img.mykernel.cn/image-20211123191513161.png)

接下来放今日新单的图

![br-1111](http://myapp.img.mykernel.cn/br-1111.gif)

![top_tu-2222](http://myapp.img.mykernel.cn/top_tu-2222.gif)



```diff
	<div class="box">
+		<img src="http://myapp.img.mykernel.cn/top_tu-2222.gif" alt="">
		<img src="http://myapp.img.mykernel.cn/adv00000.jpg" alt="">
	</div>
```

> ![image-20211123191704423](http://myapp.img.mykernel.cn/image-20211123191704423.png)
>
> 图应该压着下面的盒子

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.box {
+			position: relative;
			width: 310px;
			height: 190px;
			border: 1px solid #ccc;
			margin: 100px auto;
			padding: 10px;
		} 

		.top {
+			position: absolute;
+			top: 0px;
+			left: 0px;
		}
	</style>
</head>
<body>
	<div class="box">
+		<img class="top" src="http://myapp.img.mykernel.cn/top_tu-2222.gif" alt="">
		<img src="http://myapp.img.mykernel.cn/adv00000.jpg" alt="">
	</div>
</body>
</html>
```

> ![image-20211123191957757](http://myapp.img.mykernel.cn/image-20211123191957757.png)

右下脚

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.box {
			position: relative;
			width: 310px;
			height: 190px;
			border: 1px solid #ccc;
			margin: 100px auto;
			padding: 10px;
		} 

		.top {
			position: absolute;
			top: 0px;
			left: 0px;
		}

+		.bottom {
			position: absolute;
			bottom: 0px;
			right: 0px;
		}
	</style>
</head>
<body>
	<div class="box">
		<img class="top" src="http://myapp.img.mykernel.cn/top_tu-2222.gif" alt="">
		<img src="http://myapp.img.mykernel.cn/adv00000.jpg" alt="">
+		<img src="http://myapp.img.mykernel.cn/br-1111.gif" alt="" class="bottom">
	</div>
</body>
</html>
```

![image-20211123192129853](http://myapp.img.mykernel.cn/image-20211123192129853.png)

##### 固定定位 脱标

滚动屏幕时，固定在屏幕中某个位置

1. 完全不占标准流，浮动，绝对定位
2. 来移动位置参照点：浏览器的可视窗口
3. 滚动屏幕时， 不会移动
4. IE6 不支持固定定位，但是现在都使用360, firefox, chrome，不存在这个问题。

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.box {
			width: 300px;
			height: 300px;
			background-color: pink;
			margin: 100px;
		} 
	</style>
</head>
<body>
	<div class="box">
		<img src="http://myapp.img.mykernel.cn/sun123123.jpg" alt="">
	</div>
</body>
</html>
```

> ![image-20211123202342639](http://myapp.img.mykernel.cn/image-20211123202342639.png)
>
> 修改图片大小
>
> ```diff
> 	<div class="box">
> +		<img width="150px" src="http://myapp.img.mykernel.cn/sun123123.jpg" alt="">
> 	</div>
> ```
>
> ![image-20211123202432713](http://myapp.img.mykernel.cn/image-20211123202432713.png)

此时给img添加固定定位 

```diff
		.box img {
			position: fixed;
			top: 0;
			left: 0;
		}
```

> ![image-20211123202547035](http://myapp.img.mykernel.cn/image-20211123202547035.png)
>
> 相对浏览器

###### 与父亲是否定位无关

给父亲添加一个定位 

```diff
		.box {
			width: 300px;
			height: 300px;
			background-color: pink;
			margin: 100px;
+			position: relative;
		} 
```

> 还是![image-20211123202547035](http://myapp.img.mykernel.cn/image-20211123202547035.png)
>
> 与父母无关系

###### 浏览器滚动时，始终固定

现在给body添加高度

```diff
		body {
			height: 1500px;
		}
```

> 滚动条移动过程中，固定定位的图像始终不会动。也不会占用标准流
>
> ![image-20211123202837193](http://myapp.img.mykernel.cn/image-20211123202837193.png)

###### 浏览器的可视窗口，而非浏览器实际大小

现在让猴子到右下脚

```diff
		.box img {
			position: fixed;
			bottom: 0;
			right: 0;
		}
```

> ![image-20211123203317272](http://myapp.img.mykernel.cn/image-20211123203317272.png)
>
> 注意，浏览器的实际大小是1500px高度，从滚动条可以看到下面还有一部分属于浏览器。而我们指定的bottom, right 0 是浏览器的可视窗口的位置

##### 新浪广告 案例

###### 分析页面结构

1. 页面内容区
2. 页面广告位

![image-20211123203921577](http://myapp.img.mykernel.cn/image-20211123203921577.png)

###### 准备内容区

根据老师提供的图片的大小来定页面区大小

页面区不给定高度

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.box {
			margin: 0 auto;
			width: 1002px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div class="box">
		
	</div>
</body>
</html>
```

> 现在页面区正常了

添加图片到内容区

```diff
	<div class="box">
+		<img src="http://myapp.img.mykernel.cn/box.png" alt="">
	</div>
</body>
```

> 内容区已经搞定
>
> ![image-20211123204352212](http://myapp.img.mykernel.cn/image-20211123204352212.png)

###### 现在添加左侧广告

```diff
<body>
+	<img class="ad-l" src="http://myapp.img.mykernel.cn/ad-l.png" alt="">
	<div class="box">
		<img src="http://myapp.img.mykernel.cn/box.png" alt="">
	</div>
</body>
```

> ![image-20211123204609507](http://myapp.img.mykernel.cn/image-20211123204609507.png)
>
> 图片在box上面，标准流肯定是上下结构
>
> 现在需要固定box, 不占用标准流位置，标准流正常放

```diff
		.ad-l {
			position: fixed;
			top: 200px;
		}
```

> 现在左侧就正常了

###### 现在添加右侧广告

```diff
+		.ad-r {
			position: fixed;
			top: 200px;
			right: 0;
		}
	</style>
</head>
<body>
	<img class="ad-l" src="http://myapp.img.mykernel.cn/ad-l.png" alt="">
+	<img class="ad-r" src="http://myapp.img.mykernel.cn/ad-l.png" alt="">
	<div class="box">
		<img src="http://myapp.img.mykernel.cn/box.png" alt="">
	</div>
</body>
</html>
```

> ![image-20211123204931221](http://myapp.img.mykernel.cn/image-20211123204931221.png)

### 定位扩展

#### 绝对定位的盒子怎么居中对齐

##### 普通盒子`margin: auto`剧中对齐

普通盒子可以使用margin: auto水平居中

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			width: 200px;
			height: 200px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div></div>
</body>
</html>
```

> ![image-20211123205324932](http://myapp.img.mykernel.cn/image-20211123205324932.png)

居中

```diff
		div {
+			margin: auto;
			width: 200px;
			height: 200px;
			background-color: pink;
		}
```

> ![image-20211123205414121](http://myapp.img.mykernel.cn/image-20211123205414121.png)

##### relative相对定位居中对齐 `margin: auto`

```diff
		div {
+			position: relative;
			margin: auto;
			width: 200px;
			height: 200px;
			background-color: pink;
		}
```

> ![image-20211123205414121](http://myapp.img.mykernel.cn/image-20211123205414121.png)
>
> 会占用标准流的位置，所以正常

##### absolute绝对定位 居中对齐

```diff
		div {
+			position: absolute;
			margin: auto;
			width: 200px;
			height: 200px;
			background-color: pink;
		}
```

> 绝对定位时，`margin: auto`不能让盒子居中对齐
>
> ![image-20211123205324932](http://myapp.img.mykernel.cn/image-20211123205324932.png)

```diff
		div {
+			position: absolute;
+			top: 0;
+			left: 50%;                  /* 居中的右边*/
+			margin-left: -100px;        /* 向左走自身宽度的1半 */
			
			/*margin: auto;*/
			width: 200px;
			height: 200px;
			background-color: pink;
		}
```

![image-20211123205414121](http://myapp.img.mykernel.cn/image-20211123205414121.png)

水平居中公式的原理 `left: 50%, margin-left: -自身宽度1半`

![image-20211123210820531](http://myapp.img.mykernel.cn/image-20211123210820531.png)

absolute绝对定位 水平居中 `top: 50%, margin-top: -自身高度的1半`

![image-20211123211405375](http://myapp.img.mykernel.cn/image-20211123211405375.png)



#### 堆叠顺序

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.damao,
		.ermao,
		.sanmao {
			width: 200px;
			height: 200px;
			background-color: red;
		}

		.ermao {
			background-color: green;
		}

		.sanmao {
			background-color: blue;
		}
	</style>
</head>
<body>
	<div class="damao"></div>
	<div class="ermao"></div>
	<div class="sanmao"></div>
</body>
</html>
```

> ![image-20211123211656879](http://myapp.img.mykernel.cn/image-20211123211656879.png)
>
> 标准流的3个盒子，自上而下的显示 



##### 给所有盒子添加相对定位

```diff
		.damao,
		.ermao,
		.sanmao {
+			position: relative;
			
			width: 200px;
			height: 200px;
			background-color: red;
		}
```

> 依然占用原来的位置

##### 给所有盒子添加绝对定位, 后者居上

```diff
		.damao,
		.ermao,
		.sanmao {
+			position: absolute;

			width: 200px;
			height: 200px;
			background-color: red;
		}
```

> 只有一个盒子，蓝色。而后的定位会压着之前的盒子
>
> ![image-20211123211922726](http://myapp.img.mykernel.cn/image-20211123211922726.png)

现在给ermao, sanmao位置

```diff
		.ermao {
+			top: 50px;
+			left: 50px;
			background-color: green;
		}

		.sanmao {
+			top: 100px;
+			left: 100px;
			background-color: blue;
		}
```

> ![image-20211123212129705](http://myapp.img.mykernel.cn/image-20211123212129705.png)
>
> 现在的3个盒子已经叠加在一起了

##### 修改叠放顺序 z-index, 默认0，范围正整数/负整数/0，值越大越靠上

- 只有在定位的元素中才有z-index
- 没有单位，font-weight(400, 700)也没有单位

```diff
		.ermao {
			top: 50px;
			left: 50px;
			background-color: green;

+			z-index: 1;
		}

```

> ![image-20211124194202183](http://myapp.img.mykernel.cn/image-20211124194202183.png)

把红色放最上面

```css
		.damao {
			z-index: 2;
		}
```

![image-20211124194314164](http://myapp.img.mykernel.cn/image-20211124194314164.png)

#### 定位改变display属性

##### 标准流块元素, 不给width通览显示

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			/*标准流块元素不给宽度，通览显示*/
			height: 100px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div></div>
</body>
</html>
```

> ![image-20211124194708873](http://myapp.img.mykernel.cn/image-20211124194708873.png)

##### 转换为行内显示元素,  不给width默认为内容宽度

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
			/*标准流行内块元素不给宽度，看内容占据大小*/
+			display: inline-block;
			height: 100px;
			background-color: pink;
		}
	</style>
</head>
<body>
+	<div>1234567</div>
</body>
</html>
```

> ![image-20211124194835536](http://myapp.img.mykernel.cn/image-20211124194835536.png)

##### 标准流块转换成浮动后，类似行内块，脱标

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {

+			float: left;
			height: 100px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div>1234567</div>
</body>
</html>
```

> ![image-20211124194835536](http://myapp.img.mykernel.cn/image-20211124194835536.png)

##### 标准块转换成绝对/固定定位, 类似行内块，脱标

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
+			position: fixed;
			height: 100px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div>1234567</div>
</body>
</html>
```

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		div {
+			position: absolute;
			height: 100px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div>1234567</div>
</body>
</html>
```

> ![image-20211124195534321](http://myapp.img.mykernel.cn/image-20211124195534321.png)

### 新浪导航完善

```diff
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.box {
			margin: 0 auto;
			width: 1002px;
			height: 1760px;
			background-color: pink;
		}

		.ad-l {
			position: fixed;
			top: 200px;
		}
		.ad-r {
			position: fixed;
			top: 200px;
			right: 0;
		}
		* {
			margin: 0;
			padding: 0;
		}
+		.top {
			height: 44px;
			background-color: pink;
		}
	</style>
</head>
<body>
+	<div class="top"></div>
	<img class="ad-l" src="http://myapp.img.mykernel.cn/ad-l.png" alt="">
	<img class="ad-r" src="http://myapp.img.mykernel.cn/ad-l.png" alt="">
	<div class="box">
		<img src="http://myapp.img.mykernel.cn/box.png" alt="">
	</div>
</body>
</html>
```

> ![image-20211124200153768](http://myapp.img.mykernel.cn/image-20211124200153768.png)
>
> 现在看已经有一个top了

现在添加固定定位，一旦添加固定定位就拥有行内块特性，内容宽度就是显示的宽度，所以会消失

```diff
		.top {
+			position: fixed;
			height: 44px;
			background-color: pink;
		}
```

> ![image-20211124200319023](http://myapp.img.mykernel.cn/image-20211124200319023.png)

现在给top盒子添加图片

```diff
	<div class="top">
		<img src="http://myapp.img.mykernel.cn/top.png" alt="">
	</div>
```

> ![image-20211124200537569](http://myapp.img.mykernel.cn/image-20211124200537569.png)

#### 定位盒子缩小图时, 不会剧中了。w100%, 并居中

![image-20211124201411689](http://myapp.img.mykernel.cn/image-20211124201411689.png)

> 上面的导航在左边

行内块使用`text-align: cetner`,

```diff
		.top {
			position: fixed;
			top: 0;
			height: 44px;

			background-color: pink;
+			text-align: center;
		}
```

> ![image-20211124201411689](http://myapp.img.mykernel.cn/image-20211124201411689.png)

结果还是有问题，居中对齐没有作用，因为宽度没有给，默认的宽度是内容的宽度就是图片的宽度

此时给top盒子宽度为通览

```diff
		.top {
			position: fixed;
			top: 0;
			height: 44px;
+			width: 100%;
			background-color: pink;
			text-align: center;
		}
```

> 现在的位置就正常了
>
> ![image-20211124201703290](http://myapp.img.mykernel.cn/image-20211124201703290.png)

#### 内容区域被遮住了

绝对或固定定位，会脱标

现在把内容向下走

```diff
		.box {
+			margin: 44px auto 0;
			width: 1002px;
			height: 1760px;
			background-color: pink;
		}
```

> ![image-20211124201940281](http://myapp.img.mykernel.cn/image-20211124201940281.png)

#### 定位和浮动 解决子外边距让父盒子塌陷的问题

##### 标准流父子，margin-top会造成父盒子塌陷 解决方式

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		.father {
			width: 500px;
			height: 500px;
			background-color: pink;
		}

		.son {
			width: 200px;
			height: 200px;
			background-color: purple;
		}
	</style>
</head>
<body>
	<div class="father">
		<div class="son"></div>
	</div>
</body>
</html>
```

> ![image-20211124202343999](http://myapp.img.mykernel.cn/image-20211124202343999.png)

margin-top会造成父盒子塌陷

```diff
+		.son {
			margin-top: 100px;
			width: 200px;
			height: 200px;
			background-color: purple;
		}
```

> ![image-20211124202436014](http://myapp.img.mykernel.cn/image-20211124202436014.png)

解决方法：

- padding
- overflow
- 伪类
- 双伪类

##### son/father浮动时

###### son

```diff
		.father {
			width: 500px;
			height: 500px;
			background-color: pink;
		}
		.son {
+			float: left;
			margin-top: 100px;
			width: 200px;
			height: 200px;
			background-color: purple;
		}
```

> ![image-20211124202533557](http://myapp.img.mykernel.cn/image-20211124202533557.png)

###### fahter

```diff
		.father {
+			float: left;
			width: 500px;
			height: 500px;
			background-color: pink;
		}
		.son {
			margin-top: 100px;
			width: 200px;
			height: 200px;
			background-color: purple;
		}
```

> ![image-20211124202533557](http://myapp.img.mykernel.cn/image-20211124202533557.png)

##### son/father定位

###### 相对

会保留标准流，所以仍然会塌陷

###### 绝对定位 

```diff
		.father {
			width: 500px;
			height: 500px;
			background-color: pink;
		}
        .son {
+			position: absolute;
			margin-top: 100px;
			width: 200px;
			height: 200px;
			background-color: purple;
		}
```

> ![image-20211124202709061](http://myapp.img.mykernel.cn/image-20211124202709061.png)

或者父绝

```diff
		.father {
+			position: absolute;
			width: 500px;
			height: 500px;
			background-color: pink;
		}

		.son {
			margin-top: 100px;
			width: 200px;
			height: 200px;
			background-color: purple;
		}
```

> ![image-20211124202709061](http://myapp.img.mykernel.cn/image-20211124202709061.png)

### 淘宝轮播图综合案例

串联定位

#### 外型

现在访问：https://www.taobao.com/

![image-20211126101132765](http://myapp.img.mykernel.cn/image-20211126101132765.png)

> 1. 左右的三角点击之后可以切换图片
> 2. 下面几个圆点点击之后同样可以切换图片

切换的效果通过JS完成，目前只是完成结构

#### 结构分析

![image-20211126101424022](http://myapp.img.mykernel.cn/image-20211126101424022.png)

> 1. 大盒子
> 2. 里面有图片，产品图统一使用插入图片，而不是背景图片，不重要的小图标才使用背景图片
> 3. 左右有2个盒子垂直居中
> 4. 下面的盒子水平居中

#### 实现结构

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Document</title>
	<style>
		* {
			margin: 0;
			padding: 0;
		}

		.taobao {
			margin: 100px auto;
			width: 520px;
			height: 280px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div class="taobao">
		<img src="https://gtms03.alicdn.com/tps/i3/TB1gXd1JXXXXXapXpXXvKyzTVXX-520-280.jpg" alt="">
	</div>
</body>
</html>
```

> ![image-20211126102639166](http://myapp.img.mykernel.cn/image-20211126102639166.png)

#### 放左按钮

![image-20211126102823490](http://myapp.img.mykernel.cn/image-20211126102823490.png)

```diff
	<div class="taobao">
+		<a href="#" class="arr-l"> &lt; </a>
		<img src="https://gtms03.alicdn.com/tps/i3/TB1gXd1JXXXXXapXpXXvKyzTVXX-520-280.jpg" alt="">
	</div>
```

> ![image-20211126103003298](http://myapp.img.mykernel.cn/image-20211126103003298.png)

##### 现在需要a压住图，

- 标准流，只能上下、左右

- 浮动，脱标, 不能压住图片和文字

  ```css
  		.arr-l {
  			float: left;
  		}
  ```

- 定位：
  - 相对：与标准流差不多
  - 绝对：脱标
  - 静态：不加定位
  - 固定：脱标，滚动屏幕时，固定定位的盒子始终在可视区

##### 只能定位。

- a使用相对，自己也占了标准流，结果还是这样

  ![image-20211126103003298](http://myapp.img.mykernel.cn/image-20211126103003298.png)

- a使用绝对，自己不占标准流，图片就在恢复原样了，a也在其上

  ```css
  		.arr-l {
  			position: absolute;
  		}
  ```

  ![image-20211126104045689](http://myapp.img.mykernel.cn/image-20211126104045689.png)

```css
		.arr-l {
			position: absolute;
			/*绝对定位的盒子无需转换，直接给大小*/
			width: 20px;
			height: 30px;
			background-color: pink;
		}
```

> ![image-20211126104253356](http://myapp.img.mykernel.cn/image-20211126104253356.png)

##### 绝对定位给位置, 相对于有定位的父盒子起始

```diff
		.arr-l {
			position: absolute;
+			top: 0;
+			left: 0;
			/*绝对定位的盒子无需转换，直接给大小*/
			width: 20px;
			height: 30px;
			background-color: pink;
		}
```

> ![image-20211126104405760](http://myapp.img.mykernel.cn/image-20211126104405760.png)

##### 绝对定位相对父盒子起，子绝父相保留父位置

```diff
		.taobao {
+			position: relative;
			margin: 100px auto;
			width: 520px;
			height: 280px;
			background-color: pink;
		}
```

> ![image-20211126104510704](http://myapp.img.mykernel.cn/image-20211126104510704.png)

##### 现在让这个盒子垂直居中,  垂直居中公式计算

```diff
		.arr-l {
			position: absolute;
+			top: 50%;
+			margin-top: -15px; /* 自身盒子的一半 */
			left: 0;
			/*绝对定位的盒子无需转换，直接给大小*/
			width: 20px;
			height: 30px;
			background-color: pink;
		}
```

> ![image-20211126104633556](http://myapp.img.mykernel.cn/image-20211126104633556.png)

##### 让a半透明

```css
		.arr-l {
			position: absolute;
			top: 50%;
			margin-top: -15px;
			left: 0;
			/*绝对定位的盒子无需转换，直接给大小*/
			width: 20px;
			height: 30px;

+			background: rgba(0, 0, 0, .2);
		}
```

> ![image-20211126110459667](http://myapp.img.mykernel.cn/image-20211126110459667.png)

##### 让a 下划线，颜色白色，垂直居中, 水平居中

```diff
		.arr-l {
			position: absolute;
			top: 50%;
			margin-top: -15px;
			left: 0;
			/*绝对定位的盒子无需转换，直接给大小*/
			width: 20px;
			height: 30px;
+           text-align: center;
+			text-decoration: none;
+			color: #fff;
+			line-height: 30px;
			background: rgba(0, 0, 0, .2);
		}
```

> ![image-20211126110634862](http://myapp.img.mykernel.cn/image-20211126110634862.png)

#####  border-radius 添加圆角

```diff
		.arr-l {
			position: absolute;
			top: 50%;
			margin-top: -15px;
			left: 0;
			/*绝对定位的盒子无需转换，直接给大小*/
			width: 20px;
			height: 30px;

			text-decoration: none;
			color: #fff;
			line-height: 30px;
			background: rgba(0, 0, 0, .2);

+			border-radius: 15px;
		}
```

![image-20211126110844298](http://myapp.img.mykernel.cn/image-20211126110844298.png)

左上和左下变了，右上和右下不变

而实际是左上和左下不变，右上和右下变化。

##### border-*-radius 控制不同方位的圆角

top-left, top-right, bottom-right, bottom-left

```diff

		.arr-l {
			position: absolute;
			top: 50%;
			margin-top: -15px;
			left: 0;
			/*绝对定位的盒子无需转换，直接给大小*/
			width: 20px;
			height: 30px;

			text-decoration: none;
			color: #fff;
			line-height: 30px;
+			/*text-align: center;*/
			background: rgba(0, 0, 0, .2);

+			border-top-right-radius: 15px;
+			border-bottom-right-radius: 15px;
		}
```

1. 有点偏右了，所以不需要文本水平居中

> ![image-20211126112120388](http://myapp.img.mykernel.cn/image-20211126112120388.png)

#### 放右按钮 border-radius 简写

border-radius: 15px, 所有顶点相同弧度

border-radius: 左上 右上 右下 左下，顶点顺时针



```diff
+		.arr-r {
			position: absolute;
			top: 50%;
			margin-top: -15px;
			right: 0;
			width: 20px;
			height: 30px;

			text-align: center;
			line-height: 30px;
			text-decoration: none;
			color: #fff;
			background-color: rgba(0, 0, 0, .2);
			border-radius: 15px 0 0 15px;
		}
	</style>
</head>
<body>
	<div class="taobao">
		<a href="#" class="arr-l"> &lt; </a>
		<img src="https://gtms03.alicdn.com/tps/i3/TB1gXd1JXXXXXapXpXXvKyzTVXX-520-280.jpg" alt="">
+		<a href="#" class="arr-r"> &gt; </a>
	</div>
</body>
</html>
```

##### 优化左右按钮的代码

相同代码使用并集选择器

```css
		.arr-l, 
        .arr-r {
			position: absolute;
			top: 50%;
			margin-top: -15px;

			/*绝对定位的盒子无需转换，直接给大小*/
			width: 20px;
			height: 30px;

			text-decoration: none;
			color: #fff;
			line-height: 30px;
			background: rgba(0, 0, 0, .2);
		}

		.arr-l {
			left: 0;
			text-align: left;
			border-top-right-radius: 15px;
			border-bottom-right-radius: 15px;
		}
		.arr-r {
			right: 0;
			text-align: right;
			border-radius: 15px 0 0 15px;
		}
```

##### 左按钮和右按钮，鼠标悬浮时会变色

```diff
		.arr-l:hover, 
		.arr-r:hover {
			background-color: rgba(0, 0, 0, .4);
		}
```

鼠标在上面时会更重的颜色

![image-20211126113616184](http://myapp.img.mykernel.cn/image-20211126113616184.png)

#### 下面模块分析

![image-20211126113752055](http://myapp.img.mykernel.cn/image-20211126113752055.png)

1. 排列非常整齐
2. 外层包围内层， ul, li
3. ul大小，li给圆

#### 分析ul, li大小

![image-20211126114019366](http://myapp.img.mykernel.cn/image-20211126114019366.png)





#### 给结构

```diff
+		.circle {
			width: 70px;
			height: 13px;
			background-color: pink;
		}
	</style>
</head>
<body>
	<div class="taobao">
		<a href="#" class="arr-l"> &lt; </a>
		<img src="https://gtms03.alicdn.com/tps/i3/TB1gXd1JXXXXXapXpXXvKyzTVXX-520-280.jpg" alt="">
		<a href="#" class="arr-r"> &gt; </a>

+		<ul class="circle"></ul>
	</div>
```

> ![image-20211126114243741](http://myapp.img.mykernel.cn/image-20211126114243741.png)

#### 让其在图片上水平剧中

```diff
		.circle {
+			position: absolute;
+			bottom: 0;
+			left: 50%;
+			margin-left: -35px;			
			width: 70px;
			height: 13px;
			background-color: pink;
		}
```

> ![image-20211126114416407](http://myapp.img.mykernel.cn/image-20211126114416407.png)

下面的高度

![image-20211126114433850](http://myapp.img.mykernel.cn/image-20211126114433850.png)

```diff
		.circle {
			position: absolute;
+			bottom: 10px;
			left: 50%;
			margin-left: -35px;			
			width: 70px;
			height: 13px;
			background-color: pink;
		}
```

![image-20211126114520065](http://myapp.img.mykernel.cn/image-20211126114520065.png)

#### 给ul添加背景浅白色

```css
		.circle {
			position: absolute;
			bottom: 10px;
			left: 50%;
			margin-left: -35px;			
			width: 70px;
			height: 13px;
+			background-color: rgba(255, 255, 255, .3);
		}
```

> ![image-20211126114700567](http://myapp.img.mykernel.cn/image-20211126114700567.png)

#### 给ul的<strong style="color: red;">4边</strong>添加弧度

```diff
		.circle {
			position: absolute;
			bottom: 10px;
			left: 50%;
			margin-left: -35px;			
			width: 70px;
			height: 13px;
			background-color: rgba(255, 255, 255, .3);
+			border-radius: 15px;
}		
```

> ![image-20211126114804996](http://myapp.img.mykernel.cn/image-20211126114804996.png)

#### 往里面加li

![image-20211126114035076](http://myapp.img.mykernel.cn/image-20211126114035076.png)

```html
	<div class="taobao">
		<a href="#" class="arr-l"> &lt; </a>
		<img src="https://gtms03.alicdn.com/tps/i3/TB1gXd1JXXXXXapXpXXvKyzTVXX-520-280.jpg" alt="">
		<a href="#" class="arr-r"> &gt; </a>

		<ul class="circle">
			<li></li>
			<li></li>
			<li></li>
			<li></li>
			<li></li>
		</ul>
	</div>
```

```css
		.circle li {
			width: 8px;
			height: 8px;
			background-color: pink;
		}
```

![image-20211126115041430](http://myapp.img.mykernel.cn/image-20211126115041430.png)

#### 去掉li的默认样式

```css
		* {
			margin: 0;
			padding: 0;
		}
		li {
			list-style: none;
		}
```

![image-20211126115134993](http://myapp.img.mykernel.cn/image-20211126115134993.png)

#### li 横向排列

```diff
		.circle li {
+			float:  left;
			width: 8px;
			height: 8px;
			background-color: pink;
		}
```

![image-20211126115216878](http://myapp.img.mykernel.cn/image-20211126115216878.png)

#### li小圆点 

```diff
		.circle li {
			float:  left;
			width: 8px;
			height: 8px;
			background-color: pink;
+			border-radius: 50%;
		}
```

![image-20211126115334860](http://myapp.img.mykernel.cn/image-20211126115334860.png)

#### 给小li距离

上

![image-20211126115437761](http://myapp.img.mykernel.cn/image-20211126115437761.png)

左

![image-20211126115528561](http://myapp.img.mykernel.cn/image-20211126115528561.png)

中间

![image-20211126115543043](http://myapp.img.mykernel.cn/image-20211126115543043.png)

```diff
		.circle li {
			float:  left;
+			margin: 3px;
			width: 8px;
			height: 8px;
			background-color: pink;
			border-radius: 50%;
		}
```

![image-20211126115627197](http://myapp.img.mykernel.cn/image-20211126115627197.png)

### 鼠标选择哪个就变色

```diff
		.circle li {
			float:  left;
			margin: 3px;
			width: 8px;
			height: 8px;
+			background-color: #fff;
			border-radius: 50%;
		}

		/*当前选择*/
		.current {
+			background-color: #ff5000;
		}
	</style>
</head>
<body>
	<div class="taobao">
		<a href="#" class="arr-l"> &lt; </a>
		<img src="https://gtms03.alicdn.com/tps/i3/TB1gXd1JXXXXXapXpXXvKyzTVXX-520-280.jpg" alt="">
		<a href="#" class="arr-r"> &gt; </a>

		<ul class="circle">
			<li></li>
+			<li class="current"></li>
			<li></li>
			<li></li>
			<li></li>
		</ul>
```

1. 通过在li标签上添加一个类名为`current` 表示 当前在此li
2. 然后修改此li的颜色

![image-20211126120011103](http://myapp.img.mykernel.cn/image-20211126120011103.png)

# CSS第7天

