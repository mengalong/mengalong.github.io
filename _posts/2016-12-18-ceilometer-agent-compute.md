---
layout: post
title: ceilometer 代码分析(一) - agent-compute
category : 生活
tags : [ceilometer,agent-compute,compute,代码分析]
---


# ceilometer-agent-compute 代码分析

说明：本文基于ceilometer mitaka版本代码分析

ceilometer是openstack项目中负责计量的组件，他会通过不同的方式从系统中收集必要的计量数据。其收集数据主要有两种方式：

1. Poller：通过在不同的节点上部署agent，按照预定义的规则轮询获取系统中的计量信息，这里Poller的方式主要依赖两个agent来获取不同的数据，其中为：
   * agent-compute：部署在每个计算节点，通过image的driver来轮询获取资源使用的统计数据
   * agent-central：部署在管理节点，通过轮询调用各个组件(nova/neutron/cinder ...)的api来获取各个组件的资源使用数据
2. Notification：对应的是ceilometer-collector，运行在一个或者多个管理节点，它会监控各个组件的消息队列，队列中的通知信息会被它处理和转换为计量信息，再发回到消息系统中。

接下来我们首先来分析agent-compute的服务启动和代码运行机制

# 代码整体结构
## 代码入口
在setup.cfg中有如下的内容：

```
ceilometer-polling = ceilometer.cmd.polling:main
```
这说明ceilometer-polling的代码入口在 ceilometer/cmd/polling.py 的 main()函数

ceilometer/cmd/polling.py

```
def main():
    service.prepare_service()
    os_service.launch(CONF, manager.AgentManager(CONF.polling_namespaces,
                                                 CONF.pollster_list)).wait()
```

说明：main函数中主要做了如下几件事

1. service.prepare_service() ：预处理，主要是初始化一些参数，设置日志级别
2. manger.AgentManager() 生成agentManager 实例
    * 对于agent-compute来说，CONF.polling_namespaces = "compute", CONF.pollster_list=[]
3. 调用os_service.launch()方法加载并启动实例
4. 之后调用wait()方法等待进程结束

agent-compute的核心就在于 AgentManager()的实例化

# 代码详解：
## service.prepare_service()

```
def prepare_service(argv=None, config_files=None):
    oslo_i18n.enable_lazy()
    log.register_options(cfg.CONF)
    log_levels = (cfg.CONF.default_log_levels +
                  ['stevedore=INFO', 'keystoneclient=INFO',
                   'neutronclient=INFO'])
    log.set_defaults(default_log_levels=log_levels)
    defaults.set_cors_middleware_defaults()

    if argv is None:
        argv = sys.argv
    cfg.CONF(argv[1:], project='ceilometer', validate_default_values=True,
             version=version.version_info.version_string(),
             default_config_files=config_files)

    keystone_client.setup_keystoneauth(cfg.CONF)

    log.setup(cfg.CONF, 'ceilometer')
    # NOTE(liusheng): guru cannot run with service under apache daemon, so when
    # ceilometer-api running with mod_wsgi, the argv is [], we don't start
    # guru.
    if argv:
        gmr.TextGuruMeditation.setup_autorun(version)
    messaging.setup()
```

主要功能：

1. 初始化预定义的参数，主要设定了host，http_timeout, workers参数
2. 设置日志级别
3. 加载指定的配置文件

## AgentManager()实例化过程

### 代码入口：

ceilometer/agent/manager.py

```
class AgentManager(service_base.BaseService):

    def __init__(self, namespaces=None, pollster_list=None):
        namespaces = namespaces or ['compute', 'central']
        pollster_list = pollster_list or []
        group_prefix = cfg.CONF.polling.partitioning_group_prefix

        # features of using coordination and pollster-list are exclusive, and
        # cannot be used at one moment to avoid both samples duplication and
        # samples being lost
        if pollster_list and cfg.CONF.coordination.backend_url:
            raise PollsterListForbidden()

        super(AgentManager, self).__init__()

```

说明：

1. namespaces = ['compute']
2. pollster_list = []
3. group_prefix = ?
4. 调用父类的init方法进行初始化，父类的 __init__ 方法中是创建了线程池，默认大小为1000,线程池赋值到了 self.tg

```
class Service(ServiceBase):
    """Service object for binaries running on hosts."""

    def __init__(self, threads=1000):
        self.tg = threadgroup.ThreadGroup(threads)
```

### 加载extension：根据ceilometer.poll.compute 获取所有的插件列表
ceilometer/agent/manager.py

```
        # we'll have default ['compute', 'central'] here if no namespaces will
        # be passed
        extensions = (self._extensions('poll', namespace).extensions
                      for namespace in namespaces)
```

```
    def _extensions(self, category, agent_ns=None):
        namespace = ('ceilometer.%s.%s' % (category, agent_ns) if agent_ns
                     else 'ceilometer.%s' % category)
        return self._get_ext_mgr(namespace)
```

说明：

