---
title: "centos 6中用yum配置常用软件"
categories: [Linux]
tags: [Linux, Centos]
date: 2014-03-15T11:08:32+08:00
draft: false
---

主要有输入法、Chrome、wine

## ibus中文输入法的配置

1.输入

```shell
su root
yum install "@Chinese Support"
exit
```

2.回到桌面，system->preferences->input method<br>
3.如果没有，先注销一下<br>
4.按照提示添加输入法<br>
5.最后 再次注销，登录即可

## fcitx中文输入法的配置

1、添加软件源

```shell
# cd /etc/yum.repos.d/
# wget http://download.opensuse.org/repositories/home:/cathay4t:/misc-rhel6/CentOS_CentOS-6/home:cathay4t:misc-rhel6.repo
```

2、安装 gtk2-immodules、gtk2-immodule-xim和fcitx

```shell
# yum install gtk2-immodules gtk2-immodule-xim
# yum install fcitx
```

3、升级和修改gtk.immodules
下面的命令必须以root账户操作，不能以sudo的方式，否则会提示没有权限：

```shell
# /usr/bin/gtk-query-immodules-2.0-32 > /etc/gtk-2.0/i386-redhat-linux-gnu/gtk.immodules
```

这里假设你安装的是32位版本的centos，如果是64位版本的话，则要输入对应的命令。

4、修改xim.conf

```shell
vi /etc/X11/xinit/xinput.d/xim.conf
```

在最后面添加下面的内容，并保存退出：  

```shell
XIM=fcitx
XIM_PROGRAM=/usr/bin/fcitx
XIM_ARGS=" -d"
GTK_IM_MODULE=fcitx
QT_IM_MODULE =fcitx
```

5、分别以root和普通用户的身份，建立修改一些到xim.conf的链接

a、以root的身份  

```shell
rm -rf /etc/alternatives/xinputrc
ln -s /etc/X11/xinit/xinput.d/xim.conf /etc/alternatives/xinputrc
```

b、以普通用户的身份 

```shell
rm -rf ~/.xinputrc
ln -s /etc/X11/xinit/xinput.d/xim.conf ~/.xinputrc
```

6、在用户目录下创建一个名为.xprofile 的文件

```shell
vi ~/.xprofile
```

写入

```shell
export LC_ALL=zh_CN.UTF-8
export XMODIFIERS=@im=fcitx
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
eval `dbus-launch --sh-syntax --exit-with-session`
exec fcitx &
```

退出，重新登录，fcitx便可以使用了。

引自：<http://www.jbxue.com/LINUXjishu/4870.html>

## 安装chrome和flash-plugin
Best way to install and keep up-to-date with Google Chrome browser is use Google’s own YUM repository.

### 1) Enable google yum repository
Here creating google.repo file using vi editor

```shell
vim /etc/yum.repos.d/google.repo
```

add following content.
#### for 32-bit 

```shell
[google]
name=Google – i386
baseurl=http://dl.google.com/linux/rpm/stable/i386
enabled=1
gpgcheck=1
gpgkey=https://dl-ssl.google.com/linux/linux_signing_key.pub
```

#### for 64-bit

```shell
[google64]
name=Google – x86_64
baseurl=http://dl.google.com/linux/rpm/stable/x86_64
enabled=1
gpgcheck=1
gpgkey=https://dl-ssl.google.com/linux/linux_signing_key.pub
```

Note: Both 32-bit and 64-bit repos can be placed in the same file.

### 2) Install Google Chrome with YUM (as root user)
Install Google Chrome Stable Version

```shell
yum install google-chrome-stable
```

Install Google Chrome Beta Version

```shell
yum install google-chrome-beta
```

Install Google Chrome Unstable Version

```shell
yum install google-chrome-unstable
```

如提示缺少依赖，则需补充安装以下软件（rpm -ivh）之后再安装chrome：
chrome-deps-1.01-1.i386.rpm

Enjoy browsing google chrome :)

### 3) 安装flash 插件【一下操作基于32位的Fedora】:
添加文件 /etc/yum.repos.d/adobe-linux-i386.repo,内容如下

```shell
[adobe-linux-i386]
name=Adobe Systems Incorporated
baseurl=http://linuxdownload.adobe.com/linux/i386/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-adobe-linux
```

安装flash：

```shell
yum install flash-plugin
```

将插件应用于chrome：

```shell
ln -s /usr/lib/mozilla/plugins/libflashplayer.so /opt/google/chrome/plugins/libfla
```

引自：http://www.cnblogs.com/sn0-0py/archive/2013/05/05/3060845.html

## 安装wine

首先安装一个epel 

```shell
rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-7.noarch.rpm
```

有可能这个地址往后会失效，可浏览上层点第目录，直到找到最新第版本，例如，从http://dl.fedoraproject.org/pub/开始慢慢往下浏览。
然后安装wine 

```shell
yum install wine
```

引自：http://blackwing.iteye.com/blog/1558194