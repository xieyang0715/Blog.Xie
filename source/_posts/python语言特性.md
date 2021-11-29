---
title: python语言特性
date: 2021-01-04 08:13:54
tags:
---



python是动态语言、强类型语言



![image-20210104161600786](http://myapp.img.mykernel.cn/image-20210104161600786.png)

<!--more-->

Dynamic：动态编译语言

- 不申明类型和值，用的时候可以随意修改值和类型
- 运行时才可以看到报错。

Static：静态编译语言

- 使用前，先申明。一旦申明不可以修改类型。 

- 编译时检查。

Strong: 强类型，不同类型不可以运算，除非运算符重载。 

```python
print('a' + "b") # 相同类型运算
```

`ab`

```python
print("a" + 1) # 不同类型运算
```

```
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-14-f3cae10e7279> in <module>
----> 1 print("a" + 1)

TypeError: unsupported operand type(s) for +: 'str' and 'int'
```



Weak: 弱类型，不同类型运算时，可以隐式转换类型，可以运算。

![image-20210104162230015](http://myapp.img.mykernel.cn/image-20210104162230015.png)

