---
title: iOS搭建远程私有库
date: 2019-08-21 10:51:29
tags:
- iOS
categories: 
- iOS
photo: https://s2.ax1x.com/2019/08/19/m3QaxU.jpg
---

> iOS搭建远程私有库

<!--more-->

## 一、创建远程私有索引库

> 这个库的作用是存放索引文件

## 二、将远程索引库链接（下载）到本地

```bash
pod repo add <仓库名> <仓库源地址（SSH地址）>
```

## 三、创建远程代码私有仓库

### 1. 在本地创建 pod 项目

```bash
pod lib create <pod库名字>
```

执行结果

```bash
Cloning `https://github.com/CocoaPods/pod-template.git` into `<我的pod库名称，手动马赛克>`.
Configuring <我的pod库名称，手动马赛克> template.

------------------------------

To get you started we need to ask a few questions, this should only take a minute.

If this is your first time we recommend running through with the guide:
 - https://guides.cocoapods.org/making/using-pod-lib-create.html
 ( hold cmd and click links to open in a browser. )


What platform do you want to use?? [ iOS / macOS ]
 > iOS

What language do you want to use?? [ Swift / ObjC ]
 > ObjC

Would you like to include a demo application with your library? [ Yes / No ]
 > Yes

Which testing frameworks will you use? [ Specta / Kiwi / None ]
 > Kiwi

Would you like to do view based testing? [ Yes / No ]
 > Yes

What is your class prefix?
 > TY

Running pod install on your new library.

Analyzing dependencies
Fetching podspec for `<我的pod库名称，手动马赛克>` from `../`
Downloading dependencies
Installing <我的pod库名称，手动马赛克> (0.1.0)
Installing FBSnapshotTestCase (2.1.4)
Installing Kiwi (3.0.0)
Generating Pods project
Integrating client project

[!] Please close any current Xcode sessions and use `<我的pod库名称，手动马赛克>.xcworkspace` for this project from now on.
Sending stats
Pod installation complete! There are 3 dependencies from the Podfile and 3 total pods installed.

 Ace! you're ready to go!
 We will start you off by opening your project in Xcode
  open '<我的pod库路径，手动马赛克>'

To learn more about the template see `https://github.com/CocoaPods/pod-template.git`.
To learn more about creating a new pod, see `https://guides.cocoapods.org/making/making-a-cocoapod`.
```


### 2. 修改内容

把需要提交的代码放到路径(当前项目根目录/项目名/Classes/)下面，删除ReplaceMe.m。

### 3. 修改.podspec文件

### 4. 提交

提交代码到本地pod库

```bash
git add .
git commit -m "add file"
git remote add origin <远端pod库地址:https://gitlab.com/xxx.git>
git push -u origin master
```

打tag并提交tag到origin

```bash
git tag 0.1.0
git push --tags
```

验证本地spec文件是否有误

```bash
pod lib lint
```

验证远程spec文件是否有误

```bash
pod spec lint
```

上传.podspec索引文件到索引库

```bash
pod repo push <索引库名称> <.podspec文件名称>  --allow-warnings --use-libraries --verbose
```

**tip: **消除pod库警告`inhibit_all_warnings!`。
