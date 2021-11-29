---
title: kubernetes-etcd集群崩溃
date: 2021-03-05 01:38:49
tags:
- 个人日记
---



由于本地环境磁盘IO太低了，今天发现一台主机空闲，相着加入etcd集群来使用，我删除现在的一个etcd SSD节点，直接整个集群不可用了，猜测是

```bash
# 现有master
<!--more-->
master1 SSD
master2 SSD
master3 机械盘
master4 机械盘
master5 机械盘
```

我直接删除master2, 整个集群不可以使用



# 关闭所有node节点

连接node节点, 这样node节点就不用上报数据写etcd了，减小etcd压力

```bash
#kubectl drain <node_name>
systemctl stop containerd kubelet
```

# 调整入口

http://192.168.0.249:9999/haproxy

只留SSD的api server一个入口

![image-20210305094616262](http://myapp.img.mykernel.cn/image-20210305094616262.png)



# 删除etcd节点

保存master1 SSD节点即可，删除方式

https://www.cnblogs.com/dudu/p/12173867.html

```bash
kubectl drain master1
kubectl delete node master1
```

现在查看集群状态，可以发现master节点的etcd集群中并未移除

所以需要手工移除

```bash
# 查看etcd集群所有成员 
# kubectl exec -it -n kube-system            etcd-master01.youwoyouqu.io -- etcdctl --endpoints 127.0.0.1:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key member list


# 移除非SSD节点的其他etcd, ！！一定不要删除错了，数据会丢
# kubectl exec -it -n kube-system            etcd-master01.youwoyouqu.io -- etcdctl --endpoints 127.0.0.1:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key member remove <member_id>
```

现在查看etcd一个节点的状态现在正常了

```bash
# kubectl exec -it -n kube-system            etcd-master01.youwoyouqu.io -- etcdctl --endpoints 127.0.0.1:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key endpoint health
127.0.0.1:2379 is healthy: successfully committed proposal: took = 13.077987ms
```

现在可以查看集群中的pods了

```bash
# Kubectl get pods -A -o wide
```

可以发现大量pod是运行状态，所以需要从etcd中删除

```bash
# kubectl exec -it -n kube-system            etcd-master01.youwoyouqu.io -- etcdctl --endpoints 127.0.0.1:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key get /registry/pod --prefix --keys-only
```

> 将以上输出结果，保存在txt文件中

执行删除所有pods操作 

参考：https://www.cnblogs.com/ding2016/p/12107942.html

```bash
for i in `cat txt`; do  kubectl exec -it -n kube-system            etcd-master01.youwoyouqu.io -- etcdctl --endpoints 127.0.0.1:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key del $i; done
1
1
1
1
1
1
1
1
1
```

> 1代表删除成功
>
> 0代表没有这个key, 二次执行将没有

执行完在查看pods, 可以发现etcd1不会自动启动，所以需要修改`/etc/kubernetes/manifests/etcd.yaml` 静态pods文件，随便添加一行空白，etcd将会启动

现在查看所有pods, 所在节点均在`kubectl get nodes -o wide` 这些节点上了，只是业务数据全是Pending状态，因为node全部是NotReady状态。



# 调整入口

to Ready

![image-20210305095623415](http://myapp.img.mykernel.cn/image-20210305095623415.png)

# 先新加master节点

如果直接加node, 可能一个master的etcd抗不住。还是先加master节点，添加方式参考：

http://blog.mykernel.cn/2021/02/26/diary-%E8%AE%B0%E5%BD%95%E4%B8%80%E6%AC%A1%E9%83%A8%E7%BD%B2%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83/#%E5%BF%AB%E9%80%9F%E6%B7%BB%E5%8A%A0node-master%E8%A7%92%E8%89%B2

```bash
# 先清理
# 再加入: 生成token -> 拼接 -> 加入
```



# 查看etcd状态

leader

```bash
root@master01:~# kubectl exec -it -n kube-system            etcd-master01.youwoyouqu.io -- etcdctl --endpoints 192.168.0.27:2379,192.168.0.12:2379,192.168.0.92:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key endpoint status
192.168.0.27:2379, d53581d4fde36e5d, 3.4.13, 8.5 MB, true, false, 1779, 4944521, 4944521,  # true为leader
192.168.0.12:2379, b5393253805dd5a1, 3.4.13, 8.5 MB, false, false, 1779, 4944521, 4944521, 
192.168.0.92:2379, 4fca2493737d4ec0, 3.4.13, 8.5 MB, false, false, 1779, 4944521, 4944521, 
```

