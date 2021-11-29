---
title: 转换qcow为vmdk
date: 2021-03-03 00:29:36
tags: 
- 生产小经验
- kvm
---

```bash
[root@wzx ~]# qemu-img convert -f qcow2 /vms/ubuntu-template.qcow2  -O vmdk /vms/ubuntu-template.vmdk 
```

```bash
[root@wzx ~]# qemu-img info /vms/ubuntu-template.vmdk
image: /vms/ubuntu-template.vmdk
file format: vmdk
virtual size: 120G (128849018880 bytes)
disk size: 2.7G
cluster_size: 65536
Format specific information:
    cid: 3709011112
    parent cid: 4294967295
    create type: monolithicSparse
    extents:
        [0]:
            virtual size: 128849018880
            filename: /vms/ubuntu-template.vmdk
            cluster size: 65536
            format: 
```

