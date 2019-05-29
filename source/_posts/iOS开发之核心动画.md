---
title: iOS开发之核心动画
date: 2016-04-27 12:31:08
categories:
- iOS
tags:
- iOS
photo: https://ask.qcloudimg.com/http-save/yehe-2193280/2dlyp18yyw.png?imageView2/2/w/1620
---

> Core Animation,中文翻译为核心动画,它是一组非常强大的动画处理API,使用它能做出非常绚丽的动画效果,而且往往事半功倍.也就是说,使用少量的代码就可以实现非常强大的功能.

<!--more-->

- Core Animation的动画执行过程都是在后台操作的,不会阻塞主线程
- Core Animation是直接作用在CALayer上的,并非UIView

*所以在了解Core Animation之前,先要了解什么是CALayer*

## CALayer

> 在iOS中,你能看得见摸得着的东西基本上都是UIView,比如:按钮,文本标签等等;其实UIView之所以能显示在屏幕上,完全是因为它内部的一个图层

- 在创建UIView对象时,UIView内部会自动创建一个图层(即CALayer对象),通过UIView的layer属性可以访问这个层

```objc
@property(nonatomic,readonly,retain) CALayer *layer; 
``` 

- 当UIView需要显示到屏幕上时，会调用drawRect:方法进行绘图，并且会将所有内容绘制在自己的图层上，绘图完毕后，系统会将图层拷贝到屏幕上，于是就完成了UIView的显示;UIView本身不具备显示的功能，是它内部的层才有显示功能
- CALayer的一些属性

```objc
//宽度和高度
@property CGRect bounds;
//位置(默认指中点，具体由anchorPoint决定)
@property CGPoint position;
//锚点(x,y的范围都是0-1)，决定了position的含义
@property CGPoint anchorPoint;
//背景颜色(CGColorRef类型)
@property CGColorRef backgroundColor;
//形变属性
@property CATransform3D transform;
//边框颜色(CGColorRef类型)
@property CGColorRef borderColor;
//边框宽度
@property CGFloat borderWidth;
//圆角半径
@property CGColorRef borderColor;
//内容(比如设置为图片CGImageRef)
@property(retain) id contents;
```

> UIView和CALayer的选择

通过CALayer，就能做出跟UIImageView一样的界面效果
既然CALayer和UIView都能实现相同的显示效果，那究竟该选择谁好呢？
其实，对比CALayer，UIView多了一个事件处理的功能。也就是说，CALayer不能处理用户的触摸事件，而UIView可以.
所以，如果显示出来的东西需要跟用户进行交互的话，用UIView；如果不需要跟用户进行交互，用UIView或者CALayer都可以.
当然，CALayer的性能会高一些，因为它少了事件处理的功能，更加轻量级.

> position和anchorPoint

```objc
@property CGPoint position;
```

1. 用来设置CALayer在父层中的位置
- 以父层的左上角为原点(0, 0)

```objc
@property CGPoint anchorPoint;
```

1. 称为“定位点”、“锚点”
- 决定着CALayer身上的哪个点会在position属性所指的位置
- 以自己的左上角为原点(0, 0)
- 它的x、y取值范围都是0~1，默认值为（0.5, 0.5）

> 隐式动画

- 每一个UIView内部都默认关联着一个CALayer，我们可用称这个Layer为Root Layer（根层）
- 所有的非Root Layer，也就是手动创建的CALayer对象，都存在着隐式动画
- 什么是隐式动画？
	- 当对非Root Layer的部分属性进行修改时，默认会自动产生一些动画效果
	- 而这些属性称为Animatable Properties(可动画属性)
	- 列举几个常见的Animatable Properties：
		1. bounds：用于设置CALayer的宽度和高度。修改这个属性会产生缩放动画
		- backgroundColor：用于设置CALayer的背景色。修改这个属性会产生背景色的渐变动画
		- position：用于设置CALayer的位置。修改这个属性会产生平移动画
	- 可以通过动画事务(CATransaction)关闭默认的隐式动画效果
	
```objc
[CATransaction begin];
[CATransaction setDisableActions:YES];
self.myview.layer.position = CGPointMake(10, 10);
[CATransaction commit];
```

## Core Animation

