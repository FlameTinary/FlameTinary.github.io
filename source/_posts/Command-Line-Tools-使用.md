---
title: Command Line Tools 使用
date: 2019-08-12 15:31:06
tags:
- hide
categories:
- hide
---

## 安装

## 相关命令

### 查看目前我使用的工具XCODE版本
```bash
# command
xcode-select --print-path

# print
/Applications/Xcode.app/Contents/Developer
```

<!--more-->

### 选择XCODE中的默认版本用于我的命令行工具
```bash
# command
sudo xcode-select -switch <path/to/>Xcode.app

# 注释
`<path/to/>`是路径要使用的开发Xcode.app包

# 实例
sudo xcode-select -switch /Applications/Xcode5.1.1/Xcode.app
```

## 使用

`xcodebuild`是一个命令行工具，可以让你的工程通过`projects`、`workspaces`进行编译、测试、分析、打包。它可以运行在包含一个或多个Target的工程上面，或者在`projects` 、`workspaces`包含`scheme`上面。

### 列出所有的Target
```bash
# command
xcodebuild -list -project <your project name>.xcodeproj

# 注释
`<your project name>`是你工程的名称

# 示例
xcodebuild -list -project TYDemos.xcodeproj

# print 
Information about project "TYDemos":
    Targets:
        TYDemos
        qwer

    Build Configurations:
        Debug
        Release

    If no build configuration is specified and -scheme is not passed then "Release" is used.

    Schemes:
        qwer
        TYDemos
```

