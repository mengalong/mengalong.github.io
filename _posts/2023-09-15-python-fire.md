---
layout: post
title: Python fire 库介绍
category : 编程
tags : [Python]
---

# 介绍

Python fire 是一个Python库，只需要一行简单的命令，就可以将python的函数或者组件变成一个命令行调用。
并且提供了通用的方法增加命令行参数的解析，从此你将不需要自己写argparse

# 安装
```python
pip install fire
```
# 实例
## 场景一: fire.Fire()
在代码最后调用，可以将整个代码都加入到命令行
```python
import fire


def hello(name):
    return "Hello, {name}!".format(name=name)


def say(message):
    return "message: {message}".format(message=message)


if __name__ == "__main__":
    fire.Fire()
```

运行方法：
```python
$ python test-fire.py --help
NAME
    test-fire.py

SYNOPSIS
    test-fire.py GROUP | COMMAND

GROUPS
    GROUP is one of the following:

     fire
       The Python Fire module.

COMMANDS
    COMMAND is one of the following:

     hello

     say
    
(venv) mengalong@B000000416727Y fire-module-test $ python test-fire.py say 'hello everyone'
message: hello everyone
(venv) mengalong@B000000416727Y fire-module-test $ python test-fire.py hello "mengalong"
Hello, mengalong!
```
## 场景二： fire.Filre(<fn>)
指定某个函数作为入口，加入到命令行，函数的参数作为命令行的参数
```python
import fire


def main(name):
    return 'Hello {name}'.format(name=name)


if __name__ == "__main__":
    fire.Fire(main)
```

运行效果：
```python
$ python test-fire2.py mengalong
Hello mengalong

```
这种情况下，我们不需要在运行的时候指定main方法，因为在fire.Fire(main)已经默认指定了

## 场景三：指定一个main函数作为统一入口
另一种场景，可以使用main函数作为入口来写
```python
import fire

def hello(name):
    return "Hello {name}".format(name=name)

def main():
    fire.Fire(hello)
    
if __name__ == "__main__":
    main()
```
运行效果
```python
$ python test-fire3.py mengalong
Hello mengalong
```

## 场景四：不修改代码的情况下使用fire

假设有一个 example.py 没有引入fire库
```python
def hello(name):
    return "Hello {name}!".format(name=name)
```
运行效果：
```python
 $ python -m fire example.py hello --name="mengalong"
Hello mengalong!
```

## 场景五：多参数模式
```python
import fire


def hello(name, age):
    return "hello {name} you are {age} years old".format(name=name, age=age)

if __name__ == '__main__':
    fire.Fire(hello)


```
运行效果：
```python
 $ python test-fire5.py --name mengalong --age 14
hello mengalong you are 14 years old
```
# 实例二： 多命令行
## 场景一：多函数入口
```python
import fire

def add(x, y):
    return x + y

def multiply(x, y):
    return x * y

if __name__ == '__main__':
    fire.Fire()
```
运行效果
```python
$ python test-fire6.py add 1 5
6
$ python test-fire6.py multiply 1 5
5
```

## 场景二： dict形式指定入口
使用dict形式指定两个函数的入口
```python
import fire

def add(x, y):
    return x + y

def multiply(x, y):
    return x * y

if __name__ == '__main__':
    fire.Fire({
        'add': add,
       'multiply': multiply
    })
```
运行效果：
```python
$ python test-fire7.py multiply 3 4
12
$ python test-fire7.py add 3 4     
7
```

## 场景三： 指定一个对象作为入口

```python
import fire

class Calculator(object):

    def add(self, a, b):
        return a + b

    def multiply(self, a, b):
        return a * b


if __name__ == '__main__':
    calculator = Calculator()
    fire.Fire(calculator)
```
运行效果：
```python
$ python test-fire8.py add 5 6
11
$ python test-fire8.py multiply 5 6
30
```

未完待续，这个库还有更多有意思的用法


