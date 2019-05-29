---
title: Git基础
date: 2019-01-21 18:43:25
tags:
- Git
categories:
- Git
photo: https://git-scm.com/images/logo@2x.png
---
> Git是什么？Git是目前世界上最先进的分布式版本控制系统（没有之一）。

<!--more-->

## 在Mac上安装Git
在Mac上安装Git有两种方法：

第一种：先安装Homebrew，然后通过homebrew来安装Git，详情可以参考homebrew的[文档](https://brew.sh/)。

第二种：直接安装Xcode，Xcode集成了Git，不过默认没有安装，你需要运行Xcode，选择菜单“Xcode”->“Preferences”，在弹出窗口中找到“Downloads”，选择“Command Line Tools”，点“Install”就可以完成安装了。

## 创建版本库
什么是版本库呢？版本库又名仓库，英文名**repository**，你可以简单理解成一个目录，这个目录里面的所有文件都可以被Git管理起来，每个文件的修改、删除，Git都能跟踪，以便任何时刻都可以追踪历史，或者在将来某个时刻可以“还原”。

所以，创建一个版本库非常简单，首先，选择一个合适的地方，创建一个空目录：
```
$ mkdir MyGitPost
$ cd MyGitPost
$ pwd
/Users/username/Documents/MyGitPost
```
第二步，通过`git init`命令b把这个m目录变成Git可管理的仓库。
```
$ git init
Initialized empty Git repository in /Users/username/Documents/MyGitPost/.git/
```
## 把文件添加到版本库
首先，我们创建一个用于测试的文件
```
$ touch readme.txt
```
然后，打开`reademe.txt`文件进行编辑
```
$ vi readme.txt
```
内容如下：
```
这是一个用于测试Git的文档。
```
然后通过两步来将文件提交到Git仓库：
第一步：用命令`git add`告诉Git，把文件添加到仓库
```
$ git add readme.txt
```
执行上面的命令，没有任何显示，这就对了，Unix的哲学是“没有消息就是好消息”，说明添加成功。

第二步，用命令`git commit`告诉Git，把文件提交到仓库：
```
$ git commit -m "wrote a readme file"
[master (root-commit) 2f26a28] wrote a readme file
 1 file changed, 2 insertions(+)
 create mode 100644 readme.txt
```
`-m`后面输入的是本次提交的说明，可以输入任意内容，当然最好是有意义的，这样你就能从历史记录里方便地找到改动记录。

`git commit`命令执行成功后会告诉你，`1 file changed`：1个文件被改动（我们新添加的readme.txt文件）；`2 insertions`：插入了两行内容（readme.txt有两行内容）。

## 版本管理
现在，我们修改`readme.txt`的内容：
```
这是修改的内容
```
运行`git status`来看看修改后的结果：
```
$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
```
`git status`命令可以让我们时刻掌握仓库当前的状态，上面的命令输出告诉我们，`readme.txt`被修改过了，但还没有准备提交的修改。

使用`git diff`来查看
```
diff --git a/readme.txt b/readme.txt
index 46d49bf..75de425 100644
--- a/readme.txt
+++ b/readme.txt
@@ -1,2 +1,2 @@
这是一个用于测试Git的文档。
+这是修改的内容
(END)
```
然后就是通过`git add`和`git commit`的命令提交修改内容，在执行`git commit`之前可以使用`git status`再次查看d当前仓库的状态。

### 版本回退
在我们多次修改文件，每次`commit`都会生成一个快照，如果你对文件进行了不正当的操作，或者误删了内容，可以从最近的快照中恢复。

使用`git log`来查看每次`commit`的快照。
```
$ git log
commit 707183727c57ee8fb68199521a9b014e088af091 (HEAD -> master)
Author: Sheldon <ty24089@163.com>
Date:   Sat Jan 19 12:38:28 2019 +0800

    修改readme文件内容

commit 2f26a2852751767536b17a2f748c2081847c3578
Author: Sheldon <ty24089@163.com>
Date:   Sat Jan 19 12:04:45 2019 +0800

    wrote a readme file
```
如果嫌输出信息太多，看得眼花缭乱的，可以试试加上`--pretty=oneline`参数：
```
$ git log --pretty=oneline
707183727c57ee8fb68199521a9b014e088af091 (HEAD -> master) 修改readme文件内容
2f26a2852751767536b17a2f748c2081847c3578 wrote a readme file
```
使用`git reset`来回退到上一个版本。
```
$ git reset --hard HEAD^
HEAD is now at 2f26a28 wrote a readme file
```
如果想恢复的新的内容，使用`git reflog`查看每一次命令的记录，然后在通过`git reset --hard commit_id`命令来恢复想要的快照。

### 工作区和暂存区
工作区就是可以看的到的目录，比如在上边创建的`MyGitPost`文件夹就是工作区。

工作区有一个隐藏目录`.git`，这个不算工作区，而是Git的版本库。

Git的版本库里存了很多东西，其中最重要的就是称为`stage`（或者叫`index`）的暂存区，还有Git为我们自动创建的第一个分支`master`，以及指向`master`的一个指针叫`HEAD`。
![git_stage](./git_stage.jpeg)

### 撤销修改
如果在修改了文件，但还没有`git add`，这个时候，你不想要这个文件的修改了，想恢复未修改前的内容，那么可以使用`git checkout -- file`命令丢弃工作区的修改。
这里有两种情况：

一种是文件自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态；

一种是文件已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态。

总之，就是让这个文件回到最近一次`git commit`或`git add`时的状态。

如果在修改了文件，并且提交到了暂存区，那么可以用命令`git reset HEAD <file>`可以把暂存区的修改撤销掉（unstage），重新放回工作区。

`git reset`命令既可以回退版本，也可以把暂存区的修改回退到工作区。当我们用`HEAD`时，表示最新的版本。

再用`git status`查看一下，现在暂存区是干净的，工作区有修改。

使用`git checkout -- file`就可以恢复工作区的修改。

### 删除文件
命令`git rm`用于删除一个文件。

