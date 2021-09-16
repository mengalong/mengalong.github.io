---
layout: post
title: Linux/Mac/Windows 文件换行符转换
category : Linux
tags : [Linux]
---
Linux/Mac/Windows 文件换行符转换

文件换行符在不同操作系统下的格式分别是不一样的，具体如下：

* windows下：换行符是 \r\n
* mac下：换行符是 \r
* linux下：换行符是 \n

由于这种差异的存在，经常会导致我们在windows或者mac下写的文件拿到linux系统下会出现解析异常等问题。比如在linux下，用如下命令可以看到windows/mac下格式
的文件显示内容如下：
```commandline
$ cat -A test.txt
test^M$
test^M$

也可以用 vim -b test.txt查看文件中隐藏的换行符，可以看到存在 ^M 这种字符
```

如何将这类文件修改为linux格式的换行符呢？用以下命令即可：
* dos2unix test.txt
* mac2unix test.txt

