---
layout: post
title: 向OpenStack社区提交代码经验
category : OpenStack
tags : [OpenStack, tooz]
---

# 向OpenStack社区提交代码经验
最近给OpenStack的一个子项目tooz提交了一个patch，在和社区人沟通的过程中，看到了他们精益求精

的态度同时也学到了不少内容，这里做个简单的记录

## 准备工作
如果你还不知道如何给社区提交代码，那么可以参考: https://docs.openstack.org/infra/manual/developers.html

## tooz 功能简述

tooz项目的功能可以参见: https://github.com/openstack/tooz

简单来说，tooz实现的是一种分布式调度的任务分配能力。他的一些核心数据需要存储在指定的后端中，

tooz支持的一种存储后端就是文件(FileDriver). 我的这个patch就是为这个FileDriver实现了

heartbeat()功能.

## 一个Patch的演变

整体内容可以参见: https://review.openstack.org/#/c/453095/

### Version1:

#### 基本思路:
1. 增加timeout参数，用来控制一个member的生命周期
2. 在join_group的时候，增加 heartbeat_on 参数，用来记录每次heartbeat的时间
3. 增加heartbeat()函数，每次执行heartbeat的时候，修改heartbeat_on的时间戳
4. 修改get_members()函数，每次比较heartbeat_on的时间和当前时间之间的差值记为delta_second
   如果delta_second > timeout, 则认为这个member已经死掉了

#### 具体修改点:
1. 在 \_\_init\_\_ 方法中增加了timeout参数，用来控制单个member的失效周期
```cython
        self.timeout = int(options.get('timeout', ['10'])[0])
```

2. 在join_group()中修改meta的格式，增加heartbeat_on参数
```cython

       def _do_join_group():
            if not os.path.exists(os.path.join(group_dir, ".metadata")):
                raise coordination.GroupNotCreated(group_id)
            if os.path.isfile(me_path):
                raise coordination.MemberAlreadyExist(group_id,
                                                      self._member_id)

            current_time = datetime.datetime.now()
            details = {
                u'capabilities': capabilities,
                u'joined_on': current_time,
                u'heartbeat_on': current_time,
                u'member_id': utils.to_binary(self._member_id,
                                              encoding="utf-8")
            }
```

3. 在get_members()中增加过滤条件判断，重点关注第二个 try...except 中的内容
```cython
        def _do_get_members():
            if not os.path.isdir(group_dir):
                raise coordination.GroupNotCreated(group_id)
            members = set()
            try:
                entries = os.listdir(group_dir)
            except EnvironmentError as e:
                # Did someone manage to remove it before we got here...
                if e.errno != errno.ENOENT:
                    raise
            else:
                for entry in entries:
                    if not entry.endswith('.raw'):
                        continue
                    entry_path = os.path.join(group_dir, entry)
                    try:
                        details = self._get_member_detail(entry_path)
                        current_time = datetime.datetime.now()
                        delta_time = timeutils.delta_seconds(
                            details.get('heartbeat_on'), current_time)
                        if delta_time < 0 or delta_time > self.timeout:
                            continue
                        member_id = details.get('member_id')
                    except EnvironmentError as e:
                        if e.errno != errno.ENOENT:
                            raise
                    else:
                        members.add(member_id)
            return members
```

4. 增加heartbeat()方法:
```cython
    def heartbeat(self, group_id=None):
        if not group_id:
            return self.timeout
        safe_group_id = self._make_filesystem_safe(group_id)
        group_dir = os.path.join(self._group_dir, safe_group_id)
        safe_member_id = self._make_filesystem_safe(self._member_id)
        member_path = os.path.join(group_dir, "%s.raw" % safe_member_id)

        @_lock_me(self._driver_lock)
        def _do_heartbeat():
            details = self._get_member_detail(member_path)
            current_time = datetime.datetime.now()
            details['heartbeat_on'] = current_time
            details_blob = utils.dumps(details)
            with open(member_path, "wb") as fh:
                fh.write(details_blob)
        _do_heartbeat()
        return self.timeout
```

#### 存在的问题:
1. timeout 参数只在FileDriver内部使用，因此应该为私有变量
2. timeout 参数从options中获取的时候，应该使用现成的方法，而不是自己通过读取列表的第0个元素
3. heartbeat时间戳的记录，可以有更好的方法，不需要引入heartbeat_on 这个参数，这样会修改原始的数据结构
4. heartbeat()方法，原始接口中没有group_id的参数，因此这里引入group_id这个参数也是不合理的
5. heartbeat()方法中，获取safe_member_id 的过程实际是不需要的，因为在init方法中已经有对应的变量可以直接使用

### Version2：

#### 基本思路:
修改heartbeat的策略，不再显式记录heartbeat_on时间戳，而是通过每次heartbeat的时候修改对应文件的
时间戳，之后直接读取文件的最后修改时间作为heartbeat的时间戳

#### 具体修改点:

1. 修改init方法中timeout的处理过程，主要点：修正为私有变量，通过既有接口处理列表参数
```cython
        self._options = utils.collapse(options)
        self._timeout = int(self._options.get('timeout', 10))
```

2. 修改get_members中，获取时间戳的方法
```cython

                    try:
                        m_time = datetime.datetime.fromtimestamp(
                            os.stat(entry_path).st_mtime)
                        current_time = datetime.datetime.now()
                        delta_time = timeutils.delta_seconds(m_time,
                                                             current_time)
                        if delta_time >= 0 and delta_time <= self._timeout:
                            member_id = self._read_member_id(entry_path)
                        else:
                            continue
```

3. 修改heartbeat()方法：去掉group_id参数，修改heartbeat方法的实现
```cython
    def heartbeat(self):
        for group_id in self._joined_groups:
            safe_group_id = self._make_filesystem_safe(group_id)
            group_dir = os.path.join(self._group_dir, safe_group_id)
            member_path = os.path.join(group_dir, "%s.raw" %
                                       self._safe_member_id)

            @_lock_me(self._driver_lock)
            def _do_heartbeat():
                current_time = int(time.time())
                try:
                    os.utime(member_path, (current_time, current_time))
                except EnvironmentError as err:
                    if err.errno != errno.ENOENT:
                        raise
            _do_heartbeat()
        return self._timeout
```

#### 存在的问题
在本次修改之后，个人认为代码已经非常精简和优雅了，不过还是被社区的高手指出了问题

1.  os.utime(....) This can be replaced by: os.utime(member_path, None)
很明显，我的patch中获取current_time这一步是多余的，因此最终按照建议修改

### Version3

最终版本的代码可以参见:https://review.openstack.org/#/c/453095/7/tooz/drivers/file.py

## 总结:
在这个过程中，我主要犯了如下错误:

1. 私有变量、公共变量的使用场景的使用缺少原则
2. 提交patch，修改了原生接口的定义
3. 实现方案并不是最优化的选择
4. 使用已有模块时，对函数的使用没有仔细斟酌

如果是自己用或者在公司用，或许我的第一个版本就直接上线使用了，因为他也是解决了我们遇到的问题。
但是经不起仔细推敲和斟酌，而这种精益求精的过程或许只能在社区项目中来学习了。这次也是刷新了我的
很多观念，对个人的成长还是大有裨益。
