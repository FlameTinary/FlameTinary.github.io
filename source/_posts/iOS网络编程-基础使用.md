---
title: iOS网络编程--基础使用
date: 2016-04-12 15:11:23
tags:
- iOS
categories:
- iOS
---

>iOS中发送http请求的方案 

<!--more-->

- 苹果原生 
  - NSURLConnection 03年推出的古老技术
  - NSURLSession 13年推出iOS7之后，以取代NSURLConnection   
  - CFNetwork 底层技术、C语言的 
- 第三方框架
   - ASIHttpRequest 
   - AFNetworking 
   - MKNetworkKit

### NSURLConnection使用
- NSURLConnection同步get请求
  - 步骤
    1. 设置请求路径
    2. 创建请求对象（默认是GET请求，且已经默认包含了请求头）
    3. 使用NSURLConnection sendsync方法发送网络请求 
    4. 接收到服务器的响应后，解析响应体
  - 代码
```
//1.确定请求路径 
NSURL *url = [NSURL URLWithString:@"URLString(发送请求的地址路径)"]; 
//2.创建一个请求对象
NSURLRequest *request = [NSURLRequest requestWithURL:url];
//3.把请求发送给服务器
//sendSynchronousRequest 阻塞式的方法，会卡住线程
NSHTTPURLResponse *response = nil;NSError *error = nil;
/* 
第一个参数：请求对象 
第二个参数：响应头信息，当该方法执行完毕之后，该参数被赋值 
第三个参数：错误信息，如果请求失败，则error有值 
*/ 
//该方法是阻塞式的，会卡住线程
NSData *data = [NSURLConnection sendSynchronousRequest:request returningResponse:&response error:&error];
//4.解析服务器返回的数据
NSString *str = [[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding];
```
- NSURLConnection异步请求（get-sendAsync）
该方法不会卡住线程，网络请求任务是异步执行的
```
//1.确定请求路径 
NSURL *url = [NSURL URLWithString:@"URLString(发送请求的地址路径)"]; 
//2.创建一个请求对象 
NSURLRequest *request = [NSURLRequest requestWithURL:url]; 
//3.把请求发送给服务器,发送一个异步请求 
/* 
第一个参数：请求对象 
第二个参数：回调方法在哪个线程中执行，如果是主队列则block在主线程中执行，非主队列则在子线程中执行 
第三个参数：completionHandlerBlock块：接受到响应的时候执行该block中的代码 response：响应头信息 data：响应体 connectionError：错误信息，如果请求失败，那么该参数有值 
*/ 
[NSURLConnection sendAsynchronousRequest:request queue:[[NSOperationQueue alloc]init] completionHandler:^(NSURLResponse * __nullable response, NSData * __nullable data, NSError * __nullable connectionError) { 
      //4.解析服务器返回的数据 
      NSString *str = [[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding]; 
      //转换并打印响应头信息 
      NSHTTPURLResponse *r = (NSHTTPURLResponse *)response; 
      NSLog(@"--%zd---%@--",r.statusCode,r.allHeaderFields); 
}];
```
- NSURLConnection异步请求（get-代理）
  - 步骤
    1. 确定请求路径
    2. 创建请求对象
    3. 创建NSURLConnection对象并设置代理
    4. 遵守NSURLConnectionDataDelegate协议，并实现相应的代理方法
    5. 在代理方法中监听网络请求的响应
  - 设置代理的几种方式
```
/* 
设置代理的第一种方式：自动发送网络请求 
[[NSURLConnection alloc]initWithRequest:request delegate:self]; 
*/
/* 
设置代理的第二种方式： 
第一个参数：请求对象 
第二个参数：谁成为NSURLConnetion对象的代理
第三个参数：是否马上发送网络请求，如果该值为YES则立刻发送，如果为NO则不会发送网路请求 
NSURLConnection *conn = [[NSURLConnection alloc]initWithRequest:request delegate:self startImmediately:NO]; 
//调用该方法控制网络请求的发送 
//注意该方法内部会自动的把connect添加到当前线程的RunLoop中在默认模式下执行
[conn start]; 
*/
//设置代理的第三种方式：使用类方法设置代理，会自动发送网络请求
NSURLConnection *conn = [NSURLConnection connectionWithRequest:request delegate:self];
//取消网络请求
//[conn cancel];
```
  - 相关代理方法
