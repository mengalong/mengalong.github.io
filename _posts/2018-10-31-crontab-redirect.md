---
layout: post
title: Linux crontab 输出重定向不生效问题解决
category : Linux
tags : [crontab,重定向]
---
# 问题
近期在crontab中添加了一个定时任务，该任务执行之后默认会有正常输出。为了确保在任务执行过程中的异常信息也可以捕获，方便问题定位，因此在crontab中我写了这么一条命令：
```
01 09 * * * cd /opdir/test/ && ./test.sh &>>test.log
```
以上命令非常好理解，每天9:01执行test.sh 脚本并且将脚本的标准错误输出、标准输出全部重定向到文件 test.log中。最终发现脚本是正常执行了，但是test.log 这个日志文件中却没有任何内容。

为了解决和解释这个问题，接下来我们先简单介绍下linux系统中重定向的问题

# 概念

Linux系统中:

1: 表示标准输出(stdout)，默认输出到屏幕

2:表示标准错误输出(stderr)，默认输出到屏幕

在平时我们经常使用如下方法将脚本执行结果重定向：

```
bash test.sh >test.out     //脚本的标准输出写入到文件test.out ,标准错误输出直接打印在屏幕 等价于：bash test.sh 1>test.out
bash test.sh >test.out 2>&1 //标准输出和标准错误输出都写入到test.out并且不会互相覆盖，等价于 bash test.sh &>test.out
bash test.sh >test.out 2>test.out //标准输出和标准错误输出都写入到test.out，会出现互相覆盖的问题，正常情况不推荐这样使用
bash test.sh &>test.out //等价于第二种方法
```

比较一下以上几种的效果：
 1. 第一种：错误输出在屏幕，正常输出在文件test.out

```
root@mengalong:~/opdir/mengalong/t/t# cat test.sh
#!/bin/bash
t
date

root@mengalong:~/opdir/mengalong/t/t# bash test.sh >test.out
test.sh: line 2: t: command not found
root@mengalong:~/opdir/mengalong/t/t# cat test.out
Wed Oct 31 11:07:24 CST 2018
```

2. 第二种：错误输出和正常输出均重定向到文件test.out中

```
root@mengalong:~/opdir/mengalong/t/t# bash test.sh >test.out 2>&1
root@mengalong:~/opdir/mengalong/t/t# cat test.out
test.sh: line 2: t: command not found
Wed Oct 31 11:09:02 CST 2018
```

3. 第三种：错误输出和正常输出互相覆盖
```
root@mengalong:~/opdir/mengalong/t/t# bash test.sh >test.out 2>test.out
root@mengalong:~/opdir/mengalong/t/t# cat test.out
Wed Oct 31 11:10:36 CST 2018
ot found
```
4. 第四种，特殊情况，比较一下bash test.sh 2>&1  >test.out 和 bash test.sh >test.out 2>&1 的区别:
```
root@mengalong:~/opdir/mengalong/t/t# bash test.sh 2>&1  >test.out
test.sh: line 2: t: command not found
root@mengalong:~/opdir/mengalong/t/t# cat test.out
Wed Oct 31 11:12:13 CST 2018
```
这里只是把 2>&1 放在了 >test.out 前边，但是结果却不是像我们想象的那样，错误和正常输出都进入test.out 文件。这是因为, bash test.sh 2>&1 >test.out 这个命令中， 2>&1 的时候，只是把错误输出重定向到了标准输出，而此时标准输出的默认值是屏幕，因此实际等价于标准错误输出被重定向到了屏幕，而非文件。因此重定向需要注意顺序。

# 问题解决

接下来再回过头来看看，我写的crontab任务：
```
01 09 * * * cd /opdir/test/ && ./test.sh &>>test.log
```
按照上边的概念分析，这种写法应该等价于./test.sh >test.log 2>&1 ，脚本执行的输出和标准错误输出全部重定向到 test.log。但是实际情况却是test.log文件中并没有任何内容。

这是因为 crontab 默认使用的shell环境为 /bin/sh, 而/bin/sh 并不支持 &>>test.log 这种重定向方法，因此我们看到的效果是test.log 中没有内容。
因此解决问题的方法就是将crontab的重定向方法进行修改：
```
01 09 * * * cd /opdir/test/ && ./test.sh >>test.log 2>&1
``` 

# 啰嗦一句
crontab执行过程中，如果脚本输出没有重定向，那么会默认给系统用户发邮件，邮件内容一般存储在 /var/mail/$user 中，如果不清理就会打满服务器根分区，最终导致机器无法登陆。因此推荐的crontab命令写法如下：
```
01 09 * * * cd /opdir/test/ && ./test.sh >>test.log 2>&1 </dev/null &
```
具体后边增加了 </dev/null & ，这个的含义就不多说了，感兴趣的可以自己分析一下
