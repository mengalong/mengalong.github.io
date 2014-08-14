---
layout: post
category : Python
tagline: "tag line"
tags : [python,json]
---
{% include JB/setup %}

#python 解析包含注释的json文件

## 问题

Python 当前有默认的json包可以用来解析json文本，如果json文本中包含注释如何解析呢？

## 方法

cat json.file:

`
{        
	#this is comment        
	"a" : "1",#this is comment        
	"b" : "2", 
}
`