### 编译你工程的配置和schemes
```bash
# command
xcodebuild -scheme <your scheme name> build

# 注释
`<your scheme name>`是你选择特定的编译和执行的一个scheme

# 示例
xocdebuild -scheme TYGameSceneDemo build

# print
note: Using new build system
note: Planning build
2019-06-13 11:23:50.516 xcodebuild[61408:7436815]  DTDeviceKit: deviceType from 00008006-000F29961AC1002E was NULL
2019-06-13 11:23:50.516 xcodebuild[61408:7436815]  DTDeviceKit: deviceType from 00008006-000F29961AC1002E was NULL
2019-06-13 11:23:50.516 xcodebuild[61408:7436815]  iPhoneSupport: 📱<DVTiOSDevice (0x7ff824b6d8b0), Sheldon’s iPhone 7plus, iPhone, 12.2 (16E227), 312a7baecea95257c11a5eb61d1203138faac05c> unable to mount DeveloperDiskImage (Error Domain=com.apple.dtdevicekit Code=601 "Could not find platform" UserInfo={NSLocalizedDescription=Could not find platform})
note: Constructing build description
CreateBuildDirectory /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    builtin-create-build-directory /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products

CreateBuildDirectory /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    builtin-create-build-directory /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex

MkDir /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    /bin/mkdir -p /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app

ProcessProductPackaging /Users/sheldon/Library/MobileDevice/Provisioning\ Profiles/e91c7402-e02a-45ac-aff3-c73c563abcfb.mobileprovision /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app/embedded.mobileprovision (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    builtin-productPackagingUtility /Users/sheldon/Library/MobileDevice/Provisioning\ Profiles/e91c7402-e02a-45ac-aff3-c73c563abcfb.mobileprovision -o /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app/embedded.mobileprovision

WriteAuxiliaryFile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/DerivedSources/Entitlements.plist (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    write-file /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/DerivedSources/Entitlements.plist

ProcessProductPackaging "" /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo.app.xcent (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo


Entitlements:

{
    "application-identifier" = "CC9XV3GRSV.top.sheldon.TYGameSceneDemo";
    "com.apple.developer.team-identifier" = CC9XV3GRSV;
    "get-task-allow" = 1;
    "keychain-access-groups" =     (
        "CC9XV3GRSV.top.sheldon.TYGameSceneDemo"
    );
}


    builtin-productPackagingUtility -entitlements -format xml -o /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo.app.xcent

WriteAuxiliaryFile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/all-product-headers.yaml (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    write-file /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/all-product-headers.yaml

WriteAuxiliaryFile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo.hmap (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    write-file /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo.hmap

WriteAuxiliaryFile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-project-headers.hmap (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    write-file /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-project-headers.hmap

WriteAuxiliaryFile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-own-target-headers.hmap (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    write-file /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-own-target-headers.hmap

WriteAuxiliaryFile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-generated-files.hmap (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    write-file /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-generated-files.hmap

WriteAuxiliaryFile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-all-non-framework-target-headers.hmap (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    write-file /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-all-non-framework-target-headers.hmap

WriteAuxiliaryFile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-all-target-headers.hmap (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    write-file /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-all-target-headers.hmap

CompileC /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/GameViewController.o /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/GameViewController.m normal arm64 objective-c com.apple.compilers.llvm.clang.1_0.compiler (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    export LANG=en_US.US-ASCII
    /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang -x objective-c -arch arm64 -fmessage-length=178 -fdiagnostics-show-note-include-stack -fmacro-backtrace-limit=0 -fcolor-diagnostics -std=gnu11 -fobjc-arc -fobjc-weak -fmodules -gmodules -fmodules-cache-path=/Users/sheldon/Library/Developer/Xcode/DerivedData/ModuleCache.noindex -fmodules-prune-interval=86400 -fmodules-prune-after=345600 -fbuild-session-file=/Users/sheldon/Library/Developer/Xcode/DerivedData/ModuleCache.noindex/Session.modulevalidation -fmodules-validate-once-per-build-session -Wnon-modular-include-in-framework-module -Werror=non-modular-include-in-framework-module -Wno-trigraphs -fpascal-strings -O0 -fno-common -Wno-missing-field-initializers -Wno-missing-prototypes -Werror=return-type -Wdocumentation -Wunreachable-code -Wno-implicit-atomic-properties -Werror=deprecated-objc-isa-usage -Wno-objc-interface-ivars -Werror=objc-root-class -Wno-arc-repeated-use-of-weak -Wimplicit-retain-self -Wduplicate-method-match -Wno-missing-braces -Wparentheses -Wswitch -Wunused-function -Wno-unused-label -Wno-unused-parameter -Wunused-variable -Wunused-value -Wempty-body -Wuninitialized -Wconditional-uninitialized -Wno-unknown-pragmas -Wno-shadow -Wno-four-char-constants -Wno-conversion -Wconstant-conversion -Wint-conversion -Wbool-conversion -Wenum-conversion -Wno-float-conversion -Wnon-literal-null-conversion -Wobjc-literal-conversion -Wshorten-64-to-32 -Wpointer-sign -Wno-newline-eof -Wno-selector -Wno-strict-selector-match -Wundeclared-selector -Wdeprecated-implementations -DDEBUG=1 -DOBJC_OLD_DISPATCH_PROTOTYPES=0 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS12.2.sdk -fstrict-aliasing -Wprotocol -Wdeprecated-declarations -miphoneos-version-min=12.2 -g -Wno-sign-conversion -Winfinite-recursion -Wcomma -Wblock-capture-autoreleasing -Wstrict-prototypes -Wno-semicolon-before-method-body -Wunguarded-availability -fembed-bitcode-marker -index-store-path /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Index/DataStore -iquote /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-generated-files.hmap -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-own-target-headers.hmap -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-all-target-headers.hmap -iquote /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-project-headers.hmap -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/include -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/DerivedSources-normal/arm64 -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/DerivedSources/arm64 -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/DerivedSources -F/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos -MMD -MT dependencies -MF /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/GameViewController.d --serialize-diagnostics /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/GameViewController.dia -c /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/GameViewController.m -o /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/GameViewController.o

CompileC /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/AppDelegate.o /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/AppDelegate.m normal arm64 objective-c com.apple.compilers.llvm.clang.1_0.compiler (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    export LANG=en_US.US-ASCII
    /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang -x objective-c -arch arm64 -fmessage-length=178 -fdiagnostics-show-note-include-stack -fmacro-backtrace-limit=0 -fcolor-diagnostics -std=gnu11 -fobjc-arc -fobjc-weak -fmodules -gmodules -fmodules-cache-path=/Users/sheldon/Library/Developer/Xcode/DerivedData/ModuleCache.noindex -fmodules-prune-interval=86400 -fmodules-prune-after=345600 -fbuild-session-file=/Users/sheldon/Library/Developer/Xcode/DerivedData/ModuleCache.noindex/Session.modulevalidation -fmodules-validate-once-per-build-session -Wnon-modular-include-in-framework-module -Werror=non-modular-include-in-framework-module -Wno-trigraphs -fpascal-strings -O0 -fno-common -Wno-missing-field-initializers -Wno-missing-prototypes -Werror=return-type -Wdocumentation -Wunreachable-code -Wno-implicit-atomic-properties -Werror=deprecated-objc-isa-usage -Wno-objc-interface-ivars -Werror=objc-root-class -Wno-arc-repeated-use-of-weak -Wimplicit-retain-self -Wduplicate-method-match -Wno-missing-braces -Wparentheses -Wswitch -Wunused-function -Wno-unused-label -Wno-unused-parameter -Wunused-variable -Wunused-value -Wempty-body -Wuninitialized -Wconditional-uninitialized -Wno-unknown-pragmas -Wno-shadow -Wno-four-char-constants -Wno-conversion -Wconstant-conversion -Wint-conversion -Wbool-conversion -Wenum-conversion -Wno-float-conversion -Wnon-literal-null-conversion -Wobjc-literal-conversion -Wshorten-64-to-32 -Wpointer-sign -Wno-newline-eof -Wno-selector -Wno-strict-selector-match -Wundeclared-selector -Wdeprecated-implementations -DDEBUG=1 -DOBJC_OLD_DISPATCH_PROTOTYPES=0 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS12.2.sdk -fstrict-aliasing -Wprotocol -Wdeprecated-declarations -miphoneos-version-min=12.2 -g -Wno-sign-conversion -Winfinite-recursion -Wcomma -Wblock-capture-autoreleasing -Wstrict-prototypes -Wno-semicolon-before-method-body -Wunguarded-availability -fembed-bitcode-marker -index-store-path /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Index/DataStore -iquote /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-generated-files.hmap -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-own-target-headers.hmap -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-all-target-headers.hmap -iquote /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-project-headers.hmap -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/include -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/DerivedSources-normal/arm64 -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/DerivedSources/arm64 -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/DerivedSources -F/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos -MMD -MT dependencies -MF /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/AppDelegate.d --serialize-diagnostics /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/AppDelegate.dia -c /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/AppDelegate.m -o /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/AppDelegate.o

WriteAuxiliaryFile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/TYGameSceneDemo.LinkFileList (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    write-file /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/TYGameSceneDemo.LinkFileList

CompileC /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/main.o /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/main.m normal arm64 objective-c com.apple.compilers.llvm.clang.1_0.compiler (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    export LANG=en_US.US-ASCII
    /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang -x objective-c -arch arm64 -fmessage-length=178 -fdiagnostics-show-note-include-stack -fmacro-backtrace-limit=0 -fcolor-diagnostics -std=gnu11 -fobjc-arc -fobjc-weak -fmodules -gmodules -fmodules-cache-path=/Users/sheldon/Library/Developer/Xcode/DerivedData/ModuleCache.noindex -fmodules-prune-interval=86400 -fmodules-prune-after=345600 -fbuild-session-file=/Users/sheldon/Library/Developer/Xcode/DerivedData/ModuleCache.noindex/Session.modulevalidation -fmodules-validate-once-per-build-session -Wnon-modular-include-in-framework-module -Werror=non-modular-include-in-framework-module -Wno-trigraphs -fpascal-strings -O0 -fno-common -Wno-missing-field-initializers -Wno-missing-prototypes -Werror=return-type -Wdocumentation -Wunreachable-code -Wno-implicit-atomic-properties -Werror=deprecated-objc-isa-usage -Wno-objc-interface-ivars -Werror=objc-root-class -Wno-arc-repeated-use-of-weak -Wimplicit-retain-self -Wduplicate-method-match -Wno-missing-braces -Wparentheses -Wswitch -Wunused-function -Wno-unused-label -Wno-unused-parameter -Wunused-variable -Wunused-value -Wempty-body -Wuninitialized -Wconditional-uninitialized -Wno-unknown-pragmas -Wno-shadow -Wno-four-char-constants -Wno-conversion -Wconstant-conversion -Wint-conversion -Wbool-conversion -Wenum-conversion -Wno-float-conversion -Wnon-literal-null-conversion -Wobjc-literal-conversion -Wshorten-64-to-32 -Wpointer-sign -Wno-newline-eof -Wno-selector -Wno-strict-selector-match -Wundeclared-selector -Wdeprecated-implementations -DDEBUG=1 -DOBJC_OLD_DISPATCH_PROTOTYPES=0 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS12.2.sdk -fstrict-aliasing -Wprotocol -Wdeprecated-declarations -miphoneos-version-min=12.2 -g -Wno-sign-conversion -Winfinite-recursion -Wcomma -Wblock-capture-autoreleasing -Wstrict-prototypes -Wno-semicolon-before-method-body -Wunguarded-availability -fembed-bitcode-marker -index-store-path /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Index/DataStore -iquote /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-generated-files.hmap -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-own-target-headers.hmap -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-all-target-headers.hmap -iquote /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo-project-headers.hmap -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/include -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/DerivedSources-normal/arm64 -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/DerivedSources/arm64 -I/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/DerivedSources -F/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos -MMD -MT dependencies -MF /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/main.d --serialize-diagnostics /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/main.dia -c /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/main.m -o /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/main.o

Ld /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app/TYGameSceneDemo normal arm64 (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    export IPHONEOS_DEPLOYMENT_TARGET=12.2
    /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang -arch arm64 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS12.2.sdk -L/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos -F/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos -filelist /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/TYGameSceneDemo.LinkFileList -Xlinker -rpath -Xlinker @executable_path/Frameworks -miphoneos-version-min=12.2 -dead_strip -Xlinker -object_path_lto -Xlinker /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/TYGameSceneDemo_lto.o -Xlinker -export_dynamic -Xlinker -no_deduplicate -fembed-bitcode-marker -fobjc-arc -fobjc-link-runtime -Xlinker -dependency_info -Xlinker /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Objects-normal/arm64/TYGameSceneDemo_dependency_info.dat -o /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app/TYGameSceneDemo

Copy SceneKit assets /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/art.scnassets (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    /Applications/Xcode.app/Contents/Developer/usr/bin/copySceneKitAssets /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/art.scnassets -o /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app/art.scnassets --target-platform=iphoneos --target-version=12.2 --target-build-dir=/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos --resources-folder-path=TYGameSceneDemo.app
copySceneKitAssets: Copy ship.scn
copySceneKitAssets: Copy texture.png
copySceneKitAssets: Running scntool on /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app/art.scnassets/ship.scn

CompileAssetCatalog /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/Assets.xcassets (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    /Applications/Xcode.app/Contents/Developer/usr/bin/actool --output-format human-readable-text --notices --warnings --export-dependency-info /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/assetcatalog_dependencies --output-partial-info-plist /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/assetcatalog_generated_info.plist --app-icon AppIcon --compress-pngs --enable-on-demand-resources YES --sticker-pack-identifier-prefix top.sheldon.TYGameSceneDemo.sticker-pack. --target-device iphone --target-device ipad --minimum-deployment-target 12.2 --platform iphoneos --product-type com.apple.product-type.application --compile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/Assets.xcassets
/* com.apple.actool.compilation-results */
/Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/assetcatalog_generated_info.plist


CompileStoryboard /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/Base.lproj/Main.storyboard (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    export XCODE_DEVELOPER_USR_PATH=/Applications/Xcode.app/Contents/Developer/usr/bin/..
    /Applications/Xcode.app/Contents/Developer/usr/bin/ibtool --errors --warnings --notices --module TYGameSceneDemo --output-partial-info-plist /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Base.lproj/Main-SBPartialInfo.plist --auto-activate-custom-fonts --target-device iphone --target-device ipad --minimum-deployment-target 12.2 --output-format human-readable-text --compilation-directory /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Base.lproj /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/Base.lproj/Main.storyboard

CompileStoryboard /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/Base.lproj/LaunchScreen.storyboard (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    export XCODE_DEVELOPER_USR_PATH=/Applications/Xcode.app/Contents/Developer/usr/bin/..
    /Applications/Xcode.app/Contents/Developer/usr/bin/ibtool --errors --warnings --notices --module TYGameSceneDemo --output-partial-info-plist /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Base.lproj/LaunchScreen-SBPartialInfo.plist --auto-activate-custom-fonts --target-device iphone --target-device ipad --minimum-deployment-target 12.2 --output-format human-readable-text --compilation-directory /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Base.lproj /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/Base.lproj/LaunchScreen.storyboard

ProcessInfoPlistFile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app/Info.plist /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/Info.plist (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    builtin-infoPlistUtility /Users/sheldon/Documents/TYDemos/TYGameSceneDemo/TYGameSceneDemo/Info.plist -producttype com.apple.product-type.application -genpkginfo /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app/PkgInfo -expandbuildsettings -format binary -platform iphoneos -additionalcontentfile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Base.lproj/LaunchScreen-SBPartialInfo.plist -additionalcontentfile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Base.lproj/Main-SBPartialInfo.plist -additionalcontentfile /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/assetcatalog_generated_info.plist -requiredArchitecture arm64 -o /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app/Info.plist

LinkStoryboards (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    export XCODE_DEVELOPER_USR_PATH=/Applications/Xcode.app/Contents/Developer/usr/bin/..
    /Applications/Xcode.app/Contents/Developer/usr/bin/ibtool --errors --warnings --notices --module TYGameSceneDemo --target-device iphone --target-device ipad --minimum-deployment-target 12.2 --output-format human-readable-text --link /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Base.lproj/LaunchScreen.storyboardc /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/Base.lproj/Main.storyboardc

CodeSign /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    export CODESIGN_ALLOCATE=/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/codesign_allocate

Signing Identity:     "iPhone Developer: sheldondev@163.com (***********)"
Provisioning Profile: "iOS Team Provisioning Profile: top.sheldon.TYGameSceneDemo"
                      (e91c7402-e02a-45ac-aff3-c73c563abcfb)

    /usr/bin/codesign --force --sign A4F946689397857CA5F807E9FFB76913085216E4 --entitlements /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Intermediates.noindex/TYGameSceneDemo.build/Debug-iphoneos/TYGameSceneDemo.build/TYGameSceneDemo.app.xcent --timestamp=none /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app

Validate /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    builtin-validationUtility /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app

Touch /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app (in target: TYGameSceneDemo)
    cd /Users/sheldon/Documents/TYDemos/TYGameSceneDemo
    /usr/bin/touch -c /Users/sheldon/Library/Developer/Xcode/DerivedData/TYGameSceneDemo-fnnafecckruutfccltqcqgjtvkco/Build/Products/Debug-iphoneos/TYGameSceneDemo.app

** BUILD SUCCEEDED **
```

## 参考
[通过Xcode命令行编译](https://www.jianshu.com/p/55b80e746e38)
[用xcconfig文件配置iOS app环境变量](https://www.jianshu.com/p/9b8bc8351223)
[官方文档](https://developer.apple.com/library/archive/technotes/tn2339/_index.html#//apple_ref/doc/uid/DTS40014588-CH1-HOW_DO_I_BUILD_MY_PROJECTS_FROM_THE_COMMAND_LINE_)
