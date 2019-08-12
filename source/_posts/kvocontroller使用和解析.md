---
title: kvocontroller使用和解析
date: 2019-08-12 15:27:59
tags:
- hide
categories:
- hide
---
[facebook/KVOController](https://github.com/facebook/KVOController)是对Cocoa中KVO的封装。它提供简单、现代的API，并且是线程安全的。优点如下：
* 通知是通过block、action或者NSKeyValueObserving回调（即-observeValueForKeyPath:ofObject:change:context）来实现；
* 不会出现移除observer的异常；
* 在controller销毁时隐式地移除observer；
* 保证了线程安全。
简单地说，KVOController让我们更优雅、简单、安全地使用KVO。

<!--more-->

## 使用KVOController

```objc
// 导入框架
#import <KVOController/KVOController.h>

@interface ViewController ()
// 设置示例模型和示例按钮
@property (nonatomic, strong)TestModel *model;
@property (nonatomic, strong)UIButton *button;
@end

```

```objc
//设置KVO
FBKVOController *kvoController = [FBKVOController controllerWithObserver:self];
self.KVOController = kvoController;
//设置model
self.model = [TestModel new];
self.model.name = @"hah";
// 添加一个button
self.button = [UIButton buttonWithType:UIButtonTypeSystem];
self.button.frame = CGRectMake(100, 100, 100, 50);
[self.button setTitle:self.model.name forState:UIControlStateNormal];
[self.button addTarget:self action:@selector(buttonClick:) forControlEvents:UIControlEventTouchUpInside];
[self.view addSubview:self.button];
//添加监听
[self.KVOController observe:self.model
                    keyPath:FBKVOKeyPath(_model.name)
                    options:NSKeyValueObservingOptionNew
                      block:^(id  _Nullable observer, id  _Nonnull object, NSDictionary<NSString *,id> * _Nonnull change)
 {
     NSLog(@"%@", change);
     NSString *title = (NSString *)change[@"new"];
     [self.button setTitle:title forState:UIControlStateNormal];
 }];
```

```objc
// button 点击方法
- (void)buttonClick:(UIButton *)sender {
    self.model.name = [self.model.name isEqual: @"hah"] ? @"gogo" : @"hah";
}
```

## 源码分析

* [KVOController详解](https://www.jianshu.com/p/8deccb9c8398)
* [如何优雅地使用 KVO](https://draveness.me/kvocontroller)