---
title: iOS之数据解析
date: 2016-04-12 14:17:14
tags:
- iOS
categories:
- iOS
photo: https://upload-images.jianshu.io/upload_images/957477-26156a76b726acb3.jpg-zhengwen?imageMogr2/auto-orient/strip%7CimageView2/2/w/418/format/webp
---

>iOS开发中，几乎只要是与网络相关的应用，都离不开对网络数据的解析与应用。一般来讲，我们会从网络获取XML或者Json格式的数据，这些数据有着特定的数据结构，必须对其进行解析，得到我们可以处理的数据。所谓“解析”，就是从事先规定好的格式串中提取数据。解析的前提是数据的提供方与获取方提前约定好格式，数据提供方按照格式提供数据，数据获取方按照格式获取数据。

<!--more-->

## JSON解析

JSON是一种轻量级的数据格式，一般用于数据交互。服务器返回给客户端的数据，一般都是JSON格式或者XML格式（文件下载除外）

### 相关说明

1. JSON的格式很像OC中的字典和数组
2. 标准JSON格式key必须是双引号

### JSON解析方案

1. 第三方框架 JSONKit\SBJSON\TouchJSON
2. 苹果原生（NSJSONSerialization）

### JSON解析相关代码

#### json数据->OC对象

```objc
//把json数据转换为OC对象 
-(void)jsonToOC { 
    //1. 确定url路径
    NSURL *url = [NSURL URLWithString:@"URL字符串"];
    //2.创建一个请求对象
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    //3.使用NSURLConnection发送一个异步请求
    [NSURLConnection sendAsynchronousRequest:request queue:[NSOperationQueue mainQueue] completionHandler:^(NSURLResponse * _Nullable response, NSData * _Nullable data, NSError * _Nullable connectionError) {
        //4.当接收到服务器响应的数据后，解析数据(JSON--->OC)
        /* 
        第一个参数：要解析的JSON数据，是NSData类型也就是二进制数据 
        第二个参数: 解析JSON的可选配置参数 
        NSJSONReadingMutableContainers 解析出来的字典和数组是可变的 
        NSJSONReadingMutableLeaves 解析出来的对象中的字符串是可变的 iOS7以后有问题 
        NSJSONReadingAllowFragments 被解析的JSON数据如果既不是字典也不是数组, 那么就必须使用这个 
      */
        NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:nil]; 
        NSLog(@"%@",dict);
      }];
}
```

#### OC对象->JSON对象

``` objc
//1.要转换成JSON数据的OC对象*这里是一个字典
NSDictionary *dictM = @{@"name":@"flame", 
                        @"age":@20, 
                        @"height":@1.80 
                      };
//2.OC->JSON
/* 
注意：可以通过+ (BOOL)isValidJSONObject:(id)obj;方法判断当前OC对象能否转换为JSON数据 
具体限制： 
1.obj 是NSArray 或 NSDictionay 以及他们派生出来的子类 
2.obj 包含的所有对象是NSString,NSNumber,NSArray,NSDictionary 或NSNull 
3.字典中所有的key必须是NSString类型的 
4.NSNumber的对象不能是NaN或无穷大 
*/
/* 
第一个参数：要转换成JSON数据的OC对象，这里为一个字典 
第二个参数：NSJSONWritingPrettyPrinted对转换之后的JSON对象进行排版，无意义 
*/
NSData *data = [NSJSONSerialization dataWithJSONObject:dictM options:NSJSONWritingPrettyPrinted error:nil];
//3.打印查看Data是否有值
/* 
第一个参数：要转换为String的二进制数据 
第二个参数：编码方式，通常采用NSUTF8StringEncoding 
*/
NSString *strM = [[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding];
NSLog(@"%@",strM);
```

#### OC对象和JSON数据格式之间的一一对应关系

```objc
//OC对象和JSON数据之间的一一对应关系
-(void)oCWithJSON{ 
    //JSON的各种数据格式 
    //NSString *test = @"\"flame\""; 
    //NSString *test = @"true"; 
    NSString *test = @"{\"name\":\"flame\"}"; 
    //把JSON数据->OC对象,以便查看他们之间的一一对应关系 
    //注意点：被解析的JSON数据如果既不是字典也不是数组（比如是NSString）, 那么就必须使用这 NSJSONReadingAllowFragments 
    id obj = [NSJSONSerialization JSONObjectWithData:[test dataUsingEncoding:NSUTF8StringEncoding] options:NSJSONReadingAllowFragments error:nil]; 
    NSLog(@"%@", [obj class]); 
    /* 
    JSON数据格式和OC对象的一一对应关系 
    {} -> 字典 
    [] -> 数组 
    "" -> 字符串 
    10/10.1 -> NSNumber 
    true/false -> NSNumber 
    null -> NSNull 
    */
}
```

#### 如何查看复杂的JSON数据
方法一： 在线格式化http://tool.oschina.net/codeformat/json
方法二： 把解析后的数据写plist文件，通过plist文件可以直观的查看JSON的层次结构。 
[dictM writeToFile:@"文件路径/文件名.plist" atomically:YES];

