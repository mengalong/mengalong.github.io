---
layout: post
title: 使用pdb调试ceilometer代码
category : OpenStack
tags : [OpenStack|Ceilometer|pdb]
---

# Newton版本以前，Ceilometer代码调试方法：
ceilometer在Newton版本以前，polling-agent使用的是oslo_service模块启动的进程，因此可以直接使用python内建的模块 pdb 直接进行调试。具体调试方法如下：

1. 在需要增加断点的地方，加入一行代码:
```
import pdb;pdb.set_trace()
```
2. 之后在命令行启动进程，即可走到断点处:
```
/usr/bin/ceilometer-polling --polling-namespaces=central --logfile /var/log/ceilometer.log
```
3. 进入断点之后，pdb基本的命令如下：

```
p var : 打印指定变量var的内容
p  dict_var.__dict__ : 打印dict类型变量的内容
n : 跳过当前行继续执行下一行
s : 进入当前行对应函数的内部
```

# Newton版本以后，Ceilometer代码调试方法：

ceilometer在Newton版本开始，使用cotyledon 这个模块来启动进程框架，该模块底层是multiprocess，启动进程框架之后会进入子进程，此时使用上边的方法就不能进入到子进程的debug，此时需要使用如下方法：

1. 增加一个新的类，用于调试多线程

```
import sys
import pdb

class ForkedPdb(pdb.Pdb):
    def interaction(self, *args, **kwargs):
        _stdin = sys.stdin
        try:
            sys.stdin = open('/dev/stdin')
            pdb.Pdb.interaction(self, *args, **kwargs)
        finally:
            sys.stdin = _stdin
```
2. 在需要添加断点的地方，使用如下方法添加断点：
```
ForkedPdb().set_trace()
```
3. 之后启动进程，即可进入到pdb调试模式，具体的命令和上边一致