1. 调用self._extensions(),传递两个参数：poll，compute
2. self._extensions()，根据传递进来的参数，组装出namespace = ceilometer.poll.compute
3. 调用 self._get_ext_mar(),传递进去 ceilometer.poll.compute这个namespace


```
    @staticmethod
    def _get_ext_mgr(namespace):
        def _catch_extension_load_error(mgr, ep, exc):
            # Extension raising ExtensionLoadError can be ignored,
            # and ignore anything we can't import as a safety measure.
            if isinstance(exc, plugin_base.ExtensionLoadError):
                LOG.exception(_("Skip loading extension for %s") % ep.name)
                return
            if isinstance(exc, ImportError):
                LOG.error(_("Failed to import extension for %(name)s: "
                            "%(error)s"),
                          {'name': ep.name, 'error': exc})
                return
            raise exc

        return extension.ExtensionManager(
            namespace=namespace,
            invoke_on_load=True,
            on_load_failure_callback=_catch_extension_load_error,
        )
```

setup.cfg

```
ceilometer.poll.compute =
    disk.read.requests = ceilometer.compute.pollsters.disk:ReadRequestsPollster
    disk.write.requests = ceilometer.compute.pollsters.disk:WriteRequestsPollster
    disk.read.bytes = ceilometer.compute.pollsters.disk:ReadBytesPollster
```
说明：

1. _get_ext_mgr(),根据传递进来的namespace，调用 extension.ExtensionManager() 并返回最终的实例
2. extension是stevedore这个公共库实现的
3. namespace: ceilometer.poll.compute 对应的定义在 setup.cfg 中，对应的是compute支持的各项计量指标的采集入口，如上。最终各项指标都是通过以上插件来实现获取的。接下来先继续分析插件的加载过程

site-packages/stevedore/extension.py

```
    def __init__(self, namespace,
                 invoke_on_load=False,
                 invoke_args=(),
                 invoke_kwds={},
                 propagate_map_exceptions=False,
                 on_load_failure_callback=None,
                 verify_requirements=False):
        self._init_attributes(
            namespace,
            propagate_map_exceptions=propagate_map_exceptions,
            on_load_failure_callback=on_load_failure_callback)
        extensions = self._load_plugins(invoke_on_load,
                                        invoke_args,
                                        invoke_kwds,
                                        verify_requirements)
        self._init_plugins(extensions)
```

说明：extension.ExtensionManager()的 __init__ 方法中做了三件事：

1. 调用 __init_attributes, 初始化了三个参数，分别赋值给如下内容,这里重点是namespace = ceilometer.poll.compute

```
		self.namespace = namespace
        self.propagate_map_exceptions = propagate_map_exceptions
        self._on_load_failure_callback = on_load_failure_callback
```

2. 调用self._load_plugins加载对应namespace下的各种插件
3. 调用 self._init_plugins 将上一步load完成的extensions赋值给 self.extensions


```
    def _load_plugins(self, invoke_on_load, invoke_args, invoke_kwds,
                      verify_requirements):
        extensions = []
        for ep in self._find_entry_points(self.namespace):
            LOG.debug('found extension %r', ep)
            try:
                ext = self._load_one_plugin(ep,
                                            invoke_on_load,
                                            invoke_args,
                                            invoke_kwds,
                                            verify_requirements,
                                            )
                if ext:
                    extensions.append(ext)
            except (KeyboardInterrupt, AssertionError):
                raise
            except Exception as err:
                if self._on_load_failure_callback is not None:
                    self._on_load_failure_callback(self, ep, err)
                else:
                    # Log the reason we couldn't import the module,
                    # usually without a traceback. The most common
                    # reason is an ImportError due to a missing
                    # dependency, and the error message should be
                    # enough to debug that.  If debug logging is
                    # enabled for our logger, provide the full
                    # traceback.
                    LOG.error('Could not load %r: %s', ep.name, err,
                              exc_info=LOG.isEnabledFor(logging.DEBUG))
        return extensions
```

```
    def _load_one_plugin(self, ep, invoke_on_load, invoke_args, invoke_kwds,
                         verify_requirements):
        # NOTE(dhellmann): Using require=False is deprecated in
        # setuptools 11.3.
        if hasattr(ep, 'resolve') and hasattr(ep, 'require'):
            if verify_requirements:
                ep.require()
            plugin = ep.resolve()
        else:
            plugin = ep.load(require=verify_requirements)
        if invoke_on_load:
            obj = plugin(*invoke_args, **invoke_kwds)
        else:
            obj = None
        return Extension(ep.name, ep, plugin, obj)
```


说明：以上两个方法是整个插件的加载的关键过程

1. for ep in self._find_entry_points(self.namespace)： 这一步实际调用的是pkg_resources，在ceilometer的egg包中的 entry_points.txt 文件中找到 ceilometer.poll.compute 的详细定义。不同的操作系统entry_points.txt 文件的具体路径会有差异，但是都是在类似的 Python2.7/site-packages/xxx.dist-info 目录中
2. 遍历通过ceilometer.poll.compute找到的插件列表，对每个插件调用self._load_one_plugin()方法
3. 比如对于第一个获取到的插件：

