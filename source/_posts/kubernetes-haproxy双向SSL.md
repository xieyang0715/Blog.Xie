---
title: kubernetes haproxy双向SSL
date: 2021-03-18 06:52:11
tags:
- kubernetes
- haproxy
---



# 环境

| 操作系统    | 环境   | IP           | 角色     | VIP                    |
| ----------- | ------ | ------------ | -------- | ---------------------- |
| Ubuntu 1804 | docker | 192.168.0.81 | haproxy1 | 247接流量；249内部通信 |

# 双向通信过程

![image-20210318153048374](http://myapp.img.mykernel.cn/image-20210318153048374.png)

# 配置haproxy双向认证

## 面向kubectl

### 准备ca

验证client发来的证书，所以需要使用kubectl的签发CA

```bash
root@master01:~# scp /etc/kubernetes/pki/ca.crt 192.168.0.81:/tmp/ca.crt
ca.crt                                                        
root@master01:~# scp /etc/kubernetes/pki/ca.key 192.168.0.81:/tmp
ca.key                                                                  
```

### 准备haproxy服务端证书

使用传递过来的ca签发即可

#### 证书生成工具

```bash
root@haproxy01:/tmp# export https_proxy=http://192.168.0.33:808
root@haproxy01:/tmp# wget https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl_1.5.0_linux_amd64
root@haproxy01:/tmp# wget https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl-certinfo_1.5.0_linux_amd64
root@haproxy01:/tmp# wget https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssljson_1.5.0_linux_amd64
root@haproxy01:/tmp# chmod +x cfssl*
```

#### 配置文件

```bash
root@haproxy01:/tmp# cat cfg.json
{
    "signing": {
        "default": {
            "expiry": "876000h" 
        },
        "profiles": {
            "kubernetes-ca": { 
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth",
					"server auth"
                ]
            }
        }
    }
}
```

#### server证书申请

客户https请求api域名

```diff
root@haproxy01:/tmp# cat haproxy-server.json 
{
  "CN": "haproxy-server",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
    "O": "system:HuaYang",
    "OU": "system"
  }],
    "hosts": [
+		"kubernetes-api.youwoyouqu.io"
	]
}

```

#### 生成haproxy的证书

```bash
root@haproxy01:/tmp# ./cfssl_1.5.0_linux_amd64 gencert -ca=ca.crt -ca-key=ca.key      --config=cfg.json -profile=kubernetes-ca      haproxy-server.json | ./cfssljson_1.5.0_linux_amd64 -bare haproxy-server
2021/03/18 15:47:28 [INFO] generate received request
2021/03/18 15:47:28 [INFO] received CSR
2021/03/18 15:47:28 [INFO] generating key: rsa-2048
2021/03/18 15:47:28 [INFO] encoded CSR
2021/03/18 15:47:28 [INFO] signed certificate with serial number 496720295927558098759471391397869675046127844047
```



## 准备haproxy面向服务端的client证书

需要集群管理员权限, 所以直接使用 `system:masters` 作为证书的组即可

```diff
root@master01:~# kubectl get clusterrolebinding cluster-admin -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding

...
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
+  name: system:masters # 所以只需要在这个组即可
```

### 准备client证书申请

```diff
root@haproxy01:/tmp# cat haproxy-client.json 
{
  "CN": "haproxy-client",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
+    "O": "system:masters",
    "OU": "system"
  }],
    "hosts": [
	]
}
```

#### 生成客户端证书

```bash
root@haproxy01:/tmp# ./cfssl_1.5.0_linux_amd64 gencert -ca=ca.crt -ca-key=ca.key      --config=cfg.json -profile=kubernetes-ca      haproxy-client.json | ./cfssljson_1.5.0_linux_amd64 -bare haproxy-client
2021/03/18 15:50:25 [INFO] generate received request
2021/03/18 15:50:25 [INFO] received CSR
2021/03/18 15:50:25 [INFO] generating key: rsa-2048
2021/03/18 15:50:26 [INFO] encoded CSR
2021/03/18 15:50:26 [INFO] signed certificate with serial number 408058459368907047002860563445969480982412422654
2021/03/18 15:50:26 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

## haproxy SSL配置代理kubernetes apiserver

### 生成haproxy专用ssl证书

由于haproxy ssl需要 key和crt, 所以

```bash
root@haproxy01:/tmp# mv /tmp/haproxy-server.pem  /tmp/haproxy-server.pem.origin
root@haproxy01:/tmp# cat /tmp/haproxy-server.pem.origin /tmp/haproxy-server-key.pem | tee /tmp/haproxy-server.pem
```

```bash
root@haproxy01:/tmp# cat /tmp/haproxy-client.pem.origin /tmp/haproxy-client-key.pem | tee /tmp/haproxy-client.pem
```



### 配置haproxy

```bash
listen apiserver-6443
    bind 192.168.0.249:6443 ssl crt /tmp/haproxy-server.pem ca-file /tmp/ca.crt verify required

    mode http
    server 192.168.0.27 192.168.0.27:6443 check inter 3s fall 2 rise 5  ssl crt /tmp/haproxy-client.pem ca-file /tmp/ca.crt verify required
    server 192.168.0.92 192.168.0.92:6443 check inter 3s fall 2 rise 5                                                           
    server 192.168.0.27 192.168.0.27:6443 check inter 3s fall 2 rise 5 
```

> 注意：后面2个没有加ssl, 作为验证

重启

```bash
root@haproxy01:/tmp# systemctl restart haproxy
```

#### 客户端请求验证

```bash
root@master01:~# kubectl get node
Error from server (BadRequest): Unable to list "/v1, Resource=nodes": the server rejected our request for an unknown reason (get nodes)
root@master01:~# kubectl get node
Error from server (BadRequest): Unable to list "/v1, Resource=nodes": the server rejected our request for an unknown reason (get nodes)
root@master01:~# kubectl get node
NAME                     STATUS                     ROLES                  AGE   VERSION
master01.youwoyouqu.io   Ready                      control-plane,master   14d   v1.20.1
master02.youwoyouqu.io   Ready                      control-plane,master   13d   v1.20.1
master04.youwoyouqu.io   Ready                      control-plane,master   13d   v1.20.1
node01.youwyouqu.io      Ready,SchedulingDisabled   <none>                 14d   v1.20.1
node03.youwoyouqu.io     Ready                      <none>                 13d   v1.20.1
node07.youwoyouqu.io     Ready                      <none>                 14d   v1.20.1

```



### 完整配置haproxy

```bash
listen apiserver-6443
    bind 192.168.0.249:6443 ssl crt /tmp/haproxy-server.pem ca-file /tmp/ca.crt verify required

    mode http
    server 192.168.0.27 192.168.0.27:6443 check inter 3s fall 2 rise 5  ssl crt /tmp/haproxy-client.pem ca-file /tmp/ca.crt verify required
    server 192.168.0.92 192.168.0.92:6443 check inter 3s fall 2 rise 5 ssl crt /tmp/haproxy-client.pem ca-file /tmp/ca.crt verify required
    server 192.168.0.27 192.168.0.27:6443 check inter 3s fall 2 rise 5  ssl crt /tmp/haproxy-client.pem ca-file /tmp/ca.crt verify required
```

```bash
root@haproxy01:/tmp# systemctl restart haproxy
```

#### 验证

```bash
root@master01:~# kubectl get node
NAME                     STATUS                     ROLES                  AGE   VERSION
master01.youwoyouqu.io   Ready                      control-plane,master   14d   v1.20.1
master02.youwoyouqu.io   Ready                      control-plane,master   13d   v1.20.1
master04.youwoyouqu.io   Ready                      control-plane,master   13d   v1.20.1
node01.youwyouqu.io      Ready,SchedulingDisabled   <none>                 14d   v1.20.1
node03.youwoyouqu.io     Ready                      <none>                 13d   v1.20.1
node07.youwoyouqu.io     Ready                      <none>                 14d   v1.20.1
root@master01:~# kubectl get node
NAME                     STATUS                     ROLES                  AGE   VERSION
master01.youwoyouqu.io   Ready                      control-plane,master   14d   v1.20.1
master02.youwoyouqu.io   Ready                      control-plane,master   13d   v1.20.1
master04.youwoyouqu.io   Ready                      control-plane,master   13d   v1.20.1
node01.youwyouqu.io      Ready,SchedulingDisabled   <none>                 14d   v1.20.1
node03.youwoyouqu.io     Ready                      <none>                 13d   v1.20.1
node07.youwoyouqu.io     Ready                      <none>                 14d   v1.20.1
root@master01:~# kubectl get node
NAME                     STATUS                     ROLES                  AGE   VERSION
master01.youwoyouqu.io   Ready                      control-plane,master   14d   v1.20.1
master02.youwoyouqu.io   Ready                      control-plane,master   13d   v1.20.1
master04.youwoyouqu.io   Ready                      control-plane,master   13d   v1.20.1
node01.youwyouqu.io      Ready,SchedulingDisabled   <none>                 14d   v1.20.1
node03.youwoyouqu.io     Ready                      <none>                 13d   v1.20.1
node07.youwoyouqu.io     Ready                      <none>                 14d   v1.20.1

# 多次请求正常
```

# 后记

这样使用，haproxy会话卸载了，后端还进行SSL通信，效率应该不高。所以前端还是使用TCP最好

```bash
listen apiserver-6443
   bind 192.168.0.249:6443

   mode tcp
   server 192.168.0.12 192.168.0.12:6443 check inter 3s fall 2 rise 5
   server 192.168.0.92 192.168.0.92:6443 check inter 3s fall 2 rise 5
   server 192.168.0.27 192.168.0.27:6443 check inter 3s fall 2 rise 5
```

