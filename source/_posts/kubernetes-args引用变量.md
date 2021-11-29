---
title: "kubernetes args引用变量"
date: "2021-07-14 10:01:11"
tags:
- "kubernetes"
---

# 前言

今天配置apollo需要args添加变量，使用${}, 一直不成功，故记录一下

<!--more-->
# 解释

```bash
pos@1227f652bef0:~/yaml$ kubectl explain pod.spec.containers.args
KIND:     Pod
VERSION:  v1

FIELD:    args <[]string>

DESCRIPTION:
     Arguments to the entrypoint. The docker image's CMD is used if this is not
     provided. Variable references $(VAR_NAME) are expanded using the
     container's environment. If a variable cannot be resolved, the reference in
     the input string will be unchanged. The $(VAR_NAME) syntax can be escaped
     with a double $$, ie: $$(VAR_NAME). Escaped references will never be
     expanded, regardless of whether the variable exists or not. Cannot be
     updated. More info:
     https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#running-a-command-in-a-shell
```

传递给entrypoint的参数。

如果没有提供entrypoint, 就将参数传递给docker image的CMD

变量引用 $(VAR_NAME) 被解释为容器的环境变量

如果变量不能解析就保持不变。

变量引用可以被转义，使用2个$$, 例如：$$(VAR_NAME)，转义意味着不会被解释，不管变量是否存在或者不存在。