```
disk.read.requests = ceilometer.compute.pollsters.disk:ReadRequestsPollster
```

4. 进入 _load_one_plugin()，实际会调用 plugin = ep.load()方法加载对应插件的类，之后在实际遍历循环的过程就会调用这个插件的方法，具体在后边分析。最终返回一个 Extesion()对象，ep.name=disk.read.requests, plugin=对应的插件类实例
5. 在_load_one_plugin()返回一个extension实例之后，就会执行 extensions.append(ext)，将对应的实例append


说明：插件返回流程：

1. _load_plugins：返回 extensions = [] , extensions中是一个一个插件实例
2. ExtensionManager获取到 _load_plugins的返回值，extensions = self._load_plugins()
3. ExtensionManager调用 self._init_plugin()将 extensions赋值给 self.extensions = extensions
4. _get_ext_mgr返回的是 ExtensionManager()的结果给 _extensions 方法
5. 最终到 AgentManager的 __init__函数中，AgentManager的成员变量就得到了初始化：

```
extensions = (self._extensions('poll', namespace).extensions
                      for namespace in namespaces)
```

至此，AgentManager的 extensions 变量就得到了初始化，对应的是 ceilometer.poll.compute 下的各个插件的实例


### 加载extensions_fb:

ceilometer/agent/manager.py

```
        # get the extensions from pollster builder
        extensions_fb = (self._extensions_from_builder('poll', namespace)
                         for namespace in namespaces)
```

说明：

1. 调用 self._extensions_from_builder 传递两个参数：poll，compute

```
    def _extensions_from_builder(self, category, agent_ns=None):
        ns = ('ceilometer.builder.%s.%s' % (category, agent_ns) if agent_ns
              else 'ceilometer.builder.%s' % category)
        mgr = self._get_ext_mgr(ns)

        def _build(ext):
            return ext.plugin.get_pollsters_extensions()

        # NOTE: this seems a stevedore bug. if no extensions are found,
        # map will raise runtimeError which is not documented.
        if mgr.names():
            return list(itertools.chain(*mgr.map(_build)))
        else:
            return []
```

说明：

1. ns赋值，这里对应的是 ceilometer.builder.poll.compute
2. 调用 self._get_ext_mgr(ns), 这一步和上边加载extensions一样，根据namespace加载对应的extensions，如果有的话，最终返回的就是 ExtensionsManager实例
3. 对于compute来说，这里对应的namespace在setup.cfg中没有定义，那么 extensions_fb = [] ，是一个空列表，我们就先不深入了

### extensions赋值

```
self.extensions = list(itertools.chain(*list(extensions))) + list(
            itertools.chain(*list(extensions_fb)))
```

说明：

1. 以上加载完毕的extensions和extensions_fb都会赋值给 self.extensions


### 加载discovery_manager

ceilometer/agent/manager.py


```
self.discovery_manager = self._extensions('discover')
```

```
    def _extensions(self, category, agent_ns=None):
        namespace = ('ceilometer.%s.%s' % (category, agent_ns) if agent_ns
                     else 'ceilometer.%s' % category)
        return self._get_ext_mgr(namespace)
```

```
ceilometer.discover =
    local_instances = ceilometer.compute.discovery:InstanceDiscovery
    endpoint = ceilometer.agent.discovery.endpoint:EndpointDiscovery
    tenant = ceilometer.agent.discovery.tenant:TenantDiscovery
```

说明：

1. 调用self._extensions("discover"),传递的参数只有一个：discover
2. _extensions()组装的namespace=ceilometer.discover
3. 之后调用 self._get_ext_mgr()加载ceilometer.discover这个namespace对应的所有插件，具体的过程不再赘述
4. 最终self.discovery_manager 得到的就是 ceilometer.discover这个namespace下所有插件的实例


### notification初始化-transport获取
ceilometer/agent/manager.py

```
        self.notifier = oslo_messaging.Notifier(
            messaging.get_transport(),
            driver=cfg.CONF.publisher_notifier.telemetry_driver,
            publisher_id="ceilometer.polling")
```

说明：

1. self.notifier 是oslo_messaging.Notifier对应的一个实例，这里初始化的时候传递了三个参数：
    * 第一个参数：messaging.get_transport()
    * 第二个参数：driver = messagingv2
    * 第三个参数：publisher_id = "ceilometer.polling"
2. 接下来主要来继续分析一下 get_transport()


ceilometer/messaging.py

```
def get_transport(url=None, optional=False, cache=True):
    """Initialise the oslo_messaging layer."""
    global TRANSPORTS, DEFAULT_URL
    cache_key = url or DEFAULT_URL
    transport = TRANSPORTS.get(cache_key)
    if not transport or not cache:
        try:
            transport = oslo_messaging.get_transport(cfg.CONF, url)
        except (oslo_messaging.InvalidTransportURL,
                oslo_messaging.DriverLoadFailure):
            if not optional or url:
                # NOTE(sileht): oslo_messaging is configured but unloadable
                # so reraise the exception
                raise
            return None
        else:
            if cache:
                TRANSPORTS[cache_key] = transport
    return transport
```

