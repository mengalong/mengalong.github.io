---
layout: post
title: git log 显示颜色
category : Linux
tags : [Git]
---

git log命令可以输出比较好看的颜色，具体修改 ~/.gitconfig 配置即可，如下：

```python
[color]
        diff=auto
        status=auto
        branch=auto
        interactive=auto
        ui=auto
[alias]
        lg=log --decorate
```

