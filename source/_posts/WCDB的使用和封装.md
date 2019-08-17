---
title: WCDB的使用和封装
date: 2019-08-13 18:33:05
tags:
- iOS
categories:
- iOS
photo: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1565702556033&di=42e97cc84fdd5bac032679edd8726455&imgtype=0&src=http%3A%2F%2Fimg25.aspzz.cn%2Fuploads%2Fallimg%2Fc190624%2F156133JD62330-134U.jpg
---
公司的项目所使用的数据库是FMDB，并对其进行了二次封装；由于年久失修，遂决定改为WCDB，因此，抽时间对WCDB进行了一番研究。
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

使用WCD分为2部分：**ORM**和**CRUD**。

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

#### 约束宏

约束宏包含了**字段约束**和**表约束**。

##### 字段约束

主键约束以WCDB_PRIMARY开头，定义了数据库的主键；支持自定义主键的排序方式、是否自增。
- `WCDB_PRIMARY(className, propertyName)`是基础用法，直接使用propertyName作为数据库的主键。
- `WCDB_PRIMARY_ASC(className, propertyName)`主键升序。
- `WCDB_PRIMARY_DESC(className, propertyName)`主键降序。
- `WCDB_PRIMARY_AUTO_INCREMENT(className, propertyName)`主键自增。
- `WCDB_PRIMARY_ASC_AUTO_INCREMENT(className, propertyName)`主键自增及升序的组合。
- `WCDB_PRIMARY_DESC_AUTO_INCREMENT(className, propertyName)`主键自增及降序组合。

非空约束：`WCDB_NOT_NULL(className, propertyName)`，当该字段插入数据为空时，数据库会返回错误。
唯一约束：`WCDB_UNIQUE(className, propertyName)`，当该字段插入数据与其他列冲突时，数据库会返回错误。

##### 表约束

多主键约束以`WCDB_MULTI_PRIMARY`开头，定义了数据库的多主键，支持自定义每个主键的排序方式。
- `WCDB_MULTI_PRIMARY(className, constraintName, propertyName)`为基础用法，多个主键通过constrainName匹配。
- `WCDB_MULTI_PRIMARY_ASC(className, constraintName, propertyName)`多主键升序。
- `WCDB_MULTI_PRIMARY_DESC(className, constraintName, propertyName)`多主键降序。
  
多字段唯一约束以`WCDB_MULTI_UNIQUE`开头，定义了数据库的多字段组合唯一，支持自定义每个字段的排序方式。
- `WCDB_MULTI_UNIQUE(className, constraintName, propertyName)`基础用法。
- `WCDB_MULTI_UNIQUE_ASC(className, constraintName, propertyName)`升序。
- `WCDB_MULTI_UNIQUE_DESC(className, constraintName, propertyName)`降序。