说明：

1. transport = None，cache = None, url=None
2. transport = oslo_messaging.get_transport(cfg.CONF,url) 调用加载transport


site-packages/oslo_messaging/transport.py

```
def get_transport(conf, url=None, allowed_remote_exmods=None, aliases=None):
    allowed_remote_exmods = allowed_remote_exmods or []
    conf.register_opts(_transport_opts)

    if not isinstance(url, TransportURL):
        url = TransportURL.parse(conf, url, aliases)

    kwargs = dict(default_exchange=conf.control_exchange,
                  allowed_remote_exmods=allowed_remote_exmods)

    try:
        mgr = driver.DriverManager('oslo.messaging.drivers',
                                   url.transport.split('+')[0],
                                   invoke_on_load=True,
                                   invoke_args=[conf, url],
                                   invoke_kwds=kwargs)
    except RuntimeError as ex:
        raise DriverLoadFailure(url.transport, ex)

    return Transport(mgr.driver)
```

```
[oslo.messaging.drivers]
amqp = oslo_messaging._drivers.impl_amqp1:ProtonDriver
fake = oslo_messaging._drivers.impl_fake:FakeDriver
kafka = oslo_messaging._drivers.impl_kafka:KafkaDriver
kombu = oslo_messaging._drivers.impl_rabbit:RabbitDriver
pika = oslo_messaging._drivers.impl_pika:PikaDriver
rabbit = oslo_messaging._drivers.impl_rabbit:RabbitDriver
zmq = oslo_messaging._drivers.impl_zmq:ZmqDriver
```

说明：

1. url = rabbit
2. mgr = driver.DriverManager() 主要是两个参数，第一个参数是一个namespace: oslo.messaging.drivers,第二个参数是 rabbit
3. mgr最终是DriverManager的一个实例,其中oslo.messaging.drivers


site-packages/stevedore/driver.py

```
    def __init__(self, namespace, name,
                 invoke_on_load=False, invoke_args=(), invoke_kwds={},
                 on_load_failure_callback=None,
                 verify_requirements=False):
        on_load_failure_callback = on_load_failure_callback \
            or self._default_on_load_failure
        super(DriverManager, self).__init__(
            namespace=namespace,
            names=[name],
            invoke_on_load=invoke_on_load,
            invoke_args=invoke_args,
            invoke_kwds=invoke_kwds,
            on_load_failure_callback=on_load_failure_callback,
            verify_requirements=verify_requirements,
        )
```

说明：
1. 生成 DriverManager实例，首先调用父类的 __init__方法，其中第一个参数为 oslo.messaging.drivers,第二个参数为 rabbit
2. 代表加载的是 rabbit的driver


site-packages/stevedore/named.py

```
    def __init__(self, namespace, names,
                 invoke_on_load=False, invoke_args=(), invoke_kwds={},
                 name_order=False, propagate_map_exceptions=False,
                 on_load_failure_callback=None,
                 on_missing_entrypoints_callback=None,
                 verify_requirements=False,
                 warn_on_missing_entrypoint=True):
        self._init_attributes(
            namespace, names, name_order=name_order,
            propagate_map_exceptions=propagate_map_exceptions,
            on_load_failure_callback=on_load_failure_callback)
        extensions = self._load_plugins(invoke_on_load,
                                        invoke_args,
                                        invoke_kwds,
                                        verify_requirements)
        self._missing_names = set(names) - set([e.name for e in extensions])
        if self._missing_names:
            if on_missing_entrypoints_callback:
                on_missing_entrypoints_callback(self._missing_names)
            elif warn_on_missing_entrypoint:
                LOG.warning('Could not load %s' %
                            ', '.join(self._missing_names))
        self._init_plugins(extensions)
```

```
    def _init_attributes(self, namespace, names, name_order=False,
                         propagate_map_exceptions=False,
                         on_load_failure_callback=None):
        super(NamedExtensionManager, self)._init_attributes(
            namespace, propagate_map_exceptions=propagate_map_exceptions,
            on_load_failure_callback=on_load_failure_callback)

        self._names = names
        self._missing_names = set()
        self._name_order = name_order
```

```
   def _init_attributes(self, namespace, propagate_map_exceptions=False,
                         on_load_failure_callback=None):
        self.namespace = namespace
        self.propagate_map_exceptions = propagate_map_exceptions
        self._on_load_failure_callback = on_load_failure_callback
```

说明：
1. 首先执行 self.__init_attrivutes, 初始化几个重要参数，
2.  self.names = rabbit
3.  self.namespace = oslo.messaging.drivers
4.  调用 self._load_plugins() 加载 oslo.messaging.drivers 对应的 rabbit的driver：rabbit = oslo_messaging._drivers.impl_rabbit:RabbitDriver,加载到extensions这个字段中
5.  调用self._init_plugin将extensions初始化赋值给 self.extensions


