---
layout: post
title: git 查看指定commit修改的文件列表
category : Linux
tags : [Git]
---

# 1. 比较两次commit修改的文件列表

```python
git diff --name-only <commit-id-1> <commit-id-2>

exp:
mengalong@along-mac:~/code/git/bter$ git diff --name-only 7fa56f9c83429bc564e6d123498b14aae5c390b1 45eadc1bb962ff4d49c7c5dbf298ddb41664dd28
ChangeLog
bter/fork_pdb.py 
```


