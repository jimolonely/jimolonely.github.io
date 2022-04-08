+++ 
draft = false
date = 2022-04-08T15:49:26+08:00
title = "Hugo使用入门"
description = "安装Hugo，编译成静态网页，发布到Github Pages"
slug = "blog-by-hugo-and-github-pages"
authors = ["jimo"]
tags = ["hugo"]
categories = []
externalLink = ""
series = []
+++


# 启动服务

`-D`代表也显示草稿`draft=true`.

```shell
hugo server -D
```

正常情况应该一片空白，下面我们来创建主页。

# 创建主页

```shell
hugo new index.md
```

然后在里面随便写点什么。

# 重要配置
在 `config.toml`里配置。

## 基础配置

语言，域名，博客标题

```toml 
baseURL = 'http://jimolonely.github.io/'
languageCode = 'zh-cn'
title = 'jimo power'
```

## 配置菜单

注意和语言对应
```toml
[[languages.zh-cn.menu.main]]
name = "About"
weight = 1
url = "about/"

[[languages.zh-cn.menu.main]]
name = "Blog"
weight = 2
url = "posts/"

[[languages.zh-cn.menu.main]]
name = "Projects"
weight = 3
url = "projects/"

[[languages.zh-cn.menu.main]]
name = "Contact me"
weight = 5
url = "contact/"
```


# 自定义主题

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

# 目录层级介绍

[https://gohugo.io/getting-started/directory-structure/](https://gohugo.io/getting-started/directory-structure/)

* archetypes
* content
* config
* data
* layouts
* static
* assets

可以参考主题 `themes`下某个主题的结构。