说明：返回流程：
1. NamedExtensionManager 向上返回一个 NamedExtensionManager()实例，其中 self.extensions 对应的是rabbit 的Driver

2. get_transport()函数中：mgr 对应的就是上边加载到的rabbit的Driver实例,向上返回一个 TransPort对象

```
mgr = driver.DriverManager('oslo.messaging.drivers',
                                   url.transport.split('+')[0],
                                   invoke_on_load=True,
                                   invoke_args=[conf, url],
                                   invoke_kwds=kwargs)
return Transport(mgr.driver)
```

3. 在Transport对象的实例化过程，self._driver = driver 对应的就是上边加载到的rabbit 的driver

```
    def __init__(self, driver):
        self.conf = driver.conf
        self._driver = driver
```

4. 在ceilometer/messaging.py的 get_transport中 transport = oslo_messaging.get_transport(cfg.CONF, url)， transport对应的就是上文加载到的rabbit的driver
5. 至此，Notifier初始化过程中的第一个参数就生成了

接下来继续分析Notifier的初始化过程

### Notifer初始化

ceilometer/agent/manager.py

```
        self.notifier = oslo_messaging.Notifier(
            messaging.get_transport(),
            driver=cfg.CONF.publisher_notifier.telemetry_driver,
            publisher_id="ceilometer.polling")
```

说明：
1. 参数一：对应的是rabbit的driver
2. 参数二：对应的messagingv2


site_packages/oslo_messaging/notify/notifier.py

```
    def __init__(self, transport, publisher_id=None,
                 driver=None, topic=None,
                 serializer=None, retry=None,
                 topics=None):
        conf = transport.conf
        conf.register_opts(_notifier_opts,
                           group='oslo_messaging_notifications')

        self.transport = transport
        self.publisher_id = publisher_id
        self.retry = retry

        self._driver_names = ([driver] if driver is not None else
                              conf.oslo_messaging_notifications.driver)

        if topics is not None:
            self._topics = topics
        elif topic is not None:
            self._topics = [topic]
        else:
            self._topics = conf.oslo_messaging_notifications.topics
        self._serializer = serializer or msg_serializer.NoOpSerializer()

        self._driver_mgr = named.NamedExtensionManager(
            'oslo.messaging.notify.drivers',
            names=self._driver_names,
            invoke_on_load=True,
            invoke_args=[conf],
            invoke_kwds={
                'topics': self._topics,
                'transport': self.transport,
            }
        )

    _marker = object()
```

说明：

1. 创建了一个Notify对象，其中transport是rabbit，publisher_id=ceilometer.polling, self._driver_names=messagingv2
2. self._driver_mgr 是根据 rabbit 从 oslo.messaging.notify.drivers中加载的rabbit的driver，对应的是：

```
[oslo.messaging.notify.drivers]
log = oslo_messaging.notify._impl_log:LogDriver
messaging = oslo_messaging.notify.messaging:MessagingDriver
messagingv2 = oslo_messaging.notify.messaging:MessagingV2Driver
noop = oslo_messaging.notify._impl_noop:NoOpDriver
routing = oslo_messaging.notify._impl_routing:RoutingDriver
test = oslo_messaging.notify._impl_test:TestDriver
```


### AgentManager初始化总结

至此，AgentManager初始化的过程就结束了，整个过程中我们可以看到主要做了如下几件事：

1. self.extensions赋值： 加载ceilometer.poll.compute 这个namespace下对应的插件，实现指标的具体采集
2. self.discovery_manager赋值，加载ceilometer.discover对应的所有插件
3. self.notifier 赋值：
    * 加载了oslo_messaging下的rabbit driver赋值给 self.transport (对应的是self.notifier.transport)
    * 加载oslo_messaging下的 messagingv2 driver，赋值给 self._driver_mgr(对应的是self.notifer._driver_mgr)
    * 赋值，self.publisher_id = ceilometer.polling (对应的是self.notifier.publisher_id)
    

## 服务启动过程

ceilometer/cmd/polling.py

```
    os_service.launch(CONF, manager.AgentManager(CONF.polling_namespaces,
                                                 CONF.pollster_list)).wait()
```

说明：

1. 服务启动，调用的是os_service对应的是公共库 oslo_service
2. launch 用来启动AgentManager实例

接下来就继续来分析launch的过程


### 服务启动的入口：

site-packages/oslo_service/service.py

```
def launch(conf, service, workers=1, restart_method='reload'):

    if workers is not None and workers <= 0:
        raise ValueError(_("Number of workers should be positive!"))

    if workers is None or workers == 1:
        launcher = ServiceLauncher(conf, restart_method=restart_method)
    else:
        launcher = ProcessLauncher(conf, restart_method=restart_method)
    launcher.launch_service(service, workers=workers)

    return launcher
```  

说明：

1. 根据workers的取值，选择不同的启动方式
2. agent-compute的workers=1，对应的是通过launcher =  ServiceLauncher()实例化
3. launcher.launch_service 启动服务


### launcher实例化过程

site-packages/oslo_service/service.py

