---
layout: post
title: FreeSwitch event_socket connection refused
category : C
tags : [FreeSwitch]
---
FreeSwitch event_socket connection refused

使用esl自带的testserver.c测试FreeSwitch以外连方式连接自定义server，当通过拨号触发连接时提示：
```buildoutcfg
2021-07-14 14:34:38.647554 [NOTICE] mod_event_socket.c:452 Trying host: localhost:8040
2021-07-14 14:34:38.657412 [ERR] mod_event_socket.c:486 Socket Error: Connection refused
2021-07-14 14:34:38.657412 [ERR] mod_event_socket.c:490 Socket Error!
```

查找原因，最终是因为dialplan配置中使用的localhost:8040，这里要求需要指定实际IP
```xml
    <extension name="socket testing">
      <condition field="destination_number" expression="^1238$">
        <action application="socket" data="localhost:8040 full"/>
      </condition>
    </extension>
```
解决：修改 localhost:8040 为 127.0.0.1:8040 即可