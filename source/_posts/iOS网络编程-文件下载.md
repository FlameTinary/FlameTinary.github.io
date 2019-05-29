---
title: iOS网络编程--文件下载
date: 2016-04-12 16:50:37
tags:
- iOS
categories:
- iOS
---
>使用NSURLConnection实现下载

<!--more-->

### 1. 小文件下载
  1. 第一种方式（NSData）
```
//使用NSDta直接加载网络上的url资源（不考虑线程） 
-(void)dataDownload { 
//1.确定资源路径 
NSURL *url = [NSURL URLWithString:@"http://tupian.qqjay.com/u/2013/1127/19_222949_14.jpg"]; 
//2.根据URL加载对应的资源 
NSData *data = [NSData dataWithContentsOfURL:url]; 
//3.转换并显示数据 
UIImage *image = [UIImage imageWithData:data]; 
self.imageView.image = image; 
}
```
  2. 第二种方式（NSURLConnection-sendAsync）
```
//使用NSURLConnection发送异步请求下载文件资源 
-(void)connectDownload { 
//1.确定请求路径 
NSURL *url = [NSURL URLWithString:@"http://tupian.qqjay.com/u/2013/1127/19_222949_14.jpg"]; 
//2.创建请求对象 
NSURLRequest *request = [NSURLRequest requestWithURL:url]; 
//3.使用NSURLConnection发送一个异步请求 
[NSURLConnection sendAsynchronousRequest:request queue:[NSOperationQueue mainQueue] completionHandler:^(NSURLResponse * _Nullable response, NSData * _Nullable data, NSError * _Nullable connectionError) { 
//4.拿到并处理数据 
UIImage *image = [UIImage imageWithData:data]; 
self.imageView.image = image; 
}]; 
}
```
  3. 第三种方式（NSURLConnection-delegate）
```
//使用NSURLConnection设置代理发送异步请求的方式下载文件 
-(void)connectionDelegateDownload { 
//1.确定请求路径 
NSURL *url = [NSURL URLWithString:@"下载文件的URL地址"]; 
//2.创建请求对象 
NSURLRequest *request = [NSURLRequest requestWithURL:url]; 
//3.使用NSURLConnection设置代理并发送异步请求 
[NSURLConnection connectionWithRequest:request delegate:self]; 
}
pragma mark--NSURLConnectionDataDelegate
//当接收到服务器响应的时候调用，该方法只会调用一次
-(void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response
{
    //创建一个容器，用来接收服务器返回的数据
    self.fileData = [NSMutableData data];
    //获得当前要下载文件的总大小（通过响应头得到）
    NSHTTPURLResponse *res = (NSHTTPURLResponse *)response;
    self.totalLength = res.expectedContentLength;
    NSLog(@"%zd",self.totalLength);
    //拿到服务器端推荐的文件名称
    self.fileName = res.suggestedFilename;
}
//当接收到服务器返回的数据时会调用
//该方法可能会被调用多次
-(void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data
{
//    NSLog(@"%s",__func__);
    //拼接每次下载的数据
    [self.fileData appendData:data];
    //计算当前下载进度并刷新UI显示
    self.currentLength = self.fileData.length;
    NSLog(@"%f",1.0* self.currentLength/self.totalLength);
    self.progressView.progress = 1.0* self.currentLength/self.totalLength;
}
//当网络请求结束之后调用
-(void)connectionDidFinishLoading:(NSURLConnection *)connection
{
    //文件下载完毕把接受到的文件数据写入到沙盒中保存
    //1.确定要保存文件的全路径
    //caches文件夹路径
    NSString *caches = [NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) lastObject];
    NSString *fullPath = [caches stringByAppendingPathComponent:self.fileName];
    //2.写数据到文件中
    [self.fileData writeToFile:fullPath atomically:YES];
    NSLog(@"%@",fullPath);
}
//当请求失败的时候调用该方法
-(void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error
{
    NSLog(@"%s",__func__);
}
```

### 2. 大文件下载
  1. 实现思路
边接收数据边写文件以解决内存越来越大的问题
  2. 核心代码
