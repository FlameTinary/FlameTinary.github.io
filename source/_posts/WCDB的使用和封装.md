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

## 使用

使用WCD分为2部分：ORM和CRUD

### ORM

ORM的作用就是映射数据库字段和我们的数据模型；WCDB使用内置的宏来连接类、属性与表、字段。共有三类宏，分别对应数据库的字段、索引和约束。

#### 字段宏

字段宏以WCDB_SYNTHESIZE开头，定义了类属性与字段之间的联系。
- `WCDB_SYNTHESIZE(className, propertyName)`是最简单的的用法，直接使用propertyName作为数据库中字段的名称。
- `WCDB_SYNTHESIZE_COLUMN(className, propertyName, columnName)`支持自定义字段名。
- `WCDB_SYNTHESIZE_DEFAULT(className, propertyName, defaultValue)`自定义字段默认值（可以是任意C类型或NSString, NSData, NSNumber, NSNull）。
- `WCDB_SYNTHESIZE_COLUMN_DEFAULT(className, propertyName, cloumnName, defaultValue)`。
[字段宏参考](https://github.com/Tencent/wcdb/blob/master/objc/sample/orm/WCTSampleORM.mm)

#### 索引宏

索引宏以WCDB_INDEX开头，定义了数据库的索引属性。支持定义索引的排序方式。

- `WCDB_INDEX(className, indexSubfixName, propertyName)`是基础用法，直接指定某个字段为索引。同时，WCDB会将tableName + indexSubfixName作为该索引的名字。
- `WCDB_INDEX_ASC(className, indexSubfixName, propertyName)`升序。
- `WCDB_INDEX_DESC(className, indexSubfixName, propertyName)`降序。
- `WCDB_UNIQUE_INDEX(className, indexSubfixName, propertyName)`唯一索引。
- `WCDB_UNIQUE_INDEX_ASC(className, indexSubfixName, propertyName)`唯一索引升序排列。
- `WCDB_UNIQUE_INDEX_DESC(className, indexSubfixName, propertyName)`唯一索引降序排列。

WCDB通过`indexSubfixName`匹配多索引。相同的`indexSubfixName`会被组合为多字段索引。

例如：

```
WCDB_INDEX(WCTSampleORMIndex, "_multiIndexSubfix", multiIndexPart1)
WCDB_INDEX(WCTSampleORMIndex, "_multiIndexSubfix", multiIndexPart2)
```
[索引宏例子](https://github.com/Tencent/wcdb/blob/master/objc/sample/orm/WCTSampleORMIndex.mm)




