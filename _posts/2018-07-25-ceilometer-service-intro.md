---
layout: post
title: Ceilometer原理及介绍
category : OpenStack
tags : [OpenStack|Ceilometer]
---
注: 本文基于当前Openstack的Q版本进行分析

# 1. 背景
ceilometer项目是openstack中用来做计量计费功能的一个组件，后来又逐步发展增加了部分监控采集、告警的功能。由于种种原因，ceilometer项目在Openstack中已经处于一种没落的状态，基本没有什么新的特性开发了，原本该项目的PTL也另起炉灶开始在做Gnocchi项目(ceilometer的后端存储系统)。虽然该项目已经没有前几年活跃，但是还是在很多公有云场景中有比较多的应用，而生产环境中，可能很多公司还用的是M、N版本。

# 2. 基本概念
* meter：
针对一个资源的某些测量值，比如，一个虚拟机可以有多个meters：虚拟机在一段时间内cpu的使用时间、磁盘的请求次数等。在ceilometer中针对这些meter定义了三种类型：
   * Cumulative(累积型): 随着时间会不断增长(eg. disk I/O)
   * Gauge(测量型):离散型的值(eg. floating IPs)和浮动的值(eg. swift对象的数量)
   * Delta(变化量)：一段时间内某个采集值的变化量(eg. 带宽变化量)
* sample：
针对一个特定的meter的具体数据结构
* event:
在某个特定时间，发生的一个动作，比如：19:09:08 创建了一个虚拟机
* resource：
资源，比如instance(虚拟机)、disk(磁盘）都是资源

# 3. High-Level 架构
![ceilo-arch.png](https://upload-images.jianshu.io/upload_images/13183512-ee19f3fef5f4c1e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如上，是当前Ceilometer的一个全局概览逻辑图.
以上ceilometer的每一个服务都是基于可横向扩展来设计的。实际生产环境中，可以根据系统的负载，决定来增加实例或者增加单个实例的worker数。当前Ceilometer主要提供两个核心服务：
1. polling agent：轮询agent，是一个deamon服务，通过周期性调用Openstack内部服务的接口或者一些外部接口获取指定resource的meter数据
2. notification agent：一个deamon服务，通过监听消息队列获取相关数据，将其转换为event和sample，并根据pipeline中定义的方法将数据发送出去
这和以前的版本相比，简化了不少，以前ceilometer包含了(polling-agent,notification-agent,collector，ceilometer-api)
通过Ceilometer收集到的数据可以被发送到不同的后端。Gnocchi是用来提供对捕获到的时间序列的测量数据的存储和查询。Gnocchi未来的趋势是取代当前现存的metering数据存储接口(当前存储在mongodb、mysql等存储后端)。Aodh是一个告警服务，在满足用户设置的告警条件时，可以发送告警信息，其后端可以对接不同的数据库。Panko是用来获取系统中的各类事件并将其存储到对应后端数据库。

# 4. 数据获取的过程
## 4.1 数据采集
![1-agents.png](https://upload-images.jianshu.io/upload_images/13183512-6ef518aa9bf68952.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图展示了ceilometer-agent怎样从不同的数据源获取到数据
Ceilometer共有两种方法来收集数据：
1. Notification agent，从notification bus上获取消息，将其转换为ceilometer的sample或者event数据
2. Polling agent，周期性调用系统中的一些API或者外部工具来获取数据。轮询服务可能会对API服务带来较大的影响，因此对应的API服务需要针对这种轮询机制做一些优化。
以上第一种方法是ceilometer-agent-notification提供的，他可以监控消息队列上的信息。第二种方法是通过polling-agent实现，通过配置可以实现轮询本地虚拟化层或者远程API来获取数据。

## 4.2 notification agent监听数据
![2-1-collection-notification.png](https://upload-images.jianshu.io/upload_images/13183512-3974af1bb890721c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

notification-agent可以消费来自不同服务上报的消息数据。
这个系统的核心是notification-agent这个deamon服务，他可以监听openstack组件(比如nova、glance/cinder/neutron/等)发送到消息队列上的数据，以及ceilometer内部发送过来数据

notification-agent加载ceilometer.notification这个namespace下的一个或者多个插件.每个插件都可以监听任何topic，默认的都会监听notifications.info, notifications.sample,notifications.error.监听进程从配置的topic抓取下来消息之后，将其分发到合适的插件处理成event和sample。

基于Sample的插件提供了一个方法来获取他们所关注的事件类型，然后据此调用对应的回调方法来处理消息。注册的毁掉方法通过notification的pipeline来配置生效与否。通过插件配置的事件类型对新进来的消息进行过滤，因此回调接口最终只会收到他们所关心的数据。

## 4.3 Polling Agent轮询获取数据
![2-2-collection-poll.png](https://upload-images.jianshu.io/upload_images/13183512-db18bd685ce39acd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Polling-agent通过主动调用service接口查询来获取数据。
部署在计算节点上的polling-agent用来轮询计算资源的数据(这样agent可以更有效的和本地虚拟化层交互)，也被称为compute-agent。通过服务API轮询查询非计算资源相关数据的agent部署在控制节点上，也被称为central-agent。在all-in-one环境中，一个agent也支持同时提供以上两种角色。相反的，也可以通过部署多个agent实例来分担系统负载。Polling-agent进程可以加载ceilometer.poll.compute,ceilometer.poll.central,ceilometer.poll.ipmi这三个namespace中配置的插件。

## 4.4 数据处理
### 4.4.1 pipeline manager
![3-Pipeline.png](https://upload-images.jianshu.io/upload_images/13183512-c7597e74d3dd1752.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以上组件实现ceiloemter的pipeline。
ceilometer提供抓取数据的agent，控制数据、通过多重pipeline将数据发送到多种后端的能力。这种功能通过notification-agent来实现。

### 4.4.2 数据发送
![5-multi-publish.png](https://upload-images.jianshu.io/upload_images/13183512-99d302d197fbdbdc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图展示了一个sample数据怎样被发送到多个不同后端
当前处理过的数据可以被发送到8种不同的后端

1. gnocchi, 发送samples/event数据到Gnocchi-api
2. notifier, 发送数据到消息队列，最终可以被内部系统消费
3. udp, 通过UDP包将数据发送出去
4. http, 发送数据到REST接口
5. file, 发送数据到指定的文件
6. zaqar, 一个用于web和移动开发者的多租户云消息和通知服务
7. https, 通过SSL加密的REST接口HTTP服务
8. prometheus, 发送samples数据到 Prometheus Pushgateway

# 5. 存储/访问数据
Ceilometer是设计用来单纯的生产和序列化云数据。Ceilometer生产的数据可以被发送到通过pipeline-publishers中定义 的多重后端。推荐的方法是将数据发送到Gnocchi用于有效的时间序列存储以及资源生命周期的追踪。

参考：
[https://docs.openstack.org/ceilometer/latest/contributor/architecture.html](https://docs.openstack.org/ceilometer/latest/contributor/architecture.html)

