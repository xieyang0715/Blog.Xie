---
title: node内存管理
date: 2021-10-15 15:02:09
tags:
- 内存管理
- node
---

# 前言

node 老年代一直在使用内存

<!--more-->
# 进程空间

![image-20211015150616959](http://myapp.img.mykernel.cn/image-20211015150616959.png)

> - **Code:** the actual code being executed
> - **Stack:** contains all value types (primitives like integer or Boolean) with pointers referencing objects on the heap and pointers defining the control flow of the program
> - **Heap:** a memory segment dedicated to storing reference types like objects, strings and closures.

> http://apmblog.dynatrace.com/2015/11/04/understanding-garbage-collection-and-hunting-memory-leaks-in-node-js/
>
> https://developpaper.com/node-memory-management-and-garbage-collection/

# heap垃圾回收

## 垃圾回收前

![image-20211015150657029](http://myapp.img.mykernel.cn/image-20211015150657029.png)

> Root objects can be global objects, DOM elements or local variables

## 垃圾回收后

内存被释放

![image-20211015150730061](http://myapp.img.mykernel.cn/image-20211015150730061.png)

# nodejs当前状态

> https://stackoverflow.com/questions/12023359/what-do-the-return-values-of-node-js-process-memoryusage-stand-for

```js
var used=process.memoryUsage(); for (let key in used) console.log(`Memory: ${key} ${Math.round(used[key] / 1024 / 1024 * 100) / 100} MB`)
Memory: rss 22.59 MB         # code, heap, stack
Memory: heapTotal 4.82 MB    # V8 engine 分配的heap内存，包含以下heapUsed.
Memory: heapUsed 3.67 MB     # V8 engine 分配用于使用的内存。
Memory: external 1.64 MB     # V8 manages memory bound to JavaScript objects by C++ objects
Memory: arrayBuffers 0.01 MB
```

# 获取heap转储

https://gist.github.com/danielkhan/0a98ae0ae2b5ddfd443e/raw/c93b429e6cfe41e83cc2b8be6ae919f5bc6d80ba/heapdump.js

> 原文链接: http://apmblog.dynatrace.com/2015/11/04/understanding-garbage-collection-and-hunting-memory-leaks-in-node-js/

单个堆意义不大，不会展示堆随着时间发展的状态，使用chrome允许你比较不同时间的堆状态，通过比较查看不同。

![image-20211015153825221](http://myapp.img.mykernel.cn/image-20211015153825221.png)