```
//当接收到服务器响应的时候调用，该方法只会调用一次 
-(void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response { 
//0.获得当前要下载文件的总大小（通过响应头得到） 
NSHTTPURLResponse *res = (NSHTTPURLResponse *)response; 
self.totalLength = res.expectedContentLength; 
NSLog(@"%zd",self.totalLength); 
//创建一个新的文件，用来当接收到服务器返回数据的时候往该文件中写入数据 //1.获取文件管理者 
NSFileManager *manager = [NSFileManager defaultManager]; 
//2.拼接文件的全路径 
//caches文件夹路径 
NSString *caches = [NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) lastObject]; 
NSString *fullPath = [caches stringByAppendingPathComponent:res.suggestedFilename]; 
self.fullPath = fullPath; 
//3.创建一个空的文件 
[manager createFileAtPath:fullPath contents:nil attributes:nil]; 
} 
//当接收到服务器返回的数据时会调用 
//该方法可能会被调用多次 
-(void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data { 
//1.创建一个用来向文件中写数据的文件句柄 
//注意当下载完成之后，该文件句柄需要关闭，调用closeFile方法 
NSFileHandle *handle = [NSFileHandle fileHandleForWritingAtPath:self.fullPath]; 
//2.设置写数据的位置(追加) 
[handle seekToEndOfFile]; 
//3.写数据 [handle writeData:data]; 
//4.计算当前文件的下载进度 
self.currentLength += data.length; 
NSLog(@"%f",1.0* self.currentLength/self.totalLength); 
self.progressView.progress = 1.0* self.currentLength/self.totalLength;
}
```

### 3. 大文件断点续传
  1. 实现思路
在下载文件的时候不再是整块的从头开始下载，而是看当前文件已经下载到哪个地方，然后从该地方接着往后面下载。可以通过在请求对象中设置请求头实现。
  2. 解决方案（设置请求头）
```
//2.创建请求对象 
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url]; 
//2.1 设置下载文件的某一部分 
// 只要设置HTTP请求头的Range属性, 就可以实现从指定位置开始下载 
/* 
表示头500个字节：Range: bytes=0-499 
表示第二个500字节：Range: bytes=500-999 
表示最后500个字节：Range: bytes=-500 
表示500字节以后的范围：Range: bytes=500- 
*/ 
NSString *range = [NSString stringWithFormat:@"bytes=%zd-",self.currentLength]; 
[request setValue:range forHTTPHeaderField:@"Range"];
```
  3. 注意点（下载进度并判断是否需要重新创建文件）
```
//获得当前要下载文件的总大小（通过响应头得到） 
NSHTTPURLResponse *res = (NSHTTPURLResponse *)response; 
//注意点：res.expectedContentLength获得是本次请求要下载的文件的大小（并非是完整的文件的大小） 
//因此：文件的总大小 == 本次要下载的文件大小+已经下载的文件的大小 
self.totalLength = res.expectedContentLength + self.currentLength; 
NSLog(@"----------------------------%zd",self.totalLength); 
//0 判断当前是否已经下载过，如果当前文件已经存在，那么直接返回
 if (self.currentLength >0) { return; }
```
  4. 输出流
    1. 使用输出流也可以实现和NSFileHandle相同的功能

    2. 如何使用
```
//1.创建一个数据输出流
/*
第一个参数：二进制的流数据要写入到哪里
第二个参数：采用什么样的方式写入流数据，如果YES则表示追加，如果是NO则表示覆盖
*/
NSOutputStream *stream = [NSOutputStream outputStreamToFileAtPath:fullPath append:YES];
//只要调用了该方法就会往文件中写数据
//如果文件不存在，那么会自动的创建一个
[stream open];
self.stream = stream;
//2.当接收到数据的时候写数据
//使用输出流写数据
/*
第一个参数：要写入的二进制数据
第二个参数：要写入的数据的大小
*/
[self.stream write:data.bytes maxLength:data.length];
//3.当文件下载完毕的时候关闭输出流
//关闭输出流
[self.stream close];
self.stream = nil;
```
  5. 使用多线程下载文件的思路
- 01 开启多条线程，每条线程都只下载文件的一部分（通过设置请求头中的Range来实现）
 - 02 创建一个和需要下载文件大小一致的文件，判断当前是那个线程，根据当前的线程来判断下载的数据应该写入到文件中的哪个位置。（假设开5条线程来下载10M的文件，那么线程1下载0-2M，线程2下载2-4M一次类推，当接收到服务器返回的数据之后应该先判断当前线程是哪个线程，假如当前线程是线程2，那么在写数据的时候就从文件的2M位置开始写入）
-  03 代码相关：使用NSFileHandle这个类的seekToFileOfSet方法，来向文件中特定的位置写入数据。
 - 04 技术相关
      a.每个线程通过设置请求头下载文件中的某一个部分
      b.通过NSFileHandle向文件中的指定位置写数据
***
>使用NSURLSession实现下载

