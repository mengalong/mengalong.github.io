---
layout: post
title: Ceilometer插件采集数据原理及过程
category : OpenStack
tags : [OpenStack|Ceilometer]
---
注：
1. 本文以Openstack Q版本为准分析

# 1. 概述
前边已经对ceilometer polling-agent 的整体框架进行了分析，本文将以memory.resident这个插件为例，分析单个插件采集数据的过程。

# 2. 相关定义
1. memory.resident 插件定义如下:
```Python
memory.resident = ceilometer.compute.pollsters.instance_stats:MemoryResidentPollster
```
2. polling.yaml 中定义如下,代表该插件为300s执行一次

```
---
sources:
    - name: some_pollsters
      interval: 300
      meters:
        - memory.resident
```

# 3. 代码逻辑
## 3.1 获取resources列表
代码入口：ceilometer.polling.manager.PollingTask#poll\_and\_notify
```
  1 for source_name, pollsters in iter_random(
  2         self.pollster_matches.items()):
  3     for pollster in iter_random(pollsters):
  4         key = Resources.key(source_name, pollster)
  5         candidate_res = list(
  6             self.resources[key].get(discovery_cache))
  7         if not candidate_res and pollster.obj.default_discovery:
  8             candidate_res = self.manager.discover(
  9                 [pollster.obj.default_discovery], discovery_cache)
```
1. 在poll\_and\_notify函数的如上几行中，就是获取resources的过程
2. 第4~6行，都是在cache中获取数据，假设没有cache，首次初始化的情况，就直接第8行
3. 第8行，pollster.obj.default\_discovery 是memory.resident这个插件对象中的default\_discovery属性
ceilometer.compute.pollsters.GenericComputePollster#default\_discovery
```
    @property
    def default_discovery(self):
        return 'local_instances'
```
4. 第8行这里调用self.manager.discover传入的参数为:
   * ['local_instance']
   * {}
5. 进而去看self.manager.discover方法

## 3.2 AgentManager中discover方法的实现：
入口：ceilometer.polling.manager.AgentManager#discover
```
  1 discover(self, discovery=None, discovery_cache=None):
  2 resources = []
  3 discovery = discovery or []
  4 for url in discovery:
  5     if discovery_cache is not None and url in discovery_cache:
  6         resources.extend(discovery_cache[url])
  7         continue
  8     name, param = self._parse_discoverer(url)
  9     discoverer = self._discoverer(name)
 10     if discoverer:
 11         try:
 12             if discoverer.KEYSTONE_REQUIRED_FOR_SERVICE:
 13                 service_type = getattr(
 14                     self.conf.service_types,
 15                     discoverer.KEYSTONE_REQUIRED_FOR_SERVICE)
 16                 if not keystone_client.get_service_catalog(
 17                         self.keystone).get_endpoints(
 18                             service_type=service_type):
 19                     LOG.warning(
 20                         'Skipping %(name)s, %(service_type)s service '
 21                         'is not registered in keystone',
 22                         {'name': name, 'service_type': service_type})
 23                     continue
 24
 25             discovered = discoverer.discover(self, param)
 26
 27             if self.partition_coordinator:
 28                 discovered = [
 29                     v for v in discovered if self.hashrings[
 30                         self.construct_group_id(discoverer.group_id)
 31                     ].belongs_to_self(six.text_type(v))]
 32
 33             resources.extend(discovered)
 34             if discovery_cache is not None:
 35                 discovery_cache[url] = discovered
 36         except ka_exceptions.ClientException as e:
 37             LOG.error('Skipping %(name)s, keystone issue: '
 38                       '%(exc)s', {'name': name, 'exc': e})
 39         except Exception as err:
 40             LOG.exception('Unable to discover resources: %s', err)
 41     else:
 42         LOG.warning('Unknown discovery extension: %s', name)
 43 return resources
```
1. 第8行，解析url，当前url = 'local_instance', 解析之后name='local\_instance', param=None
2. 第9行，根据local_instance这个名字，从self.discoveries 中获取对应discovery的对象，实际对应的也就是setup.cfg中定义的该插件对象,该对象实现了一个默认的方法 discover
```
local_instances = ceilometer.compute.discovery:InstanceDiscovery
```
3. 第25行，调用该插件的discover方法，即可获取到resources列表。当前的local_instance 插件内部实现方法为调用libvirt的接口，获取当前agent-compute所在主机上的虚拟机列表，以我当前环境为例，本机上创建了一个名为mengalong的kvm虚拟机，获取到的结果如下：
```
(Pdb) p discovered
[<NovaLikeServer: mengalong>]
(Pdb) p discovered[0].__dict__
{'status': 'active', 'flavor': {'name': 'm1.tiny', 'ram': 512, 'ephemeral': 0, 'vcpus': 1, 'swap': 0, 'disk': 1, 'id': u'1'}, 'hostId': '4e63f5b48c402619503a916fc411c549a6b4337f14c7a379f24778d2', 'OS-EXT-SRV-ATTR:host': 'localhost.localdomain', 'name': 'mengalong', 'tenant_id': '2b45d8c84d24435aa29ec12e6d1426e0', 'image': {'id': 'f12c5ea1-a14c-4f71-b622-30d57c163b62'}, 'architecture': 'x86_64', 'OS-EXT-STS:vm_state': 'running', 'OS-EXT-SRV-ATTR:instance_name': 'instance-00000001', 'user_id': 'bed6963f32454af1ba30093ee415fbfe', 'os_type': 'hvm', 'id': '40b0fcf9-6770-4ebc-95e7-8a3b15ebebd0', 'metadata': {}}
```

