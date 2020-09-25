---
title: cocoapods结构解析
date: 2020-09-20 16:23:29
tags:
- iOS
categories:
- iOS
---

> CocoaPods是用 Ruby 写的，并由若干个 Ruby 包 (gems) 构成的。在解析整合过程中，最重要的几个 gems 分别是： CocoaPods/CocoaPods, CocoaPods/Core, 和 CocoaPods/Xcodeproj (是的，CocoaPods 是一个依赖管理工具 -- 利用依赖管理进行构建的！)。
> podspecs存储路径：~/.cocoapods
> 缓存文件存储路径：~/Library/Caches/CocoaPods


<!--more-->

## 核心组件

### CocoaPods/CocoaPod

这是是一个面向用户的组件，每当执行一个 pod 命令时，这个组件都将被激活。该组件包括了所有使用 CocoaPods 涉及到的功能，并且还能通过调用所有其它的 gems 来执行任务。

### CocoaPods/Core

Core 组件提供支持与 CocoaPods 相关文件的处理，文件主要是 Podfile 和 podspecs。

- Podfile：Podfile 是一个文件，用于定义项目所需要使用的第三方库。该文件支持高度定制，你可以根据个人喜好对其做出定制。更多相关信息，请查阅 Podfile 指南。
- Podspec：.podspec 也是一个文件，该文件描述了一个库是怎样被添加到工程中的。它支持的功能有：列出源文件、framework、编译选项和某个库所需要的依赖等。

### CocoaPods/Xcodeproj

这个 gem 组件负责所有工程文件的整合。它能够对创建并修改 .xcodeproj 和 .xcworkspace 文件。它也可以作为单独的一个 gem 包使用。如果你想要写一个脚本来方便的修改工程文件，那么可以使用这个 gem。


## 运行 pod install 命令

当运行 pod install 命令时会引发许多操作。要想深入了解这个命令执行的详细内容，可以在这个命令后面加上 --verbose。运行指令后会有如下操作：

### 读取Podfile文件

在安装期间，第一步是要弄清楚显示或隐式的声明了哪些第三方库。在加载 podspecs 过程中，CocoaPods 就建立了包括版本信息在内的所有的第三方库的列表。Podspecs 被存储在本地路径 ~/.cocoapods 中。

### 版本控制和冲突

如果依赖的某些库同时使用了一个第三方库，但是版本不同，那么默认是向后兼容，既高版本的兼容低版本的；
但是，总会有一些第三方库会存在不能向后兼容的问题，这时候就需要手动解决兼容问题。比如：同时依赖了一个库的1.2.5和2.1.3，那么需要用户通过明确指定使用的版本来解决冲突。

### 加载源文件

CocoaPods 执行的下一步是加载源码。每个 .podspec 文件都包含一个源代码的索引，这些索引一般包裹一个 git 地址和 git tag。它们以 commit SHAs 的方式存储在 ~/Library/Caches/CocoaPods 中。这个路径中文件的创建是由 Core gem 负责的。
CocoaPods 将依照 Podfile、.podspec 和缓存文件的信息将源文件下载到 Pods 目录中。

### 生成Pod.xcodeproj

每次 pod install 执行，如果检测到改动时，CocoaPods 会利用 Xcodeproj gem 组件对 Pods.xcodeproj 进行更新。如果该文件不存在，则用使用默认配置。否则，会将已有的配置项加载至内存中。

### 安装第三方库

当cocoapods往工程中添加一个第三方库时，不仅仅是添加了第三方库的源码，还会添加很多内容。由于每个第三方库会有一个独立的target，因此对于每个库，都会添加几个额外的文件：

- 一个包含编译选项的.xcconfig文件
- 一个同时包含编译设置和CocoaPods默认配置的私有.xcconfig文件
- 一个编译所必须的prefix.pch文件
- 一个编译必须的dummy.m文件


如果源码中包含资源bundle，那么在target的执行脚本Pods-Resources.sh中会添加bundle相关的指令。
还有一个Pods-environment.h文件，其中包含一些宏，这些宏是用来检测某个组件是否来自pod。
最后，将生成两个认可文件，一个是 plist，另一个是 markdown，这两个文件用于给最终用户查阅相关许可信息。

- Podfile.lock

该文件记录了Pod中的依赖的每个组件，以及依赖组件的版本

- Manifest.lock

这是每次运行 pod install 命令时创建的 Podfile.lock 文件的副本。如果你遇见过这样的错误 沙盒文件与 Podfile.lock 文件不同步 (The sandbox is not in sync with the Podfile.lock)。这是因为 Manifest.lock 文件和 Podfile.lock 文件不一致所引起。由于 Pods 所在的目录并不总在版本控制之下，这样可以保证开发者运行 app 之前都能更新他们的 pods，否则 app 可能会 crash，或者在一些不太明显的地方编译失败。

## xcproj

如果你已经依照我们的建议在系统上安装了 xcproj，它会对 Pods.xcodeproj 文件执行一下 touch 以将其转换成为旧的 ASCII plist 格式的文件。为什么要这么做呢？虽然在很久以前就不被其它软件支持了，但是 Xcode 仍然依赖于这种格式。如果没有 xcproj，你的 Pods.xcodeproj 文件将会以 XML 格式的 plist 文件存储，当你用 Xcode 打开它时，它会被改写，并造成大量的文件改动。


### 文件结构变化

