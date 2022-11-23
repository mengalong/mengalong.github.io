---
layout: post
title: Docker安装wordpress
category : 读书
tags : [侯世达,读书]
---
1. 下载镜像：
```commandline
docker pull mysql
docker pull wordpress
```
2. 启动mysql容器：
```commandline
docker run -d –privileged=true –name mysql-wp -v /Users/mengalong/soft/wp/data/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -p 3206:3306 mysql
```
3. 启动wordpress容器：
```commandline
docker run -d –name wordpress \
-v /Users/mengalong/soft/wp/data/wp/html:/var/www/html \
–link mysql-wp:mysql \
-p 8080:80 wordpress
```
4. 查看mysql容器的ip：
```commandline
docker inspect mysql-wp | grep -i ipaddr
```
5. 备注：mysql容器启动之后，根据上文指定的root账户和密码连接到mysql需要手动创建wordpress数据库

6. 浏览器访问：localhost:8080 , 根据指导步骤进行配置即可