### 1. NSURLSession下载文件-代理
1. 创建NSURLSession对象，设置代理（默认配置）
```
//1.创建NSURLSession,并设置代理 
/* 
第一个参数：session对象的全局配置设置，一般使用默认配置就可以 
第二个参数：谁成为session对象的代理 
第三个参数：代理方法在哪个队列中执行（在哪个线程中调用）,如果是主队列那么在主线程中执行，如果是非主队列，那么在子线程中执行 
*/ 
NSURLSession *session = [NSURLSession sessionWithConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration] delegate:self delegateQueue:[NSOperationQueue mainQueue]];
```
2. 根据Session对象创建一个NSURLSessionDataTask任务（post和get选择）
```
//创建task
NSURL *url = [NSURL URLWithString:@"http://tupian.qqjay.com/u/2013/1127/19_222949_14.jpg"];
//注意：如果要发送POST请求，那么请使用dataTaskWithRequest,设置一些请求头信息
NSURLSessionDataTask *dataTask = [session dataTaskWithURL:url];
```
3. 执行任务（其它方法，如暂停、取消等）
````
//启动task
//[dataTask resume];
//其它方法，如取消任务，暂停任务等
//[dataTask cancel];
//[dataTask suspend];
```
4. 遵守代理协议，实现代理方法（3个相关的代理方法）
```
/* 
1.当接收到服务器响应的时候调用 
session：发送请求的session对象 
dataTask：根据NSURLSession创建的task任务 
response:服务器响应信息（响应头） 
completionHandler：通过该block回调，告诉服务器端是否接收返回的数据 
*/
-(void)URLSession:(nonnull NSURLSession *)session dataTask:(nonnull NSURLSessionDataTask *)dataTask didReceiveResponse:(nonnull NSURLResponse *)response completionHandler:(nonnull void (^)(NSURLSessionResponseDisposition))completionHandler
/*
2.当接收到服务器返回的数据时调用 该方法可能会被调用多次 
*/
-(void)URLSession:(nonnull NSURLSession *)session dataTask:(nonnull NSURLSessionDataTask *)dataTask didReceiveData:(nonnull NSData *)data
/* 
3.当请求完成之后调用该方法 不论是请求成功还是请求失败都调用该方法，如果请求失败，那么error对象有值，否则那么error对象为空 
*/
-(void)URLSession:(nonnull NSURLSession *)session task:(nonnull NSURLSessionTask *)task didCompleteWithError:(nullable NSError *)error
```
5. 当接收到服务器响应的时候，告诉服务器接收数据（调用block）
```
//默认情况下，当接收到服务器响应之后，服务器认为客户端不需要接收数据，所以后面的代理方法不会调用 
//如果需要继续接收服务器返回的数据，那么需要调用block,并传入对应的策略 
/* 
NSURLSessionResponseCancel = 0, 取消任务 
NSURLSessionResponseAllow = 1, 接收任务 
NSURLSessionResponseBecomeDownload = 2, 转变成下载 
NSURLSessionResponseBecomeStream NS_ENUM_AVAILABLE(10_11, 9_0) = 3, 转变成流 
*/ 
completionHandler(NSURLSessionResponseAllow);
```

###2. NSURLSessionDownloadTask实现大文件下载
1. 使用NSURLSession和NSURLSessionDownload可以很方便的实现文件下载操作
```
/*
    第一个参数：要下载文件的url路径
    第二个参数：当接收完服务器返回的数据之后调用该block
    location:下载的文件的保存地址（默认是存储在沙盒中tmp文件夹下面，随时会被删除）
    response：服务器响应信息，响应头
    error：该请求的错误信息
    */
    //说明：downloadTaskWithURL方法已经实现了在下载文件数据的过程中边下载文件数据，边写入到沙盒文件的操作
    NSURLSessionDownloadTask *downloadTask = [session downloadTaskWithURL:url completionHandler:^(NSURL * __nullable location, NSURLResponse * __nullable response, NSError * __nullable error)
```
2. downloadTaskWithURL内部默认已经实现了变下载边写入操作，所以不用开发人员担心内存问题
3. 文件下载后默认保存在tmp文件目录，需要开发人员手动的剪切到合适的沙盒目录
4. 缺点：没有办法监控下载进度

###3. 使用NSURLSessionDownloadTask实现大文件下载-监听下载进度
1. 创建NSURLSession并设置代理，通过NSURLSessionDownloadTask并以代理的方式来完成大文件的下载
```
//1.创建NSULRSession,设置代理
self.session = [NSURLSession sessionWithConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration] delegate:self delegateQueue:[NSOperationQueue mainQueue]];
//2.创建task
NSURL *url = [NSURL URLWithString:@"URL字符串"];
self.downloadTask = [self.session downloadTaskWithURL:url];
//3.执行task
[self.downloadTask resume];
```
2. 常用代理方法的说明
```
/* 
1.当接收到下载数据的时候调用,可以在该方法中监听文件下载的进度 
该方法会被调用多次 
totalBytesWritten:已经写入到文件中的数据大小 
totalBytesExpectedToWrite:目前文件的总大小 
bytesWritten:本次下载的文件数据大小 
*/
-(void)URLSession:(nonnull NSURLSession *)session downloadTask:(nonnull NSURLSessionDownloadTask *)downloadTask didWriteData:(int64_t)bytesWritten totalBytesWritten:(int64_t)totalBytesWritten totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite
/* 
2.恢复下载的时候调用该方法 
fileOffset:恢复之后，要从文件的什么地方开发下载 
expectedTotalBytes：该文件数据的总大小 
*/
-(void)URLSession:(nonnull NSURLSession *)session downloadTask:(nonnull NSURLSessionDownloadTask *)downloadTask didResumeAtOffset:(int64_t)fileOffset expectedTotalBytes:(int64_t)expectedTotalBytes
/* 
3.下载完成之后调用该方法 
*/
-(void)URLSession:(nonnull NSURLSession *)session downloadTask:(nonnull NSURLSessionDownloadTask *)downloadTask didFinishDownloadingToURL:(nonnull NSURL *)location
/* 
4.请求完成之后调用 
如果请求失败，那么error有值 
*/
-(void)URLSession:(nonnull NSURLSession *)session task:(nonnull NSURLSessionTask *)task didCompleteWithError:(nullable NSError *)error
```
3. 实现断点下载相关代码
```
//如果任务，取消了那么以后就不能恢复了
// [self.downloadTask cancel];
//如果采取这种方式来取消任务，那么该方法会通过resumeData保存当前文件的下载信息
//只要有了这份信息，以后就可以通过这些信息来恢复下载
[self.downloadTask cancelByProducingResumeData:^(NSData * __nullable resumeData) { self.resumeData = resumeData;}];
//继续下载
//首先通过之前保存的resumeData信息，创建一个下载任务
self.downloadTask = [self.session downloadTaskWithResumeData:self.resumeData]; 
[self.downloadTask resume];
```
4. 计算当前下载进度
```
//获取文件下载进度
self.progress.progress = 1.0 * totalBytesWritten/totalBytesExpectedToWrite;
```
5. 局限性
  - 如果用户点击暂停之后退出程序，那么需要把恢复下载的数据写一份到沙盒，代码复杂度更高
  - 如果用户在下载中途未保存恢复下载数据即退出程序，则不具备可操作性

###4. 使用NSURLSessionDataTask实现大文件离线断点下载（完整）

1. 关于NSOutputStream的使用
```
//1. 创建一个输入流,数据追加到文件的屁股上
//把数据写入到指定的文件地址，如果当前文件不存在，则会自动创建
NSOutputStream *stream = [[NSOutputStream alloc]initWithURL:[NSURL fileURLWithPath:[self fullPath]] append:YES];
//2. 打开流
[stream open];
//3. 写入流数据
[stream write:data.bytes maxLength:data.length];
//4.当不需要的时候应该关闭流
[stream close];
```
2. 关于网络请求请求头的设置（可以设置请求下载文件的某一部分）
```
//1. 设置请求对象
//1.1 创建请求路径
NSURL *url = [NSURL URLWithString:@"URL字符串"];
//1.2 创建可变请求对象
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
//1.3 拿到当前文件的残留数据大小
self.currentContentLength = [self FileSize];
//1.4 告诉服务器从哪个地方开始下载文件数据
NSString *range = [NSString stringWithFormat:@"bytes=%zd-",self.currentContentLength];
NSLog(@"%@",range);
//1.5 设置请求头
[request setValue:range forHTTPHeaderField:@"Range"];
```
3. NSURLSession对象的释放
```
-(void)dealloc{ 
//在最后的时候应该把session释放，以免造成内存泄露 
// NSURLSession设置过代理后，需要在最后（比如控制器销毁的时候）调用session的invalidateAndCancel或者resetWithCompletionHandler，才不会有内存泄露 
// [self.session invalidateAndCancel];
 [self.session resetWithCompletionHandler:^{ NSLog(@"释放---");
 }];
}
```
4. 优化部分
  - 关于文件下载进度的实时更新
  - 方法的独立与抽取

