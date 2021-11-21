---
layout: post
title: Mac终端输入换行问题解决
category : Linux
tags : [Linux]
---

# 问题现象

Mac终端下输入命令行过长时，首行不能自动换行，超长后会将第一行前面的内容覆盖，直到最后再换行

![](/images/posts/linux/command-over-line.png)

# 问题原因

问题原因是因为设置了终端命令行格式的 PS1 字段中有不显示的字符。这类字符在终端计算字符长度时，会出现计算错误。导致无法正确的换行。比如我的 ~/.bash_profile 中关于 PS1 的配置如下：
```
#sets up theprompt color (currently a green similar to Linux terminal)
export PS1='\033[01;32m\u@\h\033[00m:\033[01;36m\w\033[00m\$ '
```

# 问题解决：

对PS1中的不可见字符，需要用 \\[\\] 将其包裹起来，重新加载 ~/.bash_profile 配置文件即可。修改后的结果如下：

```
export PS1='\[\033[01;32m\]\u@\h\[\033[00m:\033[01;36m\]\w\[\033[00m\]\$ '
```

