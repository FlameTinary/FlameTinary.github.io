---
title: iOS之Runloop
date: 2016-04-11 16:53:42
tags:
- iOS
categories:
- iOS
photo: https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcS-YwraZ80CtwfN7VcyKk-GPTERGXdVD8FHb0xnaDqVUaKvZuxvyQ
---
Runloop，正如其名，loop表示某种循环，和run放在一起就表示一直在运行着的循环。如果没有Runloop,那么程序一启动就会退出，什么事情都做不了；如果有了Runloop，那么相当于在内部有一个死循环，能够保证程序的持续运行。

<!--more-->

## Runloop的作用

保持程序的持续运行(ios程序为什么能一直活着不会死)

处理app中的各种事件（比如触摸事件、定时器事件【NSTimer】、selector事件【选择器·performSelector···】）

节省CPU资源，提高程序性能，有事情就做事情，没事情就休息

main函数中的Runloop ：

1. 在UIApplication函数内部就启动了一个Runloop 该函数返回一个int类型的值 
2. 这个默认启动的Runloop是跟主线程相关联的

## Runloop对象

在iOS开发中有两套api来访问Runloop 
  * foundation框架【NSRunloop】
  * core foundation框架【CFRunloopRef】

NSRunLoop和CFRunLoopRef都代表着RunLoop对象,它们是等价的，可以互相转换 

NSRunLoop是基于CFRunLoopRef的一层OC包装，所以要了解RunLoop内部结构，需要多研究CFRunLoopRef层面的API（Core Foundation层面）

## Runloop与线程

Runloop和线程的关系：一个Runloop对应着一条唯一的线程。为了让子线程不死，可以给这条子线程开启一个Runloop 

Runloop的创建：主线程Runloop已经创建好了，子线程的runloop需要手动创建 

Runloop的生命周期：在第一次获取时创建，在线程结束时销毁

## 如何获取Runloop对象

获得当前Runloop对象 

```objc
//01 NSRunloop 
NSRunLoop * runloop1 = [NSRunLoop currentRunLoop]; 
//02 CFRunLoopRef 
CFRunLoopRef runloop2 = CFRunLoopGetCurrent(); 
```

拿到当前应用程序的主Runloop（主线程对应的Runloop） 

```objc
//01 NSRunloop 
NSRunLoop * runloop1 = [NSRunLoop mainRunLoop]; 
//02 CFRunLoopRef
 CFRunLoopRef runloop2 = CFRunLoopGetMain(); 
```

**注意点**：开一个子线程创建runloop,不是通过alloc init方法创建，而是直接通过调用currentRunLoop方法来创建，它本身是一个懒加载的。 

在子线程中，如果不主动获取Runloop的话，那么子线程内部是不会创建Runloop的。可以下载CFRunloopRef的源码，搜索_CFRunloopGet0,查看代码。

Runloop对象是利用字典来进行存储，而且key是对应的线程Value为该线程对应的Runloop。

## Runloop相关类

Runloop运行图

