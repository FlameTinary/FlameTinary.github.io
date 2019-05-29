---
title: iOS开发之触摸事件
date: 2016-04-27 23:32:30
categories:
- iOS
tags:
- iOS
---

> 本文介绍了iOS中使用频率较高的触摸事件，并阐述了事件产生和传递的过程，以及响应者链的事件传递过程

<!--more-->

# 触摸事件


## 简介

- 在用户使用app过程中，会产生各种各样的事件
- iOS中的事件可以分为3大类型
	- 触摸事件
	- 加速计事件
	- 远程控制事件 

## 响应者对象

- 在iOS中不是任何对象都能处理事件，只有继承了UIResponder的对象才能接收并处理事件。我们称之为“响应者对象”
- UIApplication、UIViewController、UIView都继承自UIResponder，因此它们都是响应者对象，都能够接收并处理事件
- UIResponder
	1. 触摸事件
```objc
-(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event;
-(void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event;
-(void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event;
-(void)touchesCancelled:(NSSet *)touches withEvent (UIEvent *)event;
```
	2. 加速计事件
```objc
-(void)motionBegan:(UIEventSubtype)motion withEvent:(UIEvent *)event;
-(void)motionEnded:(UIEventSubtype)motion withEvent:(UIEvent *)event;
-(void)motionCancelled:(UIEventSubtype)motion withEvent:(UIEvent *)event;
```

	3. 远程控制事件

```objc
-(void)remoteControlReceivedWithEvent:(UIEvent *)event;
```

## UIView的触摸事件处理

- UIView是UIResponder的子类，可以覆盖下列4个方法处理不同的触摸事件:
	1. 一根或者多根手指开始触摸view，系统会自动调用view的下面方法
```objc
-(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
```
	2. 一根或者多根手指在view上移动，系统会自动调用view的下面方法（随着手指的移动，会持续调用该方法）
```objc
-(void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event
```
	3. 一根或者多根手指离开view，系统会自动调用view的下面方法
```objc
-(void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event
```
	4. 触摸结束前，某个系统事件(例如电话呼入)会打断触摸过程，系统会自动调用view的下面方法
```objc
-(void)touchesCancelled:(NSSet *)touches withEvent:(UIEvent *)event
```
    - *提示：touches中存放的都是UITouch对象*






## UITouch





### 对象和作用

- 当用户用一根手指触摸屏幕时，会创建一个与手指相关联的UITouch对象
- 一根手指对应一个UITouch对象
- `UITouch的作用`：保存着跟手指相关的信息，比如触摸的位置、时间、阶段
- 当手指移动时，系统会更新同一个UITouch对象，使之能够一直保存该手指在的触摸位置
- 当手指离开屏幕时，系统会销毁相应的UITouch对象
`*提示：iPhone开发中，要避免使用双击事件！*`

### UITouch的属性

- 触摸产生时所处的窗口
```objc
@property(nonatomic,readonly,retain) UIWindow    *window;
```
- 触摸产生时所处的视图
```objc
@property(nonatomic,readonly,retain) UIView      *view;
```
- 短时间内点按屏幕的次数，可以根据tapCount判断单击、双击或更多的点击
```objc
@property(nonatomic,readonly) NSUInteger          tapCount;
```
- 记录了触摸事件产生或变化时的时间，单位是秒
```objc
@property(nonatomic,readonly) NSTimeInterval      timestamp;
```
- 当前触摸事件所处的状态
```objc
@property(nonatomic,readonly) UITouchPhase        phase;
```

### UITouch的方法

```objc
-(CGPoint)locationInView:(UIView *)view;
```

- 返回值表示触摸在view上的位置
- 这里返回的位置是针对view的坐标系的（以view的左上角为原点(0, 0)）
- 调用时传入的view参数为nil的话，返回的是触摸点在UIWindow的位置

```objc
-(CGPoint)previousLocationInView:(UIView *)view;
```

- 该方法记录了前一个触摸点的位置


## UIEvent
- 每产生一个事件，就会产生一个UIEvent对象
- UIEvent：称为事件对象，记录事件产生的时刻和类型
- 常见属性
	- 事件类型
```objc
@property(nonatomic,readonly) UIEventType     type;
@property(nonatomic,readonly) UIEventSubtype  subtype;
```
	- 事件产生的时间
```objc
@property(nonatomic,readonly) NSTimeInterval  timestamp;
```
- UIEvent还提供了相应的方法可以获得在某个view上面的触摸对象（UITouch)


## touches和event参数

- 一次完整的触摸过程，会经历三个状态
	1. 触摸开始
```objc
-(void)touchBegan:(NSSet *)touches withEvent:(UIEvent *)event
```
	2. 触摸移动
```objc
-(void)touchMoved:(NSSet *)touches withEvent:(UIEvent *)event
```
	3. 触摸结束
```objc
-(void)touchEnded:(NSSet *)touches withEvent:(UIEvent *)event
```
	4. 触摸取消
```objc
-(void)touchCancelled:(NSSet *)touches withEvent:(UIEvent *)event
```

