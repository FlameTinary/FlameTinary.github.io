---
title: iOS之Runtime
date: 2016-04-10 18:14:53
tags:
- iOS
categories:
- iOS
photo: https://diycode.b0.upaiyun.com/photo/2019/49061ea7bcf3ad6c3bd0d991068dce3a.png
---
> RunTime简称运行时。它是一套纯C语言API,属于一个C语言库,包含了很多底层的C语言API。OC就是运行时机制，我们平时写的OC代码在运行的时候都会转换成runtime的C语言代码。

<!--more-->

对于OC的函数，属于动态调用过程，在编译的时候并不能决定真正调用哪个函数，只有在真正运行的时候才会根据函数的名称找到对应的函数来调用。事实证明：在编译阶段，OC可以调用任何函数，即使这个函数并未实现，只要声明过就不会报错。在编译阶段，C语言调用未实现的函数就会报错。

## Runtime有什么作用

在没有了解runtime之前，你会觉得它没有什么用，但当你深入的了解了OC的运行机制后，你就会觉的其实它的作用非常之大。

#### 1. 发送消息
* 方法调用的本质就是让对象发送消息
* objc_msgSend,只有对象才能发送消息，因此以objc开头
* 使用消息机制的前提，必须导入#import <objc/message.h>
* 消息机制原理:对象根据方法编号SEL去映射表查找对应的方法实现
* 消息机制代码说明：

 ```objc
// 创建person对象 
Person *p = [[Person alloc] init]; 
// 调用对象方法 
[p eat]; 
// 本质：让对象发送消息 
objc_msgSend(p, @selector(eat)); 
// 调用类方法的方式：两种 
// 第一种通过类名调用 
[Person eat]; 
// 第二种通过类对象调用 
[[Person class] eat]; 
// 用类名调用类方法，底层会自动把类名转换成类对象调用 
// 本质：让类对象发送消息 
objc_msgSend([Person class], @selector(eat));
```

#### 2.交换方法
如果系统自带的方法功能不够，可以给系统自带的方法扩展一些功能，并且保持原有的功能，有两种实现方式：
1. 继承系统的类，重写方法
2. 使用runtime,交换方法

使用Runtime实现方法交换：

* 在分类中交换方法地址，并实现方法

```objc
@implementation UIImage (Image)
//加载分类到内存中的时候调用
+(void)load{
    
    //获取imageNamed方法地址
    Method imageName = class_getClassMethod(self, @selector(imageNamed:));
    //获取imageWithName方法地址
    Method imageWithName = class_getClassMethod(self, @selector(imageWithName:));
    //交换两个方法的地址
    method_exchangeImplementations(imageName, imageWithName);   
}
// 不能在分类中重写系统方法imageNamed，因为会把系统的功能给覆盖掉，而且分类中不能调用super.
+(instancetype)imageWithName:(NSString *)name{
    // 这里调用imageWithName，相当于调用imageName
    UIImage *image = [self imageWithName:name];
    
    if (image == nil) {
        NSLog(@"图片加载为空");
    }
    return image;
}
@end
``` 

  * 调用系统方法，其实调用的是自己创建的方法

```objc
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{

    //当图片不存在的时候会打印"图片加载为空"
    UIImage * image = [UIImage imageNamed:@"222"];
    self.pageView.image = image;
}
```

#### 3. 动态添加方法

如果一个类方法非常多，加载类到内存的时候也比较耗费资源，需要给每个方法生成映射表，可以使用动态给某个类添加方法解决。

调用未实现的方法

```objc
Person * p = [[Person alloc] init];
// 默认Person没有实现eat方法，通过performSelector方法调用会报错，但是动态添加方法就不会报错
[p performSelector:@selector(eat)];
```

动态的添加方法

```objc
@implementation Person
//默认方法都有两个隐式参数
void eat (id self, SEL sel)
{
    NSLog(@"%@, %@", self, NSStringFromSelector(sel));
}
//当一个对象调用未实现的方法时，系统会调用这个方法处理，并且会把对应的方法列表传过来
+ (BOOL)resolveInstanceMethod:(SEL)sel{
    if (sel == @selector(eat)) {
        //动态添加eat方法
        // 第一个参数：给哪个类添加方法
        // 第二个参数：添加方法的方法编号
        // 第三个参数：添加方法的函数实现（函数地址）
        // 第四个参数：函数的类型，(返回值+参数类型) v:void @:对象->self :表示SEL->_cmd
        class_addMethod(self, @selector(eat), eat, "v@:");
    }
    NSLog(@"当一个对象调用未实现的方法时，会调用这个函数");
    return [super resolveInstanceMethod:sel];
}
@end
```

#### 4. 给分类添加属性
给一个类声明属性，其实本质就是给这个类添加关联，并不是直接把这个值的内存空间添加到类存空间。所以即便分类不能添加属性，但是可以使用runtime实现这一功能。

在分类中声明属性

```objc
@interface NSObject (AddName)
// @property在分类中，只会生成get,set方法的声明，不会生成下划线成员属性，和get,set方法的实现
// 因此需要自己实现
@property (nonatomic, strong) NSString *name;
@end
```

在分类中实现属性的get，set方法

```objc
//定义一个关联的key
static const char *key = "name";
@implementation NSObject (AddName)
//get方法
-(NSString *)name
{
    //根据关联的key获取关联的值
    return objc_getAssociatedObject(self, key);
}
//set方法
-(void)setName:(NSString *)name
{
    // 第一个参数：给哪个对象添加关联
    // 第二个参数：关联的key，通过这个key获取
    // 第三个参数：关联的value
    // 第四个参数:关联的策略
    objc_setAssociatedObject(self, key, name, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
@end
```

