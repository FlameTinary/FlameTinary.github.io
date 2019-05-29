---
title: iOS开发之混合开发简单介绍
date: 2016-04-18 18:39:40
categories:
- iOS
tags:
- iOS
---
>*2014年,历时8年的HTML标准制定完成.利用HTML5编写的UI界面能运行在所有拥有浏览器的平台之上,且开发迅速而简单.这样,混合开发使得iOS开发变得更简单,更快捷*

<!--more-->

### 1. Web开发新时代

- Web1.0
    主流技术:HTML+CSS
    
- Web2.0
    主流技术:Ajax(JavaScript/DOM/异步数据请求)
    
- Web3.0
    主流技术:HTML5+CSS3
    HTML5亮点: Canvas\HTML5音视频\Web存储 Geolocation \Workers多线程处理
    CSS3亮点: 设计动画\2D变形\N多新特性
    
> 网页的组成

- 一个有具体功能的完整的网页，一般由3部分组成
    - HTML
    网页的具体内容和结构
    
  - CSS
    网页的样式（美化网页最重要的一块）
    
  - JavaScript(掌握)
    网页的交互效果，比如对用户鼠标事件做出响应
    
- HTML\CSS\JavaScript学习资料：http://www.w3school.com.cn/

### 2. HTML
>什么是HTML

 - HTML的全称是HyperText Markup Language，超文本标记语言
 - 其实它就是文本，由浏览器负责将它解析成具体的网页内容

>HTML的组成

   - 跟XML类似，HTML由N个标签（节点、元素、标记）组成   

>HTML语法非常松散，目前的最新版是5.0，也就是HTML 5

  - 常见的HTML标签
    - 标题：h1、h2、h3、h4、h5....
    - 段落：p
    - 换行：br
    - 容器：div、span（用来容纳其他标签）
    - 表格：table、tr、td
    - 列表：ul、ol、li
    - 图片：img
    - 表单：input
    - 链接：a

- HTML标签类型
HTML有N多标签，根据显示的类型，主要可以分为3大类
  1. 块级标签
独占一行的标签
能随时设置宽度和高度（比如div、p、h1、h2、ul、li）

  2. 行内标签（内联标签）
多个行内标签能同时显示在一行
宽度和高度取决于内容的尺寸（比如span、a、label）

  3. 行内-块级标签（内联-块级标签）
多个行内-块级标签可以显示在同一行
能随时设置宽度和高度（比如input、button）
- CSS中有个display属性，能修改标签的显示类型
  - `none`：隐藏标签
  - `block`：让标签变为块级标签
  - `inline`：让标签变为行内标签
  - `inline-block`：让标签变为行内-块级标签（内联-块级标签）

>HTML5新增标签

- HTML5新增了27个标签元素,废弃了16个标签元素,主要包括结构性标签、级块性标签、行内语义性标签、交互性标签
  1. 结构性标签
    负责Web上下文结构的定义，确保HTML文档，包括：
         - article  文章主体内容(一篇博客、一篇论坛帖子、一段用户评论、插件)
         - header   标记头部区域内容
         - footer   标记脚部区域内容
         - section  区域章节表述
         - nav      菜单导航,链接导航
    
  2. 块级性标签
    完成Web页面区域的划分，确保内容的有效分隔，包括：
         - aside   注记,贴士,侧栏,摘要,插入的引用作为补充主体的内容
         - figure  对多个元素组合并展示的元素,常与figcaption联合使用
         - code    表示一段代码块
         - dialog  人与人之间对话,包含dt和dd两个组合元素（dt用于表示说话者、dd用于表示说话者的内容）
    
  3. 行内语义性标签
    完成Web页面具体内容的引用和表述，丰富展示内容，包括：
         - meter     特定范围内的数值，如工资、数量、百分比
         - time      时间值
         - progress  进度条，可用max、min、step进行控制，完成对进度的表示和监听
         - video     视频元素，用于视频播放，支持缓冲预载和多种视频媒体格式
         - audio     音频元素，用于音频播放，支持缓冲预载和多种音频媒体格式
    
  4. 交互性标签
    功能性内容的表达，有一定的内容和数据的关联，是各种事件的基础，包括：
         - details   表示一段具体的内容，默认不显示，通过某种方式（单击）与legend交互才会显示
         - datagrid  控制客户端数据与显示，可用于动态脚本及时更新
         - menu      用于交互菜单
         - command   用来处理命令按钮

### 3. CSS

>什么是CSS

- CSS的全称是Cascading Style Sheets，层叠样式表
- 它用来控制HTML标签的样式，在美化网页中起到非常重要的作用
- CSS的编写格式是键值对形式的，比如:
```
color: red;
background-color: blue;
font-size: 20px;
```    
  - 冒号:左边的是属性名，冒号:右边的属性值

>CSS的3种书写形式