```
class ServiceLauncher(Launcher):
    """Runs one or more service in a parent process."""
    def __init__(self, conf, restart_method='reload'):
        """Constructor.

        :param conf: an instance of ConfigOpts
        :param restart_method: passed to super
        """
        super(ServiceLauncher, self).__init__(
            conf, restart_method=restart_method)
        self.signal_handler = SignalHandler()
```

说明：

1. 调用父类的 __init__ 方法
2. 初始化 self.signal_handler ,给信号捕获初始化handle


```
class Launcher(object):

    def __init__(self, conf, restart_method='reload'):
        self.conf = conf
        conf.register_opts(_options.service_opts)
        self.services = Services()
        self.backdoor_port = (
            eventlet_backdoor.initialize_if_enabled(self.conf))
        self.restart_method = restart_method
        if restart_method not in _LAUNCHER_RESTART_METHODS:
            raise ValueError(_("Invalid restart_method: %s") % restart_method)
```

说明：

1. 配置初始化
2. 实例化 Services(),并赋值给 self.services

```
class Services(object):

    def __init__(self):
        self.services = []
        self.tg = threadgroup.ThreadGroup()
        self.done = event.Event()
```

说明：

1. 实例化self.tg 是一个进程池的对象
2. 其中self.tg 的具体实例化过程中，创建了一个默认大小为10的进程池并赋值给 self.pool（对应的是self.tg.pool）

至此launcher实例化成功，接下来看launcher启动过程,重点关注：

1. self.services 是一个大小为 10 的进程池


### launcher启动过程

site-packages/oslo_service/service.py
```
    launcher.launch_service(service, workers=workers)
```

说明：

1. 调用 launch_service 方法进行服务启动，对应传递进来的service就是AgentManager实例


```
    def launch_service(self, service, workers=1):
        if workers is not None and workers != 1:
            raise ValueError(_("Launcher asked to start multiple workers"))
        _check_service_base(service)
        service.backdoor_port = self.backdoor_port
        self.services.add(service)
```

说明：这里的关键点是 self.services.add(service)
1. 通过add方法，将这个AgentManager实例传递给 services (这个是之前创建的进程池)


```
class Services(object):
    def add(self, service):
        self.services.append(service)
        self.tg.add_thread(self.run_service, service, self.done)
```

说明：

1. self.services.append(service), 只是做一个记录，知道启动了什么进程
2. self.tg.add_thread() 将service传递给 add_thread 方法
3. add_thread方法会调用 self.run_service 这个函数


```
    @staticmethod
    def run_service(service, done):
        try:
            service.start()
        except Exception:
            LOG.exception(_LE('Error starting thread.'))
            raise SystemExit(1)
        else:
            done.wait()
```

说明：                             

1. 调用service.start()方法来启动服务，对应的就是调用AgentManager对应的start方法
2. 执行done.wait()，等待进程退出

至此，进程启动的过程就结束了


### 服务启动过程总结：

这个过程的关键点：

1. launcher实例化：通过调用oslo_service的方法来进行launcher实例化，实例化之后主要关注几个关键变量：
    * launcher.services 是一个大小为 10 的进程池
    * launcher.done是一个 event.Event()的实例，负责进程退出的相关事宜
2. launcher服务启动：通过调用 launcher.launcher_service()，根据配置的worker大小来启动AgentManager实例

AgentManager实例的启动调用的是 AgentManager的 start()方法，接下来我们来继续分析 AgentManager的实例启动过程。


## AgentManager 启动过程

ceilometer/agent/manager.py

```
    def start(self):
        self.polling_manager = pipeline.setup_polling()

        self.partition_coordinator.start()
        self.join_partitioning_groups()

        self.pollster_timers = self.configure_polling_tasks()

        self.init_pipeline_refresh()
```  

说明：

1. 根据pipeline.yaml 初始化polling_manager
2. ? partition_coordinator.start()
3. ? join_partitioning_groups()
4. 设置并启动polling_task


### pipeline初始化

ceilometer/pipeline.py

```
def setup_polling():
    """Setup polling manager according to yaml config file."""
    cfg_file = cfg.CONF.pipeline_cfg_file
    return _setup_polling_manager(cfg_file)
```

说明：

1. cfg_file = pipeline.yaml
2. 调用_setup_polling_manager(cfg_file), 生成pipeline


```
def _setup_polling_manager(cfg_file):
    if not os.path.exists(cfg_file):
        cfg_file = cfg.CONF.find_file(cfg_file)

    LOG.debug("Polling config file: %s", cfg_file)

    with open(cfg_file) as fap:
        data = fap.read()

    pipeline_cfg = yaml.safe_load(data)
    LOG.info(_LI("Pipeline config: %s"), pipeline_cfg)

    return PollingManager(pipeline_cfg)
```
说明：

1. 打开pipeline.yaml 并通过yaml加载到pipeline_cfg
2. 通过PollingManager生成pipeline对象



