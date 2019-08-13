---
title: jenkins执行GitHub中的Python代码
date: 2019-08-13 17:01:43
tags:
- jenkins
categories:
- jenkins
photo: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1565697034209&di=ee994d5bbbd74f753dfdedaa21e6868d&imgtype=0&src=http%3A%2F%2Fworkatbackbase.com%2Fwp-content%2Fuploads%2F2018%2F07%2Fjenkins.png
---

jenkins是很好用的一个自动化测试和执行代码的工具，[查看如何安装jenkins](https://sheldon.top/2019/08/12/jenkins%E6%89%93%E5%8C%85%E6%B5%81%E7%A8%8B/);

我们的代码是存储在代码仓库里面的，一般，我们使用GitHub来存放我们的代码；

那么，怎么使用jenkins来直接执行我们存放在GitHub中的代码呢？

<!--more-->

## jenkins 创建一个任务
![jenkins创建任务](https://s2.ax1x.com/2019/08/13/mCdKhR.jpg)

## 配置jenkins
### 选择GitHub项目
![填写项目地址](https://s2.ax1x.com/2019/08/13/mC0QOK.jpg)

### 填写源码管理
![源码管理](https://s2.ax1x.com/2019/08/13/mCBfCd.jpg)

如果没有凭证，就点击Credentials右边的**添加**
![添加凭证](https://s2.ax1x.com/2019/08/13/mCrrTO.jpg)
**注意：**这里添加的是SSH的私钥

#### 如何查询自己的ssh密钥？
1. 进入ssh文件夹
```
cd ~/.ssh
```
2. 查看私钥文件
```
cat id_rsa
```

### 定时构建（可选）
![定时构建](https://s2.ax1x.com/2019/08/13/mCspAU.jpg)

### 每次构建之前清空一下Jenkins工作空间，避免拉取的代码有冲突
![清空工作空间](https://s2.ax1x.com/2019/08/13/mCswCQ.jpg)

### 构建
在构建选项中选择**增加构建步骤->Execute shell**;

然后填写需要执行的脚本：
![执行脚本](https://s2.ax1x.com/2019/08/13/mCyGRJ.jpg)

### 最后
点击保存，然后就可以build我的项目啦！

