---
title: pycharm-tips-of-the-day
date: 2021-10-15 09:24:20
tags:
- pycharm
---



# 前言

今天打开pycharm看到一个窗口，之前一直忽略，今天比较好奇其中的含义

![image-20211015092507979](http://myapp.img.mykernel.cn/image-20211015092507979.png)

> 当我们点了关闭后， 可以在  Help | Tip of the Day from the main menu. 调出帮助
>
> 如果不记得以下位置，可以`ctrl + shift + A` 输入一些关键字就可以调用动作 （  Esc 和X ）
>
> 双击shift键搜索, class, file, symbol （alt+shift+g,    ctrl +x/ctrl+f, ctrl+alt+shift+n）
>
> 搜索文件，在文件视图，直接输入名称，即可快速定位文件。
>
> 搜索设置列表: 双击shift, /开头

<!--more-->
# 质量

## 代码质量分析

**Code|Inspect Code** 可以对整个项目或自定义域进行代码分析，并在独立的窗口展示结果

## 代码检查

ctrl + alt  + shift + I 可以快速检查

## 快速修复代码

Alt + Enter, 通过方向键找功能

# pycharm和代码库或问题联系

## 追踪issue

使用**YouTrack Integration**插件, 直接可以看到此项目的issue, 并且对10个团队免费。

可以持续从github/Jira，Mantis和Redmine导入issue.

官方：https://www.jetbrains.com/youtrack/download/get_youtrack.html#section=incloud

## 关联提交信息和后端issue

通过Settings/Preferences dialog Ctrl+Alt+S 进入设置后， **Version Control | Issue Navigation** 配置提交信息中的匹配模式，及对应的超链接

![image-20211015093453027](http://myapp.img.mykernel.cn/image-20211015093453027.png)

当你的提交信息带上这个ID, 然后你提交之后，在pycharm的git 修订列表中，可以通过点击超链接跳转到issue

![image-20211015093551978](http://myapp.img.mykernel.cn/image-20211015093551978.png)

>  例如github上有一个git issue: https://github.com/WorkerLivesMatter/WorkingTime/issues/65 后面是65， 提交时带上这个65, 就会在此次提交生成此issue的超链接

# File 相关

## 打印

File | Print

使用 `File - $FILE$ - $DATE$ - $TIME$` 变量可以引用日期，时间，文件名

打印结果 

![image-20211015101223818](http://myapp.img.mykernel.cn/image-20211015101223818.png)

## pycharm打开外部文件

直接把外部文件从文件浏览器托到pycharm编辑器中



# git

## 历史

VCS | Local History | Show History.

## 调出之前copy到剪切板的内容列表

windows: windows + v

pycharm: ctrl + shift + v

# 写代码

## 补全

写代码补全，通过tab完成

在补全方法时，会上浮一个视图，只有方法，可以通过 ctrl + shift + i ，打开一个详细视图

![image-20211015172431656](http://myapp.img.mykernel.cn/image-20211015172431656.png)

补全后，右下角有3个点，可以排序

![image-20211015174231702](http://myapp.img.mykernel.cn/image-20211015174231702.png)

> 右侧就是方法对应祥细的代码



## 快速查看类定义

ctrl + shift + i

## 数学运算

双击shift之后，编写`2*8+sqrt(64)`

## 方法间跳转

Alt+向上箭头 and Alt+向下箭头 , 更快的在编辑器中跳转前一个方法，后一个方法

## 快速定位此方法被哪些地方引用

Alt + F7，在方法名上面按

## 注释

ctrl + / 单行注释

ctrl + shift + / 多行注释

## 将多行变一行

选中2行之后，ctrl + shift + j

## 代码块上下移动

ctrl + shift + 上箭头

ctrl + shift + 下箭头

> 需要先选中代码

## 代码快加包装

自动生成if else/ try except, 使用      Ctrl+Alt+T (Code |       Surround With ).    

![image-20211015173705459](http://myapp.img.mykernel.cn/image-20211015173705459.png)

## 覆盖方法

Ctrl+O (Code | Override Methods ), 这样就很容易直接覆盖base class的方法

Ctrl+I (Code | Implement Methods ). 生成抽象类的方法

> ```python
> from abc import abstractmethod
> class Base:
>     @abstractmethod
>     def foo(self):
>         pass
> 
> class Derived(Base):
> ```
>
> 在`Derived` 上Ctrl + I 就可以生成方法
>
> 或者在代码中，父类某个方法抛出`NotImplementedError`异常时
>
> ```python
> class AbstractBase:
>     def abstract_method(self):
>         raise NotImplementedError
> 
> 
> class ImplClass(AbstractBase):  # Same as in example above
> ```
>
> 在`ImplClass` 上就可以快速`ctrl + I` 生成方法

## 过滤TODO

当项目中存在大量的TODO, 在 Scope-Based tab in the TODO tool ，可以基于一些方面来过滤TODO

![image-20211015095748634](http://myapp.img.mykernel.cn/image-20211015095748634.png)



## 变量批量更新

当你修改变量时，可能会在pycharm左侧有一个R图标, 可以快速更新改变的变量。

![image-20211015095927812](http://myapp.img.mykernel.cn/image-20211015095927812.png)

> 下面是factorial 而上面的变量修改为fact时，就会**提示你把其他引用的变量全部更新为fact**.

## 检查正则表达式

在正则表达式上, press Alt+Enter, and select Edit RegExp. 然后就会在你的代码上方出现一个方框，你可以输入匹配正则表达式的样例文本，如果匹配，就会有一个√图标，否则就是!图标，或者你的正则表达式包含一个问题

![image-20211015100358712](http://myapp.img.mykernel.cn/image-20211015100358712.png)

## django live template

If you are going to use Django live templates, make sure that Django is selected as the default template language i

Settings/Preferences | Languages & Frameworks | Template languages 

![image-20211015101514577](http://myapp.img.mykernel.cn/image-20211015101514577.png)



## django跳过一些目录

标识他们为resources

## 多处编辑，选择多行，选择代码的矩形区域

alt 键按着，在代码多处选择后，可以多处同时编辑

![image-20211015102044631](http://myapp.img.mykernel.cn/image-20211015102044631.png)

shift键按着，使用左右方向键选择多行文本

alt + shift  结合鼠标向下托



## 使用Emmet加速html/xml/css开发

通过 Ctrl+Alt+S 进入设置, 进入 Editor | Emmet ， 在这些Emmet | HTML, Emmet | CSS, or Emmet | JSX page 中启勾选 Enable Emmet 检验栏。

![image-20211015102801976](http://myapp.img.mykernel.cn/image-20211015102801976.png)

>  在新建demo.html 后，使用以下代码，然后`tab`键就补全了html
>
> ```html
> .nav>a*5 
> ```

# debug

## 非暂停断点

选择要记录的表达式，按住shift在左行栏点击

![image-20211015170723405](http://myapp.img.mykernel.cn/image-20211015170723405.png)

## python控制台

交互控制台，可以代码补全: alt + / 

查看文档 ctrl + q

## 项目临时文件

![image-20211015173226446](http://myapp.img.mykernel.cn/image-20211015173226446.png)

## 运行时的配置

https://www.jetbrains.com/help/pycharm/2021.1/run-debug-configurations-dialog.html#before-launch-options

## 运行时的input数据

​           Run/Debug Configuration      窗口中，选中  Redirect input     

之后选择从文件中读取数据

# 展示

## 修改pycharm代码样式

安装      EditorConfig    插件

​      To configure the style in EditorConfig, open settings (Ctrl+Alt+S       ), navigate to Editor | Code Style and       select the Enable EditorConfig support       checkbox.    

​      To export the current IDE code style settings into the .editconfig       file, click Export.    

## 工具隐藏和展开，及快速选择某个工具

你需要更多的窗口时，可以在下脚一个按钮, 来隐藏所有工具。

![image-20211015100723329](http://myapp.img.mykernel.cn/image-20211015100723329.png)



需要使用哪个工具时，可以 双击 alt键，就会临时出现所有工具了

> 当我们在这个图标上点击时，就会出现pycharm所有工具名称，可以快速选择。

## 搜索当前文件中的代码

ctrl + f

## 数据库没有库或表生成时

 Database Tools | Forget Cached Schemas. 清理缓存



## 数据库查询

当创建一个数据源，连接上数据库后，会自动生成查询控制台。要打开查询控制台， New | Query Console.

## 数据库误修改

只读模式

​       File | Data sources and select the       necessary data source in the Data Sources       list. On the Options tab, select the Read-only       checkbox.    

## 备份表

把表托到相同数据源的数据库树视图中tables 节点中，并给新名称

## 避免误删移除断点

Press Ctrl+Alt+S, go to **Build, Execution, Deployment | Debugger** and select **Drag to the editor or click with middle mouse button.**

> 现在鼠标点左边的断点不会移除，而是
>
> 1. 把断点向编辑器中托，会移除
> 2. 使用鼠标中键点断点，会移除

## 工具窗口光标到编辑器

ESC：在任何工具窗口中，要快速将光标回复到编辑器中，使用ESC

shift + ESC: 不仅会将光标回到编辑器，而且会关闭当前的工具窗口

> 当前在python 控制台
>
> ![image-20211015103851029](http://myapp.img.mykernel.cn/image-20211015103851029.png)
>
> 现在测试 esc 和shift + esc



## 快速打开编辑器文件对应的文件浏览器

在编辑器上方的tab中，ctrl 点文件名即可

![image-20211015104214816](http://myapp.img.mykernel.cn/image-20211015104214816.png)



## 主题颜色修改

ctrl + shift + A, 输入 jump, 点 Jump to Colors and Fonts

general code

![image-20211015164516343](http://myapp.img.mykernel.cn/image-20211015164516343.png)

> ctrl + ` 可以快速切换

## menu解释

当鼠标放在menu的item上时，pycharm左下角会出现当前item的简洁解释

