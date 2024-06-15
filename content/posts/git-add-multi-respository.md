---
title: "git添加多个远程仓库的几种方式"
categories: [git]
tags: [git]
date: 2024-06-15T22:08:12+08:00
draft: false
---

本文主要介绍添加git仓库的几种方式
方法一：命令行添加
方法二：通过git的config文件添加

<!--more-->

### 1. 通过命令添加多个仓库

1. 查看现有几个仓库

   ```shell
   # 命令
   git remote -v
   # 结果
   origin	https://xxxx/xxx/xxx.git (fetch)
   origin	https://xxxx/xxx/xxx.git (push)
   ```

2. 添加一个远程仓库

   添加一个远程库 名字不能是origin

   ```shell
   # 命令
   git remote add <仓库名称>  仓库地址   
   # 示例
   git remote add testAdd  https://git.test.com/tsest/test.git
   # 查询结果
   git remote -v
   testAdd	https://git.test.com/tsest/test.git (fetch)
   testAdd	https://git.test.com/tsest/test.git (push)
   origin	https://xxxx/xxx/xxx.git (fetch)
   origin	https://xxxx/xxx/xxx.git (push)
   ```

3. 推送拉取代码

   ```shell
   # 拉代码
   git pull testAdd    远程分支名：本地分支名
   # 推代码
   git push testAdd    远程分支名：本地分支名
   ```

### 2. 修改git配置文件添加多个仓库

在本地仓库的config文件夹里面找到.git文件，然后修改

    ```shell
    [core]
      repositoryformatversion = 0
      filemode = true
      bare = false
      logallrefupdates = true
      ignorecase = true
      precomposeunicode = true
    [remote "origin"]
      url = 第一个远程仓库url
      url = 第二个远程仓库url
      url = 。。。
      以此类推
    [branch "branch1"]
      remote = origin
      merge = refs/heads/branch1
    [branch "branch2"]
      remote = origin
      merge = refs/heads/branch2
    ```

推送拉取代码，与推送拉取一个仓库没有区别，直接操作就好

```shell
# 会自动拉取代码
git pull
# 会自动推送代码到两个远程仓库
git push
```

