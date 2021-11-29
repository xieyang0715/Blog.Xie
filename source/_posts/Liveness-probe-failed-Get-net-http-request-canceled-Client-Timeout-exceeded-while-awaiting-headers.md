---
title: >-
  Liveness probe failed: Get net/http: request canceled (Client.Timeout exceeded
  while awaiting headers
date: 2021-05-14 10:58:08
tags:
- kubernetes
---

# 前言

今天在收到大量pod事件报警，Readiness probe failed: Get [http://172.17.1.122:80/](http://172.17.1.122/): net/http: request canceled (Client.Timeout exceeded while awaiting headers

https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

- `initialDelaySeconds`：启动容器后，启动活动或就绪探针之前的秒数。默认为0秒。最小值为0。
- `periodSeconds`：执行探测的频率（以秒为单位）。默认为10秒。最小值为1。
- `timeoutSeconds`：探测超时的秒数。默认为1秒。最小值为1。
- `successThreshold`：探测失败后，连续最小成功探测为成功。默认值为1。对于活动性和启动探针，必须为1。最小值为1。
- `failureThreshold`：当探测失败时，Kubernetes会尝试尝试`failureThreshold`放弃。放弃活动探针意味着重新启动容器。如果准备就绪，则将Pod标记为“未就绪”。默认值为3。最小值为1。

https://github.com/kubernetes/kubernetes/issues/89898



<!--more-->
# 分析

通过以下命令获取请求时间

```bash
[root@izbp1canomrhu2o9bo8mjsz ~]# while true; do sleep 3; date && curl http://172.17.1.7:80 -s  -o /dev/null -w  "%{time_starttransfer}\n"; done
0.000
Fri May 14 11:28:14 CST 2021
0.000
Fri May 14 11:28:17 CST 2021
0.000
Fri May 14 11:28:20 CST 2021
0.000
Fri May 14 11:28:23 CST 2021
0.000
Fri May 14 11:28:26 CST 2021
0.000
Fri May 14 11:28:29 CST 2021
0.001
Fri May 14 11:28:32 CST 2021
0.000
Fri May 14 11:28:35 CST 2021
0.000
Fri May 14 11:28:38 CST 2021
0.000
Fri May 14 11:28:41 CST 2021
0.000
Fri May 14 11:28:44 CST 2021
0.000
Fri May 14 11:28:47 CST 2021
0.000
Fri May 14 11:28:50 CST 2021
0.000
Fri May 14 11:28:53 CST 2021
0.000
Fri May 14 11:28:56 CST 2021
0.000
Fri May 14 11:28:59 CST 2021
0.000
Fri May 14 11:29:02 CST 2021
0.003
Fri May 14 11:29:05 CST 2021
0.000
Fri May 14 11:29:08 CST 2021
0.000
Fri May 14 11:29:11 CST 2021
0.472

```

观察发现，只要开发访问这个web时，才会影响这个pod突然出现检测的问题



发现pod并没有重启, 所以，这个报警不需要

```bash
root@ccea447f1caa:~# kubectl get pod | grep agentweb
saasagentweb-gray-deploy-75c5f65d59-njjl6              1/1     Running   0          19h
saasagentweb-test-deploy-7c4b6f69fc-8f4lh              1/1     Running   0          13m
```

而且这个是测试环境，给的资源有限，所以不需要关心。

```bash
        resources:
          limits:
            cpu: 100m
            memory: 50Mi
          requests:
            cpu: 100m
            memory: 50Mi
            
# 测试qps cpu
```



