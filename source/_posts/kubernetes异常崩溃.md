---
title: kubernetes异常崩溃
date: 2021-02-26 02:09:41
tags:
- 生产小经验
---



最近遇到一个奇怪的现象，kubernetes在阿里云部署正常，在本地环境中部署经常会出现问题

![image-20210226101206110](http://myapp.img.mykernel.cn/image-20210226101206110.png)

查询etcd日记, 可以观察太多的too long，etcd执行时间太长，有些甚至达到1s

![image-20210226101618145](http://myapp.img.mykernel.cn/image-20210226101618145.png)

etcd pods状态

![image-20210226103307972](http://myapp.img.mykernel.cn/image-20210226103307972.png)

网络IO

```bash
mtr kubernetes-api.mykernel.cn
# 发现丢包
```

![image-20210226101220196](http://myapp.img.mykernel.cn/image-20210226101220196.png)

磁盘IO

```bash
iotop
# 发现etcd进程IO使用率打满
```



经过搜索，发现以下解决方案

https://www.programmersought.com/article/93121295768/

https://itnext.io/etcd-performance-consideration-43d98a1525a3



由于我的主机是Kvm主机，通过查看我的虚拟机的磁盘虚拟化不是virtio, 而是IDE，通过修改启动参数:

```bash
virt-install --virt-type kvm --name ubuntu-1804-template-2C-2G --ram 2048 --vcpus 2 --cdrom=/ISOs/ubuntu-18.04.3-server-amd64.iso  --disk path=/VMs/template/centos7-1804-template-2C2G.qcow2,bus=virtio,cache=none --network bridge=br0,model=virtio --graphics vnc,listen=0.0.0.0 --noautoconsole #--autostart
```

通过virtio调整，etcd的too long时间已经有改善，而且etcd pod不再崩溃

![image-20210226103251920](http://myapp.img.mykernel.cn/image-20210226103251920.png)

![image-20210226103320934](http://myapp.img.mykernel.cn/image-20210226103320934.png)

现在观察到api还是有崩溃, 获取api-server容器

```bash
root@master02:~# crictl ps
CONTAINER ID        IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID
93a1a722b6be0       bfe3a36ebd252       39 minutes ago      Running             coredns                   0                   c6bcb86a75699
8e04172b780ff       bfe3a36ebd252       39 minutes ago      Running             coredns                   0                   82ad995a34b90
4eba8ef0ee289       dee1cac4dd205       39 minutes ago      Running             kube-flannel              0                   f0376ca8d5e6b
5ee6b99e64bb2       e3f6fcd87756e       About an hour ago   Running             kube-proxy                0                   fd1ee72cc90f1
e9f5cfd473acf       75c7f71120808       About an hour ago   Running             kube-apiserver            1                   a9f4e6a4d591d
74e07ae348b69       0369cf4303ffd       About an hour ago   Running             etcd                      0                   cc5910b73fa02
18d0788466a05       4aa0b4397bbbc       About an hour ago   Running             kube-scheduler            0                   95ee3e55042c0
a6290a66af62f       2893d78e47dc3       About an hour ago   Running             kube-controller-manager   0                   ab35680c00c03
```

获取日志

```bash
root@master02:~# crictl logs --tail 2 -f e9f5cfd473acf
Trace[1789967593]: ---"About to write a response" 501ms (03:07:00.938)
Trace[1789967593]: [501.42691ms] [501.42691ms] END
I0226 03:07:23.345836       1 trace.go:205] Trace[478039569]: "GuaranteedUpdate etcd3" type:*v1.Endpoints (26-Feb-2021 03:07:22.593) (total time: 752ms):
Trace[478039569]: ---"Transaction prepared" 520ms (03:07:00.118)
Trace[478039569]: ---"Transaction committed" 227ms (03:07:00.345)
Trace[478039569]: [752.260269ms] [752.260269ms] END

```

获取搜索引擎的关键字

```bash
echo '"About to write a response" 501ms (03:07:00.938)' | sed 's@[^a-Z "]@*@g' | tr -s '*'
# 结果
https://github.com/kubernetes-monitoring/kubernetes-mixin/issues/121
您可以尝试为etcd提供更多的CPU和内存，将其放在更大的机器上，并检查这些机器上的iowait时间

# iowait观察
root@master02:~# top -n1 | grep -i %cpu
%Cpu(s):  3.4 us,  1.6 sy,  0.1 ni, 74.7 id, 20.1 wa,  0.0 hi,  0.1 si,  0.0 st
#可以看到磁盘的iowait太高了
```

所以api server崩溃由于集群组件也需要IO，所以是k8s集群运行在机械硬盘还是太慢了