![Runloop运行图](http://upload-images.jianshu.io/upload_images/1879463-dbb689a6a20153f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

五个相关的类：
1. CFRunloopRef
2. CFRunloopModeRef【Runloop的运行模式】
3. CFRunloopSourceRef【Runloop要处理的事件源】
4. CFRunloopTimerRef【Timer事件】
5. CFRunloopObserverRef【Runloop的观察者（监听者）】

## Runloop和相关类之间的关系图

![Runloop和相关类之间的关系图](http://upload-images.jianshu.io/upload_images/1879463-42827b80dec71711.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Runloop要想跑起来，它的内部必须要有一个mode,这个mode里面必须有source\observer\timer，至少要有其中的一个。

### CFRunloopModeRef
1. CFRunloopModeRef代表着Runloop的运行模式 
2. 一个Runloop中可以有多个mode,一个mode里面又可以有多个source\observer\timer等等 
3. 每次runloop启动的时候，只能指定一个mode,这个mode被称为该Runloop的当前mode 
4. 如果需要切换mode,只能先退出当前Runloop,再重新指定一个mode进入 
5. 这样做主要是为了分割不同组的定时器等，让他们相互之间不受影响 
6. 系统默认注册了5个mode 
    - kCFRunLoopDefaultMode：App的默认Mode，通常主线程是在这个Mode下运行 
    - UITrackingRunLoopMode：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响 
    - UIInitializationRunLoopMode: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用 
    - GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode，通常用不到 
    - kCFRunLoopCommonModes: 这是一个占位用的Mode，不是一种真正的Mode

### CFRunloopTimerRef
1. runloop一启动就会选中一种模式，当选中了一种模式之后其它的模式就都不管用。一个mode里面可以添加多个NSTimer,也就是说以后当创建NSTimer的时候，可以指定它是在什么模式下运行的。
2. 它是基于时间的触发器，说直白点那就是时间到了我就触发一个事件，触发一个操作。基本上说的就是NSTimer
3. 相关代码：

```objc
//NSTimer 调用了scheduledTimer方法，那么会自动添加到当前的runloop里面去，而且runloop的运行模式kCFRunLoopDefaultMode
NSTimer *timer = [NSTimer scheduledTimerWithTimeInterval:2.0 target:self selector:@selector(run) userInfo:nil repeats:YES];
//更改模式
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```
```objc
NSTimer *timer = [NSTimer timerWithTimeInterval:2.0 target:self selector:@selector(run) userInfo:nil repeats:YES];
//定时器添加到UITrackingRunLoopMode模式，一旦runloop切换模式，那么定时器就不工作
[[NSRunLoop currentRunLoop] addTimer:timer forMode:UITrackingRunLoopMode];
//定时器添加到NSDefaultRunLoopMode模式，一旦runloop切换模式，那么定时器就不工作
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
//占位模式：common modes标记
//被标记为common modes的模式 kCFRunLoopDefaultMode  UITrackingRunLoopMode
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

## GCD中的定时器

```objc
//0.创建一个队列
dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
//1.创建一个GCD的定时器
/*
第一个参数：说明这是一个定时器
第四个参数：GCD的回调任务添加到那个队列中执行，如果是主队列则在主线程执行
*/
dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
//2.设置定时器的开始时间，间隔时间以及精准度
//设置开始时间，三秒钟之后调用
dispatch_time_t start = dispatch_time(DISPATCH_TIME_NOW,3.0 *NSEC_PER_SEC);
//设置定时器工作的间隔时间
uint64_t intevel = 1.0 * NSEC_PER_SEC;
/*
第一个参数：要给哪个定时器设置
第二个参数：定时器的开始时间DISPATCH_TIME_NOW表示从当前开始
第三个参数：定时器调用方法的间隔时间
第四个参数：定时器的精准度，如果传0则表示采用最精准的方式计算，如果传大于0的数值，则表示该定时切换i可以接收该值范围内的误差，通常传0
该参数的意义：可以适当的提高程序的性能
注意点：GCD定时器中的时间以纳秒为单位（面试）
*/
dispatch_source_set_timer(timer, start, intevel, 0 * NSEC_PER_SEC);
//3.设置定时器开启后回调的方法
/*
第一个参数：要给哪个定时器设置
第二个参数：回调block
*/
dispatch_source_set_event_handler(timer, ^{
NSLog(@"------%@",[NSThread currentThread]);
});
//4.执行定时器
dispatch_resume(timer);
//注意：dispatch_source_t本质上是OC类，在这里是个局部变量，需要强引用
self.timer = timer;
```

### CFRunloopSourceRef
1. 是事件源也就是输入源，有两种分类模式；
    - 一种是按照苹果官方文档进行划分的
    - 另一种是基于函数的调用栈来进行划分的（source0和source1）。
2. 具体的分类情况
    -  以前的分法
            Port-Based Sources
            Custom Input Sources
            Cocoa Perform Selector Sources
    - 现在的分法
            Source0：非基于Port的
            Source1：基于Port的
3. 可以通过打断点的方式查看一个方法的函数调用栈

### CFRunLoopObserverRef
  - CFRunLoopObserverRef是观察者，能够监听RunLoop的状态改变
  - 如何监听:

```objc
//创建一个runloop监听者
CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(CFAllocatorGetDefault(),kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
        NSLog(@"监听runloop状态改变---%zd",activity);
    });
    //为runloop添加一个监听者
    CFRunLoopAddObserver(CFRunLoopGetCurrent(), observer, kCFRunLoopDefaultMode);
    CFRelease(observer);
```
  - 监听的状态

```objc
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
        kCFRunLoopEntry = (1UL << 0),   //即将进入Runloop
        kCFRunLoopBeforeTimers = (1UL << 1),    //即将处理NSTimer
        kCFRunLoopBeforeSources = (1UL << 2),   //即将处理Sources
        kCFRunLoopBeforeWaiting = (1UL << 5),   //即将进入休眠
        kCFRunLoopAfterWaiting = (1UL << 6),    //刚从休眠中唤醒
        kCFRunLoopExit = (1UL << 7),            //即将退出runloop
        kCFRunLoopAllActivities = 0x0FFFFFFFU   //所有状态改变
};
```

## Runloop运行逻辑

每次运行runloop，你线程的runloop对象会自动处理之前未处理的消息，并通知相关的观察者，具体顺序如下：
1. 通知观察者runloop已经启动
2. 通知观察者任何即将要开始的定时器
3. 通知观察者任何即将启动的非基于端口的源
4. 启动任何准备好的非基于端口的源
5. 如果基于端口的源准备好并处于等待状态，立即启动；并进入步骤9
6. 通知观察者线程进入休眠
7. 将线程置于休眠知道任一下面的事件发生：
    - 某一事件到达基于端口的源
    - 定时器启动
    - runloop设置的时间已经超时
    - runloop被显式唤醒
8. 通知观察者线程将被唤醒
9. 处理未处理的事件
    - 如果用户定义的定时器启动，处理定时器事件并重启runloop，进入步骤2
    - 如果输入源启动，传递相应的消息
    - 如果runloop被显式唤醒而且时间还没超时，重启runloop，进入步骤2
10. 通知观察者runloop结束

![Runloop运行逻辑](http://upload-images.jianshu.io/upload_images/1879463-f1e203b5a8358729.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Runloop应用

- NSTimer
- ImageView显示
- PerformSelector
- 常驻线程
- 自动释放池

## Runloop参考资料

- 苹果官方文档 
https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html

- CFRunLoopRef开源代码下载地址： 
http://opensource.apple.com/source/CF/CF-1151.16/










