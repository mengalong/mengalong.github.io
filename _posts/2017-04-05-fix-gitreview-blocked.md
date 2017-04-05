---
layout: post
title: 解决review.openstack.org:29418 被墙
category : OpenStack
tags : [OpenStack, gitreview, review.openstack.org]
---


# 解决review.openstack.org:29418被墙的问题

默认状态下我们向openstack项目提交代码使用的是ssh协议，对应端口为29418.但是我们发现，这个端口经常被墙，不能访问，那么

```
例如下边的例子: 
mengalong@along-mac:~/code/git/tooz$ git remote -v
gerrit     ssh://mengalong@review.openstack.org:29418/openstack/tooz.git (fetch)
gerrit     ssh://mengalong@review.openstack.org:29418/openstack/tooz.git (push)
origin     https://github.com/openstack/tooz.git (fetch)
origin     https://github.com/openstack/tooz.git (push)
```

这种情况下我们应该怎么办？

解决方法:
方法一: 如果自己有VPN 那么走VPN 

方法二: 如果没有VPN， 那么我们就可以选择不用默认的ssh://协议而使用 https://协议，具体方法如下:

```
1. 首先登陆review.openstack.org,找到 https://review.openstack.org/#/settings/http-password 设置https密码，选择生成密码
   之后系统会自动生成一个随机密码
2. 修改git review的配置，在修改之前，可以先使用 git remote -v 查看本机是否已经设置了 gerrit 相关的条目,
   如果已经设置过了,则执行如下命令进行修改
   git remote set-url gerrit https://username:password@review.openstack.org/openstack/python-cinderclient.git
   如果没有设置过，则执行如下命令进行添加
   git remote add gerrit https://username:password@review.openstack.org/openstack/python-cinderclient.git(有gerrit)
3. 设置gitreview的端口: git config --global gitreview.port 443
4. 设置gitreview的协议: git config --global gitreview.scheme https
5. 之后执行 git review -s 测试,如果没有任何输出，则说明设置成功了，那么就可以愉快的提交代码了
```

