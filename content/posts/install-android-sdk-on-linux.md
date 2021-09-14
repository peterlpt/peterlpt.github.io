---
title: "Linux安装Android SDK"
categories: [Linux, Android]
tags: [Android]
date: 2016-09-21T11:08:32+08:00
draft: false
---

主要用于服务器搭建Android环境，用于CI/CD

## 下载Android SDK

下载地址：<http://developer.android.com/sdk/index.html>，选择Linux(i386)。因为SDK只有32位的，如果装的是64位系统，则要安装ia32-libs，运行32位程序<br>
ubuntu

```shell
sudo apt-get install ia32-libs
```

centos

```shell
yum install glibc.i686
```

用xshell上传到linux的话，可新建SFTP，通过put指令上传

下载完成后解压，在终端进入到SDK的根目录，然后执行：

```shell
tools/android update sdk –no-ui
```

即可开始和windows里面一样的更新

常用命令：

```java
./android list sdk     // 可以列出所有可以安装的列表

//可以过滤指定序号的包进行下载安装
./android update sdk --no-ui --filter 1  
```

## 配置环境变量

更新完成后配置环境变量。编辑文件profile

```shell
vi /etc/profile
```

然后在下面增加下面内容：

```shell
export ANDROID_HOME=/opt/softwaretools/android-sdk-linux
export PATH=$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools:$PATH
```

/opt/softwaretools/android-sdk-linux为SDK的根目录。

使环境变量配置生效

这个配置之后，以后要执行android里面的命令，就不是需要进到这个目录，直接可以在终端里面输入。

```shell
source /etc/profile
```

在终端输入：android，如果Android SDK Manager窗口出来了，或者如果安装的命令行版本，则会报SWTError，就证明环境配置成功。

## 配置AVD
1、进入$SDK_HOME/toos目录<br>
2、命令窗口运行：./android avd

## 备忘命令

``` java
//更新指定版本build-tools
android update sdk --no-ui -a --filter build-tools-23.0.2

//待整理
android list sdk --extended --proxy-host test.com --proxy-port 1234
android update sdk --no-ui -t 1,2,3 --proxy-host test.com --proxy-port 1234
gradle -Dhttp.proxyHost=test.com -Dhttp.proxyPort=1234 -Dhttps.proxyHost=test.com -Dhttps.proxyPo
android list sdk --all --extended --proxy-host test.com --proxy-port 1234
android update sdk --no-ui --all --filter build-tools-23.0.3 --proxy-host test.com --proxy-port 1234
```

