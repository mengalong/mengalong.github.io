---
layout: post
title: mongodb中如何插入简体中文
category : 编程
tagline: "tag line"
tags : [python,json]
---



# mongodb insert simplified Chinese

## 问题

1. 系统环境是GBK编码
2. mongodb 默认(而且不能改动)用的是UTF8编码
3. 用python向数据库中插入一条包含中文的数据
4. 使用的python版本是 2.7.5

## 解决

背景知识：

1. python内部是以unicode记录字符串的，因此生产环境中如果需要做编码转换，则需要借助于unicode作为中间值，进行转换
2. python 程序第一或者第二行的 #coding=gbk 用来指明程序文件的编码格式，也是用来告诉python解释器应该用什么编码格式解析当前程序文件

编码转换举例,当前系统环境为gbk：

<!-- lang:python -->

	#!/usr/bin/env python
	#coding=gbk
	ss="中文"       #gbk 编码定义的字符串
	print "ss type:",type(ss) #类型为字符串
	print "ss value:",ss    #内容为中文字符
	print 
	ss_uni = ss.decode("gbk")  #将gbk类型的字符串转换成unicode
	print "ss_uni type:",type(ss_uni) #类型为 <type 'unicode'>
	#print "ss_uni value:",ss_uni #这里不能直接print，可以在终端下查看,因为ss_uni 本身是一个unicode的instance
	printss_utf8 = ss_uni.encode("utf8") #将unicode对象转换成utf8类型的字符串
	print "ss_utf8 type:", type(ss_utf8) #类型为字符串
	print "ss_utf8 value:", ss_utf8 #打印出来为utf8的实际编码
	
	
执行结果为：

<!-- lang:python -->
	ss type: <type ‘str’>
	ss value: 中文
	ss_uni type: <type ‘unicode’>
	ss_utf8 type: <type ‘str’>
	ss_utf8 value: 涓##

接下来看看如何讲一个包含中文的json串插入mongodb

<!-- lang:python -->
	#!/usr/bin/env python
	#coding=gbk
	import json
	import pymongo
	
	db_conn = pymongo.Connection("localhost", 1105);
	database = db_conn.test_database
	coll = database.test_collection
	coll.remove()
	
	data='{"_id":"123","text":"中文"}'   #定义gbk编码的字符串
	data_uni = data.decode("gbk")   #将gbk的字符串转换成unicode
	data_utf = data_uni.encode("utf8") #将unicode对象转换成utf8字符串
	obj = eval(data_utf) #将字符串转换成dict
	coll.insert(obj)        #执行插入
	s = coll.find_one({"_id":"123"})  #从数据库中读取出来,读出来的时dict对象
	obj = json.dumps(s,encoding="gbk",ensure_ascii=False) #将dict对象转换成unicode
	print obj.encode("gbk") #将unicode转换成gbk字符串输出
	

执行结果为：{“text”: “中文”, “_id”: “123″}

另外一种操作json串插入和修改的例子：

<!-- lang:python -->
	#!/usr/bin/env python
	#coding=gbk
	import json
	import pymongo
	
	db_conn = pymongo.Connection("localhost", 1105);
	database = db_conn.test_database
	coll = database.test_collection
	
	coll.remove()data='{"_id":"123","text":"中文"}'   #定义gbk编码的字符串
	ss = json.dumps(data,encoding="gbk",ensure_ascii=False) #将字符串转换成unicode
	obj = json.loads(ss)    #jsonload unicode编码的字符串
	
	x = eval(obj.encode('utf8')) #将unicode的字符串转换成utf8，然后再将字符串转换成dict对象
	
	coll.insert(x)  #执行插入
	i = coll.find_one({"_id":"123"})
	ss = json.dumps(i,encoding="gbk",ensure_ascii=False)
	print ss.encode("gbk")
	data = '{"a":"小明"}'
	ss = json.dumps(data,encoding="gbk",ensure_ascii=False)
	obj = json.loads(ss)
	
	coll.update({"_id":"123"},{"$set":{"text":obj}})
	i = coll.find_one({"_id":"123"})ss = json.dumps(i,encoding="gbk",ensure_ascii=False)
	print ss.encode("gbk")
	data = '鲜花'
	ss = json.dumps(data,encoding="gbk",ensure_ascii=False)
	obj = json.loads(ss)
	coll.update({"_id":"123"},{"$set":{"text":obj}})
	i = coll.find_one({"_id":"123"})
	ss = json.dumps(i,encoding="gbk",ensure_ascii=False)
	print ss.encode("gbk")
	
	
执行结果为：

<!-- lang:python -->
	{“text”: “中文”, “_id”: “123″}
	{“text”: “{\”a\”:\”小明\”}”, “_id”: “123″}
	{“text”: “鲜花”, “_id”: “123″} 