```

class PollingManager(object):
    def __init__(self, cfg):
        self.sources = []
        if not ('sources' in cfg and 'sinks' in cfg):
            raise PipelineException("Both sources & sinks are required",
                                    cfg)
        LOG.info(_LI('detected decoupled pipeline config format'))

        unique_names = set()
        for s in cfg.get('sources', []):
            name = s.get('name')
            if name in unique_names:
                raise PipelineException("Duplicated source names: %s" %
                                        name, self)
            else:
                unique_names.add(name)
                self.sources.append(SampleSource(s))
        unique_names.clear()
```
说明：

1.  初始化所有的sources，每一个sources就是一个SampleSource对象
 
 
```
class SampleSource(Source):
    def __init__(self, cfg):
        super(SampleSource, self).__init__(cfg)
        # Support 'counters' for backward compatibility
        self.meters = cfg.get('meters', cfg.get('counters'))
        try:
            self.interval = int(cfg.get('interval', 600))
        except ValueError:
            raise PipelineException("Invalid interval value", cfg)
        if self.interval <= 0:
            raise PipelineException("Interval value should > 0", cfg)

        self.resources = cfg.get('resources') or []
        if not isinstance(self.resources, list):
            raise PipelineException("Resources should be a list", cfg)

        self.discovery = cfg.get('discovery') or []
        if not isinstance(self.discovery, list):
            raise PipelineException("Discovery should be a list", cfg)
        self.check_source_filtering(self.meters, 'meters')
```

说明：

1. super的 __init__ 函数中，做了几个变量初始化：
    * self.cfg = cfg
    * self.name = cfg['name']
    * self.sinks = cfg.get('sinks')
    
    如下，一个pipeline.yaml的实例:
    
    * sources中定义了要采集的各项指标(meters), 采集周期(interval)，采集的后置处理sinks
    * sinks：定义了一类采集项采集到数据知乎的发布方式以及对应到系统中的meter_name, unit, type, scale
    
```
---
sources:
    - name: cpu_source
      interval: 600
      meters:
          - "cpu"
      sinks:
          - cpu_sink
          - cpu_delta_sink
sinks:
    - name: cpu_sink
      transformers:
          - name: "rate_of_change"
            parameters:
                target:
                    name: "cpu_util"
                    unit: "%"
                    type: "gauge"
                    scale: "100.0 / (10**9 * (resource_metadata.cpu_number or 1))"
      publishers:
          - notifier://
```

2. 初始化meters对象
3. 初始化interval，默认给赋值600
4. self.resources = [], self.discovery = []

总结：

每一个SampleSources对象主要包含如下几部分内容：

1. name： cpu_source
2. interval: 600
3. meters: ['cpu']
4. sinks: cpu_sink, cpu_delta_sink


最终 self.polling_manager 就是PollingManager()的一个实例，这个实例中，有一个关键变量，self.sources，而self.sources 是一个[] 集合，每个元素都是一个 SampleSources()对象

### ? partition_coordinator.start()

TODO：暂时还没搞清楚

### ? join_partitioning_groups()

TODO: 暂时还没搞清楚

### 设置并启动polling_task

ceilometer/agent/manager.py

```
self.pollster_timers = self.configure_polling_tasks()
```

说明： 

1. 设置polling_tasks 

### polling_task 的获取，并赋值给data变量

#### 详细过程

```
    def configure_polling_tasks(self):
        # allow time for coordination if necessary
        delay_start = self.partition_coordinator.is_active()

        # set shuffle time before polling task if necessary
        delay_polling_time = random.randint(
            0, cfg.CONF.shuffle_time_before_polling_task)

        pollster_timers = []
        data = self.setup_polling_tasks()

```
说明：

1. 前几步暂时忽略，从 data = self.setup_polling_tasks()开始
2. data = self.setup_polling_tasks()


```
    def setup_polling_tasks(self):
        polling_tasks = {}
        for source in self.polling_manager.sources:
            polling_task = None
            for pollster in self.extensions:
                if source.support_meter(pollster.name):
                    polling_task = polling_tasks.get(source.get_interval())
                    if not polling_task:
                        polling_task = self.create_polling_task()
                        polling_tasks[source.get_interval()] = polling_task
                    polling_task.add(pollster, source)
        return polling_tasks
```

说明：
1. 遍历polling_manager.sources
2. 对每一个sources而言，需要把所有的extensions遍历一遍(setup.cfg 中定义的所有采集项)，判断如果对应的extension的name在当前的source中，那么就创建一个polling_task
3. 如果这个polling_task 不在 polling_tasks中，则polling_tasks[interval] = polling_task
4. polling_task中add进入当前这个source和pollster，这样就给在pipeline.yaml中定义的一个meter找到了他所对应的pollster(extenstion)
5. 最终返回polling_tasks
6. 接下来继续看polling_task的创建过程


```

    def create_polling_task(self):
        """Create an initially empty polling task."""
        return PollingTask(self)
``` 

说明：

1. 每一个polling_task 都是 PollingTask的一个实例

