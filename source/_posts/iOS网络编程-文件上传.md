---
title: iOS网络编程--文件上传
date: 2016-04-12 17:31:37
tags:
- iOS
categories:
- iOS
---

>NSURLConnection实现文件上传

<!--more-->

1. 文件上传步骤 
  `1. `确定请求路径
  `2. `根据URL创建一个可变的请求对象
  `3. `设置请求对象，修改请求方式为POST
  `4. `设置请求头，告诉服务器我们将要上传文件（Content-Type）
  `5. `设置请求体（在请求体中按照既定的格式拼接要上传的文件参数和非文件参数等数据）
     -  拼接文件参数
     -  拼接非文件参数
     -  添加结尾标记

  `6. `使用NSURLConnection sendAsync发送异步请求上传文件 
  `7. `解析服务器返回的数据

2. 文件上传设置请求体的数据格式
```
//请求体拼接格式 
//分隔符：----WebKitFormBoundaryhBDKBUWBHnAgvz9c 
//01.文件参数拼接格式 
--分隔符 
Content-Disposition:参数 
Content-Type:参数 
空行 
文件参数 
//02.非文件拼接参数 
--分隔符 
Content-Disposition:参数 
空行 
非文件的二进制数据 
//03.结尾标识 
--分隔符--
```
3. 文件上传相关代码
```
-(void)upload { 
//1.确定请求路径 
NSURL *url = [NSURL URLWithString:@"URL字符串"]; 
//2.创建一个可变的请求对象 
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url]; 
//3.设置请求方式为POST 
request.HTTPMethod = @"POST"; 
//4.设置请求头 
NSString *filed = [NSString stringWithFormat:@"multipart/form-data; boundary=%@",Kboundary]; 
[request setValue:filed forHTTPHeaderField:@"Content-Type"]; 
//5.设置请求体 
NSMutableData *data = [NSMutableData data]; 
//5.1 文件参数 
/*
 --分隔符 
Content-Disposition:参数 
Content-Type:参数 
空行 
文件参数 
*/ 
[data appendData:[[NSString stringWithFormat:@"--%@",Kboundary] dataUsingEncoding:NSUTF8StringEncoding]]; 
[data appendData:KnewLine]; 
[data appendData:[@"Content-Disposition: form-data; name=\"file\"; filename=\"test.png\"" dataUsingEncoding:NSUTF8StringEncoding]];
[data appendData:KnewLine]; 
[data appendData:[@"Content-Type: image/png" dataUsingEncoding:NSUTF8StringEncoding]];
[data appendData:KnewLine]; 
[data appendData:KnewLine]; 
[data appendData:KnewLine]; 
UIImage *image = [UIImage imageNamed:@"test"]; 
NSData *imageData = UIImagePNGRepresentation(image); 
[data appendData:imageData]; 
[data appendData:KnewLine]; 
//5.2 非文件参数 
/* 
--分隔符 
Content-Disposition:参数 
空行 
非文件参数的二进制数据 
*/ 
[data appendData:[[NSString stringWithFormat:@"--%@",Kboundary] dataUsingEncoding:NSUTF8StringEncoding]]; 
[data appendData:KnewLine]; 
[data appendData:[@"Content-Disposition: form-data;name=\"username\"" dataUsingEncoding:NSUTF8StringEncoding]];
[data appendData:KnewLine]; 
[data appendData:KnewLine];
 [data appendData:KnewLine]; 
NSData *nameData = [@"wendingding" dataUsingEncoding:NSUTF8StringEncoding]; 
[data appendData:nameData]; 
[data appendData:KnewLine];
//5.3 结尾标识 
//--分隔符-- 
[data appendData:[[NSString stringWithFormat:@"--%@--",Kboundary] dataUsingEncoding:NSUTF8StringEncoding]]; 
[data appendData:KnewLine]; 
request.HTTPBody = data; 
//6.发送请求 
[NSURLConnection sendAsynchronousRequest:request queue:[NSOperationQueue mainQueue] completionHandler:^(NSURLResponse * __nullable response, NSData * __nullable data, NSError * __nullable connectionError) { 
//7.解析服务器返回的数据 
NSLog(@"%@",[NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:nil]);
 }];
 }
```
4. 如何获得文件的MIMEType类型
  1. 直接对该对象发送一个异步网络请求，在响应头中通过response.MIMEType拿到文件的MIMEType类型
