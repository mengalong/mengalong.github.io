---
layout: post
title: shell 脚本不同变量截取方法
category : 编程
tags : [Shell]
---

# 背景
在 shell 脚本中，经常有变量截取的各种处理需求,下面给出一些常见的字符串截取操作办法。

# 举例
1. 从头截取指定某几个字符
```
variable="Hello, World"
result="${variable:0:5}"
echo $result

输出结果: Hello
作用: 截取的是 [0,5) 对应的字符
```
2. 从指定位置截取到字符串末尾
```
variable="Hello, World"
result="${variable:7}"
echo $result
输出结果:World
作用: 截取 [7,] 的字符,从下标为7的字符开始到末尾
```
3. 从末尾截取最后 5 个字符
```
variable="Hello, World"
result="${variable: -5}"
echo $result

输出结果: World
作用: 注意 -5 的前边有一个空格，代表从后往前截取5个字符
```
4. 从末尾截取并指定长度
```
variable="Hello, World"
result="${variable: -5:2}"
echo $result

输出结果: Wo 
作用：先用 -5 从末尾截取出最后的5个字符，然后用 2 截取前一步结果的前两个字符
```
5. 从第一个指定字符开始往后截取
```
variable="Hello-World-Test"
result="${variable#*-}"
echo $result

输出：World-Test
作用：从第一个中划线开始截取到字符串末尾, 注意这里 *- ，*在中划线前边，代表把第一个中划线之前的内容全部删掉
```
6. 从最后一个指定字符开始往前截取
```
variable="Hello-World-Test"
result="${variable%-*}"
echo $result

输出: Hello-World
作用：从最后一个中划线开始截取到字符串的开头，注意这里 -* ， * 在中划线后边，代表把最后一个中划线往后的内容全部删除
```