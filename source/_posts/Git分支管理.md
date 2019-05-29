---
title: Git分支管理
date: 2019-01-21 18:52:55
tags:
- Git
categories:
- Git
photo: https://git-scm.com/images/logo@2x.png
---
> 分支就是科幻电影里面的平行宇宙，当你正在电脑前努力学习Git的时候，另一个你正在另一个平行宇宙里努力学习SVN。
如果两个平行宇宙互不干扰，那对现在的你也没啥影响。不过，在某个时间点，两个平行宇宙合并了，结果，你既学会了Git又学会了SVN！

<!--more-->

## 创建于合并分支
查看分支：`git branch`

创建分支：`git branch <name>`

切换分支：`git checkout <name>`

创建+切换分支：`git checkout -b <name>`

合并某分支到当前分支：`git merge <name>`

删除分支：`git branch -d <name>`

##解决冲突
当两个分支同时做了修改，这个时候合并分支会出现冲突。

可以查看冲突文件

Git用`<<<<<<<`，`=======`，`>>>>>>>`标记出不同分支的内容

修改后再提交，用`git log --graph --pretty=oneline --abbrev-commit`命令查看分支的合并情况。

## 分支管理策略
通常，合并分支时，如果可能，Git会用`Fast forward`模式，但这种模式下，删除分支后，会丢掉分支信息。

如果要强制禁用`Fast forward`模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。
```
git merge --no-ff -m "merge with no-ff" dev
```
`--no-ff`参数，表示禁用`Fast forward`。

因为本次合并要创建一个新的commit，所以加上`-m`参数，把commit描述写进去。

合并后，我们用`git log`查看分支历史。

在实际开发中，我们应该按照几个基本原则进行分支管理：

首先，`master`分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；

那在哪干活呢？干活都在`dev`分支上，也就是说，`dev`分支是不稳定的，到某个时候，比如1.0版本发布时，再把`dev`分支合并到`master`上，在`master`分支发布1.0版本；

你和你的小伙伴们每个人都在`dev`分支上干活，每个人都有自己的分支，时不时地往`dev`分支上合并就可以了。

所以，团队合作的分支看起来就像这样：

![分支策略](./分支策略.png)

## bug分支
如果当前你正在进行开发工作，这个时候需要fix一个bug，那么可以使用`git stash`命令先把当前的工作现场保存。

然后切换到需要修复bug的分支，创建临时分支。

修复完毕后，将临时分支合并到需要修复的bug分支，完成bug的修复。

用`git stash list`命令查看保存的工作现场。

工作现场还在，Git把stash内容存在某个地方了，但是需要恢复一下，有两个办法：

一是用`git stash apply`恢复，但是恢复后，stash内容并不删除，你需要用`git stash drop`来删除；

另一种方式是用`git stash pop`，恢复的同时把stash内容也删了。

## 多人协作
要查看远程库的信息，用`git remote`或者，用`git remote -v`显示更详细的信息

### 推送分支
推送主分支
```
$ git push origin master
```
推送其他分支，比如`dev`，就改成：
```
$ git push origin dev
```

### 抓取分支
使用`git clone`将远程仓库克隆到本地，使用`git branch`查看分支，会发现只有`master`分支
```
$ git branch
* master
```
如果要在`dev`分支上开发，就必须创建远程`origin`的`dev`分支到本地，用这个命令创建本地`dev`分支：
```
$ git checkout -b dev origin/dev
```
如果你的小伙伴已经向origin/dev分支推送了他的提交，而碰巧你也对同样的文件作了修改，推送的时候也许会发生冲突。

先用`git pull`把最新的提交从`origin/dev`抓下来，然后，在本地合并，解决冲突，再推送。

如果`git pull`也失败了，原因是没有指定本地`dev`分支与远程`origin/dev`分支的链接，根据提示，设置`dev`和`origin/dev`的链接：
```
$ git branch --set-upstream-to=origin/dev dev
Branch 'dev' set up to track remote branch 'dev' from 'origin'.
```

**多人协作的工作模式通常是这样：**

首先，可以试图用`git push origin <branch-name>`推送自己的修改；

如果推送失败，则因为远程分支比你的本地更新，需要先用`git pull`试图合并；

如果合并有冲突，则解决冲突，并在本地提交；

没有冲突或者解决掉冲突后，再用`git push origin <branch-name>`推送就能成功！

如果`git pull`提示`no tracking information`，则说明本地分支和远程分支的链接关系没有创建，用命令`git branch --set-upstream-to <branch-name> origin/<branch-name>`。

这就是多人协作的工作模式，一旦熟悉了，就非常简单。

## rebase
rebase操作可以把本地未push的分叉提交历史整理成直线；

rebase的目的是使得我们在查看历史提交的变化时更容易，因为分叉的提交需要三方对比。
