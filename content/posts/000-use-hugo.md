+++ 
draft = false
date = 2022-04-08T15:49:26+08:00
title = ""
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

创建文章

```shell
hugo new posts/000-use-hugo.md
```

启动服务, `-D`代表也显示草稿`draft=true`.

```shell
hugo server -D
```

自定义主题

```shell
git init
git submodule add https://github.com/luizdepra/hugo-coder.git themes/hugo-coder
```

然后在 `config.toml`里配置主题名字：

```shell
theme = "hugo-coder"
```

# 构建静态网页

```shell
hugo -D
```

输出位于 `public`目录下。


