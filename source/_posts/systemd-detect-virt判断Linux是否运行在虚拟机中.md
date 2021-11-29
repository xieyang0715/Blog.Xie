---
title: systemd-detect-virt判断Linux是否运行在虚拟机中
date: 2020-11-24 10:16:15
tags: 日常挖掘
---

`systemd` 已经提供了一个命令来帮你完成判断Linux是否运行在虚拟机中，那就是 `systemd-detect-virt`.

```bash
# systemd-detect-virt
none # 物理机
kvm
vmware
```



转载: https://www.lujun9972.win/blog/2020/08/08/%E4%BD%BF%E7%94%A8systemd-detect-virt%E5%88%A4%E6%96%ADlinux%E6%98%AF%E5%90%A6%E8%BF%90%E8%A1%8C%E5%9C%A8%E8%99%9A%E6%8B%9F%E6%9C%BA%E4%B8%AD/index.html