---
title: k8s filebeat收集pod日志
date: 2020-12-11 06:53:57
tags: 生产小经验
---

今天开发反馈elk观察不到pod报错信息。1. k8s dashboard授权只读权限给开发。2. 将Pod启动逻辑修改。



之前直接启动java服务，将日志追加到日志文件，可能是因为崩溃太快，filebeat还没有完全收集完日志就停了。

```bash
# 启动脚本, 不依赖服务挂起。

# 错误才打印日志
/usr/share/filebeat/bin/filebeat -e -c /etc/filebeat/filebeat.yml -path.home /usr/share/filebeat -path.config /etc/filebeat -path.data /var/lib/filebeat -path.logs /var/log/filebeat > /dev/null & 
# 标准输出和错误输出全写到日志
nohup /usr/local/jdk/bin/java -jar /apps/app/admin-0.0.1-SNAPSHOT.jar --spring.profiles.active=pro  &>> /apps/logs/app.log &
tail -f /etc/hosts

```

```bash 
# 服务是否在线通过liveness或readness
livenessProbe:
  initialDelaySeconds: 80
  periodSeconds: 3
  httpGet:
    port: 8080
    scheme: HTTP
    path: /
readinessProbe:
  initialDelaySeconds: 80 # 初始化延迟时间，告诉kubelet在执行第一次探测前应该等待多少秒, 容器启动后，多久开始探测。
  successThreshold: 1 # 失败后，成功多少次才算成功, 成功就会被接入流量。
  failureThreshold: 3 # 成功后，不成功几次就停。
  timeoutSeconds: 1
  periodSeconds: 3    # 执行这个就绪性探针的间隔. 小点早点发现pod不可用。
  httpGet:
    port: 8080
    scheme: HTTP
    path: /
lifecycle: #保证在read/live检测后停止pod前睡眠一会，保证filebeat收集日志。
  preStop:
    exec:
      command: ["/bin/bash", "-c", "sleep 20"]

```

