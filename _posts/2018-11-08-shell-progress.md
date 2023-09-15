---
layout: post
title: Linux bash 输出进度条
category : 编程
tags : [Linux,shell]
---

# 背景

如题，需求很简单，最近在工作中写个脚本在后台执行时间比较长，需要增加一个进度条以确定脚本是挂死了还是在正常运行中。

# 代码实现
```
#!/bin/bash
i=0
icon=''
arr=('|' '/' '-' '\\')
index=0
while [ $i -le 100 ]
do
	index=`echo $i%4`
	printf "[%-74s][%d%%][%c]\r" "$icon" "$i" "${arr[$index]}"
	icon='#'$icon
	((i++))
	sleep 0.1
done
echo
```