```
class PollingTask(object):

    def __init__(self, agent_manager):
        self.manager = agent_manager

        # elements of the Cartesian product of sources X pollsters
        # with a common interval
        self.pollster_matches = collections.defaultdict(set)

        # we relate the static resources and per-source discovery to
        # each combination of pollster and matching source
        resource_factory = lambda: Resources(agent_manager)
        self.resources = collections.defaultdict(resource_factory)

        self._batch = cfg.CONF.batch_polled_samples
        self._telemetry_secret = cfg.CONF.publisher.telemetry_secret
```

说明：

1. PollingTask的类定义
2. 这里有个疑问，self.manager = agent_manager, 这个agent_manager 从哪里来？


#### 总结

1. data = self.setup_polling_tasks() 最终获取到的是PollingTask 对象
2. data的组织形式是一个hash，poling_tasks, 对应的是：

```
{
    interval1:[polling_task1, polling_task2],
    interval2:[polling_task3, polling_task4]
}
```


### polling_tasks 加入定时器

```
       for interval, polling_task in data.items():
            delay_time = (interval + delay_polling_time if delay_start
                          else delay_polling_time)
            pollster_timers.append(self.tg.add_timer(interval,
                                   self.interval_task,
                                   initial_delay=delay_time,
                                   task=polling_task))
        self.tg.add_timer(cfg.CONF.coordination.heartbeat,
                          self.partition_coordinator.heartbeat)

        return pollster_timers
```

说明：

1. 遍历polling_tasks，按照不同的interval，将对应的polling_task加入不同的定时器队列


```
					self.tg.add_timer(interval,
        							self.interval_task,
                                   initial_delay=delay_time,
                                   task=polling_task)
```

说明：

1. 调用self.interval_task, 并传入interval变量和polling_task

```
    def interval_task(self, task):
        self._keystone = None
        self._keystone_last_exception = None

        task.poll_and_notify()
```


```
    def poll_and_notify(self):
        """Polling sample and notify."""
        cache = {}
        discovery_cache = {}
        poll_history = {}
        for source_name in self.pollster_matches:
            for pollster in self.pollster_matches[source_name]:
                key = Resources.key(source_name, pollster)
                candidate_res = list(
                    self.resources[key].get(discovery_cache))
                if not candidate_res and pollster.obj.default_discovery:
                    candidate_res = self.manager.discover(
                        [pollster.obj.default_discovery], discovery_cache)

                # Remove duplicated resources and black resources. Using
                # set() requires well defined __hash__ for each resource.
                # Since __eq__ is defined, 'not in' is safe here.
                polling_resources = []
                black_res = self.resources[key].blacklist
                history = poll_history.get(pollster.name, [])
                for x in candidate_res:
                    if x not in history:
                        history.append(x)
                        if x not in black_res:
                            polling_resources.append(x)
                poll_history[pollster.name] = history

                # If no resources, skip for this pollster
                if not polling_resources:
                    p_context = 'new ' if history else ''
                    LOG.info(_LI("Skip pollster %(name)s, no %(p_context)s"
                                 "resources found this cycle"),
                             {'name': pollster.name, 'p_context': p_context})
                    continue

                LOG.info(_LI("Polling pollster %(poll)s in the context of "
                             "%(src)s"),
                         dict(poll=pollster.name, src=source_name))
                try:
                    samples = pollster.obj.get_samples(
                        manager=self.manager,
                        cache=cache,
                        resources=polling_resources
                    )
                    sample_batch = []

                    for sample in samples:
                        sample_dict = (
                            publisher_utils.meter_message_from_counter(
                                sample, self._telemetry_secret
                            ))
                        if self._batch:
                            sample_batch.append(sample_dict)
                        else:
                            self._send_notification([sample_dict])

                    if sample_batch:
                        self._send_notification(sample_batch)

                except plugin_base.PollsterPermanentError as err:
                    LOG.error(_(
                        'Prevent pollster %(name)s for '
                        'polling source %(source)s anymore!')
                        % ({'name': pollster.name, 'source': source_name}))
                    self.resources[key].blacklist.extend(err.fail_res_list)
                except Exception as err:
                    LOG.warning(_(
                        'Continue after error from %(name)s: %(error)s')
                        % ({'name': pollster.name, 'error': err}),
                        exc_info=True)

```

说明：

1. interval_task 周期性的任务
2. 调用对应的pollster对应的 get_sample()方法获取对应的samples数据
3. 对每一个sample数据调用ceilometer/publisher/util.py中的 meter_message_from_counter()方法，组装一个msg
4. 如果配置文件中定义了批量发送sample数据，则将这个sample数据append到sample_batch,否则直接调用self._send_notification()进行数据发送
5. 在 self._send_notification()中，调用self.manager.notifier.sample()将数据发送出去, 这里的manager是AgentManager的一个实例，notifier是上文通过transport、driver创建的Notification对象，sample函数是用来发送数据的


以上的定时任务，最终通过 self.tg.add_timer， 加入了定时任务的队列中，定期执行


### self.init_pipeline_refresh()

定期刷新pipeline

TODO： 待后续详细分析

