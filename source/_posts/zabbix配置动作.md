---
title: zabbix配置动作
date: 2021-04-28 13:36:01
tags:
- zabbix
- 生产小经验
---



# zabbix配置报警的步骤

## 用户订阅媒介

用户订阅报警媒介，可以传递一个参数 {ALERT.SENDTO}

![1](http://myapp.img.mykernel.cn/1.png)

## 报警媒介

报警媒介只需要脚本和参数

![2xxx](http://myapp.img.mykernel.cn/2xxx.png)

## 动作引用

动作中引用用户及媒介

![image-20210428134554640](http://myapp.img.mykernel.cn/image-20210428134554640.png)

> 持续就是，这一步经历多久，就进入下一步。如果在持续时间内恢复，就进行恢复的操作。

# 动作配置

## 操作

> 注意 内容有个空行

```bash
{EVENT.NAME}故障，长达5分钟!
```

```bash

告警主机:{HOST.NAME}
告警地址:{HOST.IP}
监控项目:{ITEM.NAME}
监控取值:{ITEM.LASTVALUE}
告警等级:{TRIGGER.SEVERITY}
当前状态:{TRIGGER.STATUS}
告警时间:{EVENT.DATE} {EVENT.TIME}
事件ID:{EVENT.ID}
```

## 恢复

> 注意 内容有个空行

```bash
{EVENT.NAME}已恢复!
```

```bash

告警主机:{HOST.NAME}
告警地址:{HOST.IP}
监控项目:{ITEM.NAME}
监控取值:{ITEM.LASTVALUE}
告警等级:{TRIGGER.SEVERITY}
当前状态:{TRIGGER.STATUS}
告警时间:{EVENT.DATE} {EVENT.TIME}
恢复时间:{EVENT.RECOVERY.DATE} {EVENT.RECOVERY.TIME}
持续时间:{EVENT.AGE}
事件ID:{EVENT.ID}
```

## 配置故障持续5分钟，才开始报警

![image-20210428135349805](http://myapp.img.mykernel.cn/image-20210428135349805.png)

# dingding脚本

```python
#!/usr/bin/env python3
# encoding: utf-8
"""
@author: songliangcheng
@qq:     2192383945@qq.com
@file:   dingding.py
@time:   2021-04-28 11:00:06
@version: v1.0.1
@description:
给一个人发告警:  dingding.py  --telephone 17502890627 "标题" "内容"
给多个人发告警:  dingding.py  --telephone 17502890627 192168 "标题" "内容"
"""
import argparse
import datetime

import requests
import json
import time
import hmac
import hashlib
import base64
import urllib.parse


class DingDing:
    """
    dingding调用的封装
    NORMAL 表示接口响应正常
    """
    NORMAL = 0

    def __init__(self, webhook, secret=None, keyword=None):
        """
        参考https://developers.dingtalk.com/document/app/custom-robot-access/title-zob-eyu-qse
        :param webhook: 钉钉的webhook
        :param secret:  生成的签名
        :param keyword:  关键字，没有就不写
        """
        if secret:
            self.timestamp = str(round(time.time() * 1000))
            secret_enc = secret.encode('utf-8')
            string_to_sign = '{}\n{}'.format(self.timestamp, secret)
            string_to_sign_enc = string_to_sign.encode('utf-8')
            hmac_code = hmac.new(secret_enc, string_to_sign_enc, digestmod=hashlib.sha256).digest()
            self.sign = urllib.parse.quote_plus(base64.b64encode(hmac_code))
            self.keyword = keyword
            # webhook url
            self.webhook = "{}&timestamp={}&sign={}".format(webhook, self.timestamp, self.sign)
        else:
            self.webhook = webhook

    def req(self, content, atusers):
        """
        发起一个请示
        :param content: 内容
        :param atusers: @哪些人
        :return:
        """
        payload = json.dumps({
            "msgtype": "text",
            "text": {
                "content": "{} {}".format(content, self.keyword if self.keyword else "")
            },
            "at": {
                "atMobiles": atusers,
                "isAtAll": True if 'all' in atusers else False
            }
        })
        headers = {
            'Content-Type': 'application/json'
        }
        response = requests.request("POST", self.webhook, headers=headers, data=payload)
        _ = json.loads(response.text)
        if _['errcode'] == DingDing.NORMAL:
            return 0
        return _


if __name__ == '__main__':
    # 位置参数解析
    parser = argparse.ArgumentParser(description="zabbix dingding alert", add_help=True)
    parser.add_argument("telephone", help="告警 @谁 或 all")
    parser.add_argument("title", help="告警的标题" )
    parser.add_argument("content", help="告警的内容")
    parser.add_argument("webhook", help="webhook")
    parser.add_argument("secret", help="secret")

    args = parser.parse_args()
    content = """\
{}
{}\
""".format(args.title, args.content)
    # 钉钉告警
    dd = DingDing(
        webhook=args.webhook,
        secret=args.secret, keyword=Config.KEYWORD)
    rex = dd.req(content, atusers=args.telephone.split("|"))
    print(rex)
```