```
//如果想要及时拿到该数据，那么可以发送一个同步请求
-(NSString *)getMIMEType{ 
NSString *filePath = @"文件所在路径";
NSURLResponse *response = nil; 
[NSURLConnection sendSynchronousRequest:[NSURLRequest requestWithURL:[NSURL fileURLWithPath:filePath]] returningResponse:&response error:nil]; 
return response.MIMEType;
}
//对该文件发送一个异步请求，拿到文件的MIMEType
-(void)MIMEType{ 
// NSString *file = @"文件所在路径"; 
[NSURLConnection sendAsynchronousRequest:[NSURLRequest requestWithURL:[NSURL fileURLWithPath:@"/Users/文顶顶/Desktop/test.png"]] queue:[NSOperationQueue mainQueue] completionHandler:^(NSURLResponse * __nullable response, NSData * __nullable data, NSError * __nullable connectionError) { 
// response.MIMEType 
NSLog(@"%@",response.MIMEType);
 }];
}
```
  2. 通过UTTypeCopyPreferredTagWithClass方法
```
//注意：需要依赖于框架MobileCoreServices
-(NSString *)mimeTypeForFileAtPath:(NSString *)path{
    if (![[[NSFileManager alloc] init] fileExistsAtPath:path]) {
        return nil;
    }   
CFStringRef UTI = UTTypeCreatePreferredIdentifierForTag(kUTTagClassFilenameExtension, (__bridge CFStringRef)[path pathExtension], NULL);    CFStringRef MIMEType = UTTypeCopyPreferredTagWithClass (UTI, kUTTagClassMIMEType);
    CFRelease(UTI);
    if (!MIMEType) {
        return @"application/octet-stream";
    }
    return (__bridge NSString *)(MIMEType);
}
```
***
>NSURLSession实现文件上传

1. 实现文件上传的方法
```
  /*
第一个参数：请求对象
第二个参数：请求体（要上传的文件数据）
block回调：
NSData:响应体
NSURLResponse：响应头
NSError：请求的错误信息
*/
NSURLSessionUploadTask *uploadTask =  [session uploadTaskWithRequest:request fromData:data completionHandler:^(NSData * __nullable data, NSURLResponse * __nullable response, NSError * __nullable error)
```
2. 设置代理，在代理方法中监听文件上传进度
```
/*
调用该方法上传文件数据
如果文件数据很大，那么该方法会被调用多次
参数说明：
    totalBytesSent：已经上传的文件数据的大小
    totalBytesExpectedToSend：文件的总大小
*/
-(void)URLSession:(nonnull NSURLSession *)session task:(nonnull NSURLSessionTask *)task didSendBodyData:(int64_t)bytesSent totalBytesSent:(int64_t)totalBytesSent totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend
{
    NSLog(@"%.2f",1.0 * totalBytesSent/totalBytesExpectedToSend);
}
```
3. 关于NSURLSessionConfiguration相关
  - 作用：可以统一配置NSURLSession,如请求超时等
  - 创建的方式和使用
```
//创建配置的三种方式
+(NSURLSessionConfiguration *)defaultSessionConfiguration;
+(NSURLSessionConfiguration *)ephemeralSessionConfiguration;
+(NSURLSessionConfiguration *)backgroundSessionConfigurationWithIdentifier:(NSString *)identifier NS_AVAILABLE(10_10, 8_0);
//统一配置NSURLSession
-(NSURLSession *)session
{
    if (_session == nil) {
        //创建NSURLSessionConfiguration
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        //设置请求超时为10秒钟
        config.timeoutIntervalForRequest = 10;
        //在蜂窝网络情况下是否继续请求（上传或下载）
        config.allowsCellularAccess = NO;
        _session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:[NSOperationQueue mainQueue]];
    }
    return _session;
}
```

