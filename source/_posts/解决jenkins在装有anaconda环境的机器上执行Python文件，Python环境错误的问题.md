---
title: 解决jenkins在装有anaconda环境的机器上执行Python文件，Python环境错误的问题
date: 2019-08-13 11:12:50
tags:
- jenkins
categories:
- jenkins
photo: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1565677144522&di=6954ebf5259d33dc50f673ac2cfa35ab&imgtype=0&src=http%3A%2F%2Fke.atguigu.com%2Ffiles%2Fdefault%2F2018%2F06-06%2F0938284d7ec9621229.jpg
---

使用jenkins定时执行某些任务或者执行测试是很方便的一个工作，但是有些时候如果碰到一些环境或者冲突方面的问题也是比较头疼的。
<!--more-->

python分为2.7和3.0以上两个大版本，所以一些朋友的电脑中会同时存在这两种版本。由于我的电脑是MacBook pro， 所以系统中自动安装的2.7的版本。

但我需要使用3.7的版本作为主要开发，且系统中的2.7版本也不能删除，因为系统的一些功能需要依赖2.7的版本。

所以，在经过双版本的折腾后，还是选择了Anaconda的怀抱。

以前一直使用Jenkins自动执行iOS程序，最近写了一个Python的工具，需要定时执行，所以也是用Jenkins来操作；但是，问题来了，在Jenkins中的`Execute shell`中直接写了：

```bash
python "文件路径"
```
这样却报错了...

![jenkins错误](https://s2.ax1x.com/2019/08/13/m9JIQe.jpg)

可是明明在Terminal中执行没有问题啊！！

第一次猜测可能是Jenkins找不到python的路径，所以在Jenkins -> Manage Jenkins -> Configure System -> 全局属性 -> Environment variables -> 中添加了Python的路径，然并卵！

![配置python在jenkins中的路径](https://s2.ax1x.com/2019/08/13/m9JxSS.jpg)

继续折腾，既然找不到包，那么我就把包放在工程文件所在的路径中，最后还是找不到!

最后直接使用python绝对路径执行文件

![路径执行python](https://s2.ax1x.com/2019/08/13/m9YAYV.jpg)

```bash
/anaconda3/bin/python ~/Documents/TYPythonTools/TYChartRobot.py
```