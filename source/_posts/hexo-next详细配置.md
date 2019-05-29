---
title: hexo+next详细配置
date: 2019-01-21 11:42:41
categories:
- Hexo
tags:
- Hexo
- Next
photo: https://i.imgur.com/OE5CXF7.png
---
> hexo安装好后，默认的主题是[landscape](https://github.com/hexojs/hexo-theme-landscape)，如果不喜欢，我们也可以进行更换。本文主要配置Next([GitHub](https://github.com/iissnan/hexo-theme-next)，[官方文档](http://theme-next.iissnan.com/))主题。

<!--more-->

## Next安装与启用主题
### 安装Next
切换到Blog主目录文件夹下

通过git安装

```
$ cd your-hexo-site
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
```
### 启用主题
与所有 Hexo 主题启用的模式一样。 当 克隆/下载 完成后，打开 _config.yml(站点配置文件)， 找到 `theme` 字段，并将其值更改为 `next`。

### 验证主题
首先启动 Hexo 本地站点，并开启调试模式（即加上 --debug），整个命令是 `hexo s --debug`。 在服务启动的过程，注意观察命令行输出是否有任何异常信息，如果你碰到问题，这些信息将帮助他人更好的定位错误。 当命令行输出中提示出：
```
INFO  Hexo is running at http://0.0.0.0:4000/. Press Ctrl+C to stop.
```
此时即可使用浏览器访问 http://localhost:4000，检查站点是否正确运行。

## 设置主题
目前 NexT 支持四种 Scheme，他们是：
```
# Schemes
# scheme: Muse
# scheme: Mist
# scheme: Pisces
# scheme: Gemini
```
`Muse` - 默认 Scheme，这是 NexT 最初的版本，黑白主调，大量留白

`Mist` - Muse 的紧凑版本，整洁有序的单栏外观

`Pisces` - 双栏 Scheme，小家碧玉似的清新

`Gemini` - 双栏 Scheme，比`Pisces`略大

Scheme 的切换通过更改`主题配置文件`，搜索 `scheme` 关键字。 你会看到有三行 scheme 的配置，将你需用启用的 scheme 前面注释 `#` 去除即可。

## 设置菜单
```
menu:
  home: / || home
  about: /about/ || user
  tags: /tags/ || tags
  categories: /categories/ || th
  archives: /archives/ || archive
  # schedule: /schedule/ || calendar
  # sitemap: /sitemap.xml || sitemap
  # commonweal: /404/ || heartbeat
```

|键值|设定值|显示文本|
|:-----:|:-----:|:-----:|
|home|home: /|主页|
|archives|archives: /archives|归档页|
|categories|categories: /categories|分类页（需要手动创建页面）|
|tags|tags: /tags|标签页（需要手动创建页面）|
|about|about: /about|关于页面（需要手动创建页面）| 
|commonweal|commonweal: /404.html|公益 404（需要手动创建页面）|

## 设置侧栏
修改 ``主题配置文件` 中的 `sidebar` 字段来控制侧栏的行为。侧栏的设置包括两个部分，其一是侧栏的位置， 其二是侧栏显示的时机。

- 设置侧栏的位置，修改 sidebar.position 的值，支持的选项有：
    - left - 靠左放置
    - right - 靠右放置
```
sidebar:
  position: left
```
- 设置侧栏显示的时机，修改 `sidebar.display` 的值，支持的选项有：

    - post - 默认行为，在文章页面（拥有目录列表）时显示
    - always - 在所有页面中都显示
    - hide - 在所有页面中都隐藏（可以手动展开）
    - remove - 完全移除
```
sidebar:
  display: post
```

## 设置动态背景
`主题配置文件`中找到`canvas_nest`，设置成`ture`就OK啦。
```
# Canvas-nest
canvas_nest: ture
```
## 在右上角或者左上角实现fork me on github
![](https://upload-images.jianshu.io/upload_images/1879463-67e8c3b4657c24da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在[GitHub Ribbons](https://github.blog/2008-12-19-github-ribbons/)或[GitHub Corners](http://tholman.com/github-corners/)选择一款你喜欢的挂饰，拷贝方框内的代码
![](https://upload-images.jianshu.io/upload_images/1879463-214a8a6dfcbf6baf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将刚刚复制的挂饰代码，添加到`Blog/themes/next/layout/_layout.swig`文件中，添加到`<div class="headband"></div>`的下方。
## 设置RSS
NexT 中 RSS 有三个设置选项，满足特定的使用场景。 更改 `主题配置文件`，设定 `rss` 字段的值：

- false：禁用 RSS，不在页面上显示 RSS 连接。
- 留空：使用 Hexo 生成的 Feed 链接。 你可以需要先安装 `hexo-generator-feed` 插件。
- 具体的链接地址：适用于已经烧制过 Feed 的情形。

## 修改文章内链接文本样式

修改`Blog/themes/next/source/css/_common/components/post/post.styl`，在末尾添加CSS样式：

```
// 文章内链接文本样式
.post-body p a{
  color: #0593d3; //原始链接颜色
  border-bottom: none;
  border-bottom: 1px solid #0593d3; //底部分割线颜色
  &:hover {
    color: #fc6423; //鼠标经过颜色
    border-bottom: none;
    border-bottom: 1px solid #fc6423; //底部分割线颜色
  }
}
```

## 修改底部标签样式
修改`Blog\themes\next\layout\_macro\post.swig`中文件，`command+f`搜索`rel="tag">#`，将`#`替换成`<i class="fa fa-tag"></i>`。
![](https://upload-images.jianshu.io/upload_images/3072214-36dcbfd6cd83d271.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/386/format/webp)

## 在文章末尾添加“文章结束”标记
- 在路径`Blog\themes\next\layout\_macro`文件夹中新建`passage-end-tag.swig`文件。
- 在`passage-end-tag.swig`添加以下内容，直接用文本编辑器打开，粘贴以下内容后保存
```
<div>
    {% if not is_index %}
        <div style="text-align:center;color: #ccc;font-size:14px;">-------------感谢您的阅读<i class="fa fa-paw"></i>-------------</div>
    {% endif %}
</div>
```
- 打开`Blog\themes\next\layout\_macro\post.swig`，在`post-body`之后，`post-footer`之前（`post-footer`之前两个DIV），添加以下代码：
```
<div>
  {% if not is_index %}
    {% include 'passage-end-tag.swig' %}
  {% endif %}
</div>
```
具体添加位置：
```
    <div>
    {% if not is_index %}
        {% include 'passage-end-tag.swig' %}
    {% endif %}
    </div>

    {% if (theme.alipay or theme.wechatpay or theme.bitcoin) and not is_index %}
      <div>
        {% include 'reward.swig' %}
      </div>
    {% endif %}

    {% if theme.post_copyright.enable and not is_index %}
      <div>
        {% include 'post-copyright.swig' with { post: post } %}
      </div>
    {% endif %}

    <footer class="post-footer">
```
- 修改主题配置文件_config.yml，在末尾添加：

```
# 文章末尾添加“本文结束”标记
passage_end_tag:
  enabled: true
```
- 显示效果：

![](https://upload-images.jianshu.io/upload_images/1879463-8629419c5007d532.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 修改作者头像并旋转

打开`\themes\next\source\css\_common\components\sidebar\sidebar-author.styl`，在里面添加如下代码：
```
.site-author-image {
  display: block;
  margin: 0 auto;
  padding: $site-author-image-padding;
  max-width: $site-author-image-width;
  height: $site-author-image-height;
  border: $site-author-image-border-width solid $site-author-image-border-color;
  /* 头像圆形 */
  border-radius: 80px;
  -webkit-border-radius: 80px;
  -moz-border-radius: 80px;
  box-shadow: inset 0 -1px 0 #333sf;
  /* 设置循环动画 [animation: (play)动画名称 (2s)动画播放时长单位秒或微秒 (ase-out)动画播放的速度曲线为以低速结束 
    (1s)等待1秒然后开始动画 (1)动画播放次数(infinite为循环播放) ]*/
 
  /* 鼠标经过头像旋转360度 */
  -webkit-transition: -webkit-transform 1.0s ease-out;
  -moz-transition: -moz-transform 1.0s ease-out;
  transition: transform 1.0s ease-out;
}
img:hover {
  /* 鼠标经过停止头像旋转 
  -webkit-animation-play-state:paused;
  animation-play-state:paused;*/
  /* 鼠标经过头像旋转360度 */
  -webkit-transform: rotateZ(360deg);
  -moz-transform: rotateZ(360deg);
  transform: rotateZ(360deg);
}
/* Z 轴旋转动画 */
@-webkit-keyframes play {
  0% {
    -webkit-transform: rotateZ(0deg);
  }
  100% {
    -webkit-transform: rotateZ(-360deg);
  }
}
@-moz-keyframes play {
  0% {
    -moz-transform: rotateZ(0deg);
  }
  100% {
    -moz-transform: rotateZ(-360deg);
  }
}
@keyframes play {
  0% {
    transform: rotateZ(0deg);
  }
  100% {
    transform: rotateZ(-360deg);
  }
}
```

## 主页文章添加阴影效果
效果图：

![](https://upload-images.jianshu.io/upload_images/1879463-5de81a87bce76925.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体实现方法：

打开`\themes\next\source\css\_custom\custom.styl`,向里面加入：
```
// 主页文章添加阴影效果
 .post {
   margin-top: 60px;
   margin-bottom: 60px;
   padding: 25px;
   -webkit-box-shadow: 0 0 5px rgba(202, 203, 203, .5);
   -moz-box-shadow: 0 0 5px rgba(202, 203, 204, .5);
  }
  ```

## 设置网站的图标Favicon

在[easyicon](https://www.easyicon.net/)或者[阿里巴巴矢量图标库](https://www.iconfont.cn/)找一张你喜欢的图标（大：32x32 小：16x16），将下载下来的小图和中图放在Blog/themes/next/source/images，将默认的两张图片替换掉。命名和默认的一样也可以自己定义：
![](https://upload-images.jianshu.io/upload_images/3072214-cf67d50c1502275d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/786/format/webp)

## 底部隐藏由Hexo强力驱动、主题--NexT.Mist

打开`Blog/themes/next/layout/_partials/footer.swig`，注释掉相应代码。
```
//用下面的符号注释，注释代码用下面括号括起来
<!-- -->

<!--
<span class="post-meta-divider">|</span>

{% if theme.footer.powered %}
  <div class="powered-by">{#
  #}{{ __('footer.powered', '<a class="theme-link" target="_blank" href="https://hexo.io">Hexo</a>') }}{#
#}</div>
{% endif %}

{% if theme.footer.powered and theme.footer.theme.enable %}
  <span class="post-meta-divider">|</span>
{% endif %}

{% if theme.footer.theme.enable %}
  <div class="theme-info">{#
  #}{{ __('footer.theme') }} &mdash; {#
  #}<a class="theme-link" target="_blank" href="https://github.com/iissnan/hexo-theme-next">{#
    #}NexT.{{ theme.scheme }}{#
  #}</a>{% if theme.footer.theme.version %} v{{ theme.version }}{% endif %}{#
#}</div>
{% endif %}

{% if theme.footer.custom_text %}
  <div class="footer-custom">{#
  #}{{ theme.footer.custom_text }}{#
#}</div>
{% endif %}
-->
```

## 添加侧栏推荐阅读

编辑主题配置文件，如下配置即可：
```
# Blog rolls
links_icon: link
links_title: 推荐阅读
#links_layout: block
links_layout: inline
links:
  Swift 4: https://developer.apple.com/swift/
  Objective-C: https://developer.apple.com/documentation/objectivec
```

## Hexo博客添加站内搜索

![](https://upload-images.jianshu.io/upload_images/3072214-aff43f0e07e85647.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/972/format/webp)

NexT主题支持集成 Swiftype、 微搜索、Local Search 和 Algolia。下面介绍Local Search的安装吧。

切换到根目录

安装 hexo-generator-search
```
npm install hexo-generator-search --save
```

安装 hexo-generator-searchdb
```
npm install hexo-generator-searchdb --save
```

编辑`站点配置文件`，添加以下内容：
```
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```
编辑`主题配置文件`，设置`Local searchenable`为`ture`
```
# Local search
# Dependencies: https://github.com/flashlab/hexo-generator-search
local_search:
  enable: ture
  # if auto, trigger search by changing input
  # if manual, trigger search by pressing enter key or search button
  trigger: auto
  # show top n results per article, show all results by setting to -1
  top_n_per_article: 1
```


## 参考文章

- [hexo的next主题个性化教程:打造炫酷网站](http://shenzekun.cn/hexo%E7%9A%84next%E4%B8%BB%E9%A2%98%E4%B8%AA%E6%80%A7%E5%8C%96%E9%85%8D%E7%BD%AE%E6%95%99%E7%A8%8B.html)
- [Hexo-NexT配置超炫网页效果](https://www.jianshu.com/p/9f0e90cc32c2)