- 4个触摸事件处理方法中，都有NSSet * touches和UIEvent * event两个参数
	- 一次完整的触摸过程中，只会产生一个事件对象，4个触摸方法都是同一个event参数
	- 如果两根手指同时触摸一个view，那么view只会调用一次touchesBegan:withEvent:方法，touches参数中装着2个UITouch对象
	- 如果这两根手指一前一后分开触摸同一个view，那么view会分别调用2次touchesBegan:withEvent:方法，并且每次调用时的touches参数中只包含一个UITouch对象
	- 根据touches中UITouch的个数可以判断出是单点触摸还是多点触摸

# 事件产生和传递
---

- 发生触摸事件后，系统会将该事件加入到一个由UIApplication管理的事件队列中
- UIApplication会从事件队列中取出最前面的事件，并将事件分发下去以便处理，通常，先发送事件给应用程序的主窗口（keyWindow）
- 主窗口会在视图层次结构中找到一个最合适的视图来处理触摸事件，这也是整个事件处理过程的第一步
- 找到合适的视图控件后，就会调用视图控件的touches方法来作具体的事件处理
	- touchesBegan…
	- touchesMoved…
	- touchedEnded…

- **如果父控件不能接收触摸事件，那么子控件就不可能接收到触摸事件**
- UIView不接收触摸事件的三种情况：
	1. `不接收用户交互` userInteractionEnabled = NO
	2. `隐藏` hidden = YES
	3. `透明` alpha = 0.0 ~ 0.01

	*提示：UIImageView的userInteractionEnabled默认就是NO，因此UIImageView以及它的子控件默认是不能接收触摸事件的*


# 响应者链条
---

- 响应者链条：是由多个响应者对象连接起来的链条
- 作用：能很清楚的看见每个响应者之间的联系，并且可以让一个事件多个对象处理。
- 响应者对象：能处理事件的对象



## 响应者链条示意图

![](http://upload-images.jianshu.io/upload_images/1879463-1e597f7a72595802.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 事件传递的完整过程

1. 先将事件对象由上往下传递(由父控件传递给子控件)，找到最合适的控件来处理这个事件。
2. 调用最合适控件的touches….方法
3. 如果调用了[super touches….];就会将事件顺着响应者链条往上传递，传递给上一个响应者
4. 接着就会调用上一个响应者的touches….方法

- **如何判断上一个响应者**
	1. 如果当前这个view是控制器的view,那么控制器就是上一个响应者
	2. 如果当前这个view不是控制器的view,那么父控件就是上一个响应者

## 响应者链的事件传递过程

1. 如果view的控制器存在，就传递给控制器；如果控制器不存在，则将其传递给它的父视图
2. 在视图层次结构的最顶级视图，如果也不能处理收到的事件或消息，则其将事件或消息传递给window对象进行处理
3. 如果window对象也不处理，则其将事件或消息传递给UIApplication对象
4. 如果UIApplication也不能处理该事件或消息，则将其丢弃

# 手势识别
---

>为了完成手势识别，必须借助于手势识别器----UIGestureRecognizer;
利用UIGestureRecognizer，能轻松识别用户在某个view上面做的一些常见手势

- UIGestureRecognizer是一个抽象类，定义了所有手势的基本行为，使用它的子类才能处理具体的手势
	- UITapGestureRecognizer(敲击)
	- UIPinchGestureRecognizer(捏合，用于缩放)
	- UIPanGestureRecognizer(拖拽)
	- UISwipeGestureRecognizer(轻扫)
	- UIRotationGestureRecognizer(旋转)
	- UILongPressGestureRecognizer(长按)

- 每一个手势识别器的用法都差不多，比如UITapGestureRecognizer的使用步骤如下：
	- 创建手势识别器对象
```objc
UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] init];
```
	- 设置手势识别器对象的具体属性
```objc
// 连续敲击2次
tap.numberOfTapsRequired = 2;
// 需要2根手指一起敲击
tap.numberOfTouchesRequired = 2;
```
	- 添加手势识别器到对应的view上
```objc
[self.iconView addGestureRecognizer:tap];
```
	- 监听手势的触发
```objc
[tap addTarget:self action:@selector(tapIconView:)];
```

- 手势识别的状态
```objc
typedef NS_ENUM(NSInteger, UIGestureRecognizerState) {
    // 没有触摸事件发生，所有手势识别的默认状态
    UIGestureRecognizerStatePossible,
    // 一个手势已经开始但尚未改变或者完成时
    UIGestureRecognizerStateBegan,
    // 手势状态改变
    UIGestureRecognizerStateChanged,
    // 手势完成
    UIGestureRecognizerStateEnded,
    // 手势取消，恢复至Possible状态
    UIGestureRecognizerStateCancelled, 
    // 手势失败，恢复至Possible状态
    UIGestureRecognizerStateFailed,
    // 识别到手势识别
    UIGestureRecognizerStateRecognized = UIGestureRecognizerStateEnded
};
```