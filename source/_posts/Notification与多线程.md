---
title: Notification与多线程
date: 2019-08-12 15:26:06
tags:
- hide
categories:
- hide
---
原文：
> In a multithreaded application, notifications are always delivered in the thread in which the notification was posted, which may not be the same thread in which an observer registered itself.

翻译：
> 在多线程应用中，Notification在哪个线程中post，就在哪个线程中被转发，而不一定是在注册观察者的那个线程中。

<!--more-->

举个🌰：

```objc
// 主线程中add
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(observerSelect) name:@"qwer" object:nil];
NSLog(@"1.current thread -- %@", [NSThread currentThread]);

// 子线程中post
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3.0 * NSEC_PER_SEC)), dispatch_get_global_queue(0, 0), ^{
    NSLog(@"3.current thread -- %@", [NSThread currentThread]);
   [[NSNotificationCenter defaultCenter] postNotificationName:@"qwer" object:nil];
});
```

```objc
- (void)observerSelect {
    NSLog(@"2.current thread -- %@", [NSThread currentThread]);
}
```
打印结果：
```objc
2019-03-26 11:24:29.987543+0800 timerTest[46207:2785353] 1.current thread -- <NSThread: 0x600002814d00>{number = 1, name = main}
2019-03-26 11:24:33.270912+0800 timerTest[46207:2785409] 3.current thread -- <NSThread: 0x60000284e880>{number = 3, name = (null)}
2019-03-26 11:24:33.271095+0800 timerTest[46207:2785409] 2.current thread -- <NSThread: 0x60000284e880>{number = 3, name = (null)}
```

虽然我们在主线程中注册了通知者，但在全局队列中post，并不是在主线程中处理的，所以，我们在处理回调结果时，一定要注意线程问题。

那么，我们如何能在子线程中post，在主线程中进行处理呢？

看看官方文档：
> For example, if an object running in a background thread is listening for notifications from the user interface, such as a window closing, you would like to receive the notifications in the background thread instead of the main thread. In these cases, you must capture the notifications as they are delivered on the default thread and redirect them to the appropriate thread.

翻译：
> 例如，如果在后台线程中运行的对象正在侦听来自用户界面的通知（例如关闭窗口），则您希望在后台线程而不是主线程中接收通知。在这些情况下，您必须捕获在默认线程上传递的通知，并将它们重定向到相应的线程。


根据官方文档修改我们上面的例子：



```objc
@interface ViewController () <NSMachPortDelegate>
@property (nonatomic) NSMutableArray    *notifications;         // 通知队列
@property (nonatomic) NSThread          *notificationThread;    // 期望线程
@property (nonatomic) NSLock            *notificationLock;      // 用于对通知队列加锁的锁对象，避免线程冲突
@property (nonatomic) NSMachPort        *notificationPort;      // 用于向期望线程发送信号的通信端口
@end
```

```objc
// 初始化
self.notifications = [[NSMutableArray alloc] init];
self.notificationLock = [[NSLock alloc] init];
    
self.notificationThread = [NSThread currentThread];
self.notificationPort = [[NSMachPort alloc] init];
self.notificationPort.delegate = self;
    
// 往当前线程的run loop添加端口源
// 当Mach消息到达而接收线程的run loop没有运行时，则内核会保存这条消息，直到下一次进入run loop
[[NSRunLoop currentRunLoop] addPort:self.notificationPort
                            forMode:(__bridge NSString *)kCFRunLoopCommonModes];
    
    
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(observerSelectNotification:) name:@"qwer" object:nil];
NSLog(@"1.current thread -- %@", [NSThread currentThread]);
    
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3.0 * NSEC_PER_SEC)), dispatch_get_global_queue(0, 0), ^{
    NSLog(@"3.current thread -- %@", [NSThread currentThread]);
    [[NSNotificationCenter defaultCenter] postNotificationName:@"qwer" object:nil];
});
```

```objc
// Delegates are sent this if they respond, otherwise they
// are sent handlePortMessage:; argument is the raw Mach message
- (void)handleMachMessage:(void *)msg{
    [self.notificationLock lock];
    
    NSLog(@"5.current thread -- %@", [NSThread currentThread]);
    while ([self.notifications count]) {
        NSNotification *notification = [self.notifications objectAtIndex:0];
        [self.notifications removeObjectAtIndex:0];
        [self.notificationLock unlock];
        [self observerSelectNotification:notification];
        [self.notificationLock lock];
    };
    
    [self.notificationLock unlock];

}
```

```objc
- (void)observerSelectNotification:(NSNotification *)notification {
    
    if ([NSThread currentThread] != _notificationThread) {
        NSLog(@"4.current thread -- %@", [NSThread currentThread]);
        // Forward the notification to the correct thread.
        [self.notificationLock lock];
        [self.notifications addObject:notification];
        [self.notificationLock unlock];
        [self.notificationPort sendBeforeDate:[NSDate date]
                                   components:nil
                                         from:nil
                                     reserved:0];
    }
    else {
        NSLog(@"2.current thread -- %@", [NSThread currentThread]);
        NSLog(@"process notification");
    }
}
```

运行结果：
```objc
2019-03-26 11:58:30.826178+0800 timerTest[47621:2830678] 1.current thread -- <NSThread: 0x6000023c93c0>{number = 1, name = main}
2019-03-26 11:58:33.826488+0800 timerTest[47621:2830764] 3.current thread -- <NSThread: 0x600002388740>{number = 3, name = (null)}
2019-03-26 11:58:33.826702+0800 timerTest[47621:2830764] 4.current thread -- <NSThread: 0x600002388740>{number = 3, name = (null)}
2019-03-26 11:58:33.826899+0800 timerTest[47621:2830678] 5.current thread -- <NSThread: 0x6000023c93c0>{number = 1, name = main}
2019-03-26 11:58:33.827061+0800 timerTest[47621:2830678] 2.current thread -- <NSThread: 0x6000023c93c0>{number = 1, name = main}
2019-03-26 11:58:33.827151+0800 timerTest[47621:2830678] process notification
```

可以看到，我们在全局dispatch队列中抛出的Notification，如愿地在主线程中接收到了。

这种实现方式的具体解析及其局限性大家可以参考官方文档[Delivering Notifications To Particular Threads](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Notifications/Articles/Threading.html#//apple_ref/doc/uid/20001289-CEGJFDFG)，在此不多做解释。当然，更好的方法可能是我们自己去子类化一个NSNotificationCenter，或者单独写一个类来处理这种转发。

参考：
* [Notification Programming Topics](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Notifications/Articles/Notifications.html)
* [Threading Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Multithreading/ThreadSafetySummary/ThreadSafetySummary.html)
* [Notification与多线程](http://southpeak.github.io/2015/03/14/nsnotification-and-multithreading/)