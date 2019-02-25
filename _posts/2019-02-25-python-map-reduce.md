---
layout: post
title: Python map、reduce函数用法
category : Python
tags : [Linux,Python]
---

最近解决两个小问题的时候用到了python的map和reduce函数，简单用法做个记录：
# map 函数
## 语法：map(function, iterable,...)
用法：
对列表中的每一个元素应用function函数，并返回对应新的列表

参数：
function -- 函数的引用
iterable -- 一个或者多个序列

返回值：
一个新的列表

## 实例：
1. 对一个列表的所有元素计算平方
```
	def sequare(x):
	    return x*x
	
	print map(sequare, [1,2,3])
	
	结果为：
	[1, 4, 9]
```
1. 将一个数值列表的每个元素转换为字符串之后用'#'连接
```
	print "#".join(map(str, [1,2,3]))
	
	结果为:
	1#2#3
```
1. 将两个列表对位的元素相加，并返回新列表
```
	def add_list(x, y):
	    return x + y
	
	print map(add, [1,2,3], [4,5,6])
	
	结果为：
	[5, 7, 9]
```

# reduce 函数
## 语法：reduce(function, iterable[, initializer])
用法：
对列表中的所有元素执行function的进行累积计算，返回最终计算结果

参数：
function -- 函数的引用
iterable -- 列表或者可迭代对象

返回值：
最终的计算结果

## 实例

1. 计算列表所有元素的和
普通方法可以通过for循环遍历即可实现，用reduce只需要简单的一行
```
	def add(x, y):
	    return x + y
	print reduce(add, [1,2,3])
	
	结果为：
	6
```
