---
title: WCDB的使用和封装
date: 2019-08-13 18:33:05
tags:
- iOS
categories:
- iOS
photo: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1565702556033&di=42e97cc84fdd5bac032679edd8726455&imgtype=0&src=http%3A%2F%2Fimg25.aspzz.cn%2Fuploads%2Fallimg%2Fc190624%2F156133JD62330-134U.jpg
---

[WCDB](https://github.com/Tencent/wcdb)是腾讯开源的一款供客户端使用的数据库，基于[SQLCipher](https://github.com/sqlcipher/sqlcipher)，支持iOS，macOS和Android。

<!--more-->

## 特点

### 易用
WCDB通过深度封装数据库语言，使开发者可以通过OC或者Swift的代码方式使用数据库，无需为了拼接SQL而写一大坨代码；

### 高效
WCDB通过框架层和[SQLCipher](https://github.com/sqlcipher/sqlcipher)源码优化，使其有更高效的表现；

### 完整
WCDB覆盖了数据库使用过程中大多数的使用场景和需求；

* **加密：**WCDB提供基于[SQLCipher](https://github.com/sqlcipher/sqlcipher)的数据库加密。
* **损坏修复：**WCDB内建了Repair Kit用于修复损坏的数据库。
* **反注入：**WCDB内建了对SQL注入的保护。

## 安装WCDB
使用Cocoapods安装WCDB：
1. 更新pod repo：`pod repo update`
2. 在Podfile对应的target中，添加`pod 'WCDB'`
3. 执行`pod install --verbose`
4. 打开workspace
5. 引入头文件'#import <WCDB/WCDB.h>'