#### 字典转模型框架

1. 相关框架
  - Mantle 需要继承自MTModel
  - JSONModel 需要继承自JSONModel
  - MJExtension 不需要继承，无代码侵入性
2. 自己设计和选择框架时需要注意的问题
  - 侵入性
  - 易用性，是否容易上手
  - 扩展性，很容易给这个框架增加新的功能

## XML解析

### XML简单介绍

XML：可扩展标记语言
http://baike.baidu.com/link?url=TbqIa8egIx6fbzxIPnajB253ae80Nli9O7sLob2LW9zt-lN1WKxuhfV5srC-lyaJpBgQXezNbHN8-QDmMY46dqQSI9SfOrzgKxB8GXCuhtmmoC1gpkqmJ1Z1PhWgV_TD

### XML解析

XML解析的两种方式 

1. SAX:从根元素开始，按顺序一个元素一个元素的往下解析，可用于解析大、小文件 
2. DOM:一次性将整个XML文档加载到内存中，适合较小的文件 

解析XML的工具 

1. 苹果原生NSXMLParser:使用SAX方式解析，使用简单 
2. 第三方框架 

libxml2:纯C语言的，默认包含在iOS SDK中，同时支持DOM和SAX的方式解析 GDataXML:采用DOM方式解析，该框架由Google开发，是基于xml2的

### XML解析代码

使用NSXMLParser解析XML步骤和代理方法

```objc
//解析步骤：
//4.1 创建一个解析器
NSXMLParser *parser = [[NSXMLParser alloc]initWithData:data];
//4.2 设置代理
parser.delegate = self;
//4.3 开始解析
[parser parse];
```

```objc
//1.开始解析XML文档
-(void)parserDidStartDocument:(nonnull NSXMLParser *)parser
//2.开始解析XML中某个元素的时候调用，比如<video>
-(void)parser:(nonnull NSXMLParser *)parser didStartElement:(nonnull NSString *)elementName namespaceURI:(nullable NSString *)namespaceURI qualifiedName:(nullable NSString *)qName attributes:(nonnull NSDictionary<NSString *,NSString *> *)attributeDict{ 
    if ([elementName isEqualToString:@"videos"]) { return; } 
    //字典转模型 
    TYVideo *video = [TYVideo objectWithKeyValues:attributeDict];   
    [self.videos addObject:video];
}
//3.当某个元素解析完成之后调用，比如</video>
-(void)parser:(nonnull NSXMLParser *)parser didEndElement:(nonnull NSString *)elementName namespaceURI:(nullable NSString *)namespaceURI qualifiedName:(nullable NSString *)qName
//4.XML文档解析结束
-(void)parserDidEndDocument:(nonnull NSXMLParser *)parser
```

使用GDataParser解析XML的步骤和方法

```objc
//配置环境
// 1 先导入框架，然后按照框架使用注释配置环境
// 2 GDataXML框架是MRC的，所以还需要告诉编译器以MRC的方式处理GDataXML的代码
//加载XML文档（使用的是DOM的方式一口气把整个XML文档都吞下）
GDataXMLDocument *doc = [[GDataXMLDocument alloc]initWithData:data options:kNilOptions error:nil];
//获取XML文档的根元素，根据根元素取出XML中的每个子元素 
NSArray * elements = [doc.rootElement elementsForName:@"video"];
//取出每个子元素的属性并转换为模型
for (GDataXMLElement *ele in elements) { 
    TYVideo *video = [[TYVideo alloc]init]; 
    video.name = [ele attributeForName:@"name"].stringValue; 
    video.length = [ele attributeForName:@"length"].stringValue.integerValue; 
    video.url = [ele attributeForName:@"url"].stringValue;
    video.image = [ele attributeForName:@"image"].stringValue; 
    video.ID = [ele attributeForName:@"id"].stringValue; 
    //把转换好的模型添加到tableView的数据源self.videos数组中 
    [self.videos addObject:video];
}
```

## 多值参数和中文输出问题

### 多值参数如何设置请求路径

```objc
/*
如果一个参数对应着多个值，那么直接按照"参数=值&参数=值"的方式拼接 
*/
-(void)test{ 
    //1.确定
    URL NSURL *url = [NSURL URLWithString:@"URL字符串"]; 
    //2.创建请求对象 
    NSURLRequest *request = [NSURLRequest requestWithURL:url]; 
    //3.发送请求 
    [NSURLConnection sendAsynchronousRequest:request queue:[NSOperationQueue mainQueue] completionHandler:^(NSURLResponse * _Nullable response, NSData * _Nullable data, NSError * _Nullable connectionError) { 
        //4.解析 
        NSLog(@"%@",[NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:nil]); 
    }];
}
```

### 如何解决字典和数组中输出乱码的问题

给字典和数组添加一个分类，重写descriptionWithLocale方法，在该方法中拼接元素格式化输出。 

```objc
-(nonnull NSString *)descriptionWithLocale:(nullable id)locale
```
