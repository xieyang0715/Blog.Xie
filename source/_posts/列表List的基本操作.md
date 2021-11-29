---
title: 列表List的基本操作
date: 2021-01-12 10:20:26
tags:
toc: true
---



# list函数

```python
In [10]: help(list)
Help on class list in module builtins:

class list(object)
 |  list() -> new empty list # -> new empty list python3.5之后，增加的描述方式，调用list()返回一个新的空列表
 |  list(iterable) -> new list initialized from iterable's items
 |  
 |  Methods defined here:
 |  
 |  __add__(self, value, /)
 |      Return self+value.

```

> iterable, 可迭代对象
>
> 可以数清楚，但是未必是有序的。
>
> range
>
> 顺序的 sequence
>
> 无序的

帮助获取

![image-20210113100722922](http://myapp.img.mykernel.cn/image-20210113100722922.png)![image-20210113101023699](http://myapp.img.mykernel.cn/image-20210113101023699.png)

<!--more-->

# 定义

- [x] 赋值即定义
- [x] 列表不能一开始定义大小。Java/c++其他语言中，会让你初始化大小。告诉你列表你想开辟多大空间，语言的优化。
- [x] 索引：下标
- [x] 第1个元素，索引是0

```bash
正索引：从0开始
负索引：0被占了，只能从-1开始
	C/c++不提供负索引
		早期，找最大索引，依次-反着拿元素。
		现在，直接从-1开始拿
```

- [x] 所有线性结构：左边或下边界     ----------> 右边或上边界
- [x] 访问索引不可以超界，否则抛出indexerror



```python
In [11]: lst1 = list()

In [12]: lst1
Out[12]: []

In [13]: lst2 = []

In [14]: lst2
Out[14]: []

In [19]: lst3 = list(range(1,5,2))

In [20]: lst3
Out[20]: [1, 3]

In [21]: lst = [1,2,'ab',None]

In [22]: lst
Out[22]: [1, 2, 'ab', None]

    
    
# range是可迭代对象, 
In [15]: list(range(5))
Out[15]: [0, 1, 2, 3, 4]

#  list(range())   不常用，一般用来取奇偶数
In [16]: list(range(1,5,2))
Out[16]: [1, 3]

In [17]: list(range(2,5,2))
Out[17]: [2, 4]

# ipython的增强特性,_取上次的out输出
In [18]: _
Out[18]: [2, 4]
    

    
# 列表保证整齐，可变。里面放的是引用
# 可变：如果里面放个列表，也是无限大，就撑爆了，所以第5个元素是一个引用
In [23]: lst = [1,2,'ab',None,[1,2,'ab',None]]

In [24]: lst
Out[24]: [1, 2, 'ab', None, [1, 2, 'ab', None]]

    
    
# lst重新赋值a, b时
# 原来以上几个元素将不被引用，引用计数为0，就是垃圾了。垃圾回收就会回收
# gc就起作用了，在达到一定垃圾时，就会进行gc, python效率降低，除非你之前手工调用gc

In [23]: lst = [1,2,'ab',None,[1,2,'ab',None]]

In [24]: lst
Out[24]: [1, 2, 'ab', None, [1, 2, 'ab', None]]


In [25]: a = 5

In [26]: b = 'ab'

In [27]: lst = [a,b]

In [28]: lst
Out[28]: [5, 'ab']

```



# 访问 list[index]

类似其他语言的数组访问，数组也是线性的, 只是数组的元素只能是相同的类型

```python
In [28]: lst
Out[28]: [5, 'ab']

In [29]: lst[1] # 上界元素的索引 = 元素个数（长度）- 1
Out[29]: 'ab'
```



# 列表查询

## index

- [x] 默认：从左到右搜索，索引从0开始，区间 [0, +oo)
- [x] index(value,[start[,stop]])
- [x] 返回正索引

```python
# index
In [30]: lst = [ 1, 2, 3, 2, 2, 5, 2 ]

In [31]: lst
Out[31]: [1, 2, 3, 2, 2, 5, 2]

  # index(value)
In [32]: lst.index(2)
Out[32]: 1
  # index(value,start) [start,
In [33]: lst.index(2,1)
Out[33]: 1

In [34]: lst.index(2,2)
Out[34]: 3
   # index(value,start) [start, 反向也一样，对应最后一个元素，索引为6
In [35]: lst.index(2,-1)
Out[35]: 6
In [36]: lst.index(2,-3)
Out[36]: 4
```

## count

- [x] count(value)
- [x] 返回列表中匹配value的次数

```python
In [37]: lst
Out[37]: [1, 2, 3, 2, 2, 5, 2]

In [38]: lst.count(2)
Out[38]: 4

```



## len

- [x] 对象的元数据
- [x] len(iterable)

```python
In [37]: lst
Out[37]: [1, 2, 3, 2, 2, 5, 2]

In [39]: len(lst)
Out[39]: 7
```

# 列表元素修改

索引访问修改

- [x] list[index]=value
- [x] 索引不要超界

```python
In [40]: lst
Out[40]: [1, 2, 3, 2, 2, 5, 2]

In [41]: lst[len(lst)-1]=6

In [42]: lst
Out[42]: [1, 2, 3, 2, 2, 5, 6]

In [43]: lst[9]
---------------------------------------------------------------------------
IndexError                                Traceback (most recent call last)
<ipython-input-43-6b5942bbc420> in <module>
----> 1 lst[9]

IndexError: list index out of range

```

# 列表增加、插入元素

对列表有影响，影响其长度

```python
In [45]: help(list.append)
Help on method_descriptor:

append(...)
    L.append(object) -> None -- append object to end # python3.5 之后，新增。表示调用append返回一个None对象
```

> 返回None，表示没有新的列表产生，所以就是就地修改

## append

```python
In [46]: lst
Out[46]: [1, 2, 3, 2, 2, 5, 6]

In [47]: a  = lst.append(10)

In [48]: type(a)
Out[48]: NoneType

In [49]: a # ipython中可以省略print，打印出None

In [50]: print(a) # cpython打印None为None
None
	
    # 就地修改
In [52]: lst
Out[52]: [1, 2, 3, 2, 2, 5, 6, 10]


```

## insert

- [x] 索引可以超界
  - [x] 超下界，在前面插入
  - [x] 超上界，在尾部插入

```python
insert(index,object) -> None
```

```python
# 超下界
In [54]: lst
Out[54]: [1, 2, 3, 2, 2, 5, 6, 10]

In [55]: lst.insert(-100,123)

In [56]: lst
Out[56]: [123, 1, 2, 3, 2, 2, 5, 6, 10]
# 超上界
In [57]: lst.insert(1100,1223)

In [58]: lst
Out[58]: [123, 1, 2, 3, 2, 2, 5, 6, 10, 1223]

```










## *

重复

产生新列表

```python
In [62]: lst1 = []

In [63]: lst1*5
Out[63]: []

In [64]: lst1 = [1]

In [65]: lst1*5
Out[65]: [1, 1, 1, 1, 1]
```

## +

本质调用__add__()方法

产生新列表

连接操作

```python
In [66]: lst1=[1,12,13]

In [67]: lst
Out[67]: [123, 1, 2, 3, 2, 2, 5, 6, 10, 1223, 20]

In [68]: lst + lst1
Out[68]: [123, 1, 2, 3, 2, 2, 5, 6, 10, 1223, 20, 1, 12, 13]

```
## extend

- [x] 追加
- [x] 就地修改

```python
extend(iterable) -> None
```

```python
In [59]: lst
Out[59]: [123, 1, 2, 3, 2, 2, 5, 6, 10, 1223]

In [60]: lst.append(20) # 没有Out，表示就地修改

In [61]: lst
Out[61]: [123, 1, 2, 3, 2, 2, 5, 6, 10, 1223, 20]

    
    
In [69]: lst
Out[69]: [123, 1, 2, 3, 2, 2, 5, 6, 10, 1223, 20]

In [70]: lst1
Out[70]: [1, 12, 13]

In [71]: lst1.extend(lst) # 没有Out，表示就地修改

In [72]: lst1      # 追加  
Out[72]: [1, 12, 13, 123, 1, 2, 3, 2, 2, 5, 6, 10, 1223, 20]

```



# 列表删除元素

对列表有影响，影响其长度

## remove

从左至右查找第1个匹配value的值，移除第1个匹配。

就地修改

```python
In [73]: lst1
Out[73]: [1, 12, 13, 123, 1, 2, 3, 2, 2, 5, 6, 10, 1223, 20]

In [74]: lst1.remove(2)

In [75]: lst1                #第1个2被移除
Out[75]: [1, 12, 13, 123, 1, 3, 2, 2, 5, 6, 10, 1223, 20]

```



## pop

- [ ] pop([index]) -> item
- [ ] 不指定索引，从尾部弹出一个
- [ ] 指定index, 从索引处弹一个

```python
# 尾部弹出
In [76]: lst1
Out[76]: [1, 12, 13, 123, 1, 3, 2, 2, 5, 6, 10, 1223, 20]

In [77]: lst1.pop()
Out[77]: 20

In [78]: lst1
Out[78]: [1, 12, 13, 123, 1, 3, 2, 2, 5, 6, 10, 1223]

    
# 索引处弹出
In [79]: lst1
Out[79]: [1, 12, 13, 123, 1, 3, 2, 2, 5, 6, 10, 1223]

In [80]: lst1.pop(0)
Out[80]: 1

In [81]: lst1
Out[81]: [12, 13, 123, 1, 3, 2, 2, 5, 6, 10, 1223]

```

## clear

清除列表所有元素，剩下空列表

```python
In [82]: lst1
Out[82]: [12, 13, 123, 1, 3, 2, 2, 5, 6, 10, 1223]

In [83]: lst1.clear()

In [84]: lst1
Out[84]: []
```

# 列表其它操作

## reversed

- [ ] 反转列表元素，返回None
- [ ] 就地修改

```python
In [85]: lst
Out[85]: [123, 1, 2, 3, 2, 2, 5, 6, 10, 1223, 20]

In [86]: lst.reverse() # 就地反转

In [87]: lst                  # 此元素不动，两边对调。
Out[87]: [20, 1223, 10, 6, 5, 2, 2, 3, 2, 1, 123]
```



## sort

- [ ] sort(key=None, reverse=False) -> None
- [ ] 对列表元素进行排序，就地修改，默认升序
- [ ] reverse为True, 反转，降序

```python
# 缺省升序，传递None, False
In [88]: lst
Out[88]: [20, 1223, 10, 6, 5, 2, 2, 3, 2, 1, 123]

In [89]: lst.sort()

In [90]: lst
Out[90]: [1, 2, 2, 2, 3, 5, 6, 10, 20, 123, 1223]

# 降序
In [91]: lst.sort(reverse=True)

In [92]: lst
Out[92]: [1223, 123, 20, 10, 6, 5, 3, 2, 2, 2, 1]

    
# key函数
In [92]: lst
Out[92]: [1223, 123, 20, 10, 6, 5, 3, 2, 2, 2, 1]

In [93]: lst.append("a")

In [94]: lst
Out[94]: [1223, 123, 20, 10, 6, 5, 3, 2, 2, 2, 1, 'a']

In [95]: lst.sort() # 不同类型不能排序
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-95-b7de4ff5ffae> in <module>
----> 1 lst.sort()

TypeError: '<' not supported between instances of 'str' and 'int'

In [96]: lst.sort(key=str) 
#1.调用函数对所有元素处理
#2.将所有元素处理的结果，在排序
#key 复杂函数，或lambda表达式来处理

In [97]: lst
Out[97]: [1, 10, 1223, 123, 2, 2, 2, 20, 3, 5, 6, 'a']

In [98]: lst.sort(key=str,reverse=True)

In [99]: lst
Out[99]: ['a', 6, 5, 3, 20, 2, 2, 2, 123, 1223, 10, 1]

```



## in

- [ ] [3,4] in [1,2,[3,4]]
- [ ] for x in [1,2,3,4]
- [ ] 返回bool，可以做条件

```python
In [100]: lst
Out[100]: ['a', 6, 5, 3, 20, 2, 2, 2, 123, 1223, 10, 1]

In [101]: 20 in lst
Out[101]: True

In [102]: 30 in lst
Out[102]: False

In [103]: 30 in []
Out[103]: False

In [104]: 30 in [30,20,10]
Out[104]: True
    
# 条件
In [105]: if 30 in [30,20,10]:
     ...:     pass # pass占位值 ，当前 不写逻辑。后面才写
     ...: 
  
# not in
In [106]: 30 not in [30,20,10]
Out[106]: False

```

# 列表复制( copy, 浅拷贝)

## 内存地址比较和内容比较

```python
In [107]: lst0 = list(range(4))

In [108]: id(lst0) # id标识符，内存地址
Out[108]: 139669301785544

In [109]: lst1 = list(range(4))

In [110]: id(lst1)
Out[110]: 139669299915912

In [111]: hash(lst0) # 列表不可哈希 hash函数，不是所有类型可以使用
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-111-69a8889a75a8> in <module>
----> 1 hash(lst0)

TypeError: unhashable type: 'list'

        
# 比较内存地址
In [112]: lst0 is lst1
Out[112]: False

# 比较内容
In [113]: lst0 == lst1
Out[113]: True

```

## 列表复制

### 指向同一空间

```python
In [114]: lst0
Out[114]: [0, 1, 2, 3]

In [115]: lst1
Out[115]: [0, 1, 2, 3]

In [116]: lst1 = lst0 # 将lst1和lst0的内存地址指向同一段内存空间。所以is, ==为True

In [117]: lst1[2] = 10

In [118]: lst1
Out[118]: [0, 1, 10, 3]

In [119]: lst0
Out[119]: [0, 1, 10, 3]

In [120]: lst1 is lst0
Out[120]: True

In [121]: lst1 == lst0
Out[121]: True

```

### 列表复制

```python
In [125]: lst0 = list(range(4))

In [126]: lst0
Out[126]: [0, 1, 2, 3]

In [127]: lst5=lst0.copy() # 不是复制内存地址，简单类型复制内容。复杂类型复制地址

In [128]: lst5
Out[128]: [0, 1, 2, 3]

In [129]: lst5 is lst0
Out[129]: False

In [130]: lst5 == lst0
Out[130]: True

    
    
# 复杂类型
In [145]: lst0 = [1,[2,3,4,],5]

In [146]: lst5 = lst0.copy() # 拷贝

In [147]: lst5
Out[147]: [1, [2, 3, 4], 5]

In [148]: lst0
Out[148]: [1, [2, 3, 4], 5]

In [149]: lst5 == lst0
Out[149]: True

In [150]: lst5 is lst0 # 不是拷贝地址，而是内容
Out[150]: False

In [151]: 

In [151]: lst5[2] = 20 # lst5和lst0地址不一样，修改结果不一样。

In [152]: lst5
Out[152]: [1, [2, 3, 4], 20]

In [153]: lst0
Out[153]: [1, [2, 3, 4], 5]

In [154]: lst5[2] = 5

In [155]: lst5
Out[155]: [1, [2, 3, 4], 5]

In [156]: lst0
Out[156]: [1, [2, 3, 4], 5]

In [157]: 

In [157]: 

In [157]: lst5[1][1] = 20  # lst5和lst0地址不一样，修改结果应该不一样。 但是此处是修改复杂类型，结果一样，说明复杂类型是复制的地址

In [158]: lst5
Out[158]: [1, [2, 20, 4], 5]

In [159]: lst0
Out[159]: [1, [2, 20, 4], 5]

In [160]: lst5 == lst0
Out[160]: True

In [161]: lst5 is lst0
Out[161]: False

```



## 列表复制( copy.deepcopy, 深拷贝)

无论是否复杂类型，都拷贝内容

```python
In [1]: import copy

In [2]: lst0 = [1, [2, 20, 4], 5]

In [3]: lst5 = copy.deepcopy(lst0) # 深拷贝

In [4]: lst5
Out[4]: [1, [2, 20, 4], 5]

In [5]: lst0
Out[5]: [1, [2, 20, 4], 5]

In [6]: lst5 == lst0
Out[6]: True

In [7]: lst5 is lst0 # 内存地址不一样
Out[7]: False

In [8]: 

In [8]: 

In [8]: lst5[2] = 20 # 修改简单类型

In [9]: lst5
Out[9]: [1, [2, 20, 4], 20]

In [10]: lst0
Out[10]: [1, [2, 20, 4], 5]

In [11]: lst5 is lst0
Out[11]: False

In [12]: lst5[2] = 5

In [13]: lst5
Out[13]: [1, [2, 20, 4], 5]

In [14]: lst0
Out[14]: [1, [2, 20, 4], 5]

In [15]: 

In [15]: 

In [15]: lst5[1][1] = 3 # 修改复杂类型

In [16]: lst5   # 结果
Out[16]: [1, [2, 3, 4], 5]

In [17]: lst0   # 结果
Out[17]: [1, [2, 20, 4], 5]

    
# 结果不一致，说明复杂类型也是复制本身不是地址
```

