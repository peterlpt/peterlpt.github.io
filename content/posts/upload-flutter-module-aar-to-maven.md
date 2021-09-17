---
title: "上传Flutter module aar到maven私服"
categories: [Flutter]
tags: [Flutter, add-to-app]
date: 2020-05-03T10:08:32+08:00
draft: false
---

本文主要描述如何编写shell脚本找到所有Flutter module aar，并调用maven上传命令上传到私服。

<!--more-->

### 1. 开发环境说明

Flutter版本，经测试适用于`Flutter (Channel stable, v1.12.13)`至目前如下`Flutter (Channel stable, v1.12.13`

```shell
$ flutter doctor -v
[✓] Flutter (Channel beta, v1.17.0-3.2.pre, on Mac OS X 10.15.3 19D76, locale
    zh-Hans-CN)
    • Flutter version 1.17.0-3.2.pre at /Users/peter/Data/flutter
    • Framework revision 2a7bc389f2 (11 days ago), 2020-04-21 20:34:20 -0700
    • Engine revision 4c8c31f591
    • Dart version 2.8.0 (build 2.8.0-dev.20.10)
```

maven版本，其他mavn版本未测试，但只要上传命令一致，应该都适用。

```shell
$ mvn -v
Apache Maven 3.6.3 (cecedd343002696d0abb50b32b541b8a6ba2883f)
Maven home: /usr/local/Cellar/maven/3.6.3/libexec
Java version: 1.8.0_201, vendor: Oracle Corporation, runtime: /Library/Java/JavaVirtualMachines/jdk1.8.0_201.jdk/Contents/Home/jre
Default locale: zh_CN, platform encoding: UTF-8
OS name: "mac os x", version: "10.15.3", arch: "x86_64", family: "mac"
```

### 2. Flutter module add-to-app 的依赖配置方式

根据"[Add Flutter to existing app](https://flutter.dev/docs/development/add-to-app)"指引，我们以下三种依赖配置方式：

* 使用Andorid Studio向导，经测试不论是新建一个Flutter Module还是引入一个已有的，最后还是会指向以下两个方式
* 手工配置AAR依赖，允许团队其他未参与Flutter Module开发的成员无需安装Flutter SDK
* 手工配置Module源码依赖，可方便同时在Host App、Flutter Module开发，实时迭代，但要求所有使用此依赖配置的团队成员均安装Flutter SDK

个人建议：在多成员协同开发的APP中，所有参与Flutter Module开发成员本地配置为源码依赖，提交仓库的代码配置为aar依赖，指向maven私服，对团队其他非Flutter开发成员零影响。

源码依赖方式，无过多需要注意的，按指引进行配置，这里主要是对aar依赖方式记录经验明细。

### 3. aar构建

```shell
$ flutter build aar --build-number 1.0.0-SNAPSHOT
Running Gradle task 'assembleAarDebug'...                               
Running Gradle task 'assembleAarDebug'... Done                     18.3s
✓ Built build/host/outputs/repo.
Running Gradle task 'assembleAarProfile'...                             
Running Gradle task 'assembleAarProfile'... Done                   36.1s
✓ Built build/host/outputs/repo.
Running Gradle task 'assembleAarRelease'...                             
Running Gradle task 'assembleAarRelease'... Done                   50.6s
✓ Built build/host/outputs/repo.

Consuming the Module
  1. Open <host>/app/build.gradle
  2. Ensure you have the repositories configured, otherwise add them:

      repositories {
        maven {
            url '/Users/peter/Data/AsPrj/flutter_module/build/host/outputs/repo'
        }
        maven {
            url 'https://storage.googleapis.com/download.flutter.io'
        }
      }

  3. Make the host app depend on the Flutter module:

    dependencies {
      debugImplementation 'com.cx580.fluttermodule:flutter_debug:1.0.0-SNAPSHOT'
      profileImplementation 'com.cx580.fluttermodule:flutter_profile:1.0.0-SNAPSHOT'
      releaseImplementation 'com.cx580.fluttermodule:flutter_release:1.0.0-SNAPSHOT'
    }


  4. Add the `profile` build type:

    android {
      buildTypes {
        profile {
          initWith debug
        }
      }
    }

To learn more, visit https://flutter.dev/go/build-aar
```

注意：