```
/* 
1.当接收到服务器响应的时候调用 
第一个参数connection：监听的是哪个NSURLConnection对象 
第二个参数response：接收到的服务器返回的响应头信息 
*/ 
-(void)connection:(nonnull NSURLConnection *)connection didReceiveResponse:(nonnull NSURLResponse *)response 
/* 
2.当接收到数据的时候调用，该方法会被调用多次 
第一个参数connection：监听的是哪个NSURLConnection对象 
第二个参数data：本次接收到的服务端返回的二进制数据（可能是片段） 
*/ 
-(void)connection:(nonnull NSURLConnection *)connection didReceiveData:(nonnull NSData *)data 
/* 
3.当服务端返回的数据接收完毕之后会调用 通常在该方法中解析服务器返回的数据 */
 -(void)connectionDidFinishLoading:(nonnull NSURLConnection *)connection 
/*
4.当请求错误的时候调用（比如请求超时） 
第一个参数connection：NSURLConnection对象 
第二个参数：网络请求的错误信息，如果请求失败，则error有值 
*/ 
-(void)connection:(nonnull NSURLConnection *)connection didFailWithError:(nonnull NSError *)error
```
  - 如何控制代理方法在哪个线程调用*****
```
//说明：默认情况下，代理方法会在主线程中进行调用（为了方便开发者拿到数据后处理一些刷新UI的操作不需要考虑到线程间通信） 
//设置代理方法的执行队列 
[connect setDelegateQueue:[[NSOperationQueue alloc]init]];
```
  - 开子线程发送网络请求的注意点，适用于自动发送网络请求模式
```
//使用GCD开启一个子线程来发送网络请求 
dispatch_async(dispatch_get_global_queue(0, 0), ^{ 
//使用非自动发送网络请求模式,发送请求OK 
/* 
//创建NSURLConnection对象，设置代理，暂不发送 
NSURLConnection *connect = [[NSURLConnection alloc]initWithRequest:request delegate:self startImmediately:NO]; 
//设置代理方法的执行队列 [connect setDelegateQueue:[[NSOperationQueue alloc]init]]; 
//调用start发送网络请求 
[connect start]; 
*/ 
//使用自动发送网络请求模式，发送请求失败（需要改造代码） 
//WHY? 
/*
01 网络请求发送和数据接收是否成功，和一些因素相关，比如客户端的网速、服务器端的查询速度等等。 
02 而在子线程中创建的NSURLConnection对象是一个临时变量，当请求发送完成之后就被释放了，所以这个时候它的代理方法不会调用用。 
03 为什么使用非自动发送网络请求模式是OK的。 因为在该模式中，调用了start来开始发送网络请求，该方法内部会自动将当前的connect作为一个Source添加到当前线程所在的Runloop中,如果当前线程是子线程（即当前线程的runloop并未创建），那么该方法内部会默认先创建当前线程的Runloop,设置在runloop的默认模式下运行。 此时runloop会对这个Connect对象进行强引用，保证了代理方法被调用的前提 
*/ 
NSURLConnection *connect = [[NSURLConnection alloc]initWithRequest:request delegate:self]; 
[connect setDelegateQueue:[[NSOperationQueue alloc]init]]; 
//创建当前线程的runloop，并开启runloop 
[[NSRunLoop currentRunLoop] run]; 
});
```
  - 其他知识
```
01 关于消息弹窗第三方框架的使用 
SVProgressHUD 
02 字符串截取相关方法 
-(NSRange)rangeOfString:(NSString *)searchString; 
-(NSString *)substringWithRange:(NSRange)range;
```
- NSURLConnection发送POST请求
  - 发送POST请求步骤
    1. 确定URL路径
    2. 创建请求对象（可变对象）
    3. 修改请求对象的方法为POST，设置请求体（Data）
    4. 发送一个异步请求
    5. 补充：设置请求超时，处理错误信息，设置请求头（如获取客户端的版本等等,请求头是可设置可不设置的）
  - 代码
