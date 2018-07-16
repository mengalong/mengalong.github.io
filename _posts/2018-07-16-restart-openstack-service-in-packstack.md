---
layout: post
title: packstack安装的openstack服务如何重启
category : OpenStack
tags : [OpenStack|packstack]
---

&nbsp;&nbsp;&nbsp;&nbsp;最近使用packstack在virtualbox的centos7虚拟机中安装了单节点openstack服务实验使用openstack。
&nbsp;&nbsp;&nbsp;&nbsp;具体怎样使用packstack安装openstack服务，可以参见：[RDO官方文档](https://www.rdoproject.org/install/packstack/)，以前使用devstack安装过，怎奈花了很久的时间也没成功，最终放弃了，近期看到packstack了，拿来试用了一下，还是非常好用的，基本实现了一键安装的能力。服务部署完成之后，在centos下，要重启openstack指定服务，当前有两种方法，具体如下：

### 方法一：

1. yum安装openstack-utils:yum install openstack-utils -y
2. 安装完成之后，可以使用opestack-service 命令对基础服务进行管理
3. 具体可以尝试如下命令：
```
openstack-service --help
openstack-service list
openstack-service restart openstack-ceilometer-polling
```
4. openstack-service 实际是一个shell脚本，底层实际调用的是centos的systemctl工具，因此我们也可以使用systemctl工具来重启服务

### 方法二:
1. 使用systemctl列出所有openstack服务
```
systemctl list-unit-files --type=service --ful --no-legend --no-pager | egrep "^(openstack|neutron|quantum)" | grep -v 'neutron-.*-cleanup' | grep enable
```
2. 查看指定服务当前的运行状态
```
systemctl status openstack-ceilometer-polling
```
3. 重启指定服务
```
systemctl restart openstack-ceilometer-polling
```
4. 停止指定服务
```
systemctl stop openstack-ceilometer-polling
```