执行pod install 后，我们的工程结构发生了变化，在以前的基础上添加了如下文件：

1. PodFile：依赖描述文件
2. Podfile.lock：当前安装的依赖库的版本
3. xxx.xcworkspace：xcworkspace文件，使用CocoaPod管理依赖的项目，XCode只能使用workspace编译项目，如果还只打开以前的xcodeproj文件进行开发，编译会失败 xcworkspace文件实际是一个文件夹，实际Workspace信息保存在contents.xcworkspacedata里，该文件的内容非常简单，实际上只指示它所使用的工程的文件目录

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Workspace
   version = "1.0">
   <FileRef
      location = "group:CardPlayer/CardPlayer.xcodeproj">
   </FileRef>
   <FileRef
      location = "group:Pods/Pods.xcodeproj">
   </FileRef>
</Workspace>
```

4. Pods/

- Pods.xcodeproj，所有的第三方库都由Pods工程构建，每个三方库对应一个Pods工程中的Target。
- 每个第三方库都会在Pods目录下对应一个目录。
- Headers，在Headers文件中有两个目录，Private和Public，其中包含第三方库的私有和公有的头文件，这些文件都是快捷连接，其真正的位置在每个三方库的文件夹目录下。

![header结构](https://s1.ax1x.com/2020/09/25/09o0VP.jpg)

5. Target Support Files 支撑Target的文件

![Target Support Files](https://s1.ax1x.com/2020/09/25/09oYgH.jpg)


## 工程结构的变化

### Pods工程变化

Pods工程会为每个依赖的第三方库定义一个Target，还会定义一个Pods-xxx的Target，每个Target会生成一个静态库
![Pods结构](https://s1.ax1x.com/2020/09/25/09oUKA.png)

Pods工程会新建两个Configuration，每个Configuration会为不同的Target设置不同的xcconfig，xcconfig指出了头文件查找的目录，要链接的第三方，链接目录等。
![7](https://s1.ax1x.com/2020/09/25/09osPS.png)
AFNetworking.xcconfig文件的内容：

CONFIGURATION_BUILD_DIR = $PODS_CONFIGURATION_BUILD_DIR/AFNetworking
GCC_PREPROCESSOR_DEFINITIONS = $(inherited) COCOAPODS=1
HEADER_SEARCH_PATHS = "${PODS_ROOT}/Headers/Private" "${PODS_ROOT}/Headers/Private/AFNetworking" "${PODS_ROOT}/Headers/Public" "${PODS_ROOT}/Headers/Public/AFNetworking"
OTHER_LDFLAGS = -framework "CoreGraphics" -framework "MobileCoreServices" -framework "Security" -framework "SystemConfiguration"
PODS_BUILD_DIR = $BUILD_DIR
PODS_CONFIGURATION_BUILD_DIR = $PODS_BUILD_DIR/$(CONFIGURATION)$(EFFECTIVE_PLATFORM_NAME)
PODS_ROOT = ${SRCROOT}
PODS_TARGET_SRCROOT = ${PODS_ROOT}/AFNetworking
PRODUCT_BUNDLE_IDENTIFIER = org.cocoapods.${PRODUCT_NAME:rfc1034identifier}
SKIP_INSTALL = YES

Header_SERACH_PATHS：编译时查找头文件的目录
OTHER_LD_FLAGS：指明要链接的framework

Pods-CardPlayer.debug.xcconfig文件的内容：

GCC_PREPROCESSOR_DEFINITIONS = $(inherited) COCOAPODS=1
HEADER_SEARCH_PATHS = $(inherited) "${PODS_ROOT}/Headers/Public" "${PODS_ROOT}/Headers/Public/AFNetworking"
LIBRARY_SEARCH_PATHS = $(inherited) "$PODS_CONFIGURATION_BUILD_DIR/AFNetworking"
OTHER_CFLAGS = $(inherited) -isystem "${PODS_ROOT}/Headers/Public" -isystem "${PODS_ROOT}/Headers/Public/AFNetworking"
OTHER_LDFLAGS = $(inherited) -ObjC -l"AFNetworking" -framework "CoreGraphics" -framework "MobileCoreServices" -framework "Security" -framework "SystemConfiguration"
PODS_BUILD_DIR = $BUILD_DIR
PODS_CONFIGURATION_BUILD_DIR = $PODS_BUILD_DIR/$(CONFIGURATION)$(EFFECTIVE_PLATFORM_NAME)
PODS_PODFILE_DIR_PATH = ${SRCROOT}/..
PODS_ROOT = ${SRCROOT}/../Pods

OTHER_LDFLAGS：说明了编译Pods时需要链接的第三方库，还需要链接其它framework
所以我们在xcode里能看到AFNetworking依赖的framework:

![6](https://s1.ax1x.com/2020/09/25/09odbt.png)

### 主工程变化
引入CocoaPods后，主工程的设置也会发生相应的变化，引入之前：
![4](https://s1.ax1x.com/2020/09/25/09oBUf.png)

Debug和Release中的Configuration没有设置任何配置文件，引入CocoaPods后的变化：
![3](https://s1.ax1x.com/2020/09/25/09oaDI.jpg)
可以看到，采用CocoaPods之后，Debug和Release设置了相应的配置文件，这些配置文件指明了头文件的查找目录和要链接的第三方库

## 编译并链接第三方库

[Podfile语法中文参考文档](http://www.srcmini.com/3965.html)

