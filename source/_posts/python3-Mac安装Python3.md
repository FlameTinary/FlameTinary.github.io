---
title: python3-Mac安装Python3
date: 2019-01-21 19:04:37
tags:
- Python
categories:
- 后台
photo: https://www.python.org/static/img/python-logo.png
---

> 
因为Python是跨平台的，它可以运行在Windows、Mac和各种Linux/Unix系统上。在Windows上写Python程序，放到Linux上也是能够运行的。
要开始学习Python编程，首先就得把Python安装到你的电脑里。安装后，你会得到Python解释器（就是负责运行Python程序的），一个命令行交互环境，还有一个简单的集成开发环境。

<!--more-->

## 安装Python3
系统自带的Python版本是2.7。要安装最新的Python 3.7，有两个方法：

方法一：从Python官网下载Python 3.7的[安装程序](https://www.python.org/ftp/python/3.7.0/python-3.7.0-macosx10.9.pkg)（网速慢的同学请移步[国内镜像](https://pan.baidu.com/s/1kU5OCOB#list/path=%2Fpub%2Fpython)），双击运行并安装；

方法二：如果安装了[Homebrew](https://brew.sh/)，直接通过命令`brew install python3`安装即可。

## 直接运行.py文件
在文件开头添加：
```
#!/usr/bin/env python3
```
然后通过命令给文件权限：
```
chmod a+x hello.py
```
就可以通过命令`./hello.py`直接运行.py文件了。