![Core Animation结构图](http://7xtfqf.com2.z0.glb.clouddn.com/Snip20160427_1.png)

- 核心动画中所有类都遵守CAMediaTiming
- CAAnaimation是个抽象类，不具备动画效果，必须用它的子类才有动画效果
- CAAnimationGroup和CATransition才有动画效果
	- CAAnimationGroup是个动画组，可以同时进行缩放，旋转。
	- CATransition是转场动画，界面之间跳转都可以用转场动画。
- CAPropertyAnimation也是个抽象类，本身不具备动画效果，只有子类才有
	- CABasicAnimation和CAKeyframeAnimation
		1. CABasicAnimation基本动画，做一些简单效果
		- CAKeyframeAnimation帧动画，做一些连续的流畅的动画

> 开发步骤

_如果不是xcode5之后的版本，使用它需要先添QuartzCore.framework和引入对应的框架<QuartzCore/QuartzCore.h>_

1. 首先得有CALayer
- 初始化一个CAAnimation对象，并设置一些动画相关属性
- 通过调用CALayer的addAnimation:forKey:方法，增加CAAnimation对象到CALayer中，这样就能开始执行动画了
- 通过调用CALayer的removeAnimationForKey:方法可以停止CALayer中的动画

### CAAnimation

> 简介

- 是所有动画对象的父类，负责控制动画的持续时间和速度，是个抽象类，不能直接使用，应该使用它具体的子类
- 属性说明：(红色代表来自CAMediaTiming协议的属性)
	- duration：动画的持续时间
	- repeatCount：重复次数，无限循环可以设置HUGE_VALF或者MAXFLOAT
	- repeatDuration：重复时间
	- removedOnCompletion：默认为YES，代表动画执行完毕后就从图层上移除，图形会恢复到动画执行前的状态。如果想让图层保持显示动画执行后的状态，那就设置为NO，不过还要设置fillMode为kCAFillModeForwards
	- fillMode：决定当前对象在非active时间段的行为。比如动画开始之前或者动画结束之后
	- beginTime：可以用来设置动画延迟执行时间，若想延迟2s，就设置为CACurrentMediaTime()+2，CACurrentMediaTime()为图层的当前时间
	- timingFunction：速度控制函数，控制动画运行的节奏
	- delegate：动画代理

```objc
#pragma mark 暂停CALayer的动画
-(void)pauseLayer:(CALayer*)layer
{
    CFTimeInterval pausedTime = [layer convertTime:CACurrentMediaTime() fromLayer:nil];

    // 让CALayer的时间停止走动
      layer.speed = 0.0;
    // 让CALayer的时间停留在pausedTime这个时刻
    layer.timeOffset = pausedTime;
}

```
```objc

#pragma mark 恢复CALayer的动画
-(void)resumeLayer:(CALayer*)layer
{
    CFTimeInterval pausedTime = layer.timeOffset;
    // 1. 让CALayer的时间继续行走
      layer.speed = 1.0;
    // 2. 取消上次记录的停留时刻
      layer.timeOffset = 0.0;
    // 3. 取消上次设置的时间
      layer.beginTime = 0.0;    
    // 4. 计算暂停的时间(这里也可以用CACurrentMediaTime()-pausedTime)
    CFTimeInterval timeSincePause = [layer convertTime:CACurrentMediaTime() fromLayer:nil] - pausedTime;
    // 5. 设置相对于父坐标系的开始时间(往后退timeSincePause)
      layer.beginTime = timeSincePause;
}
```

> 动画填充模式
 
- fillMode属性值（要想fillMode有效，最好设置removedOnCompletion = NO）
- kCAFillModeRemoved 这个是默认值，也就是说当动画开始前和动画结束后，动画对layer都没有影响，动画结束后，layer会恢复到之前的状态
- kCAFillModeForwards 当动画结束后，layer会一直保持着动画最后的状态 
- kCAFillModeBackwards 在动画开始前，只需要将动画加入了一个layer，layer便立即进入动画的初始状态并等待动画开始。
- kCAFillModeBoth 这个其实就是上面两个的合成.动画加入后开始之前，layer便处于动画初始状态，动画结束后layer保持动画最后的状态

> 速度控制函数

- 速度控制函数(CAMediaTimingFunction) 
	1. kCAMediaTimingFunctionLinear（线性）：匀速，给你一个相对静态的感觉
	- kCAMediaTimingFunctionEaseIn（渐进）：动画缓慢进入，然后加速离开
	- kCAMediaTimingFunctionEaseOut（渐出）：动画全速进入，然后减速的到达目的地
	- kCAMediaTimingFunctionEaseInEaseOut（渐进渐出）：动画缓慢的进入，中间加速，然后减速的到达目的地。这个是默认的动画行为。

> 动画代理方法

```objc
@interface NSObject (CAAnimationDelegate)

/* Called when the animation begins its active duration. */

- (void)animationDidStart:(CAAnimation *)anim;

/* Called when the animation either completes its active duration or
 * is removed from the object it is attached to (i.e. the layer). 'flag'
 * is true if the animation reached the end of its active duration
 * without being removed. */

- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag;

@end
```

#### 1. CAPropertyAnimation

- 是CAAnimation的子类，也是个抽象类，要想创建动画对象，应该使用它的两个子类：
	1. CABasicAnimation
	- CAKeyframeAnimation

- 属性说明：
	- keyPath：通过指定CALayer的一个属性名称为keyPath（NSString类型），并且对CALayer的这个属性的值进行修改，达到相应的动画效果。比如，指定@“position”为keyPath，就修改CALayer的position属性的值，以达到平移的动画效果

> CABasicAnimation

- 基本动画,是CAPropertyAnimation的子类

- 属性说明:
	- fromValue：keyPath相应属性的初始值
	- toValue：keyPath相应属性的结束值

- 动画过程说明：
	- 随着动画的进行，在长度为duration的持续时间内，keyPath相应属性的值从fromValue渐渐地变为toValue
	- keyPath内容是CALayer的可动画Animatable属性
	- 如果fillMode=kCAFillModeForwards同时removedOnComletion=NO，那么在动画执行完毕后，图层会保持显示动画执行后的状态。但在实质上，图层的属性值还是动画执行前的初始值，并没有真正被改变。

> 关键帧动画

- 关键帧动画，也是CAPropertyAnimation的子类，与CABasicAnimation的区别是：
	- CABasicAnimation只能从一个数值（fromValue）变到另一个数值（toValue），而CAKeyframeAnimation会使用一个NSArray保存这些数值

- 属性说明：
	- values：上述的NSArray对象。里面的元素称为“关键帧”(keyframe)。动画对象会在指定的时间（duration）内，依次显示values数组中的每一个关键帧
	- path：可以设置一个CGPathRef、CGMutablePathRef，让图层按照路径轨迹移动。path只对CALayer的anchorPoint和position起作用。如果设置了path，那么values将被忽略
	- keyTimes：可以为对应的关键帧指定对应的时间点，其取值范围为0到1.0，keyTimes中的每一个时间值都对应values中的每一帧。如果没有设置keyTimes，各个关键帧的时间是平分的

_CABasicAnimation可看做是只有2个关键帧的CAKeyframeAnimation_

#### 2. CAAnimationGroup

- 动画组，是CAAnimation的子类，可以保存一组动画对象，将CAAnimationGroup对象加入层后，组中所有动画对象可以同时并发运行

- 属性说明：
	- animations：用来保存一组动画对象的NSArray
	- 默认情况下，一组动画对象是同时运行的，也可以通过设置动画对象的beginTime属性来更改动画的开始时间

#### 3. CATransition

- CATransition是CAAnimation的子类，用于做转场动画，能够为层提供移出屏幕和移入屏幕的动画效果。iOS比Mac OS X的转场动画效果少一点
- UINavigationController就是通过CATransition实现了将控制器的视图推入屏幕的动画效果
- 动画属性:
	- type：动画过渡类型
	- subtype：动画过渡方向
	- startProgress：动画起点(在整体动画的百分比)
	- endProgress：动画终点(在整体动画的百分比)
	- 转场动画过渡效果

|类型字符串 | 效果说明 | 关键字 | 方向 |
|:----------:|:---------:|:-------:|:-----:|
|fade|交叉淡化过渡|YES||
|push|新视图把旧视图推出去|YES||
|moveIn|新视图移到旧视图上面|YES||
|reveal|将旧视图移开,显示下面的新视图|YES||
|cube|立方体翻滚效果|||
|oglFlip|上下左右翻转效果|||
|suckEffect|收缩效果，如一块布被抽走||NO|
|rippleEffect|水滴效果|| NO |
|pageCurl|向上翻页效果|||
|pageUnCurl|向下翻页效果|||
|facameraIrisHollowOpende|相机镜头打开效果|| NO |
|cameraIrisHollowClose|相机镜头关闭效果|| NO |

> 使用UIView动画函数实现转场动画——单视图

```objc
+ (void)transitionWithView:(UIView *)view duration:(NSTimeInterval)duration options:(UIViewAnimationOptions)options animations:(void (^)(void))animations completion:(void (^)(BOOL finished))completion;
```
- 参数说明：
	- duration：动画的持续时间
	- view：需要进行转场动画的视图
	- options：转场动画的类型
	- animations：将改变视图属性的代码放在这个block中
	- completion：动画结束后，会自动调用这个block

> 使用UIView动画函数实现转场动画——双视图

```objc
+ (void)transitionFromView:(UIView *)fromView toView:(UIView *)toView duration:(NSTimeInterval)duration options:(UIViewAnimationOptions)options completion:(void (^)(BOOL finished))completion;
```
- 参数说明：
	- duration：动画的持续时间
	- options：转场动画的类型
	- animations：将改变视图属性的代码放在这个block中
	- completion：动画结束后，会自动调用这个block