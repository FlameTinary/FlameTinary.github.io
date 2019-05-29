---
title: iOS开发之数据存储
date: 2016-04-12 23:14:10
categories:
- iOS
tags:
- iOS
photo: https://user-gold-cdn.xitu.io/2018/4/21/162e62a8e26ca9ed?imageView2/1/w/1304/h/734/q/85/format/webp/interlace/1
---

> 作为移动端开发工程师，所需要的数据几乎全部都是通过网络获取，而且网络请求都有时耗；在网络好的情况下这种时耗虽然不足考虑，但是一旦网络环境不好，会很影响产品体验。网络环境无法控制，但是对于一些数据不经常变动的网络请求或没必要实时更新的数据，我们可以选择将网络数据缓存本地，适时更新。

<!--more-->

## iOS应用数据存储的常用方式

* XML属性列表(plist)归档
* Preference(偏好设置)
* NSKeyedArchiver归档(NSCoding)
* SQLite 3
* Core Data

## 应用沙盒

每个iOS应用都有自己的应用沙盒(应用沙盒就是文件系统目录),与其他文件系统隔离,应用必须待在自己的沙盒里,其他应用不能访问沙盒

应用沙盒的文件系统目录,如下图所示(假设应用的名称叫Layer)

![应用沙盒的文件系统目录.png](http://upload-images.jianshu.io/upload_images/1879463-10f4ca6f7d0712b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 应用沙盒结构分析

**应用程序包:** (上图中的Layer)包含了所有的资源文件和可执行文件
**Documents:** 保存应用运行时生成的需要持久化的数据,itunes同步设备时会备份该目录,例如:游戏应用可以将游戏存档保存在该目录
**tmp:** 保存应用运行时所需的临时数据, 使用完毕后再见相应的文件从该目录删除.应用没有运行时,系统也可能会清除该目录下的文件.itunes同步设备的时候不会备份该文件夹
**Library/Caches:** 保存应用运行时生成的需要持久化的数据,iTunes同步设备的时候不会备份该目录.一般存储体积大,不需要备份的非重要数据
**Library/Preferences:** 保存应用的所有偏好设置,iOS的settings(设置)应用会在该目录中查找应用的设置信息.iTunes同步设备时会备份该目录

### 应用沙盒目录的常见获取方式

#### 沙盒根目录

```objc
NSString * home = NSHomeDirectory();
```

#### Documents(2种方式)

1. 利用沙盒根目录拼接"Documents"字符串

```objc
 NSString * home = NSHomeDirectory();
    NSString * documents = [home stringByAppendingPathComponent:@"Documents"];
```

*`不建议采用,因为新版本的操作系统可能会修改目录名`*

2. 利用NSSearchPathForDirectoriesInDomains函数

```objc
//NSUserDomainMask代表从用户文件夹下找
//YES,代表展开路径中的波浪字符"~"
NSArray * arr = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, NO);
//在iOS中,只有一个目录跟传入的参数匹配,所以这个集合里面只有一个元素
NSString * documents = [arr objectAtIndex:0];
```

#### tmp

```objc
NSString * tmp = NSTemporaryDirectory();
```

#### Library/Caches(跟Documents相似的2种方法)

1. 利用沙盒根目录拼接"Caches"字符串
2. 利用NSSearchPathForDirectoriesInDomains函数 (将函数的第一个参数改为NSCachesDirectory即可)

#### Library/Preferences 

通过NSUserDefaults类存取该目录下的设置信息

## 属性列表

属性列表是一种XML格式的文件, 拓展名为plist

如果对象是NSString\NSDictionary\NSArray\NSData\NSNumber等类型, 就可以使用writeToFile:atomically:方法直接将对象写到属性列表文件中

属性列表 - 归档NSDictionary

将一个NSDictionary对象归档到一个plist属性列表中

```objc
//将数据封装成字典
NSMutableDictionary * dic = [NSMutableDictionary dictionary];
[dic setObject:@"小米" forKey:@"name"];
[dic setObject:@"11111111111" forKey:@"phone"];
[dic setObject:@"6" forKey:@"size"];
//将字典持久化到Documents/phone.plist文件中
[dic writeToFile:path atomically:YES];
```

NSDictionary的存储和读取过程

![NSDictionary的存储和读取过程.png](http://upload-images.jianshu.io/upload_images/1879463-9397a6736de2278e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 偏好设置

很多iOS应用都支持偏好设置,比如保存用户名\密码\字体大小等设置, iOS提供了一套标准的解决方案来为应用加入偏好设置功能

每个应用都有个NSUserDefaults实例，通过它来存取偏好设置

比如，保存用户名、字体大小、是否自动登录

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
[defaults setObject:@"flame" forKey:@"username"];
[defaults setFloat:18.0f forKey:@"text_size"];
[defaults setBool:YES forKey:@"auto_login"];
```

读取上次保存的设置

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
NSString *username = [defaults stringForKey:@"username"];
float textSize = [defaults floatForKey:@"text_size"];
BOOL autoLogin = [defaults boolForKey:@"auto_login"];
```

**注意**：UserDefaults设置数据时，不是立即写入，而是根据时间戳定时地把缓存中的数据写入本地磁盘。所以调用了set方法之后数据有可能还没有写入磁盘应用程序就终止了。出现以上问题，可以通过调用synchornize方法强制写入

```objc
[defaults synchornize];
```

## NSKeyedArchiver

如果对象是NSString、NSDictionary、NSArray、NSData、NSNumber等类型，可以直接用NSKeyedArchiver进行归档和恢复

不是所有的对象都可以直接用这种方法进行归档，只有遵守了NSCoding协议的对象才可以

NSCoding协议有2个方法：
1. encodeWithCoder
    每次归档对象时，都会调用这个方法。一般在这个方法里面指定如何归档对象中的每个实例变量，可以使用encodeObject:forKey:方法归档实例变量
2. initWithCoder
    每次从文件中恢复(解码)对象时，都会调用这个方法。一般在这个方法里面指定如何解码文件中的数据为对象的实例变量，可以使用decodeObject:forKey方法解码实例变量

归档一个NSArray对象到Documents/array.archive

```objc
NSArray *array = [NSArray arrayWithObjects:@”a”,@”b”,nil];
[NSKeyedArchiver archiveRootObject:array toFile:path];
```

恢复(解码)NSArray对象

```objc
NSArray *array = [NSKeyedUnarchiver unarchiveObjectWithFile:path];
```

## 归档对象

NSKeyedArchiver-归档Person对象（Person.h）

```objc
@interface Person : NSObject<NSCoding>
@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) int age;
@property (nonatomic, assign) float height;
@end
```  
  
NSKeyedArchiver-归档Person对象（Person.m）

```objc
@implementation Person
-(void)encodeWithCoder:(NSCoder *)encoder {
     [encoder encodeObject:self.name forKey:@"name"];
     [encoder encodeInt:self.age forKey:@"age"];
     [encoder encodeFloat:self.height forKey:@"height"];
}
-(id)initWithCoder:(NSCoder *)decoder {
     self.name = [decoder decodeObjectForKey:@"name"];
     self.age = [decoder decodeIntForKey:@"age"];
     self.height = [decoder decodeFloatForKey:@"height"];
     return self;
}
-(void)dealloc {
     [super dealloc];
     [_name release];
}
@end
```

NSKeyedArchiver-归档Person对象（编码和解码）

归档(编码)

```objc
Person *person = [[[Person alloc] init] autorelease];
person.name = @"TY";
person.age = 27;
person.height = 1.83f;
[NSKeyedArchiver archiveRootObject:person toFile:path];
```

恢复(解码)

```objc
Person *person = [NSKeyedUnarchiver unarchiveObjectWithFile:path];
```

NSKeyedArchiver-归档对象的注意

如果父类也遵守了NSCoding协议，请注意：
* 应该在encodeWithCoder:方法中加上一句

```objc
[super encodeWithCode:encode];
```

确保继承的实例变量也能被编码，即也能被归档
* 应该在initWithCoder:方法中加上一句

```objc
self = [super initWithCoder:decoder];
```
确保继承的实例变量也能被解码，即也能被恢复

## NSData

使用archiveRootObject:toFile:方法可以将一个对象直接写入到一个文件中，但有时候可能想将多个对象写入到同一个文件中，那么就要使用NSData来进行归档对象
NSData可以为一些数据提供临时存储空间，以便随后写入文件，或者存放从磁盘读取的文件内容。可以使用[NSMutableData data]创建可变数据空间

![流程图.png](http://upload-images.jianshu.io/upload_images/1879463-30ccb4ca364a510e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


NSData-归档2个Person对象到同一文件中

归档（编码）

```objc
// 新建一块可变数据区
NSMutableData *data = [NSMutableData data];
// 将数据区连接到一个NSKeyedArchiver对象
NSKeyedArchiver *archiver = [[[NSKeyedArchiver alloc] initForWritingWithMutableData:data] autorelease];
// 开始存档对象，存档的数据都会存储到NSMutableData中
[archiver encodeObject:person1 forKey:@"person1"];
[archiver encodeObject:person2 forKey:@"person2"];
// 存档完毕(一定要调用这个方法)
[archiver finishEncoding];
// 将存档的数据写入文件
[data writeToFile:path atomically:YES];
```

恢复（解码）

```objc
// 从文件中读取数据
NSData *data = [NSData dataWithContentsOfFile:path];
// 根据数据，解析成一个NSKeyedUnarchiver对象
NSKeyedUnarchiver *unarchiver = [[NSKeyedUnarchiver alloc] initForReadingWithData:data];
Person *person1 = [unarchiver decodeObjectForKey:@"person1"];
Person *person2 = [unarchiver decodeObjectForKey:@"person2"];
// 恢复完毕
[unarchiver finishDecoding];
```

利用归档实现深复制

比如对一个Person对象进行深复制

```objc
// 临时存储person1的数据
NSData *data = [NSKeyedArchiver archivedDataWithRootObject:person1];
// 解析data，生成一个新的Person对象
Student *person2 = [NSKeyedUnarchiver unarchiveObjectWithData:data];
// 分别打印内存地址
NSLog(@"person1:0x%x", person1); // person1:0x7177a60
NSLog(@"person2:0x%x", person2); // person2:0x7177cf0
```
![深复制解析.png](http://upload-images.jianshu.io/upload_images/1879463-69b590233c6a36ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## SQLite3
    
SQLite3是一款开源的嵌入式关系型数据库，可移植性好、易使用、内存开销小
SQLite3是无类型的，意味着你可以保存任何类型的数据到任意表的任意字段中。比如下列的创表语句是合法的：

create table t_person(name, age);
为了保证可读性，建议还是把字段类型加上：
create table t_person(name text, age integer);

SQLite3常用的5种数据类型：text、integer、float、boolean、blob

在iOS中使用SQLite3，首先要添加库文件libsqlite3.dylib和导入主头文件

```objc
#import <sqlite3.h>
```

创建、打开、关闭数据库

创建或打开数据库

```objc
// path为：~/Documents/person.db
sqlite3 *db;
int result = sqlite3_open([path UTF8String], &db);
```

**代码解析:**
sqlite3_open()将根据文件路径打开数据库，如果不存在，则会创建一个新的数据库。如果result等于常量SQLITE_OK，则表示成功打开数据库

sqlite3 *db：一个打开的数据库实例

数据库文件的路径必须以C字符串(而非NSString)传入


关闭数据库：

```objc
sqlite3_close(db);
```

执行不返回数据的SQL语句

执行创表语句

```objc
char *errorMsg;  // 用来存储错误信息
char *sql = "create table if not exists t_person(id integer primary key autoincrement, name text, age integer);";
int result = sqlite3_exec(db, sql, NULL, NULL, &errorMsg);
```
    
**代码解析：**
sqlite3_exec()可以执行任何SQL语句，比如创表、更新、插入和删除操作。但是一般不用它执行查询语句，因为它不会返回查询到的数据

sqlite3_exec()还可以执行的语句：
1. 开启事务：begin transaction;
2. 回滚事务：rollback;
3. 提交事务：commit;
    
带占位符插入数据

```objc
char *sql = "insert into t_person(name, age) values(?, ?);";
sqlite3_stmt *stmt;
if (sqlite3_prepare_v2(db, sql, -1, &stmt, NULL) == SQLITE_OK) {
     sqlite3_bind_text(stmt, 1, "母鸡", -1, NULL);
     sqlite3_bind_int(stmt, 2, 27);
}
if (sqlite3_step(stmt) != SQLITE_DONE) {
     NSLog(@"插入数据错误");
}
sqlite3_finalize(stmt);
```
    
**代码解析：**
sqlite3_prepare_v2()返回值等于SQLITE_OK，说明SQL语句已经准备成功，没有语法问题

sqlite3_bind_text()：大部分绑定函数都只有3个参数
1. 第1个参数是sqlite3_stmt *类型
2. 第2个参数指占位符的位置，第一个占位符的位置是1，不是0
3. 第3个参数指占位符要绑定的值
4. 第4个参数指在第3个参数中所传递数据的长度，对于C字符串，可以传递-1代替字符串的长度
5. 第5个参数是一个可选的函数回调，一般用于在语句执行后完成内存清理工作

sqlite_step()：执行SQL语句，返回SQLITE_DONE代表成功执行完毕

sqlite_finalize()：销毁sqlite3_stmt *对象

查询数据

```objc
char *sql = "select id,name,age from t_person;";
sqlite3_stmt *stmt;
if (sqlite3_prepare_v2(db, sql, -1, &stmt, NULL) == SQLITE_OK) {
     while (sqlite3_step(stmt) == SQLITE_ROW) {
         int _id = sqlite3_column_int(stmt, 0);
         char *_name = (char *)sqlite3_column_text(stmt, 1);
         NSString *name = [NSString stringWithUTF8String:_name];
         int _age = sqlite3_column_int(stmt, 2);
         NSLog(@"id=%i, name=%@, age=%i", _id, name, _age);
     }
}
sqlite3_finalize(stmt);
```
    
**代码解析:**

sqlite3_step()返回SQLITE_ROW代表遍历到一条新记录

sqlite3_column_*()用于获取每个字段对应的值，第2个参数是字段的索引，从0开始
