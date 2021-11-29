---
title: range函数
date: 2021-01-06 02:04:02
tags:
toc: true
---



首先观察for循环代码

```python
for i in range(10):
    print(i+1)
```

```
1
2
3
4
5
6
7
8
9
10
```

for是从iterable中拿元素，依次进入循环中。



range函数怎么使用？



# 1. 获取函数帮助

在学习一个函数时，不建议读源码，因为python官方是cpython, c源码实现。初学者看几个指针就蒙了。

```python
# 准备3.6.1 虚拟环境
pyenv  install -v 3.6.1

pyenv versions
pyenv virtualenv 3.6.1 magedu361 
pyenv local magedu361 

pip install ipython


# 获取帮助
(magedu361) [python@19d125e0548a learn]$ ipython
In [1]: help(range)
Help on class range in module builtins:

class range(object)
 |  range(stop) -> range object
 |  range(start, stop[, step]) -> range object
 |  
 |  Return an object that produces a sequence of integers from start (inclusive)
 |  to stop (exclusive) by step.  range(i, j) produces i, i+1, i+2, ..., j-1.
 |  start defaults to 0, and stop is omitted!  range(4) produces 0, 1, 2, 3.
 |  These are exactly the valid indices for a list of 4 elements.
 |  When step is given, it specifies the increment (or decrement).
    返回一个对象，该对象逐步生成从开始（含）到停止（不含）的整数序列。 range（i，j）产生i，i + 1，i + 2，...，j-1。 start默认为0，stop被忽略！ range（4）产生0、1、2、3。这正是4个元素列表的有效索引。 给定step时，它指定增量（或减量）。
```

- [ ] [ ] 中的内容表示可以不给值 

- [ ] -> 返回值

  - range(stop) -> range object, start默认为0。 区间 [0,stop)

    ```python
    In [1]: range(4)
    Out[1]: range(0, 4)
    
    In [2]: list(range(4)) # 结尾被忽略
    Out[2]: [0, 1, 2, 3] # 0，1，2，3也是这4个元素的有效索引
    ```

  - range(start, stop) -> range object，指定start,则从start开始。索引还是从0开始。区间 [start,stop)

  - range(start, stop, step) -> range object, step可以是增量，或减量。区间 [start,stop)

    ```python
    In [3]: list(range(4,1,-1)) # [4,1)
    Out[3]: [4, 3, 2] 
    ```





# 2. range 求奇数或偶数

```python
list(range(0,10,2))
```

```
[0, 2, 4, 6, 8] # 不包含10
```

```python
list(range(1,10,2))
```

```
[1, 3, 5, 7, 9]
```



# 3. range函数高阶应用

http://blog.mykernel.cn/2021/01/06/%E7%BB%83%E4%B9%A0/#13-%E6%89%93%E8%8F%B1%E5%BD%A2

range的范围不能被0或1束缚住

# 4. 获取帮助

```python
In [4]: help(print)
Help on built-in function print in module builtins:

print(...)
    print(value, ..., sep=' ', end='\n', file=sys.stdout, flush=False)
    
    Prints the values to a stream, or to sys.stdout by default.
    Optional keyword arguments:
    file:  a file-like object (stream); defaults to the current sys.stdout.
    sep:   string inserted between values, default a space.
    end:   string appended after the last value, default a newline.
    flush: whether to forcibly flush the stream.

```

help可以接变量、对象、类名、函数名、方法名

获取a的帮助

```python
In [5]: a = 5

In [6]: help(a)
Help on int object:

class int(object)
 |  int(x=0) -> integer
 |  int(x, base=10) -> integer
 |  
 |  Convert a number or string to an integer, or return 0 if no arguments
 |  are given.  If x is a number, return x.__int__().  For floating point
 |  numbers, this truncates towards zero.
 |  
 |  If x is not a number or if base is given, then x must be a string,
 |  bytes, or bytearray instance representing an integer literal in the
 |  given base.  The literal can be preceded by '+' or '-' and be surrounded
 |  by whitespace.  The base defaults to 10.  Valid bases are 0 and 2-36.
 |  Base 0 means to interpret the base from the string as an integer literal.
 |  >>> int('0b100', base=0)
 |  4

```

![image-20210107134735805](http://myapp.img.mykernel.cn/image-20210107134735805.png)