```
//1.确定请求路径 
NSURL *url = [NSURL URLWithString:@"请求路径字符串"];
//2.创建请求对象
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
//2.1更改请求方法
request.HTTPMethod = @"POST";
//2.2设置请求体
request.HTTPBody = [@"username=111&pwd=222" dataUsingEncoding:NSUTF8StringEncoding];
//2.3请求超时
request.timeoutInterval = 5;
//2.4设置请求头
[request setValue:@"ios 9.0" forHTTPHeaderField:@"User-Agent"];
//3.发送请求
[NSURLConnection sendAsynchronousRequest:request queue:[NSOperationQueue mainQueue] completionHandler:^(NSURLResponse * __nullable response, NSData * __nullable data, NSError * __nullable connectionError) 
{ 
          //4.解析服务器返回的数据 
          if (connectionError) { 
              NSLog(@"--请求失败-"); 
          }else { 
              NSLog(@"%@",[[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding]);
          }
}];
```

### NSURLSession使用
- 使用步骤
  - 使用NSURLSession创建task，然后执行task
- 关于task
  - NSURLSessionTask是一个抽象类，本身不能使用，只能使用它的子类
  - NSURLSessionDataTask\NSURLSessionUploadTask\NSURLSessionDownloadTask
- 发送get请求
```
//1.创建NSURLSession对象（可以获取单例对象） 
NSURLSession *session = [NSURLSession sharedSession]; 
//2.根据NSURLSession对象创建一个Task 
NSURL *url = [NSURL URLWithString:@"URL字符串"]; 
NSURLRequest *request = [NSURLRequest requestWithURL:url]; 
//方法参数说明 
/* 
注意：该block是在子线程中调用的，如果拿到数据之后要做一些UI刷新操作，那么需要回到主线程刷新 
第一个参数：需要发送的请求对象 
block:当请求结束拿到服务器响应的数据时调用block 
block-NSData:该请求的响应体 
block-NSURLResponse:存放本次请求的响应信息，响应头，真实类型为NSHTTPURLResponse 
block-NSErroe:请求错误信息 
*/ 
NSURLSessionDataTask * dataTask = [session dataTaskWithRequest:request completionHandler:^(NSData * __nullable data, NSURLResponse * __nullable response, NSError * __nullable error) { 
//拿到响应头信息 
NSHTTPURLResponse *res = (NSHTTPURLResponse *)response; 
//4.解析拿到的响应数据 
NSLog(@"%@\n%@",[[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding],res.allHeaderFields); 
}]; 
//3.执行Task 
//注意：刚创建出来的task默认是挂起状态的，需要调用该方法来启动任务（执行任务） 
[dataTask resume];
```
- 发送get请求的第二种方式
```
//注意：该方法内部默认会把URL对象包装成一个NSURLRequest对象（默认是GET请求） 
//方法参数说明 
/* 
//第一个参数：发送请求的URL地址 
//block:当请求结束拿到服务器响应的数据时调用block 
//block-NSData:该请求的响应体 
//block-NSURLResponse:存放本次请求的响应信息，响应头，真实类型为NSHTTPURLResponse 
//block-NSErroe:请求错误信息 
*/ 
-(nullable NSURLSessionDataTask *)dataTaskWithURL:(NSURL *)url completionHandler:(void (^)(NSData * __nullable data, NSURLResponse * __nullable response, NSError * __nullable error))completionHandler;
```
- 发送post请求
```
//1.创建NSURLSession对象（可以获取单例对象） 
NSURLSession *session = [NSURLSession sharedSession]; 
//2.根据NSURLSession对象创建一个Task 
NSURL *url = [NSURL URLWithString:@"URL字符串"]; 
//创建一个请求对象，并这是请求方法为POST，把参数放在请求体中传递 NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url]; 
request.HTTPMethod = @"POST"; 
request.HTTPBody = [@"username=111&pwd=222&type=JSON" dataUsingEncoding:NSUTF8StringEncoding]; 
NSURLSessionDataTask *dataTask = [session dataTaskWithRequest:request completionHandler:^(NSData * __nullable data, NSURLResponse * __nullable response, NSError * __nullable error) { 
//拿到响应头信息 
NSHTTPURLResponse *res = (NSHTTPURLResponse *)response; 
//解析拿到的响应数据 
NSLog(@"%@\n%@",[[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding],res.allHeaderFields); 
}]; 
//3.执行Task 
//注意：刚创建出来的task默认是挂起状态的，需要调用该方法来启动任务（执行任务） 
[dataTask resume];
```
