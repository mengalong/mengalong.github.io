---
layout: post
title: Ceilometer Compute Agent 代码分析
category : OpenStack
tags : [OpenStack|Ceilometer]
---
注：
1. 本文以Openstack Q版本为准分析
2. 本文内容较长，通过本文，你可以从代码层面了解到polling-agent的整个启动运行过程，其中也包含了部分代码的细节分析

# 1. 背景:
前边我们说过，Ceilometer中的agent主要是两个，Polling Agent 和 Notification Agent,而Polling Agent中，主要又包含两个：Compute Agent + Central Agent, 本文就来分析Polling Agent的代码基本原理。通过对代码的分析，来进一步了解Polling Agent的工作机制。

# 2. 服务启动命令
本文测试环境是通过packstack在虚拟机环境中安装的一套 all-in-one Openstack环境。
Polling Agent的启动命令如下：
```python
/usr/bin/ceilometer-polling --logfile /var/log/ceilometer/polling.log
```

# 3. 进程启动基本流程
启动代码的入口在
ceilometer.cmd.polling.main
```python
def main():
    conf = cfg.ConfigOpts()
    conf.register_cli_opts(CLI_OPTS)
    service.prepare_service(conf=conf)
    sm = cotyledon.ServiceManager()
    sm.add(create_polling_service, args=(conf,))
    oslo_config_glue.setup(sm, conf)
    sm.run()
```
从这里可以看到，整体的基本过程为：
1. 初始化配置文件，注册默认的namespaces配置项
2. 调用service.prepare_service方法，初始化基础配置项，主要包含：keystone认证、日志配置项、配置文件等
3. 启动进程框架
4. 通过sm.add(create_polling_service,args=(conf,))初始化进程实例AgentManager
5. 通过sm.run() 启动数据采集进程
以上，这里边最主要的在于2，4，5这三步，接下来分别分析。

# 4. 代码分析
## 4.1 配置初始化：
配置初始化的过程主要在service.prepare_service()这个方法中，这里我们只简单说明一下基础配置项如何进行初始化
在ceilometer.service.prepare_service函数中：
```
def prepare_service(argv=None, config_files=None, conf=None):
    if argv is None:
        argv = sys.argv

    if conf is None:
        conf = cfg.ConfigOpts()
...
    for group, options in opts.list_opts():
        conf.register_opts(list(options),
                           group=None if group == "DEFAULT" else group)
 ...

    conf(argv[1:], project='ceilometer', validate_default_values=True,
         version=version.version_info.version_string(),
         default_config_files=config_files)
```

