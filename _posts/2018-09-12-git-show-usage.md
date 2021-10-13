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

# 2. 获取历史所有的修改记录
```python
git log
```

# 3. 单行显示历史所有的commit
```python
git log --pretty=oneline

exp:
mengalong@along-mac:~/code/git/bter$ git log --pretty=oneline
ea8a0f7eb5af01c6daf718029ede465b0911dda5 (HEAD -> master) modify the publish policy
7fa56f9c83429bc564e6d123498b14aae5c390b1 (origin/master) add fork_pdb for debug the project
45eadc1bb962ff4d49c7c5dbf298ddb41664dd28 modify the insert function in impl_mysql
637a829011a3d3b7bfdf1ce24299f631a8e7741a update Changelog info
5c78ccbab445e4ba0575133d8ee68e69df72a786 (tag: 0.0.2) update README.md for database
0eab7e5214d5302d8ee6fe01b3e28837fb57c781 update the Changelog
390b1555af4373a2abe0ee243230b21cdd7ff655 add dist and bter.egg.* into .gitignore
f1ffcd3ef07ec8cfe00a03f286ef3d6c8486bc31 add mysql-sync for table create
```

# 4. 查看指定文件所有的历史修改
```
git log <file-path>
```

# 5. 单行显示指定文件历史所有commit
```
git log --pretty=oneline <file-path>

exp:
mengalong@along-mac:~/code/git/bter/bter$ git log --pretty=oneline utils.py
8160a4919e75420d32234ec136fe77e5f5281888 implement period task
6a0e823b446a2e88adeec37bae11b45928e80f85 add conf parser and license header
bc5e932862f25688c09f0c90df74d505efb5bcc3 init the bter demo
```

# 6. 显示指定commit的修改内容
```buildoutcfg
git show <commit-id>
```