[表约束示例](https://github.com/Tencent/wcdb/blob/master/objc/sample/orm/WCTSampleORMTableConstraint.mm)

#### 修改字段
由于SQLite只支持增加字段，并不支持删除和重命名字段。因此，WCDB在删除字段的时候只是将其定义删除；删除定义后，WCDB只是将该字段忽略，其旧数据依然在数据库中，但新增的数据基本**不会因为该字段产生额外的性能和空间损耗**。
由于SQLite不支持修改字段名称，所以WCDB采用重新映射的方式。

### CRUD

WCDB提供了三个基础类进行数据库操作：`WCTDatabase`、`WCTTable`、`WCTTransaction`。它们的接口都是**线程安全的**。

#### WCTDatabase

表示一个数据库，可以进行所有的数据库操作。

##### 创建数据库

`WCTDatabase`通过`initWithPath:`接口进行创建。该接口会同时创建path中不存在的目录。

```OBJC
NSString* path = @"<这里是数据库路径>";
WCTDatabase *database = [[WCTDatabase alloc] initWithPath:path];
```

##### 打开数据库

WCDB采用懒加载的方式管理对象，因此SQLite连接会在第一次被访问的时候打开。开发者**不需要手动打开数据库**。

```OBJC
// 判断数据库是否能够打开
if ([database canOpen]) {
  //...
}
// 判断数据库是否已经打开
if ([database isOpened]) {
  //...
}
```

##### 关闭数据库
```OBJC
[database close];
```

**注意：**由于WCDB支持多线程访问数据库，因此，该接口会阻塞等待所有线程均已操作结束。

对于一个特定路径的数据库，WCDB会在所有对象对其的引用结束时，自动关闭数据库，并且回收内存和SQLite连接。因此，大多数的时候，开发者并**不需要手动关闭数据库**。

##### 文件操作
WCDB提供了删除数据库、移动数据库、获取数据库占用空间和使用路径的文件操作接口。

```OBJC
- (BOOL)removeFilesWithError:(WCTError **)error;
- (BOOL)moveFilesToDirectory:(NSString *)directory withExtraFiles:(NSArray<NSString *> *)extraFiles andError:(WCTError **)error;
- (NSArray<NSString *> *)getPaths;
- (NSUInteger)getFilesSizeWithError:(WCTError **)error;
```

若是一个线程正在操作数据库，而另一个线程进行移动数据库的操作，可能会导致数据库的损坏；因此，文件操作通常放在关闭数据库后。

```OBJC
[database close:^{
  WCTError *error = nil;
  BOOL ret = [database moveFilesToDirectory:otherDirectory withError:&error];
  if (!ret) {
      NSLog(@"Move files Error %@", error);
  }
}];
```

#### WCTTable
表示一个表。等价于预设了`class`和`tableName`的`WCTDatabase`，仅可以进行数据库的CRUD。

```OBJC
WCTTable* table = [database getTableOfName:tableName
                                 withClass:WCTSampleTable.class];
```

#### WCTTransaction
表示一个事务。

```OBJC
WCTTransaction *transaction = [database getTransaction];
```

与WCTDatabase的事务不同，WCTTransaction可以在函数和对象之间传递，实现跨线程的事务。

```OBJC
//You can do a transaction in different threads using WCTTransaction.
//But it's better to run serially, or an inner thread mutex will guarantee this.
BOOL ret = [transaction begin];
dispatch_async(dispatch_queue_create("other thread", DISPATCH_QUEUE_SERIAL), ^{
  WCTSampleTransaction *object = [[WCTSampleTransaction alloc] init];
  BOOL ret = [transaction insertObject:object
                                  into:tableName];
  if (ret) {
      [transaction commit];
  } else {
      [transaction rollback];
  }
});
```

#### CRUD操作

##### 创建数据库

```OBJC
BOOL ret = [database createTableAndIndexesOfName:tableName
                                       withClass:WCTSampleTable.class];

// 或者

BOOL ret = [database createTableOfName:tableName
                     withColumnDefList:{
                     WCTSampleTable.intValue.def(WCTColumnTypeInteger32),
                     WCTSampleTable.stringValue.def(WCTColumnTypeString)
           }];
```

##### 将数据插入表

- `insertObject:into:和insertObjects:into:`，插入单个或多个对象
- `insertOrReplaceObject:into`和`insertOrReplaceObjects:into`，插入单个或多个对象。当对象的主键在数据库内已经存在时，更新数据；否则插入对象。
- `insertObject:onProperties:into:`和`insertObjects:onProperties:into:`，插入单个或多个对象的部分属性
- `insertOrReplaceObject:onProperties:into`和`insertOrReplaceObjects:onProperties:into`，插入单个或多个对象的部分属性。当对象的主键在数据库内已经存在时，更新数据；否则插入对象。

##### 删除数据

- `deleteAllObjectsFromTable:`删除表内的所有数据
- `deleteObjectsFromTable:`后可组合接 where、orderBy、limit、offset以删除部分数据

##### 更新数据
- `updateAllRowsInTable:onProperties:withObject:`，通过object更新数据库中所有指定列的数据
- `updateRowsInTable:onProperties:withObject:`后可组合接 where、orderBy、limit、offset以通过object更新指定列的部分数据
- `updateAllRowsInTable:onProperty:withObject:`，通过object更新数据库某一列的数据
- `updateRowsInTable:onProperty:withObject:`后可组合接 where、orderBy、limit、offset以通过object更新某一列的部分数据
- `updateAllRowsInTable:onProperties:withRow:`，通过数组更新数据库中的所有指定列的数据
- `updateRowsInTable:onProperties:withRow:`后可组合接 where、orderBy、limit、offset以通过数组更新指定列的部分数据
- `updateAllRowsInTable:onProperty:withRow:`，通过数组更新数据库某一列的数据
- `updateRowsInTable:onProperty:withRow:`后可组合接 where、orderBy、limit、offset以通过数组更新某一列的部分数

##### 查找数据
- `getOneObjectOfClass:fromTable:`后可接 where、orderBy、limit、offset以从数据库中取出一行数据并组合成object
- `getOneObjectOnResults:fromTable:`后可接 where、orderBy、limit、offset以从数据库中取出一行数据的部分列并组合成object
- `getOneRowOnResults:fromTable:`后可接 where、orderBy、limit、offset以从数据库中取出一行数据的部分列并组合成数组
- `getOneColumnOnResult:fromTable:`后可接 where、orderBy、limit、offset以从数据库中取出一列数据并组合成数组
- `getOneDistinctColumnOnResult:fromTable:`后可接 where、orderBy、limit、offset以从数据库中取出一列数据，并取distinct后组合成数组。
- `getOneValueOnResult:fromTable:`后可接 where、orderBy、limit、offset以从数据库中取出一行数据的某一列
- `getAllObjectsOfClass:fromTable:`，取出所有数据，并组合成object
- `getObjectsOfClass:fromTable:`后可接 where、orderBy、limit、offset以从数据库中取出一部分数据，并组合成object
- `getAllObjectsOnResults:fromTable:`，取出所有数据的指定列，并组合成object
- `getObjectsOnResults:fromTable:`后可接where、orderBy、limit、offset以从数据库中取出一部分数据的指定列，并组合成object
- `getAllRowsOnResults:fromTable:`，取出所有数据的指定列，并组合成数组
- `getRowsOnResults:fromTable:`后可接where、orderBy、limit、offset以从数据库中取出一部分数据的指定列，并组合成数组

[CRUD示例](https://github.com/Tencent/wcdb/blob/master/objc/sample/convenient/sample_convenient_main.mm)

##### 链式接口
WCDB对于增删改查操作，都提供了对应的类以实现链式调用

- WCTInsert
- WCTDelete
- WCTUpdate
- WCTSelect
- WCTRowSelect
- WCTMultiSelect

```OBJC
WCTSelect *select = [database prepareSelectObjectsOnResults:Message.localID.max()
                                                  fromTable:@"message"];
NSArray<Message *> *objects = [[[[select where:Message.localID > 0] 
                               			groupBy:{Message.content}]
                                	    orderBy:Message.createTime.order()] 
                               		      limit:10].allObjects;
```


### 其他

#### 事务
#### 调试SQL
[WCTStatistics SetGlobalSQLTrace:]会监控所有执行的SQL，该接口可用于调试，确定SQL是否执行正确。

```OBJC
//SQL Execution Monitor
[WCTStatistics SetGlobalSQLTrace:^(NSString *sql) {
	NSLog(@"SQL: %@", sql);
}];
```

#### WINQ
#### 自定义类型
#### 隔离C++代码
#### 关于数据库加密
#### 高级用法
##### 主键自增(Auto Increment)
##### as重定向
##### 多表查询
#### 基础类共享
对于同一个路径的数据库，不同的`WCTDatabase`、`WCTTable`、`WCTTransaction`对象共享同一个WCDB核心。

```OBJC
WCTDatabase* database1 = [[WCTDatabase alloc] initWithPath:path];
WCTDatabase* database2 = [[WCTDatabase alloc] initWithPath:path];
database1.tag = 1;
NSLog(@"%d", database2.tag);//print 1
```
