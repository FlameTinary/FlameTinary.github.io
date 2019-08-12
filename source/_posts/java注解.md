---
title: java注解
date: 2019-08-12 15:29:19
tags:
- hide
catergories:
-hide
---
## 什么是注解
注解，顾名思义，注解,就是对某一事物进行添加注释说明，会存放一些信息，这些信息可能对以后某个时段来说是很有用处的。
<!--more-->
## java中的注解
Java注解又叫java标注，java提供了一套机制，使得我们可以对方法、类、参数、包、域以及变量等添加标准(即附上某些信息)。且在以后某个时段通过反射将标注的信息提取出来以供使用。

## 自定义注解
### 为什么自定义注解
Java从1.5版本以后默认内置三个标注：
- `@Override`:只能用在方法之上的，用来告诉别人这一个方法是改写父类的。
- `@Deprecated`:建议别人不要使用旧的API的时候用的,编译的时候会用产生警告信息,可以设定在程序里的所有的元素上。
- `@SuppressWarnings`:这一个类型可以来暂时把一些警告信息消息关闭。
但是，仅仅这三个标注是不能满足我们开发时一些需求的。所以java允许我们自定义注解来使用。
### 怎么自定义注解
自定义步骤大致分为两步：
1. 通过`@interface`关键字声明注解名称，以及注解的成员属性或者叫做注解的参数。
2. 使用java内置的四个元注解对这个自定义标注的功能和范围进行一些限制。

#### 元注解
元注解，就是定义注解的注解，也就是说这些元注解是的作用就是专门用来约束其它注解的注解。请区别上面那三个注解，他们也是通过元注解定义而来的。
元注解有哪些呢，主要有四个`@Target`,`@Retention`,`@Documented`,`@Inherited`

- `@Target` 表示该注解用于什么地方，可能的 ElemenetType 参数包括： 
    - `ElemenetType.CONSTRUCTOR` 构造器声明 
    - `ElemenetType.FIELD` 域声明（包括 enum 实例） 
    - `ElemenetType.LOCAL_VARIABLE` 局部变量声明 
    - `ElemenetType.METHOD` 方法声明 
    - `ElemenetType.PACKAGE` 包声明 
    - `ElemenetType.PARAMETER` 参数声明 
    - `ElemenetType.TYPE` 类，接口（包括注解类型）或enum声明 
- `@Retention` 表示在什么级别保存该注解信息。可选的 RetentionPolicy 参数包括： 
    - `RetentionPolicy.SOURCE` 注解将被编译器丢弃 
    - `RetentionPolicy.CLASS` 注解在class文件中可用，但会被VM丢弃 
    - `RetentionPolicy.RUNTIME` VM将在运行期也保留注释，因此可以通过反射机制读取注解的信息。 
- `@Documented` 将此注解包含在 javadoc 中 
- `@Inherited` 允许子类继承父类中的注解 
#### 自定义

```java
import java.lang.annotation.*;

@Target(ElementType.TYPE) // 这个标注用于类
@Retention(RetentionPolicy.RUNTIME) //这个标注会一直保留到运行时
@Documented // 将此注解包含在javadoc中

public @interface TYDescription {

    String Value();

}
```

```java
import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented

public @interface TYName {
    String originate();
    String community();
}
```
#### 使用注解

```java
@TYName(originate = "Sheldon", community = "sheldon.com")
@TYDescription(Value = "desc String!!!")
```
#### 注意
## 参考