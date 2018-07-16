---
layout: post
title: Gnocchi+keystone认证失败
category : OpenStack
tags : [Openstack|Gnocchi]
---

# gnocchi 使用keystone认证时提示认证失败

gnocchi.conf中配置的auth\_mode=keystone
执行gnocchi resource list 提示信息如下：
```
Unauthorized: The request you have made requires authentication. (HTTP 401)
```

其原因为export的环境变量中没有指定OS\_AUTH\_TYPE
执行如下命令即可:
```
export OS_AUTH_TYPE=password
```
具体可以参见gnocchi的官方文档:
```
 http://gnocchi.xyz/gnocchiclient/shell.html#authentication-method
```
