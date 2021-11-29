---
title: python 基础语法
date: 2021-01-04 08:53:27
tags:
toc: true
---



# 1. 注释 

python是解释运行，#后面的代码，将不会被解释器运行。

```python
print(123) # 在后，一般常用对代码解释 

#print(123)  # 在前，整行注释
```



# 2. 数字

整数

```bash
# c
char
unsigned char
signed char
int
unsigned int
short
unsigned short
long
unsgined long

# python3
int     # 只有int 任意长度
bool    # True False
complex # 实数+虚数*j

# python2
int     #带符号
long    # 长整型
```

浮点数

```bash
# c
float
double
long double

# python
float
```



# 3. 字符串

1. '  或 "中的字符

2. """或''' 中的字符，并且其中可以包含' 或" , 可以跨行

3. **r或R前缀**的字符，里面所有字符为普通字符， 即使有转义符

4. \ 在引号中表示**续行**, 不在引号中表示**转义**

   - `\\`   表示本身\

   - `\t`    表示tab

   - `\r`    表示回车

   - `\n`    换行

   - `\'`     '

   - `\"`     "

     

```python
print('a' + "b") # print
```

`ab`

```python
print("a" + "b")
```

`ab`

' '和" "一起操作并无差别



```python
''' welcome "to" 
1\
2
3
'python' '''
```

`' welcome "to" \n12\n3\n\'python\' '`

- 1和2没有\n断开，因为\续行，相当于1行

- 字符跨行，记录为\n



```python
print(' welcome "to" \n12\n3\n\'python\' ')
```

```
 welcome "to" 
12
3
'python' 
```

print遇到\n会转义，但是加上r/R不会转义

```python
print(R' welcome "to" \n12\n3\n\'python\' ')
```

`welcome "to" \n12\n3\n\'python\' `

# 4. 缩进

1. C/java 大括号嵌套多

2. Python 缩进几个，就是一层。 今天约定死，4个空格为一层缩进。



在python中if, else, def, class等关键字起始，后面冒号结尾，下一句必须是缩进。



# 5. 标识符

保存字符串的东西，为其取一个名字



命名规则：

1. 字母、数字、下划线，只能使用字母和下划线起始字符

2. 区分大小写，尽量命名规范统一

人为约定

1. 标识符不使用中文
2. 不使用歧义单词：class_
3. 不要随便使用下划线开头的标识符



# 6. 常量和变量

都是标识符

常量，定义后不可以改变。python没有常量，一般叫字面常量，因为在内存中固定的位置。其他语言加const, final

​	python中可以随便改，所以python错误非常难以发现

变量：赋值后，可以在修改，让标识符对应的量可以修改。



# 7. 运算符

## 7.1 算术

+

-

*

/ python2是整除，python3是自然除。

​	// 整除,C/java 这些语言，没有//, 两个整数相除结果是整数。一个小数，结果是小数

% 拿余数

**  多少次方，有的语言是 ^



```bash
[root@openstack-controller2 ~]# python2
Python 2.7.5 (default, Nov 16 2020, 22:23:17) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-44)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 1/2
0


(magedu-pypy-36-731) [python@19d125e0548a web]$ python3
Python 3.6.9 (2ad108f17bdb, Apr 07 2020, 02:59:05)
[PyPy 7.3.1 with GCC 7.3.1 20180303 (Red Hat 7.3.1-5)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>> 1/2 
0.5 # 自然除，结果是人易明白的
>>>> 1//2
0

```



cell中

```python
# 自然除
1/2 
```

`0.5`

```python
# 整除
1//2
```

`0`

```python
# 取余数
5%3
```

`2`

```python
# 次方
2**3
```

`8`

```python
# 开平方或math的函数
2**0.5
```

`1.4142135623730951`













## 7.2 位运算符

&     位与，乘法

|     位或，+法

~     取反

^     异或，相同为0，不同为1

`<<`  数字转换成2进制后，向左移

`>>`  数字转换成2进制后，向右移

相对于补码计算， 32//4, 32>>2结果一样，效率相当，因为计算经过解释 器优化。

## 7.3 比较运算符

== 

!= 

`>` 

`<` 

`>=`

`<=`

相同类型比较，结果是bool

不同类型比较：除了==, !=可以完成不同类型比较，其他均需要相同类型比较，除非运算符重载。

**所以python要注意，==, != 就算不同类型也可以比较**

python中没有++, --.  c, c++, java都有



## 7.4 逻辑运算符

and 第1个表达式False, 恒为False

or   第1个表达式True, 恒为True

## 7.5 赋值运算符

python赋值即定义，赋值了同时也定义了类型。但是python是动态编译语言，类型不固定，可以随意修改。

有些语言，先申明类型，而后赋值。并且类型不可以随意修改。

右边的值赋值给左边的标识符

```python
a = min(3,5)
```

```bash
X = y = z = 10 连等，建议少用，如果是引用时，出大事。
```

## 7.6 成员运算符

in   谁在谁里面

not in  谁不在谁里面

## 7.7 身份运算符

is

is not

# 8. 运算优先级

算术> 逻辑>赋值

记不住，使用括号



# 9. 表达式

由运算符组合

表达式的条件，会用bool函数隐式求值

## 9.1 单分支 if, if else

```markdown
if condition:
   if-true
```

## 9.2 多分支 if elif else

```markdown
if condition:
   if-true
elif conditoin:
   elif-true
...
else:
   all-false
```

```python
a = -5
if a < 0:
    print("negative")
elif a == 0:
    print("zero")
else:
    print("positive")
```

`negative`



## 9.3 分支嵌套

```python
score = 80
if score < 0:
    print("wrong")
else:
    if score == 0:
        print("egg")
    elif score <= 100:
        print("right")
    else:
        print('too big')
```

`right`

给定一定不超过5位的正整数，判断其有几位。

- 使用input函数

```python
#num = 1234
num = int(input('>>> '))
if (num / 1000) > 1:
    print(4)
elif (num / 100) > 1:
    print(3)
elif (num / 10) > 1:
    print(2)    
else:
    print(1)
```

`4`

```python
data = num / 100

if data > 1:
    if data > 10:
        print(4)
    else:
        print(3)
else: # <=1
    data = num / 10
    if data < 1:
        print(1)
    else:
        print(2)
```

`4`

如果搜索太多了，折半效率高

如果折的多，就需要封装和递归

```python
val = input('>>> ')
val = int(val)
if val >= 1000: # fold 折半
    if val >= 10000:
        print(5)
    else:
        print(4)
else:
    if val >= 100:
        print(3)
    elif val >= 10:
        print(2)
    else:
        print(1)
```





## 9.4 三目运算符

```python
a = int(input("a>>> "))
b = int(input("b>>> "))
print(a if a < b else b)
```



## 9.5 真值表

假:

1. 空集合："", {}, [], ()
2. 0
3. None 特殊的对象。 引用对象没有赋值就是None

真：非假

```python
bool(-1)
```

`True`



# 10. 循环

python只有2个循环while, for循环，而且格式固定



代码执行顺序

1. 手画
2. 网站：http://pythontutor.com/

## while

1. while true 死循环
2. while false 无用语句
3. 变量需要初始化，退出边界



语法 

```python
while condition: # condition表示表达式
    while-true   # 当condition为true时，进入循环
```

示例：打印10到1

```python
flag = 10
while flag:
    print(flag)
    flag -= 1
```

```
10
9
8
7
6
5
4
3
2
1
```

如果flag=-10怎么改造？

```python
flag = -10
while flag:
    print(flag)
    flag += 1
```

```
-10
-9
-8
-7
-6
-5
-4
-3
-2
-1
```





## for

1. 两层或多层for，需要变量初始化，退出条件

语法 

```python
for element in iterable: # 从命令中拿一个元素
    for-in               # 拿到元素就进入，拿不到就不进入
```

- [x] iterable，可迭代，可以顺序结构，可以非顺序结构。
- [x] iterable中的元素是否重复无所谓，但是内存中重复字符仅一份。



打印1~10

```python
for i in range(10): # range(10)，for会依次遍历0，1，2，3，4，5，6，7，8，9. 不需要知道原因，请参考range函数http://blog.mykernel.cn/2021/01/06/range%E5%87%BD%E6%95%B0/
    print(i+1
         ) # 括号可以换行
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

打印10~1

```python
for i in range(10): # range(10)，for会依次遍历0，1，2，3，4，5，6，7，8，9. 不需要知道原因，请参考range函数http://blog.mykernel.cn/2021/01/06/range%E5%87%BD%E6%95%B0/
    print(10-i)
```

```
10
9
8
7
6
5
4
3
2
1
```



## break

中断循环

```python
for i in range(10): # range(10)，for会依次遍历0，1，2，3，4，5，6，7，8，9. 不需要知道原因，请参考range函数http://blog.mykernel.cn/2021/01/06/range%E5%87%BD%E6%95%B0/
    if i == 5:
        break # 当遍历至i是5时，就跳出循环，不向后
    print(i)
```

```
0
1
2
3
4
```



**计算1000以内的被7整除的前20个数（for循环）**

界定：1000以内, range(1000)

被7整数：条件 i%7 == 0

前20个数：计数20，满足就退出循环break

```python
count = 0
for i in range(1000):
    if i%7:
        continue
    print(i)
    #count += 1
    count = count + 1 # count必须先定义，后使用
    if count >= 20:   # 边界问题，使用>= 范围大，避免没有考虑到边界
        break
```



**计算1000以内的被7整除的前20个数（while循环）**

1. 变量初始化
2. 边界

```python
num   = 0
count = 0

while num < 1000:          # 边界
    print(num)             # 前面打印，从初始值开始
    num+=7 # 修正变量
#     print(num)           # 后面打印，从修正后打印
    count+=1
    if count >= 20:        # 边界20
        break
    
```

```
0
7
14
21
28
35
42
49
56
63
70
77
84
91
98
105
112
119
126
133
```



## continue

跳过本次循环

```bash
for i in range(10): # range(10)，for会依次遍历0，1，2，3，4，5，6，7，8，9. 不需要知道原因，请参考range函数http://blog.mykernel.cn/2021/01/06/range%E5%87%BD%E6%95%B0/
    if i == 5:
        continue  # 当遍历至i是5时，就跳过本次循环，进入下一个元素的循环
    print(i)
```

```
0
1
2
3
4 # 没有5
6
7
8
9
```

## else

循环没有异常时或break时，打印

```python
for i in range(10):
    print(i)
else:
    print("ok")
```

```
0
1
2
3
4
5
6
7
8
9
ok
```

存在break不打印

```python
for i in range(10):
    if i == 5:
        break
    print(i)
else:
    print("ok")
```

```
0
1
2
3
4
```

不存在break打印

```python
for i in range(10):
    if i == 5:
        continue
    print(i)
else:
    print("ok")
```

```
0
1
2
3
4
6
7
8
9
ok
```