---
title: iOS之文件的压缩和解压缩
date: 2016-04-12 20:41:11
categories:
- iOS
tags:
- iOS
photo: https://img.pconline.com.cn/images/upload/upc/tx/pcdlc/1612/22/c109/33077087_1482396951548.jpg
---
>使用ZipArchive来压缩和解压缩文件需要添加依赖库（libz）,使用需要包含Main文件，如果使用cocoaPoads来安装框架，那么会自动的配置框架的使用环境

<!--more-->

## 压缩文件的第一种方式

```objc
//压缩文件的第一种方式
/*
第一个参数：压缩文件要保存的位置
第二个参数：要压缩哪几个文件
*/
[Main createZipFileAtPath:fullpath withFilesAtPaths:arrayM];
```

## 压缩文件的第二种方式

```objc
//压缩文件的第二种方式
/*
第一个参数：文件压缩到哪个地方
第二个参数：要压缩文件的全路径
*/
[Main createZipFileAtPath:fullpath withContentsOfDirectory:zipFile];
```

## 如何对压缩文件进行解压

```objc
//如何对压缩文件进行解压
/*
第一个参数：要解压的文件
第二个参数：要解压到什么地方
*/
[Main unzipFileAtPath:unZipFile toDestination:fullpath];
```