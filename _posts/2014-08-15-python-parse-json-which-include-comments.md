---
layout: post
title: python 解析包含注释的json文件
category : Python
tagline: "tag line"
tags : [python,json]
---


# python 解析包含注释的json文件

## 问题

Python 当前有默认的json包可以用来解析json文本，如果json文本中包含注释如何解析呢？

## 方法

cat json.file:

<!-- lang:python-->
	{        
		#this is comment        
		"a" : "1",#this is comment        
		"b" : "2", 
	}

cat parse_json.py

<!-- lang:python-->
	#!/usr/bin/env python
	import jsonfh = open("json.file", "r")
	str = json.dumps(eval(fh.read()))
	print str
	obj = json.loads(str)
	print json.dumps(obj,indent=4)
		aaa

执行结果：

<!-- lang:python-->
	python parse_json.py 
	{“a”: “1″, “b”: “2″}
	{
    	“a”: “1″, 
    	“b”: “2″
	}

enjoy it~
