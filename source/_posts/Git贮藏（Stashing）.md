---
title: Git贮藏（Stashing）
date: 2019-08-22 14:53:30
tags:
- Git
categories:
- Git
photo: https://git-scm.com/images/logo@2x.png
---

> 我们在工作中经常会遇到一种情况，当我们在项目的当前分支修改了一些代码，工作进行到一半，又需要去其他的分支处理一些事情；这个时候，你不想把修改了一半的代码提交到git，但不提交又无法切到其他分支；怎么办呢？

<!--more-->

我们可以使用Git的一个功能——Stashing（贮藏）。

## 作用

Stashing可以将当前工作区已修改状态下的文件储存起来，在栈中增加一条存储。

## 用法

### 列出所有的存储列表

```bash
git stash list
```

### 应用存储

```bash
git stash pop
```

重新应用最后一次的存储，同时，将其从栈中移走。

```bash
git stash apply stash@{0}
```

通过存储名应用到当前工作区，但并不会从栈中移除。

### 取消贮藏

```bash
git stash show -p stash@{0} | git apply -R
```

取消之前所应用贮藏的修改。

### 移除贮藏

```bash
git stash drop <stash name>
```

从栈中移除贮藏

### 通过贮藏创建新分支

```bash
git stash branch <branch name>
```

创建一个新的分支，并且检出你贮藏工作时所做的提交，重新应用于你的工作，如果成功，将会丢弃贮藏。