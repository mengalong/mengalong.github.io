---
layout: post
title: Django 使用xadmin时设置初始后台用户名密码
category : 编程
tags : [Python, Django]
---

# Django 配置好xadmin之后设置后台初始密码

* 执行如下命令初始化数据库
```
python manage.py makemigrations
python manage.py migrate
```

* 执行如下命令，设置管理员账户信息
```
python manage.py createsuperuser
```
