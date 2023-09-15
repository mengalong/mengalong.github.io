---
layout: post
title: grep获取文件中指定key的内容
category : 系统应用
tags : [Linux, grep]
---

# grep获取文件中指定key的内容
如下文件内容中我们想获取key4的值

```html
mengalong@along-mac:~$ cat urfile
2017-04-12 some describe key1=22 key3=34 some describe key4=12 key5=xxx
2017-04-12 key1=222 key4=342 some describe key2=122
```

从上边格式可以看到，key4所在文件中的第几列是不固定的，因此不能直接使用awk，那么就可以用grep来解决了

```cython
mengalong@along-mac:~$ egrep -o "key4=\d+" urfile
key4=12
key4=342
```