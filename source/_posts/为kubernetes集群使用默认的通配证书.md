---
title: 为kubernetes集群使用默认的通配证书
date: 2021-08-13 10:42:28
tags:
- 生产小经验
---

# 前言

如果你的k8s集群为公司部署，如果有通配名证书：例如：`*.google.com`, 你所有部署的子域名`a.google.com`,`b.google.com`。

<!--more-->
# 准备tls secret

```bash
kubectl create secret -h
```

# 配置nginx-ingress-controller

> https://kubernetes.github.io/ingress-nginx/user-guide/tls/#default-ssl-certificate

```bash
kubectl edit deploy -n kube-system      nginx-ingress-controller 
```



```diff
      containers:
      - args:
        - /nginx-ingress-controller
        - --election-id=ingress-controller-leader
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/nginx-configuration
        - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
        - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
        - --annotations-prefix=nginx.ingress.kubernetes.io
        - --publish-service=$(POD_NAMESPACE)/nginx-ingress-lb
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
+        - --default-ssl-certificate=default/google.com
```

> 此选项的含义，当你在配置ingress.yml时，如果不提供secretName字段，将默认使用此配置的默认证书。选项的值: `<namespace>/<secret>`
>
> For instance, if you have a TLS secret `foo-tls` in the `default` namespace, add `--default-ssl-certificate=default/foo-tls` in the `nginx-controller` deployment.
>
> The default certificate will also be used for ingress `tls:` sections that do not have a `secretName` option.

