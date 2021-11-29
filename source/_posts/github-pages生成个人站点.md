---
title: 0元撸github-pages生成个人站点
tags:
- hexo
- CI
- github
photos:
- http://myapp.img.mykernel.cn/github-pages.png
date: 2021-11-15 10:27:58
---

# 前言

>  发起此文灵感的项目: https://github.com/JamesIves/github-pages-deploy-action#--github-pages-deploy-action-rocket
>
> - [Github Actions](https://github.com/features/actions) 后面会简介Actions。
> -  [GitHub Pages](https://pages.github.com/) 此网页中有会视频介绍。

了解`CICD工具: jenkins`的朋友肯定知道`webhook`

![Generic Webhook Trigger Usage Example](http://myapp.img.mykernel.cn/68747470733a2f2f696d672e796f75747562652e636f6d2f76692f386d724a4e6b6f667871342f302e6a7067)

1. jenkins会生成一个类似格式的URL: `JENKINS_URL/generic-webhook-trigger/invoke`
2. git仓库可以基于git事件触发这个URL
   - 可以触发webook
     - [Bitbucket Cloud](https://confluence.atlassian.com/bitbucket/manage-webhooks-735643732.html)
     - [Bitbucket Server](https://confluence.atlassian.com/bitbucketserver/managing-webhooks-in-bitbucket-server-938025878.html)
     - [GitHub](https://developer.github.com/webhooks/)
     - [GitLab](https://docs.gitlab.com/ce/user/project/integrations/webhooks.html)
     - [Gogs](https://gogs.io/docs/features/webhook) and [Gitea](https://docs.gitea.io/en-us/webhooks/)
     - [Assembla](https://blog.assembla.com/AssemblaBlog/tabid/12618/bid/107614/Assembla-Bigplans-Integration-How-To.aspx)
     - [Jira](https://developer.atlassian.com/server/jira/platform/webhooks/)
     - And many many more!
   - git事件: https://docs.github.com/en/developers/webhooks-and-events/events/github-event-types
     - push 无论啥分支只是commit之后，push到远程仓库，远程仓库就会有push事件
     - pr/pull/merge事件，只是从一个分支发起一个pull request, 就会产生此事件
3. jenkins完成拉代码、构建、上传打包结果到存储库、邮件通知



上面这个git event触发远程URL是一个通用的操作，而git仓库也有自带的对事件发生之后，执行一系列的脚本命令的方法。这个叫`Github Actions`. 阿里云有一个流水线。gitlab有一个`gitlib-ci流水线`

以下将简介`github actions`用法

<!--more-->



# Github Actions

## github工作流

此名称其实就是定义git仓库的一系列事件，并伴随一系列的命令，帮助开发者完成CI/CD， 完成自定义的工作流。

![overview-actions-simple](http://myapp.img.mykernel.cn/overview-actions-simple.png)

1. git事件会触发一个工作流
2. 一个工作流中包含多个job，每个job是并行执行
3. 每个job中使用steps内嵌多个item, 每个item可以定义脚本。每个job会在相同的runner上串行运行。



## 先看一个示例

1. In your repository, create the `.github/workflows/` directory to store your workflow files.
2. In the `.github/workflows/` directory, create a new file called `learn-github-actions.yml` and add the following code. 

```yaml
name: learn-github-actions 
on: [push] 
jobs:
  check-bats-version:
    runs-on: ubuntu-latest 
    steps:

    - uses: actions/checkout@v2 # 下载push代码到runner
    - uses: actions/setup-node@v2 # 安装指定包
      with:
        node-version: '14'
    - run: npm install -g bats 
    - run: bats -v
  my-job:
    name: My job
    runs-on: ubuntu-latest
    steps:
    - name: Setup Node.js environment
      uses: actions/setup-node@v2.4.1
      with:
        # Set always-auth in npmrc
        always-auth: false # optional, default is false
        # Version Spec of the version to use.  Examples: 12.x, 10.15.1, >=10.15.0
        node-version: 17 # optional
```

3. Commit these changes and push them to your GitHub repository.

## workflow文件理解

`name: learn-github-actions`   名称会在`github`中的Actions标签中出现

![image-20211115123842975](http://myapp.img.mykernel.cn/image-20211115123842975.png)

> 对于这个Actions界面，更详细的介绍, 参考: https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#viewing-the-jobs-activity

`on: [push]` 指定触发此工作流的事件，此处使用push,表示只要推了代码就会触发。只在某些分支上触发时，参考：https://docs.github.com/actions/reference/workflow-syntax-for-github-actions#onpushpull_requestpaths

`jobs` 可以归组多个job

`runs-on: ubuntu-latest`  runner是ubuntu, 可以使用其它runner, https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idruns-on

`- uses: actions/checkout@v2` 可以调用公开仓库、与代码同仓库、docker hub上的image，这些作为action, 更详细的参见后面的actions介绍 

`- run: npm install -g bats `  多个并列的run关键字，通知runner执行啥子命令



以上文件的第1个job的图表

![overview-actions-event](http://myapp.img.mykernel.cn/overview-actions-event.png)

## action

action获取

- https://github.com/marketplace/actions/ 
- 在github上编辑workflow, 右侧会有获取action的市场

更新action: https://docs.github.com/en/github/administering-a-repository/keeping-your-actions-up-to-date-with-dependabot

> 让action始终保持最新

引用action:

- A public repository： 建议使用SHA, https://docs.github.com/en/actions/learn-github-actions/finding-and-customizing-actions#using-release-management-for-your-custom-actions
- The same repository where your workflow file references the action：自定义action: https://docs.github.com/en/actions/learn-github-actions/finding-and-customizing-actions#using-inputs-and-outputs-with-an-action
- A published Docker container image on Docker Hub https://docs.github.com/en/actions/learn-github-actions/finding-and-customizing-actions#referencing-a-container-on-docker-hub





# 示例github workflow

引用此仓库的示例: https://github.com/slcnx/github-pages-deploy-action

代码结构如下:

```
|-- github-pages-deploy-action (repository)
|   |__ .github
|       └── workflows
|           └── learn-github-actions.yml
```

示例的`learn-github-actions.yml`

```yaml
name: learn-github-actions 
on: [push] 
jobs:
  check-bats-version:
    runs-on: ubuntu-latest 
    steps:

    - uses: actions/checkout@v2 # 下载push代码到runner
    - uses: actions/setup-node@v2 # 安装指定包
      with:
        node-version: '14'
    - run: npm install -g bats 
    - run: bats -v
```

当我们推送代码之后，就可以在actions界面中查看执行结果

https://github.com/slcnx/github-pages-deploy-action/runs/4206695692?check_suite_focus=true

![image-20211115125147294](http://myapp.img.mykernel.cn/image-20211115125147294.png)

> 注意：这个是我第一次做示例的结果，而代码仓库中多了一个job, 是另一个示例的，对应的workflow不是当前这个。

# 配置github-pages

原理应该是通过这个流水线：使用hexo构建、将源代码上传到`github-pages`的仓库中

因为目前我使用hexo, 自己搭建的nginx + 自己定制的接收webhook处理的一系列命令，完全够用了，而且还可以基于自己的nginx做认证限制，故就不使用此方案了。

> 对`hexo` +`nginx`方式感兴趣，可以使用看此文。
>
> http://blog.mykernel.cn/2020/09/27/%E6%90%AD%E5%BB%BA%E8%87%AA%E5%8A%A8%E5%8F%91%E5%B8%83%E5%8D%9A%E5%AE%A2hexotypora%E4%B8%83%E7%89%9B%E4%BA%91%E5%AD%98%E5%82%A8/

# github-pages访问慢

此项目的代理可以解决

https://github.com/slcnx/FastGithub
