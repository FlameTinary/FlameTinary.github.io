---
title: iOS开发--AFN框架基本使用
date: 2016-04-12 20:37:02
tags:
- iOS
categories:
- iOS
photo: https://camo.githubusercontent.com/1560be050811ab73457e90aee62cd1cd257c7fb9/68747470733a2f2f7261772e6769746875622e636f6d2f41464e6574776f726b696e672f41464e6574776f726b696e672f6173736574732f61666e6574776f726b696e672d6c6f676f2e706e67
---

> AFNetworking is a delightful networking library for iOS, macOS, watchOS, and tvOS. It's built on top of the [Foundation URL Loading System](https://developer.apple.com/documentation/foundation/url_loading_system), extending the powerful high-level networking abstractions built into Cocoa. It has a modular architecture with well-designed, feature-rich APIs that are a joy to use.

<!--more-->

## AFN内部结构


### NSURLConnection 

* AFURLConnectionOperation 
* AFHTTPRequestOperation 
* AFHTTPRequestOperationManager(封装了常用的 HTTP 方法) 
    * 属性 
        * baseURL :AFN建议开发者针对 AFHTTPRequestOperationManager 自定义个一个单例子类，设置 baseURL, 所有的网络访问，都只使用相对路径即可 
        * requestSerializer :请求数据格式/默认是二进制的 HTTP 
        * responseSerializer :响应的数据格式/默认是 JSON 格式 
        * operationQueue 
        * reachabilityManager :网络连接管理器 
    * 方法 
      * manager :方便创建管理器的类方法 
      * HTTPRequestOperationWithRequest :在访问服务器时，如果要告诉服务器一些附加信息，都需要在 Request 中设置 
      * GET 
      * POST 

### NSURLSession 
AFURLSessionManager 
AFHTTPSessionManager(封装了常用的 HTTP 方法) 

* GET 
* POST 
* UIKit + AFNetworking 分类 
* NSProgress :利用KVO 

半自动的序列化&反序列化的功能 

* AFURLRequestSerialization :请求的数据格式/默认是二进制的 
* AFURLResponseSerialization :响应的数据格式/默认是JSON格式 

附加功能 

安全策略 

* HTTPS 
* AFSecurityPolicy 
+ 网络检测 
* 对苹果的网络连接检测做了一个封装 
* AFNetworkReachabilityManager

## AFN的基本使用

### 发送GET请求的两种方式（POST同）

```objc
-(void)get1{ 
    //1.创建AFHTTPRequestOperationManager管理者 
    //AFHTTPRequestOperationManager内部是基于NSURLConnection实现的 
    AFHTTPRequestOperationManager *manager = [AFHTTPRequestOperationManager manager]; 
    //2.发送请求 
    /* 
    http://120.25.226.186:32812/login?username=ee&pwd=ee&type=JSON 
    第一个参数：NSString类型的请求路径，AFN内部会自动将该路径包装为一个url并创建请求对象 
    第二个参数：请求参数，以字典的方式传递，AFN内部会判断当前是POST请求还是GET请求，以选择直接拼接还是转换为NSData放到请求体中传递 
    第三个参数：请求成功之后回调Block 
    第四个参数：请求失败回调Block 
    */ 
    NSDictionary *param = @{ @"username":@"520it", 
                                @"pwd":@"520it" 
                              }; 
    //注意：字符串中不能包含空格 
    [manager GET:@"url字符串" parameters:param success:^(AFHTTPRequestOperation * _Nonnull operation, id _Nonnull responseObject) {
        NSLog(@"请求成功---%@",responseObject); 
    } failure:^(AFHTTPRequestOperation * _Nonnull operation, NSError * _Nonnull error) {
        NSLog(@"失败---%@",error); 
    }];
}

-(void)get2{ 
    //1.创建AFHTTPSessionManager管理者 
    //AFHTTPSessionManager内部是基于NSURLSession实现的 
    AFHTTPSessionManager *manager = [AFHTTPSessionManager manager]; 
    //2.发送请求 
    NSDictionary *param = @{ @"username":@"520it",
                                @"pwd":@"520it" 
                              }; 
    //注意：responseObject:请求成功返回的响应结果（AFN内部已经把响应体转换为OC对象，通常是字典或数组） 
    [manager GET:@"url字符串" parameters:param success:^(NSURLSessionDataTask * _Nonnull task, id _Nonnull responseObject) {
        NSLog(@"请求成功---%@",[responseObject class]); 
    } failure:^(NSURLSessionDataTask * _Nonnull task, NSError * _Nonnull error) {
        NSLog(@"失败---%@",error);
    }];
}
```

### 使用AFN下载文件

```objc
-(void)download{ 
    //1.创建一个管理者 
    AFHTTPSessionManager *manage = [AFHTTPSessionManager manager]; 
    //2.下载文件 
    /* 
    第一个参数：请求对象 
    第二个参数：下载进度 
    第三个参数：block回调，需要返回一个url地址，用来告诉AFN下载文件的目标地址 
    targetPath：AFN内部下载文件存储的地址，tmp文件夹下 
    response：请求的响应头 
    返回值：文件应该剪切到什么地方 
    第四个参数：block回调，当文件下载完成之后调用 
    response：响应头 
    filePath：文件存储在沙盒的地址 == 第三个参数中block的返回值 
    error：错误信息 
    */ 
    //2.1 创建请求对象 
    NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:@"url字符串"]];
    //2.2 创建下载进度，并监听 
    NSProgress *progress = nil; 
    NSURLSessionDownloadTask *downloadTask = [manage downloadTaskWithRequest:request progress:&progress destination:^NSURL * _Nonnull(NSURL * _Nonnull targetPath, NSURLResponse * _Nonnull response) { 
        NSString *caches = [NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) lastObject]; 
        //拼接文件全路径 
        NSString *fullpath = [caches stringByAppendingPathComponent:response.suggestedFilename]; 
        NSURL *filePathUrl = [NSURL fileURLWithPath:fullpath]; 
        return filePathUrl; 
    } completionHandler:^(NSURLResponse * _Nonnull response, NSURL * _Nonnull filePath, NSError * _Nonnull error) { 
        NSLog(@"文件下载完毕---%@",filePath); 
    }]; 
    //2.3 使用KVO监听下载进度 
    [progress addObserver:self forKeyPath:@"completedUnitCount" options:NSKeyValueObservingOptionNew context:nil]; 
    //3.启动任务 [downloadTask resume];
}

//获取并计算当前文件的下载进度
-(void)observeValueForKeyPath:(NSString *)keyPath ofObject:(NSProgress *)progress change:(NSDictionary<NSString *,id> *)change context:(void *)context{ 
    NSLog(@"%zd--%zd--%f",progress.completedUnitCount,progress.totalUnitCount,1.0 * progress.completedUnitCount/progress.totalUnitCount);
}
```