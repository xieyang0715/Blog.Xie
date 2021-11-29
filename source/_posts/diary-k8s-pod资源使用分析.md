---
title: "k8s pod资源使用分析"
date: 2021-05-12 10:39:38
tags:
- 个人日记
---

# 前言
<!--more-->

今天在启动k8s时，遇到一个问题，自己制作的nginx镜像，启动在k8s环境中，和测试环境一样的资源使用率，竟然启动不了, 后来调整nginx打开的文件描述符后正常。



<!--more-->
# 不同资源测试不同的并发

```bash
        resources:
          limits:
            cpu: 10m
            memory: 50Mi
          requests:
            cpu: 10m
            memory: 50Mi
```

```bash
# go-stress-testing-linux -c 200 -n 100 -u 172.18.51.183
qps只有300多，但是如果启动50个副本，同样可以完成请求。副本多了，更新慢
```

```bash
        resources:
          limits:
            cpu: 1
            memory: 50Mi
          requests:
            cpu: 1
            memory: 50Mi
```

```bash
# go-stress-testing-linux -c 200 -n 100 -u 172.18.51.183
qps只有2000多
```



[nginx](https://nginx.org/en/)官方文档中描述内存使用`10,000 inactive HTTP keep-alive connections take about 2.5M memory;`，并在调整不同的cpu限制，发现`` qps的值，在1核心是2000多。10m是300多



# pod真正使用的资源

## 内存

```bash

[root@izbp16exe6p6dolijnp7csz ~]# podname=saasagentweb-test-deploy-6f776fd7bf-cnz77;  echo $(cat $(find /sys -name "*$(kubectl get pod ${podname} -o yaml | grep uid | tail -n1 | tr '-' '_' | awk '{print $2}')*" | grep mem)/memory.usage_in_bytes)bytes
20668416bytes

[root@izbp16exe6p6dolijnp7csz ~]# echo $[20668416/1024/1024]
19M


# 获取id
root@a3f71b491a2e:~# podname=saasapi-prod-deploy-7698dbc475-mkkj2;  kubectl get pod ${podname} -o yaml | grep uid | tail -n1 | tr '-' '_' | awk '{print $2}'
691efd36_b93c_11eb_90cf_022063e39af7

# 获取大小
find /sys -name "*691efd36_b93c_11eb_90cf_022063e39af7*" | grep mem
/sys/fs/cgroup/memory/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod691efd36_b93c_11eb_90cf_022063e39af7.slice 

cat /sys/fs/cgroup/memory/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod691efd36_b93c_11eb_90cf_022063e39af7.slice/memory.usage_in_bytes
147M

# 和这个并无差异
root@a3f71b491a2e:~# kubectl top pod
NAME                                           CPU(cores)   MEMORY(bytes)             
saasapi-prod-deploy-7698dbc475-mkkj2           53m          147Mi           
saasapi-prod-deploy-7698dbc475-nsgd8           83m          133Mi 
```

一般前期生产，给足量的CPU和内存，固定replicas，prometheus观察pod一段时间再调整噻

```bash
14天观察 `deploy`最高内存700Mi, CPU最高300m, 所以配置limit和reqeust为1核心，1024Mi

pod1: 最高cpu: 100m, 最高内存150Mi
pod2: 最高cpu: 150m, 最高内存150Mi
deployment: 最高cpu: 300m, 最高内存 700Mi


ingress-controller，一般只需要请求资源，不会做限制。
    resources:
      requests:
        cpu: 100m
        memory: 70Mi

```

## cpu

```bash
[root@izbp16exe6p6dolijnp7csz ~]# podname=saasagentweb-test-deploy-6f776fd7bf-cnz77;  cat $(find /sys -name "*$(kubectl get pod ${podname} -o yaml | grep uid | tail -n1 | tr '-' '_' | awk '{print $2}')*" | grep cpuacct)/cpuacct.stat
user 15
system 16
```



