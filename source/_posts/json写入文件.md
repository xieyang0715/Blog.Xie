---
title: json写入文件
tags:
  - python常用模块
photos:
  - http://myapp.img.mykernel.cn/images-123.png
date: 2021-11-14 11:06:18
---

# 前言

python中的`json.dump`写入文件的格式有两种:

- 单行
- indent缩进
- ascii编码

但是在文本中如果想每个对象，仅单行展示就比较麻烦了, 细品下面的代码

<!--more-->



# 区别dumps和dump

```python
    with open(f"{name}.json", 'w', encoding='utf-8') as f:
        for line in parsed:
            f.write(json.dumps(line,ensure_ascii=False)+'\n')
        # json.dump(parsed, f, ensure_ascii=False, indent=1)
```

> 1. 直接dump
> 2. dumps的结果是字符，通过write写字符