---
title: jenkins 打包流程
date: 2019-08-12 15:02:02
tags:
- Jenkins
catergories:
- iOS
---

## 安装jenkins
我们使用Homebrew安装jenkins

<!--more-->

### 安装Homebrew
1. 首先确认机器有没有安装Homebrew

```
brew -v
```

2. 如果未安装，则安装Homebrew

```
$ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

3. 安装java环境

```
brew install Caskroom/cask/java
```
由于现在的java最高版本为12.0，jenkins不支持，所以我们安装一个第三方的java

```
brew cask install homebrew/cask-versions/adoptopenjdk8
```

4. 安装jenkins

```
brew install jenkins
```

5. 链接文件

```
ln -sfv /usr/local/opt/jenkins/*.plist ~/Library/LaunchAgents
```

6. 启动jenkins

```
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.jenkins.plist
```



