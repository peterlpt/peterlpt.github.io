---
title: "Flutter之遇到的问题记录"
categories: [Flutter, Dart]
tags: [Flutter]
date: 2020-02-10T13:09:12+08:00
draft: false
---

记录在Flutter开发中遇到的一系列问题、分析过程及解决方法。

<!--more-->

### 1. 给现有Flutter应用添加Web支持，运行报bad import错

{{< admonition failure 错误信息 >}} 

```shell
$ flutter run -d chrome
Launching lib/main.dart on Chrome in debug mode...
                                                                        
Unable to find modules for some sources, this is usually the result of either a
bad import, a missing dependency in a package (or possibly a dev_dependency
needs to move to a real dependency), or a build failure (if importing a
generated file).

Please check the following imports:

`import 'generated_plugin_registrant.dart';` from flutter_jzcf|lib/utils/encrypt_web_entrypoint.dart at 5:1

Failed after 2.3s                                                       
Building application for the web...                                12.2s
Failed to build application for the Web.

```

 {{< /admonition >}}

* 问题解决

原因是app中存在多个main()函数，注释掉其他不必要的测试main()函数，只保留一个即可。

* 参考： [Enable web in flutter import errors](https://stackoverflow.com/questions/59968935/enable-web-in-flutter-import-errors)

### 2. 设置CocoaPods报ruby: bad interpreter: No such file or directory

{{< admonition failure 错误信息 >}} 

在配置iOS环境时，安装CocoaPods后，设置报如下错误

```shell
$ pod setup
-bash: /usr/local/bin/pod: /System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/bin/ruby: bad interpreter: No such file or directory
```

 {{< /admonition >}}

* 问题分析

确认pod路径

```shell
$ which pod
/usr/local/bin/pod
```

确认cocoapods安装状态

```shell
$ gem list --local |grep coco
cocoapods (1.8.4)
cocoapods-core (1.8.4)
cocoapods-deintegrate (1.0.4)
cocoapods-downloader (1.3.0)
cocoapods-plugins (1.0.0)
cocoapods-search (1.0.0)
cocoapods-stats (1.1.0)
cocoapods-trunk (1.4.1)
cocoapods-try (1.1.0)
```

说明cocoapods有安装，再确认一下cocoapods 信息

```shell
$ gem info cocoapods

*** LOCAL GEMS ***

cocoapods (1.8.4)
    Authors: Eloy Duran, Fabio Pelosin, Kyle Fuller, Samuel Giddins
    Homepage: https://github.com/CocoaPods/CocoaPods
    License: MIT
    Installed at: /usr/local/lib/ruby/gems/2.6.0

    The Cocoa library package manager.
```

初步发现问题：目前path上获取的pod是在/usr/local/bin/pod的，而我们安装的在/usr/local/lib/ruby/gems/2.6.0，说明并非我们安装的版本，回想起之前处理升级到Catalina的兼容问题时（见[修复升级Catalina后Jekyll本地预览启动失败](https://ptlpt.gitee.io/fix-jekyll-local-exec-fail-on-catalina/)）修改过ruby后，所有安装的gems均已经在此目录下，在其中的bin中存放了各个包的可执行入口，包含了这里的pod：

```shell
$ ls /usr/local/lib/ruby/gems/2.6.0/bin
bundle          github-pages    nokogiri        sandbox-pod     xcodeproj
bundler         httpclient      pod             sass
commonmarker    jekyll          rake            sass-convert
fuzzy_match     kramdown        rougify         scss
gemoji          listen          safe_yaml       update_rubygems
```

{{< admonition success 问题解决 >}} 

修改Paths

```shell
$ echo 'export PATH="/usr/local/lib/ruby/gems/2.6.0/bin:$PATH"' >> ~/.bash_profile
$ source ~/.bash_profile

# 再次执行pod相关命令则不会再报错了
$ pod --version
1.8.4
```

{{< /admonition >}}

### 3. AppBar 不自动显示返回键等leading Widget

{{< admonition failure 异常情况 >}} 

关于Scaffold AppBar的automaticallyImplyLeading参数，默认为true，根据doc注释：

```dart
  /// Controls whether we should try to imply the leading widget if null.
  /// If true and [leading] is null, automatically try to deduce what the leading
  /// widget should be. If false and [leading] is null, leading space is given to [title].
  /// If leading widget is not null, this parameter has no effect.
```

automaticallyImplyLeading=true，同时leading Widget未设置，系统应该会自动根据需要显示返回键等leading Widget默认实现。

但有时却不会显示？

{{< /admonition >}}

* 问题原因

只有Scaffold作为根Widget时，AppBar automaticallyImplyLeading=true，才能根据存在上级路由时自动显示返回键

{{< admonition success 问题解决 >}} 

两种方式：

1. 将Scaffold作为根Widget
2. 如果布局需要无法作为根Widget，则显式指定leading Widget，如返回键：

```dart
Scaffold(
  appBar: AppBar(
    //显式指定返回键
    leading: BackButton(
      onPressed: () => Navigator.pop<bool>(context, false),
    ),
  ),
))
```

{{< /admonition >}}

### 4. Waiting for another flutter command to release the startup lock...

{{< admonition failure 问题现象 >}} 

AndroidStudio的时候顶部的模拟器一直是loading状态，即使已经打开了模拟器。运行flutter doctor 提示：

```shell
$ flutter doctor
Waiting for another flutter command to release the startup lock...
```

{{< /admonition >}}

{{< admonition success 问题解决 >}} 

从提示中得知，因为有上一条命令未结束而锁定了，如果确认无其他在执行的命令，那可以到Flutter安装目录/bin/cache/，删除lockfile文件，再执行Flutter命令就正常了

{{< /admonition >}}

### 5. asset在debug、profile下可以读取，但到了release就无法读取，报Unable to load asset的问题

背景：flutter的buildtype默认只有debug profile release三个，而我是add-to-app的，而原宿主app是有debug alpha beta release4个buildtype，add-to-app后，就在宿主app上定义了debug profile alpha beta release共5个

{{< admonition failure 遇到的问题 >}} 

我在宿主app上debug profile release三个buildtype下asset读取无异常，但flutter module上没有的alpha beta上时就所有asset都无法读取。

{{< /admonition >}}

Flutter版本信息如下：

```shell
$ flutter doctor -v
[✓] Flutter (Channel beta, v1.17.0-3.2.pre, on Mac OS X 10.15.3 19D76, locale
    zh-Hans-CN)
    • Flutter version 1.17.0-3.2.pre at /Users/peter/Data/flutter
    • Framework revision 2a7bc389f2 (11 days ago), 2020-04-21 20:34:20 -0700
    • Engine revision 4c8c31f591
    • Dart version 2.8.0 (build 2.8.0-dev.20.10)
```

暂时未找到有效办法