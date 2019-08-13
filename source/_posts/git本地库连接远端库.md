---
title: git本地库连接远端库
date: 2019-08-13 14:55:27
tags:
- git
categories:
- git
---

我们使用git有时候会有这种情况：
我们只在本地创建了git仓库，并没有在GitHub上创建远端仓库；当我们在本的的git仓库提交了几次修改后，想要将此git仓库放在GitHub，那么我们需要怎么做呢？
<!--more-->

## 连接远程仓库

假设我们已经在GitHub上创建了一个空的仓库

先cd到我们本地git仓库所在的目录,然后执行git连接操作

```
git remote add origin git@github.com:FlameTinary/TYPythonTools.git
```

## 错误步骤

然后执行`git pull`，会发现，报错了：

```
warning: no common commits
remote: Enumerating objects: 4, done.
remote: Counting objects: 100% (4/4), done.
remote: Compressing objects: 100% (4/4), done.
Unpacking objects: 100% (4/4), done.
remote: Total 4 (delta 0), reused 0 (delta 0), pack-reused 0
From github.com:FlameTinary/TYPythonTools
 * [new branch]      master     -> origin/master
There is no tracking information for the current branch.
Please specify which branch you want to merge with.
See git-pull(1) for details.

    git pull <remote> <branch>

If you wish to set tracking information for this branch you can do so with:

    git branch --set-upstream-to=origin/<branch> master

 ✘ sheldon@tianyudeMacBook-Pro  ~/Documents/TYPythonTools   master  git pull remote master
fatal: 'remote' does not appear to be a git repository
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

执行`git pull origin master --allow-unrelated-histories`也不行；

那么怎么办呢？

## 正确步骤

原来是我们本的的branch和远端的branch无法对应起来；

可以执行`git branch --set-upstream-to=origin/master master`，会出现：

```
Branch 'master' set up to track remote branch 'master' from 'origin'.
```

然后我们再执行`git pull origin master --allow-unrelated-histories`；

最后将本地内容推送到远端：

```
git push -u origin master
```

大功告成！！

## 总结：
### 1. 连接远端仓库
```
git remote add origin git@github.com:FlameTinary/TYPythonTools.git
```
### 2. 连接分支
```
git branch --set-upstream-to=origin/master master
```
### 3. 拉取远端代码
```
git pull origin master --allow-unrelated-histories
```
### 4. 推送本地代码到远端
```
git push -u origin master
```