为属性赋值

```objc
// 给系统NSObject类动态添加属性name
NSObject *objc = [[NSObject alloc] init];
objc.name = @"flame";
NSLog(@"%@",objc.name);
```

#### 5. 使用runtime字典转模型

提供一个分类，专门根据字典生成模型对应的属性字符串。

```objc
// 自动打印属性字符串
+ (void)resolveDict:(NSDictionary *)dict{
    // 拼接属性字符串代码
    NSMutableString *strM = [NSMutableString string];
    // 1.遍历字典，把字典中的所有key取出来，生成对应的属性代码
    [dict enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull key, id  _Nonnull obj, BOOL * _Nonnull stop) {
         NSString *type;
        if ([obj isKindOfClass:NSClassFromString(@"__NSCFString")]) {
            type = @"NSString";
        }else if ([obj isKindOfClass:NSClassFromString(@"__NSCFArray")]){
            type = @"NSArray";
        }else if ([obj isKindOfClass:NSClassFromString(@"__NSCFNumber")]){
            type = @"int";
        }else if ([obj isKindOfClass:NSClassFromString(@"__NSCFDictionary")]){
            type = @"NSDictionary";
        }else if ([obj isKindOfClass:NSClassFromString(@"__NSCFBoolean")]){
            type = @"Bool";
        }
        // 属性字符串
        NSString *str;
        if ([type containsString:@"NS"]) {
            str = [NSString stringWithFormat:@"@property (nonatomic, strong) %@ *%@;",type,key];
        }else{
            str = [NSString stringWithFormat:@"@property (nonatomic, assign) %@ %@;",type,key];
        }
        // 每生成属性字符串，就自动换行。
        [strM appendFormat:@"\n%@\n",str];
    }];
    // 把拼接好的字符串打印出来，就好了。
    NSLog(@"%@",strM);
}
```

使用runtime字典转模型

思路：利用运行时，遍历模型中所有属性，根据模型的属性名，去字典中查找key，取出对应的值，给模型的属性赋值。

```objc
#import <objc/message.h>
@implementation NSObject (Model)
+ (instancetype)modelWithDict:(NSDictionary *)dict
{
    // 思路：遍历模型中所有属性-》使用运行
    // 0.创建对应的对象
    id objc = [[self alloc] init];
    // 1.利用runtime给对象中的成员属性赋值  
    // class_copyIvarList:获取类中的所有成员属性
    // Ivar：成员属性的意思
    // 第一个参数：表示获取哪个类中的成员属性
    // 第二个参数：表示这个类有多少成员属性，传入一个Int变量地址，会自动给这个变量赋值
    // 返回值Ivar *：指的是一个ivar数组，会把所有成员属性放在一个数组中，通过返回的数组就能全部获取到。
    unsigned int count;
    // 获取类中的所有成员属性
    Ivar *ivarList = class_copyIvarList(self, &count);
    for (int i = 0; i < count; i++) {
        // 根据角标，从数组取出对应的成员属性
        Ivar ivar = ivarList[i];
        // 获取成员属性名
        NSString *name = [NSString stringWithUTF8String:ivar_getName(ivar)];
        // 处理成员属性名->字典中的key
        // 从第一个角标开始截取
        NSString *key = [name substringFromIndex:1];
        // 根据成员属性名去字典中查找对应的value
        id value = dict[key];
        // 二级转换:如果字典中还有字典，也需要把对应的字典转换成模型
        // 判断下value是否是字典
        if ([value isKindOfClass:[NSDictionary class]]) {
            // 字典转模型
            // 获取模型的类对象，调用modelWithDict
            // 模型的类名已知，就是成员属性的类型
            // 获取成员属性类型
           NSString *type = [NSString stringWithUTF8String:ivar_getTypeEncoding(ivar)];
          // 生成的是这种@"@\"User\"" 类型 -》 @"User"  在OC字符串中 \" -> "，\是转义的意思，不占用字符
            // 裁剪类型字符串
            NSRange range = [type rangeOfString:@"\""];
           type = [type substringFromIndex:range.location + range.length];
            range = [type rangeOfString:@"\""];
            // 裁剪到哪个角标，不包括当前角标
          type = [type substringToIndex:range.location];
            // 根据字符串类名生成类对象
            Class modelClass = NSClassFromString(type);
            if (modelClass) { // 有对应的模型才需要转
                // 把字典转模型
                value  =  [modelClass modelWithDict:value];
            }
        }
        // 三级转换：NSArray中也是字典，把数组中的字典转换成模型.
        // 判断值是否是数组
        if ([value isKindOfClass:[NSArray class]]) {
            // 判断对应类有没有实现字典数组转模型数组的协议
            if ([self respondsToSelector:@selector(arrayContainModelClass)]) {  
                // 转换成id类型，就能调用任何对象的方法
                id idSelf = self;
                // 获取数组中字典对应的模型
                NSString *type =  [idSelf arrayContainModelClass][key];
                // 生成模型
               Class classModel = NSClassFromString(type);
                NSMutableArray *arrM = [NSMutableArray array];
                // 遍历字典数组，生成模型数组
                for (NSDictionary *dict in value) {
                    // 字典转模型
                  id model =  [classModel modelWithDict:dict];
                    [arrM addObject:model];
                }
                // 把模型数组赋值给value
                value = arrM;
            }
        }
        if (value) { // 有值，才需要给模型的属性赋值
            // 利用KVC给模型中的属性赋值
            [objc setValue:value forKey:key];
        }
    }
    return objc;
}
@end
```

