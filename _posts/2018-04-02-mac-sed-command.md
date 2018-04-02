---
layout: post
title: Mac下sed -i参数的使用方法
category : linux
tags : [Linux]
---

# mac下sed -i参数: command a expects \ followed by text

mac下想要使用-i参数直接替换文件中的内容，使用的命令为：
```
sed -i 's/source-string/dest-string/g' urfile
```
执行之后，提示的错误信息为：
```
sed: 1: "a": command a expects \ followed by text
```
该命令，在redhat、ubuntu下执行则没有问题，这个问题产生的原因是Mac是基于FreeBSD,和RedHat不是一个系列。
查看mac下 sed 帮助文档，显示如下内容：
```
     -i extension
             Edit files in-place, saving backups with the specified extension.  If a zero-length extension is given, no backup will be saved.  It is not recommended to give a zero-
             length extension when in-place editing files, as you risk corruption or partial content in situations where disk space is exhausted, etc.
```

以上代表，mac下，-i 参数之后指定的字符串是用来指定备份文件的后缀名，如果不想备份文件,那么-i参数也需要指定一个空字符串：''.
因此，修改之后的命令格式为：
```
sed -i '' 's/source-string/dest-string/g' urfile
```