- 行内样式：（内联样式）直接在标签的style属性中书写
```
    <body style="color: red;">
```
- 页内样式：在本网页的style标签中书写
```
    <style>
    body {
    color: red;
    }
    </style>
    ```
- 外部样式：在单独的CSS文件中书写，然后在网页中用link标签引用
```
    <link rel="stylesheet" href="index.css">
```

>CSS的两大重点

1. 属性
    通过属性的复杂叠加才能做出漂亮的网页
  - CSS有N多属性，根据继承性，主要可以分为2大类
    - 可继承属性
父标签的属性值会传递给子标签
一般是文字控制属性
```
所有标签可继承
visibility、cursor
内联标签可继承
letter-spacing、word-spacing、white-space、line-height、color、font、font-family、font-size、font-style、font-variant、font-weight、text-decoration、text-transform、direction
 块级标签可继承
text-indent、text-align
列表标签可继承
list-style、list-style-type、list-style-position、list-style-image
```

    - 不可继承属性
父标签的属性值不能传递给子标签
一般是区块控制属性
```
display、margin、border、padding、background
height、min-height、max-height、width、min-width、max-width
overflow、position、left、right、top、bottom、z-index
float、clear
table-layout、vertical-align
page-break-after、page-bread-before
unicode-bidi
```
    
2. 选择器
    通过选择器找到对应的标签设置样式*(具体选择器内容和教程请参考:http://www.w3school.com.cn/css/index.asp)*

> CSS布局

-  默认情况下，所有的网页标签都在标准流布局中,从上到下，从左到右
- 脱离标准流的方法有
  - float属性
    - float属性的常用取值有:
        `left`：脱离标准流，浮动在父标签的最左边
`right`：脱离标准流，浮动在父标签的最右边
  - position属性

    ![position属性](http://upload-images.jianshu.io/upload_images/1879463-98d3215b81c3df23.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>CSS3新增特性

- **RGBA透明度**
   RGB(红色R+绿色G+蓝色B),RGBA则在其基础上增加了Alpha通道，可用于设置透明值

- **块阴影与圆角阴影**
   box-shadow  text-shadow

- **圆角**
    border-radius

- **边框图片**
     border-image

- **形变**
   transform: none | <transform-function>[<transform-fuction>]

### 4. 盒子模型

![盒子模型](http://upload-images.jianshu.io/upload_images/1879463-dcd26e170da66975.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 网页上的每一个标签都是一个盒子
- 每个盒子都有四个属性
  1. 内容（content）

![内容属性](http://upload-images.jianshu.io/upload_images/1879463-fd9b30c8df171c7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  2. 填充（padding，内边距）

![填充属性](http://upload-images.jianshu.io/upload_images/1879463-5689bf59b9e0f5fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  3. 边框（border）：盒子本身
  4. 边界（margin，外边距）

![边界属性](http://upload-images.jianshu.io/upload_images/1879463-2803e390750974e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![标准盒子模型](http://upload-images.jianshu.io/upload_images/1879463-7d4bf71b5a129753.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![IE盒子模型](http://upload-images.jianshu.io/upload_images/1879463-0ad03d90f842a6aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 4. JavaScript
>什么是JavaScript

- JavaScript是一门广泛用于浏览器客户端的脚本语言，由Netspace公司设计，当时跟Sun公司合作，所以名字起得像Java。业内一般简称JS

>JS的常见用途

- HTML DOM操作（节点操作，比如添加、修改、删除节点）
- 给HTML网页增加动态功能，比如动画
- 事件处理：比如监听鼠标点击、鼠标滑动、键盘输入

>JavaScript的书写方式

- JS常见的书写方式有2种
  - 页内JS：在当前网页的script标签中编写
```
<script type="text/javascript">
</script>
```
  - 外部JS
```
<script src="index.js"></script>
```

>Canvas

-  HTML
```
    <canvas id="canvas"></canvas>
```
- JS
```
   var canvas = document.getElementById('canvas');
   var context = canvas.getContext('2d');
```
    - `context`是一个绘图的上下文环境
    - `2d`是二维图形

- Canvas绘制直线
  - 起点
```
   context.moveTo(100,100);
```
  - 终点
```  
context.lineTo(400, 400);
```
  - 绘制
```
    context.stroke();
```
  - 设置线条颜色和宽度
```
    context.strokeStyle = 'red';
   context.lineWidth = 5;
```
  - 设置填充色
```
    context.fillStyle = 'blue';
```

- Canvas绘制弧线
 ```  
 context.arc(
          centerX, centerY, radius,
          startingAngle, endingAngle, 
          anticlockwise=false     
       )
```
  - `centerX, centerY`: 圆心的坐标
  - `radius`: 半径
  - `startingAngle`, `endingAngle`: 开始角度,结束角度
  - `anticlockwise`: false顺时针 true逆时针

![Canvas绘制弧线](http://upload-images.jianshu.io/upload_images/1879463-b56a7746859fd186.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