## 3.3 调用memory.resident插件的get_sample方法
以上两步获取到了虚拟机列表之后，再回到之前的 poll_and_notify方法中，下一步就是调用memory.resident 插件对应的get_sample 方法来采集对应虚拟机的详细数据
入口：ceilometer/polling/manager.py:183
```
    samples = pollster.obj.get_samples(
                        manager=self.manager,
                        cache=cache,
                        resources=polling_resources
                    )
```

## 3.4 插件memory.resident的get_sample方法的实现：
入口：ceilometer.compute.pollsters.GenericComputePollster#get_samples
```
  1 def get_samples(self, manager, cache, resources):
  2     self._inspection_duration = self._record_poll_time()
  3     for instance in resources:
  4         try:
  5             polled_time, result = self._inspect_cached(
  6                 cache, instance, self._inspection_duration)
  7             if not result:
  8                 continue
  9             for stats in self.aggregate_method(result):
 10                 yield self._stats_to_sample(instance, stats, polled_time)
```
1. 第2行，获取该插件本次执行时距离上次执行的时间差
2. 第3行，遍历虚拟机列表
3. 调用self.\_inspect_cached方法获取指定虚拟机的相关监控数据

## 3.5 插件memory.resident的inspect方法
1. 在该插件的父类中，定义了inspector_method = 'inspect_instance'
3. 在插件初始化的时候，self.inspector = self._get_inspector(self.conf) 初始化了本插件对应的inspector
4. inspector对象初始化的时候，会根据配置文件中定义的hypervisor_inspector从ceilometer.compute.virt中加载对应的inspector。当前我的环境中hypervisor_inspector=libvirt(具体在ceilometer/compute/virt/inspector.py中定义)

## 3.6 通过本插件的insperctor的inspect_method对应的方法获取指定虚拟机的监控数据
1. 本插件定义的inspect_method='inspect_instance'
2. 加载的inspector为libvirt对应的对象
```
libvirt = ceilometer.compute.virt.libvirt.inspector:LibvirtInspector
```
3. 调用ceilometer.compute.virt.libvirt.inspector.LibvirtInspector#inspect_instance 这个方法获取到的数据为：
```
{'cpu_number': 1, 'cpu_time': 561600559525L, 'memory_swap_in': None, 'memory_usage': None, 'memory_bandwidth_total': None, 'memory_bandwidth_local': None, 'cpu_cycles': None, 'cache_references': None, 'cpu_util': None, 'memory_swap_out': None, 'cache_misses': None, 'cpu_l3_cache_usage': None, 'memory_resident': 8L, 'instructions': None}
```
4. 本实例中会再获取返回数据中的 memory_resident 对应的值，层层返回到poll_and_notify方法中，对应的sample结构为：

```
(Pdb) p sample
<name: memory.resident, volume: 181, resource_id: 40b0fcf9-6770-4ebc-95e7-8a3b15ebebd0, timestamp: None>
(Pdb) p sample.__dict__
{'user_id': 'bed6963f32454af1ba30093ee415fbfe', 'name': 'memory.resident', 'resource_id': '40b0fcf9-6770-4ebc-95e7-8a3b15ebebd0', 'timestamp': None, 'id': '93cad640-920e-11e8-aa10-080027e77a00', 'volume': 181L, 'source': 'openstack', 'monotonic_time': 2703.940111794, 'project_id': '2b45d8c84d24435aa29ec12e6d1426e0', 'type': 'gauge', 'resource_metadata': {'status': 'active', 'disk_gb': 1, 'instance_host': 'localhost.localdomain', 'image': {'id': 'f12c5ea1-a14c-4f71-b622-30d57c163b62'}, 'ephemeral_gb': 0, 'host': '4e63f5b48c402619503a916fc411c549a6b4337f14c7a379f24778d2', 'flavor': {'name': 'm1.tiny', 'ram': 512, 'ephemeral': 0, 'vcpus': 1, 'swap': 0, 'disk': 1, 'id': u'1'}, 'task_state': u'', 'image_ref_url': None, 'memory_mb': 512, 'root_gb': 1, 'display_name': 'mengalong', 'name': 'instance-00000001', 'vcpus': 1, 'instance_id': '40b0fcf9-6770-4ebc-95e7-8a3b15ebebd0', 'instance_type': 'm1.tiny', 'state': 'running', 'image_ref': 'f12c5ea1-a14c-4f71-b622-30d57c163b62', 'architecture': 'x86_64', 'os_type': 'hvm'}, 'unit': 'MB'}
```

## 3.7 数据入库
### 3.7.1 调用agent的notifier的sample方法，将数据发送到消息队列
代码入口：ceilometer.polling.manager.PollingTask#_send_notification
```
    def _send_notification(self, samples):
        self.manager.notifier.sample(
            {},
            'telemetry.polling',
            {'samples': samples}
        )
```
这里self.manager.notifier 是oslo_messaging.Notifier的对象
### 3.7.2 notifier的sample方法：
代码入口：oslo_messaging.notify.notifier.Notifier#sample
这就对应的是公共库oslo.messaging库中的方法,最终将数据发送到了消息队列的 telemetry.polling 队里上

## 3.8 数据最终入库:
截止到上边3.7.2，插件采集到的sample数据都被发送到了消息队列，之后ceilometer-agent-notification会监听消息队列上的内容，获取到消息数据之后，根据对应数据的定义将其发送到合适的后端入库。具体过程下一篇文章再继续分析。


