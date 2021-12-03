---
title: Jenkins批量删除历史构建
date: 2021-12-03 20:08:36
categories: Jenkins
toc: true
---

&emsp;&emsp;路径：项目管理 ----》 脚本命令行 ---》放入下面的脚本
    

```
def jobName = "cdzq"   //删除的项目名称
def maxNumber = 600    // 保留的最小编号，意味着小于该编号的构建都将被删除

Jenkins.instance.getItemByFullName(jobName).builds.findAll {
  it.number <= maxNumber
}.each {
  it.delete()
}
```
