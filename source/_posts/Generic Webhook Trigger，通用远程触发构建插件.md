---
title: Generic Webhook Trigger，通用远程触发构建插件
date: 2021-12-03 19:48:46
categories: Jenkins
toc: true
---

&emsp;&emsp;如果使用jenkins自由风格构建的GitHub hook trigger for GITScm polling+Use secret text(s) or file(s)的方式，那么1个git仓库就只能有一个webhook钩子，无法在1个仓库下针对多个项目设置钩子，也无法针对分支名等参数选择构建。
    
	
# 插件优点   

>  1.接收任何 JENKINS_URL/generic-webhook-trigger/invoke格式的HTTP 请求   
>  2.提取来自POST带有JSONPath或XPath 、headers或者query参数的值   
>  3.可以使用这些值作为变量触发构建，如：使用触发构建的分支参数（ref）过滤构建指定分支  
>  4.可以使用token参数区别不同的构建项目 


# jenkins设置

>  1.下载generic-webhook-trigger插件，路径：系统管理 > 插件管理   
>  2.在项目中设置对应git仓库的token（Token Credential使用secret text设置token和直接设置一个token的作用和效果是一样的，设置secret text的secret就等同于直接设置token，并且两者的调用方式都是一样的，通过url的param方式），路径：构建触发器 > Generic Webhook Trigger   
*****
&emsp;&emsp;备注：设置前两步并且设置完git就已经完成了。  
&emsp;&emsp;&emsp;&emsp;&emsp;这里还可以设置参数过滤，就比如我只允许master的分支构建。
>  1.设置Post content parameters，Expression是git的请求参数，Variable是设置接收Expression的变量。注意接收参数使用JSONPath格式，并且git也需要设置请求时使用application/json格式   
>  2.设置Optional filter，Expression正则匹配规则，Text为与Expression正则验证的字符串。   

# git设置

>  1.给git项目添加webhooks，路径：项目中的设置 > webhooks > Add webhook   
>![](/images/GenericWebhookTrigger/git-step-1.png)   
>  2.添加Payload URL，格式：http://JENKINS_URL/generic-webhook-trigger/invoke?token=TOKEN_HERE  
>  3.设置Content type为application/json  
>![](/images/GenericWebhookTrigger/git-step-2.png)   
>  4.点击Add webhook添加一个webhook触发钩子  
*****
&emsp;&emsp;备注：添加完成后可以在路径：项目中的设置 > webhooks > Add webhook页面查看钩子和jenkins是否通信成功。 
![](/images/GenericWebhookTrigger/git-step-3.png)   


[Generic Webhook Trigge官方文档](https://plugins.jenkins.io/generic-webhook-trigger/)
