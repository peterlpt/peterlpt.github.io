---
title: "修复升级Catalina后Jekyll本地预览启动失败"
categories: [Mac]
tags: [Jekyll, Homebrew, Catalina]
date: 2019-12-02T16:09:12+08:00
draft: false
---

修复MacOS从Mojave升级到Catalina后，bundle启动Jekyll本地预览失败，提示“-bash: /usr/local/bin/bundle: /System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/bin/ruby: bad interpreter: No such file or directory”问题

这两天相对有空，决定升级MacOS到Catalina，之前已经留意到该系统相关的兼容问题，已经有所心理准备：MacOS Catalina（10.15）不再支持 32 位应用，这也是首个**只支持 64 位应用程序**的 macOS 版本，亦即意味着有数量相当可观的一批旧应用将不能在新系统中运行。[Apple 列出 235 个与 macOS Catalina 不兼容的应用](https://www.oschina.net/news/110544/235-apps-incompatible-with-catalina)

### 问题现象

用Bundle启动Jekyll本地预览报如下错误：

```shell
$ bundle exec jekyll serve
-bash: /usr/local/bin/bundle: /System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/bin/ruby: bad interpreter: No such file or directory
```

### 问题分析解决

刚更新了系统，因此想到需要更新一下gem：

```shell
$ gem update --system
Updating rubygems-update
Fetching rubygems-update-3.0.6.gem
ERROR:  While executing gem ... (Gem::FilePermissionError)
    You don't have write permissions for the /Library/Ruby/Gems/2.6.0 directory.
```

那应该是指向了系统的ruby去了，验证一下果不其然：

```shell
$ which ruby
/usr/bin/ruby
```

接着想找回原来的ruby安装目录，brew list一下发现找不到，如果可以找到，我们应该只需要修改一下环境变量就好。

我们用brew重新安装ruby

```shell
$ brew install ruby
…
You may want to add this to your PATH.

ruby is keg-only, which means it was not symlinked into /usr/local,
because macOS already provides this software and installing another version in
parallel can cause all kinds of trouble.

If you need to have ruby first in your PATH run:
  echo 'export PATH="/usr/local/opt/ruby/bin:$PATH"' >> ~/.bash_profile

For compilers to find ruby you may need to set:
  export LDFLAGS="-L/usr/local/opt/ruby/lib"
  export CPPFLAGS="-I/usr/local/opt/ruby/include"

For pkg-config to find ruby you may need to set:
  export PKG_CONFIG_PATH="/usr/local/opt/ruby/lib/pkgconfig"
```

按上面提示设置一下环境变量

gem、bundle随着ruby的安装而安装，再执行更新gem：

```shell
$ which gem
/usr/local/opt/ruby/bin/gem
$ which bundle
/usr/local/opt/ruby/bin/bundle
$ gem update --system
Latest version already installed. Done.
```

尝试启动jekyll

```shell
$ bundle exec jekyll serve
Traceback (most recent call last):
2: from /usr/local/opt/ruby/bin/bundle:23:in `<main>'
1: from /usr/local/Cellar/ruby/2.6.5/lib/ruby/2.6.0/rubygems.rb:303:in `activate_bin_path'
/usr/local/Cellar/ruby/2.6.5/lib/ruby/2.6.0/rubygems.rb:284:in `find_spec_for_exe': Could not find 'bundler' (2.0.1) required by your /Users/xxx/Data/jekyllPrj/peterlpt.github.io/Gemfile.lock. (Gem::GemNotFoundException)
To update to the latest version installed on your system, run `bundle update --bundler`.
To install the missing version, run `gem install bundler:2.0.1`
```

按提示我们更新一下bundle、安装缺失版本

```shell
$ bundle update --bundler
You must use Bundler 2 or greater with this lockfile.
$ gem install bundler:2.0.1
Fetching bundler-2.0.1.gem
bundler's executable "bundle" conflicts with /usr/local/lib/ruby/gems/2.6.0/bin/bundle
Overwrite the executable? [yN]  y
bundler's executable "bundler" conflicts with /usr/local/lib/ruby/gems/2.6.0/bin/bundler
Overwrite the executable? [yN]  y
Successfully installed bundler-2.0.1
Parsing documentation for bundler-2.0.1
Installing ri documentation for bundler-2.0.1
Done installing documentation for bundler after 3 seconds
1 gem installed
```

再次尝试启动jekyll

```shell
$ bundle exec jekyll serve
Could not find concurrent-ruby-1.1.5 in any of the sources
Run `bundle install` to install missing gems.
```

安装项目依赖的所有gem包

注：此命令会尝试更新系统中已存在的gem包

```shell
$ bundle install
……
Bundle complete! 1 Gemfile dependency, 85 gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.
```

终于可以运行：

```shell
$ bundle exec jekyll serve
Configuration file: /Users/xxx/Data/jekyllPrj/peterlpt.github.io/_config.yml
       Deprecation: The 'gems' configuration option has been renamed to 'plugins'. Please update your config file accordingly.
            Source: /Users/xxx/Data/jekyllPrj/peterlpt.github.io
       Destination: /Users/xxx/Data/jekyllPrj/peterlpt.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
       Jekyll Feed: Generating feed for posts
   GitHub Metadata: No GitHub API authentication could be found. Some fields may be missing or have incorrect data.
    Liquid Warning: Liquid syntax error (line 2): Expected dotdot but found pipe in "{{(site.github.public_repositories | sort: 'stargazers_count') | reverse }}" in pages/open-source.md
                    done in 6.972 seconds.
 Auto-regeneration: enabled for '/Users/xxx/Data/jekyllPrj/peterlpt.github.io'
    Server address: http://127.0.0.1:4000
  Server running... press ctrl-c to stop.
```

