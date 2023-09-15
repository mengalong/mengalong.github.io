---
layout: post
title: Linux 和 mac 下date时间戳转换
category : 系统应用
tags : [Linux]
---

# 获取当前时间戳
```
mac:
$ date +%s
1637149454

linux:
$ date +%s
1637149454
```

# 查看指定时间的时间戳
```
mac:
$ date -j -f "%Y-%m-%d %H:%M:%S" "2021-11-17 19:44:14" +%s
1637149454

linux:
$ date -d "2021-11-17 19:44:14" +%s
1637149454
```

# 时间戳转换为指定的格式
```
mac:
$ date -r 1637149454 +"%F %T"
2021-11-17 19:44:14

linux:
$ date -d @1637149454 +"%F %T"
2021-11-17 19:44:14
```