* 这里使用了flutter channel beta的新功能指定版本号打包， 如果使用Channel stable, v1.12.13，暂时只能使用`flutter build aar`命令，打出来的aar固定版本号为1.0，如需要调整，可参考[Flutter 1.12后 上传aar至maven私服](https://juejin.im/post/5e3c0ad351882549087d90b5)在打好包后写脚本修改一下pom文件
* 指定的版本号`1.0.0-SNAPSHOT`，为快照版本，日常工作中会自动获取最新的快照，在开发阶段建议使用快照版本，发布生产时再切换回release版本。关于快照版本，可参考[Maven 快照(SNAPSHOT)](https://www.runoob.com/maven/maven-snapshots.html)进一步了解
* 在完成打包后已经显示了host app配置依赖的完整步骤，如果不考虑将aar部署到私服，按上面步骤配置即可，aar直接指向本机本地仓库

### 3. maven 私服搭建及本地上传配置

使用Nexus搭建，是比较成熟的方案，网上已有比较多的搭建经验分享，这里就不班门弄斧了，假定已经具备maven私服，并且为本次Flutter Module的aar管理已经创建好对应的仓库。

本地上传相关账号配置，也可参考[Flutter 1.12后 上传aar至maven私服](https://juejin.im/post/5e3c0ad351882549087d90b5)

### 4. 打包并上传aar脚本实现

进入`flutter build aar`结果打印的本地仓库目录`/Users/peter/Data/AsPrj/flutter_module/build/host/outputs/repo`，可以发现打包工具会将我们所本配置的第三方依赖分别打出我们指定版本的aar，并配套生成对应的pom文件，一个一般复杂功能的Flutter module，aar的文件随便就有几十个。

而我们如果要将打包出来的aar上传到私服并能够完整使用，就必须将本次构建的所有aar及pom文件全部上传，这一步使用手工上传是不现实的，编写脚本找到所有aar及pom文件并逐一上传显得很有必要。

本文脚本实现思路主是利用shell脚本语法进行遍历找所有`.aar`结尾及`.pom`结尾文件，并做完整性判断，以免上传不全导致私服仓库被污染。

下为脚本明细：

```shell
#!/bin/bash

flutter clean
flutter build aar --build-number 1.0.1-SNAPSHOT

# 定义用于aar、pom文件目录存放的数组
aars=()
poms=()
# 指定打包后本地仓库的目录，由于这里将此脚本放在flutter module根目录，因此直接配置了flutter module根目录下相对目录
targetPath="build/host/outputs/repo"

# 定义遍历找到所有pom文件和aar文件的函数
# 参数$1：当前查找的目录名
function findAarPom(){
	echo "查找此目录是否有aar及pom：$1"
	targetDir=`ls $1`
	for fileName in $targetDir
	do
		if [[ -d  $1"/"$fileName ]]; then
			# 还是目录，则递归找下一级
			findAarPom $1"/"$fileName
		else
		  # 如果是文件，判断后缀，如果符合期望，则将文件路径拼接好放于对应数组最后一位
			if [[ ${fileName:0-4} == '.aar' ]]; then
				aars[${#aars[@]}]=$1"/"$fileName
			elif [[ ${fileName:0-4} == '.pom' ]]; then
				poms[${#poms[@]}]=$1"/"$fileName
			fi
		fi
	done
}

findAarPom $targetPath
echo "============"
echo "aar有：《共${#aars[@]}个》"
echo "${aars[@]}"
echo "pom有：《共${#poms[@]}个》"
echo "${poms[@]}"
echo "============"

# 一个aar文件必然对应会有一个pom文件，如果数量不对，一定是打包出错
if [[ ${#aars[@]} -ne ${#poms[@]} ]]; then
	echo "-- !!! pom文件与aar不对称，请检查aar打包配置，上传任务 退出 !!! --"
    exit 1
fi
if [[ ${#aars[@]} == 0 ]]; then
	echo "-- !!! 未找到aar文件，请检查aar打包配置，上传任务 退出 !!! --"
    exit 1
fi

# 定义将目标pom及aar上传到maven指定仓库的函数
# 参数$1：为pom文件
# 参数$2：为aar文件
function upload(){
	echo "开始上传："
	echo $1
	echo $2
	# mvn上传命令，这里由于将上传用户名密码配置于全局maven settings.xml，则无需再指定用户名密码
	mvn deploy:deploy-file \
	-DpomFile="$1" \
	-DgeneratePom=false \
	-Dfile="$2" \
	-Durl="http://xxx.xxx.xxx.xxx:8081/repository/app-snapshot" \
	-DrepositoryId="nexus-releases" \
	-Dpackaging=aar
}

# 循环上传
for (( i=0;i<${#aars[@]};i++ )); do
    echo "正在处理第$[$i+1]个，共${#aars[@]}个"
    upload "${poms[$i]}" "${aars[$i]}"
done
```







