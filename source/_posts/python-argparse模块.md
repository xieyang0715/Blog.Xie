---
title: "python argparse模块"
date: 2021-04-20 13:47:59
tags:
- "python常用模块"
---

# argparse

解析位置参数

# 位置参数

## 必给位置参数

### 1个

```python
import argparse

parser = argparse.ArgumentParser(description="my test script", add_help=True)
parser.add_argument("file",help="给一个或多个文件")
args = parser.parse_args(["abc"])
print(args)
print("=========")
args = parser.parse_args(["-h"])
print(args)
```

```python
Namespace(file='abc')
=========
usage: cat.py [-h] file

my test script

positional arguments:
  file        给一个或多个文件

optional arguments:
  -h, --help  show this help message and exit
```



### 多个

```python
import argparse

parser = argparse.ArgumentParser(description="my test script", add_help=True)
parser.add_argument("file",help="给一个或多个文件",nargs="+")
args = parser.parse_args(["abc","cdb"])
print(args)
print("=========")
args = parser.parse_args(["-h"])
print(args)
```

```python
Namespace(file=['abc', 'cdb'])
=========
usage: cat.py [-h] file [file ...]

my test script

positional arguments:
  file        给一个或多个文件

optional arguments:
  -h, --help  show this help message and exit
```

## 可选位置参数

### 1个

#### 无默认

默认None

```python
import argparse

parser = argparse.ArgumentParser(description="my test script", add_help=True)
parser.add_argument("file",help="给一个或多个文件",nargs="?")
args = parser.parse_args([])
print(args)
print("=========")
args = parser.parse_args(["-h"])
print(args)
```

```python
Namespace(file=None)
=========
usage: cat.py [-h] [file]

my test script

positional arguments:
  file        给一个或多个文件

optional arguments:
  -h, --help  show this help message and exit
```



#### 有默认

```python
import argparse

parser = argparse.ArgumentParser(description="my test script", add_help=True)
parser.add_argument("file",help="给一个或多个文件",nargs="?",default="1231")
args = parser.parse_args([])
print(args)
print("=========")
args = parser.parse_args(["-h"])
print(args)
```

```python
Namespace(file='1231')
=========
usage: cat.py [-h] [file]

my test script

positional arguments:
  file        给一个或多个文件

optional arguments:
  -h, --help  show this help message and exit
```

### 多个

#### 无默认

默认空列表

```python
import argparse

parser = argparse.ArgumentParser(description="my test script", add_help=True)
parser.add_argument("file",help="给一个或多个文件",nargs="*")
args = parser.parse_args([])
print(args)
print("=========")
args = parser.parse_args(["-h"])
print(args)
```

```python
Namespace(file=[])
=========
usage: cat.py [-h] [file [file ...]]

my test script

positional arguments:
  file        给一个或多个文件

optional arguments:
  -h, --help  show this help message and exit

```

#### 默认

```python
import argparse

parser = argparse.ArgumentParser(description="my test script", add_help=True)
parser.add_argument("file",help="给一个或多个文件",nargs="*",default=[1,2,3])
args = parser.parse_args([])
print(args)
print("=========")
args = parser.parse_args(["-h"])
print(args)
```

```python
Namespace(file=[1, 2, 3])
=========
usage: cat.py [-h] [file [file ...]]

my test script

positional arguments:
  file        给一个或多个文件

optional arguments:
  -h, --help  show this help message and exit

```

# 选项参数

## 选项必给

```bash
import argparse

parser = argparse.ArgumentParser(description="my test script", add_help=True)
parser.add_argument("-f","--file",help="给一个或多个文件",required=True)
args = parser.parse_args(["-f"])
print(args)
print("=========")
args = parser.parse_args(["-h"])
print(args)
```

## 选项可选

```bash
import argparse

parser = argparse.ArgumentParser(description="my test script", add_help=True)
parser.add_argument("-f","--file",help="给一个或多个文件",required=False)
args = parser.parse_args(["-f"])
print(args)
print("=========")
args = parser.parse_args(["-h"])
print(args)
```

## 几个参数/默认值

> 选项后的一个，多个，必给，可选，默认值，由位置参数一样控制

## 在有限集合中选选项

```python
    parser = argparse.ArgumentParser(description="操作destinationrule的脚本", add_help=True)
    parser.add_argument("-m", "--method", choices=["add","del"],help="dr操作的方法", required=True)
    args = parser.parse_args()
```

> 输出
>
> ```bash
> usage: test.py [-h] -n NAMESPACE -dr DESTINATIONRULE -v VERSION -m {add,del}
> test.py: error: the following arguments are required: -n/--namespace, -dr/--destinationrule, -v/--version, -m/--method
> ```

