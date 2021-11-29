---
title: 生产k8s的deploy检测配置
date: 2020-11-13 07:51:15
tags: 生产小经验
---



当initialDelaySeconds太小时, 会频繁出错

```yaml
        livenessProbe:
          initialDelaySeconds: 80
          periodSeconds: 3
          httpGet:
            port: 80
            scheme: HTTP
            path: /
        readinessProbe:
          initialDelaySeconds: 80 # 初始化延迟时间，告诉kubelet在执行第一次探测前应该等待多少秒, 容器启动后，多久开始探测。
          successThreshold: 1 # 失败后，成功多少次才算成功, 成功就会被接入流量。
          failureThreshold: 3 # 成功后，不成功几次就停。
          timeoutSeconds: 1
          periodSeconds: 3    # 执行这个就绪性探针的间隔. 小点早点发现pod不可用。
          httpGet:
            port: 80
            scheme: HTTP
            path: /
        lifecycle: #我们这里使用 preStop 设置了一个 20s 的宽限期，Pod 在真正销毁前会先 sleep 等待 20s，这就相当于留了时间给 Endpoints 控制器和 kube-proxy 更新去 Endpoints 对象和转发规则，这段时间 Pod 虽然处于 Terminating 状态，即便在转发规则更新完全之前有请求被转发到这个 Terminating 的 Pod，依然可以被正常处理，因为它还在 sleep，没有被真正销毁。https://www.qikqiak.com/post/zero-downtime-rolling-update-k8s/
          preStop:
            exec:
              command: ["/bin/bash", "-c", "sleep 20"]
```

> 注意：修改配置后，需要kubectl get deploy -n xxx xxx -o yaml验证