这里基本的过程为:
1. 初始化一个oslo_config的实例conf
2. 对ceilometer.opts.list_opts中定义的各类参数进行遍历初始化默认值(具体初始化的方法，这里不做赘述
3. 最终的效果是在conf这个实例中，获取list_opts所列出来的各类参数项以及他们的默认值初始化到conf中
4. 调用conf(argv[1:],project='ceilometer',......) 方法，用argv[1]中指定配置文件的配置项对conf中的默认值进行覆盖完成配置项初始化

## 4.2. AgentManager初始化
### 4.2.1 AgentManager初始化入口：
代码路径：ceilometer.cmd.polling.create_polling_service
```
def create_polling_service(worker_id, conf):
    return manager.AgentManager(worker_id,
                                conf,
                                conf.polling_namespaces)
```
这个函数中，调用AgentManager进行初始化,传入三个参数：
worker_id: 这个参数在哪里初始化的还没搞清楚
conf：即上一步初始化的配置对象
conf.polling_namespaces: 使用默认的配置，即：['compute', 'central']

### 4.2.2 AgentManager初始化过程
代码路径：ceilometer.polling.manager.AgentManager#__init__
```
 1     def __init__(self, worker_id, conf, namespaces=None):
  2         namespaces = namespaces or ['compute', 'central']
  3         group_prefix = conf.polling.partitioning_group_prefix
  4
  5         super(AgentManager, self).__init__(worker_id)
  6
  7         self.conf = conf
  8
  9         if type(namespaces) is not list:
 10             namespaces = [namespaces]
 11
 12         # we'll have default ['compute', 'central'] here if no namespaces will
 13         # be passed
 14         extensions = (self._extensions('poll', namespace, self.conf).extensions
 15                       for namespace in namespaces)
 16         # get the extensions from pollster builder
 17         extensions_fb = (self._extensions_from_builder('poll', namespace)
 18                          for namespace in namespaces)
 19
 20         self.extensions = list(itertools.chain(*list(extensions))) + list(
 21             itertools.chain(*list(extensions_fb)))
 22
 23         if not self.extensions:
 24             LOG.warning('No valid pollsters can be loaded from %s '
 25                         'namespaces', namespaces)
 26
 27         discoveries = (self._extensions('discover', namespace,
 28                                         self.conf).extensions
 29                        for namespace in namespaces)
 30         self.discoveries = list(itertools.chain(*list(discoveries)))
 31         self.polling_periodics = None
 32
 33         self.hashrings = None
 34         self.partition_coordinator = None
 35         if self.conf.coordination.backend_url:
 36             # XXX uuid4().bytes ought to work, but it requires ascii for now
 37             coordination_id = str(uuid.uuid4()).encode('ascii')
 38             self.partition_coordinator = coordination.get_coordinator(
 39                 self.conf.coordination.backend_url, coordination_id)
 40
 41         # Compose coordination group prefix.
 42         # We'll use namespaces as the basement for this partitioning.
 43         namespace_prefix = '-'.join(sorted(namespaces))
 44         self.group_prefix = ('%s-%s' % (namespace_prefix, group_prefix)
 45                              if group_prefix else namespace_prefix)
 46
 47         self.notifier = oslo_messaging.Notifier(
 48             messaging.get_transport(self.conf),
 49             driver=self.conf.publisher_notifier.telemetry_driver,
 50             publisher_id="ceilometer.polling")
 51
 52         self._keystone = None
 53         self._keystone_last_exception = None
```

1. 第2行，初始化namespaces为 ['compute', 'central']
2. 第3行，是用于partition功能的，暂时忽略
3. 第14~21行，加载数据采集插件，加载了ceilometer.poll.compute, ceilometer.poll.central, ceilometer.builder.poll.central 这三个namespaces下对应的插件，具体包含哪些插件可以去看setup.cfg中的定义
4. 加载完成之后，self.extensions 是一个list，其中是多个元素，每个元素就是一个插件对象
```
(Pdb) p self.extensions[1]
<stevedore.extension.Extension object at 0x7f1cc9ef9310>
(Pdb) p self.extensions[1].__dict__
{'obj': <ceilometer.network.services.fwaas.FirewallPollster object at 0x7f1cc9ef92d0>, 'entry_point': EntryPoint.parse('network.services.firewall = ceilometer.network.services.fwaas:FirewallPollster'), 'name': 'network.services.firewall', 'plugin': <class 'ceilometer.network.services.fwaas.FirewallPollster'>}
```
5. 第30行，加载discovery插件，加载了ceilometer.discover.central,ceilometer.discover.central 这两个namespaces下对应的插件。discovery插件在数据采集的时候会用到，这里不展开，后续单独介绍。加载完成后，self.discoveries 和 self.extensions的类型一样，都是stevedore.extension.Extension 对象。
6. 第47~50行，初始化self.notifier 对象，这个对象主要是用来和消息队列进行交互的，默认消息队列使用的是rabbitmq。该对象的作用主要是未来在数据采集完成之后，会调用这里self.notifier的接口将数据发送到rabbitmq。该对象的主要变量是：
```
(Pdb) p self.notifier.__dict__
{'_serializer': <oslo_messaging.serializer.NoOpSerializer object at 0x7f1cc9aa3710>, '_driver_mgr': <stevedore.named.NamedExtensionManager object at 0x7f1cc9aa3890>, 'retry': -1, '_driver_names': ['messagingv2'], '_topics': ['notifications'], 'publisher_id': 'ceilometer.polling', 'transport': <oslo_messaging.transport.NotificationTransport object at 0x7f1cc9b26550>}
(Pdb) p self.notifier._driver_mgr
<stevedore.named.NamedExtensionManager object at 0x7f1cc9aa3890>
(Pdb) p self.notifier._driver_mgr.__dict__
{'_extensions_by_name_cache': None, '_names': ['messagingv2'], 'namespace': 'oslo.messaging.notify.drivers', '_on_load_failure_callback': None, 'extensions': [<stevedore.extension.Extension object at 0x7f1cc9aa3c10>], 'propagate_map_exceptions': False, '_name_order': False, '_missing_names': set([])}
```
   * self.notifier.topics = ['notifications']
   * self.notifier._driver_names = 'messagingv2'(定义在ceilometer/publisher/messaging.py 中的 telemetry_driver这个变量)，代表的是ceilometer向消息队列发送消息时使用的驱动类型
   * self.notifier.publisher_id = 'ceilometer.polling'
   * self.notifier._driver_mgr 是从oslo.messaging.notify.drivers 中加载名称为messagingv2对应的插件
   * self.notifier.transport 是从oslo.messaging.drivers 中加载名称为 rabbit对应的插件。因为ceilometer.conf 中 transport_url定义是：transport_url=rabbit://guest:guest@192.168.2.120:5672/

## 4.3. AgentManager服务启动
### 4.3.1 AgentManager 服务启动入口:

代码路径: ceilometer.polling.manager.AgentManager#run

```
    def run(self):
        super(AgentManager, self).run()
        self.polling_manager = PollingManager(self.conf)
        if self.partition_coordinator:
            self.partition_coordinator.start()
            self.join_partitioning_groups()
        self.start_polling_tasks()
```

这里主要是第三行和最后一行，进入进程启动的具体过程，其中：
第3行，初始化polling_manager
第4行，启动周期任务

### 4.3.2 初始化polling_manager
1. 入口：ceilometer/polling/manager.py:385

```
self.polling_manager = PollingManager(self.conf)
```

通过PollingManager+conf文件初始化polling_manager
2. 在ceilometer/polling/manager.py:54中定义了polling.cfg_file 的初始化为 polling.yaml,该配置文件对应的格式如下,其中定义了不同插件的采集周期，这里的meters中的名称需要和extensions中加载的插件名对应，只有在这里定义了采集周期的插件，最终才会周期性调度

```
{"sources": [{"name": source_1,
                      "interval": interval_time,
                      "meters" : ["meter_1", "meter_2"],
                      "resources": ["resource_uri1", "resource_uri2"],
                     },
                     {"name": source_2,
                      "interval": interval_time,
                      "meters" : ["meter_3"],
                     },
                    ]}
        }
```

3. 根据加载的配置文件，初始化sources,
入口：ceilometer.polling.manager.PollingSource#__init__

```
        super(PollingManager, self).__init__(conf)
        cfg = self.load_config(conf.polling.cfg_file)
        self.sources = []
        if 'sources' not in cfg:
            raise PollingException("sources required", cfg)
        for s in cfg.get('sources'):
            self.sources.append(PollingSource(s))

```

1.  按照polling.yaml 中的分组，对sources进行初始化，最终对应的就是self.polling_manager.sources
2. self.polling_manager.sources中的每一个对象就是一个PollingSource,每一个PollingSource关键的参数：
   * meters: 即为上边polling.yaml 中每一组source中的meter
   * interval：即为polling.yaml 中的interval

```
(Pdb) p self.polling_manager.sources
[<ceilometer.polling.manager.PollingSource object at 0x7f71a2f48150>]
(Pdb) p self.polling_manager.sources[0].__dict__
{'name': 'some_pollsters', 'cfg': {'interval': 300, 'meters': ['cpu', 'cpu_l3_cache', 'memory.usage', 'network.incoming.bytes', 'network.incoming.packets', 'network.outgoing.bytes', 'network.outgoing.packets', 'disk.device.read.bytes', 'disk.device.read.requests', 'disk.device.write.bytes', 'disk.device.write.requests', 'hardware.cpu.util', 'hardware.memory.used', 'hardware.memory.total', 'hardware.memory.buffer', 'hardware.memory.cached', 'hardware.memory.swap.avail', 'hardware.memory.swap.total', 'hardware.system_stats.io.outgoing.blocks', 'hardware.system_stats.io.incoming.blocks', 'hardware.network.ip.incoming.datagrams', 'hardware.network.ip.outgoing.datagrams'], 'name': 'some_pollsters'}, 'interval': 300, 'meters': ['cpu', 'cpu_l3_cache', 'memory.usage', 'network.incoming.bytes', 'network.incoming.packets', 'network.outgoing.bytes', 'network.outgoing.packets', 'disk.device.read.bytes', 'disk.device.read.requests', 'disk.device.write.bytes', 'disk.device.write.requests', 'hardware.cpu.util', 'hardware.memory.used', 'hardware.memory.total', 'hardware.memory.buffer', 'hardware.memory.cached', 'hardware.memory.swap.avail', 'hardware.memory.swap.total', 'hardware.system_stats.io.outgoing.blocks', 'hardware.system_stats.io.incoming.blocks', 'hardware.network.ip.incoming.datagrams', 'hardware.network.ip.outgoing.datagrams'], 'resources': [], 'discovery': []}
```

### 4.3.3 AgentManager服务启动过程：
启动入口：ceilometer.polling.manager.AgentManager#start_polling_tasks

```
  1 def start_polling_tasks(self):
  2     data = self.setup_polling_tasks()
  3
  4     # Don't start useless threads if no task will run
  5     if not data:
  6         return
  7
  8     # One thread per polling tasks is enough
  9     self.polling_periodics = periodics.PeriodicWorker.create(
 10         [], executor_factory=lambda:
 11         futures.ThreadPoolExecutor(max_workers=len(data)))
 12
 13     for interval, polling_task in data.items():
 14
 15         @periodics.periodic(spacing=interval, run_immediately=True)
 16         def task(running_task):
 17             self.interval_task(running_task)
 18
 19         self.polling_periodics.add(task, polling_task)
 20
 21     utils.spawn_thread(self.polling_periodics.start, allow_empty=True)
```

服务启动的具体过程主要是如下两步:
1. 通过setup_polling_tasks()获取需要加入周期任务的插件列表
2. 按照设置的插件运行周期进行分组，同一周期的插件加入到相同的运行队列中启动运行

#### 4.3.3.1 获取周期任务列表
代码入口：ceilometer.polling.manager.AgentManager#setup_polling_tasks

```
  1 def setup_polling_tasks(self):
  2     polling_tasks = {}
  3     for source in self.polling_manager.sources:
  4         for pollster in self.extensions:
  5             if source.support_meter(pollster.name):
  6                 polling_task = polling_tasks.get(source.get_interval())
  7                 if not polling_task:
  8                     polling_task = PollingTask(self)
  9                     polling_tasks[source.get_interval()] = polling_task
 10                 polling_task.add(pollster, source)
 11     return polling_tasks
```

1. 遍历self.polling_manager.sources中的source
2. 遍历self.extensions，查看如果对应插件在对应source的meters列表中,代表该插件需要定期执行，获取该插件定义的执行周期，将其加入到polling_tasks中
3. 最终polling_tasks格式如下：

```
(Pdb) p data
{300: <ceilometer.polling.manager.PollingTask object at 0x7fa6103394d0>}
(Pdb) p data[300]
<ceilometer.polling.manager.PollingTask object at 0x7fa6103394d0>
(Pdb) p data[300].__dict__
{'manager': <ceilometer.polling.manager.AgentManager object at 0x7fa610fb47d0>, '_telemetry_secret': '2e30d044a69343e0', 'pollster_matches': defaultdict(<type 'set'>, {'some_pollsters': set([<stevedore.extension.Extension object at 0x7fa610689850>, <stevedore.extension.Extension object at 0x7fa610697090>, <stevedore.extension.Extension object at 0x7fa61066c0d0>, <stevedore.extension.Extension object at 0x7fa61066c150>, <stevedore.extension.Extension object at 0x7fa610690190>, <stevedore.extension.Extension object at 0x7fa61066c190>, <stevedore.extension.Extension object at 0x7fa610690050>, <stevedore.extension.Extension object at 0x7fa61066ca50>, <stevedore.extension.Extension object at 0x7fa610690290>, <stevedore.extension.Extension object at 0x7fa61066c2d0>, <stevedore.extension.Extension object at 0x7fa610690890>, <stevedore.extension.Extension object at 0x7fa610733bd0>, <stevedore.extension.Extension object at 0x7fa610667c10>, <stevedore.extension.Extension object at 0x7fa610667490>, <stevedore.extension.Extension object at 0x7fa6106970d0>, <stevedore.extension.Extension object at 0x7fa610689550>, <stevedore.extension.Extension object at 0x7fa610667dd0>, <stevedore.extension.Extension object at 0x7fa610689e50>, <stevedore.extension.Extension object at 0x7fa610667e10>, <stevedore.extension.Extension object at 0x7fa610667e50>, <stevedore.extension.Extension object at 0x7fa610667e90>, <stevedore.extension.Extension object at 0x7fa610683790>])}), 'resources': defaultdict(<function <lambda> at 0x7fa610330b90>, {'some_pollsters-cpu_l3_cache': <ceilometer.polling.manager.Resources object at 0x7fa610339110>, 'some_pollsters-disk.device.read.bytes': <ceilometer.polling.manager.Resources object at 0x7fa6103398d0>, 'some_pollsters-hardware.memory.cached': <ceilometer.polling.manager.Resources object at 0x7fa610340310>, 'some_pollsters-disk.device.write.requests': <ceilometer.polling.manager.Resources object at 0x7fa6103401d0>, 'some_pollsters-hardware.cpu.util': <ceilometer.polling.manager.Resources object at 0x7fa6103404d0>, 'some_pollsters-memory.usage': <ceilometer.polling.manager.Resources object at 0x7fa610339790>, 'some_pollsters-hardware.system_stats.io.outgoing.blocks': <ceilometer.polling.manager.Resources object at 0x7fa610340410>, 'some_pollsters-hardware.memory.buffer': <ceilometer.polling.manager.Resources object at 0x7fa610340150>, 'some_pollsters-hardware.memory.swap.total': <ceilometer.polling.manager.Resources object at 0x7fa610340490>, 'some_pollsters-network.incoming.bytes': <ceilometer.polling.manager.Resources object at 0x7fa610339c10>, 'some_pollsters-cpu': <ceilometer.polling.manager.Resources object at 0x7fa610340190>, 'some_pollsters-disk.device.write.bytes': <ceilometer.polling.manager.Resources object at 0x7fa610340290>, 'some_pollsters-hardware.network.ip.outgoing.datagrams': <ceilometer.polling.manager.Resources object at 0x7fa610340210>, 'some_pollsters-disk.device.read.requests': <ceilometer.polling.manager.Resources object at 0x7fa610339a10>, 'some_pollsters-hardware.memory.swap.avail': <ceilometer.polling.manager.Resources object at 0x7fa610340090>, 'some_pollsters-network.incoming.packets': <ceilometer.polling.manager.Resources object at 0x7fa610339d10>, 'some_pollsters-hardware.system_stats.io.incoming.blocks': <ceilometer.polling.manager.Resources object at 0x7fa610340590>, 'some_pollsters-network.outgoing.bytes': <ceilometer.polling.manager.Resources object at 0x7fa610339a50>, 'some_pollsters-hardware.network.ip.incoming.datagrams': <ceilometer.polling.manager.Resources object at 0x7fa610340350>, 'some_pollsters-hardware.memory.total': <ceilometer.polling.manager.Resources object at 0x7fa610340250>, 'some_pollsters-network.outgoing.packets': <ceilometer.polling.manager.Resources object at 0x7fa6103400d0>, 'some_pollsters-hardware.memory.used': <ceilometer.polling.manager.Resources object at 0x7fa6103403d0>}), '_batch': True}
```

#### 4.3.3.2 启动周期性任务
代码入口: ceilometer.polling.manager.AgentManager#start_polling_tasks

```
  1 def start_polling_tasks(self):
  2     data = self.setup_polling_tasks()
  3
  4     # Don't start useless threads if no task will run
  5     if not data:
  6         return
  7
  8     # One thread per polling tasks is enough
  9     self.polling_periodics = periodics.PeriodicWorker.create(
 10         [], executor_factory=lambda:
 11         futures.ThreadPoolExecutor(max_workers=len(data)))
 12
 13     for interval, polling_task in data.items():
 14
 15         @periodics.periodic(spacing=interval, run_immediately=True)
 16         def task(running_task):
 17             self.interval_task(running_task)
 18
 19         self.polling_periodics.add(task, polling_task)
 20
 21     utils.spawn_thread(self.polling_periodics.start, allow_empty=True)
```

1. 第9~11行，根据data中任务组的个数，创建N个线程池大小
2. 第13~19行，遍历data中的任务，调用将其加入到线程池中的线程调度任务中，主要在于self.interval_task,在self.interval_task 中实际调用的是task.poll_and_notify, 这里的task就是上边的 ceilometer.polling.manager.PollingTask 对象.
3. 第21行，启动线程的周期调度，最终进程框架中的多个线程就周期性的调用对应线程绑定的PollingTask 中的 poll_and_notify 方法,调度插件周期性采集数据

#### 4.3.3.3 周期性采集任务
代码入口：ceilometer.polling.manager.PollingTask#poll_and_notify

```
  1 def poll_and_notify(self):
  2     """Polling sample and notify."""
  3     cache = {}
  4     discovery_cache = {}
  5     poll_history = {}
  6     for source_name, pollsters in iter_random(
  7             self.pollster_matches.items()):
  8         for pollster in iter_random(pollsters):
  9             key = Resources.key(source_name, pollster)
 10             candidate_res = list(
 11                 self.resources[key].get(discovery_cache))
 12             if not candidate_res and pollster.obj.default_discovery:
 13                 candidate_res = self.manager.discover(
 14                     [pollster.obj.default_discovery], discovery_cache)
 15
 16             # Remove duplicated resources and black resources. Using
 17             # set() requires well defined __hash__ for each resource.
 18             # Since __eq__ is defined, 'not in' is safe here.
 19             polling_resources = []
 20             black_res = self.resources[key].blacklist
 21             history = poll_history.get(pollster.name, [])
 22             for x in candidate_res:
 23                 if x not in history:
 24                     history.append(x)
 25                     if x not in black_res:
 26                         polling_resources.append(x)
 27             poll_history[pollster.name] = history
 28
 29             # If no resources, skip for this pollster
 30             if not polling_resources:
 31                 p_context = 'new ' if history else ''
 32                 LOG.debug("Skip pollster %(name)s, no %(p_context)s"
 33                           "resources found this cycle",
 34                           {'name': pollster.name, 'p_context': p_context})
 35                 continue
 36
 37             LOG.info("Polling pollster %(poll)s in the context of "
 38                      "%(src)s",
 39                      dict(poll=pollster.name, src=source_name))
 40             try:
 41                 polling_timestamp = timeutils.utcnow().isoformat()
 42                 samples = pollster.obj.get_samples(
 43                     manager=self.manager,
 44                     cache=cache,
 45                     resources=polling_resources
 46                 )
47                 sample_batch = []
 48
 49                 for sample in samples:
 50                     # Note(yuywz): Unify the timestamp of polled samples
 51                     sample.set_timestamp(polling_timestamp)
 52                     sample_dict = (
 53                         publisher_utils.meter_message_from_counter(
 54                             sample, self._telemetry_secret
 55                         ))
 56                     if self._batch:
 57                         sample_batch.append(sample_dict)
 58                     else:
 59                         self._send_notification([sample_dict])
 60
 61                 if sample_batch:
 62                     self._send_notification(sample_batch)
 63
 64             except plugin_base.PollsterPermanentError as err:
 65                 LOG.error(
 66                     'Prevent pollster %(name)s from '
 67                     'polling %(res_list)s on source %(source)s anymore!',
 68                     dict(name=pollster.name,
 69                          res_list=str(err.fail_res_list),
 70                          source=source_name))
 71                 self.resources[key].blacklist.extend(err.fail_res_list)
 72             except Exception as err:
 73                 LOG.error(
 74                     'Continue after error from %(name)s: %(error)s'
 75                     % ({'name': pollster.name, 'error': err}),
 76                     exc_info=True)
```

1. 第6~7 行，对上文的data中的pollster_matches进行遍历，其格式如下：

```
(Pdb) p data[300].pollster_matches
defaultdict(<type 'set'>, {'some_pollsters': set([<stevedore.extension.Extension object at 0x7fa610689850>, <stevedore.extension.Extension object at 0x7fa610697090>, <stevedore.extension.Extension object at 0x7fa61066c0d0>, <stevedore.extension.Extension object at 0x7fa61066c150>, <stevedore.extension.Extension object at 0x7fa610690190>, <stevedore.extension.Extension object at 0x7fa61066c190>, <stevedore.extension.Extension object at 0x7fa610690050>, <stevedore.extension.Extension object at 0x7fa61066ca50>, <stevedore.extension.Extension object at 0x7fa610690290>, <stevedore.extension.Extension object at 0x7fa61066c2d0>, <stevedore.extension.Extension object at 0x7fa610690890>, <stevedore.extension.Extension object at 0x7fa610733bd0>, <stevedore.extension.Extension object at 0x7fa610667c10>, <stevedore.extension.Extension object at 0x7fa610667490>, <stevedore.extension.Extension object at 0x7fa6106970d0>, <stevedore.extension.Extension object at 0x7fa610689550>, <stevedore.extension.Extension object at 0x7fa610667dd0>, <stevedore.extension.Extension object at 0x7fa610689e50>, <stevedore.extension.Extension object at 0x7fa610667e10>, <stevedore.extension.Extension object at 0x7fa610667e50>, <stevedore.extension.Extension object at 0x7fa610667e90>, <stevedore.extension.Extension object at 0x7fa610683790>])})
```

可以看到，第6行的source_name 即为polling.yaml 中的每一组的sourcename，pollsters为对应组中meters所对应的extensions对象的集合
2. 第8行开始对所有的pollsters进行遍历，调度插件采集数据
3. 每一个插件采集数据的时候有1个必备元素为：
   * 该插件采集的数据所对应的对象(resources),比如虚拟机列表，该列表是通过对应插件定义的discovery方法获取
4. 第9~39行，获取该插件所对应的resources，即数据采集对象列表
5. 第42~46行，调用该插件的get_samples方法，获取对应的采集数据(sample)
6. 第49~62行，调用指定的notifier方法将数据发送出去

# 5. 总结
至此，Compute Agent整体启动过程，插件调度过程就完成了。下一篇文章再对单个插件的调度过程进行分析说明。





