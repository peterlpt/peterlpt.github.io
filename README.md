我的个人博客：<http://ptlpt.gitee.io>。

[Hugo](https://gohugo.io/)框架，[LoveIt](https://github.com/dillonzq/LoveIt.git)主题

### 本地预览博客

使用以下命令启动网站:

```bash
hugo serve
```

去查看 [http://localhost:1313](http://localhost:1313)

当文件内容更改时, 页面会随着更改自动刷新。

由于本主题使用了 Hugo 中的 `.Scratch` 来实现一些特性, 可为 `hugo server` 命令添加 `--disableFastRender` 参数来实时预览正在编辑的文章页面。

```bash
hugo serve --disableFastRender
```

`hugo serve` 的默认运行环境是 `development`, 而 `hugo` 的默认运行环境是 `production`.

由于本地 `development` 环境的限制, **评论系统**, **CDN** 和 **fingerprint** 不会在 `development` 环境下启用.

可以使用 `hugo serve -e production` 命令来开启这些特性。

```bash
hugo serve -e production
```

### 更多博客主题配置说明

[LoveIt主题配置说明](https://hugoloveit.com/zh-cn/posts/)

