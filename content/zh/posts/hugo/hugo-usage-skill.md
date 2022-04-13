---
title: "Hugo使用技巧"
date: 2022-04-10T08:20:32+08:00
tags: [ "hugo" ]
featured_image: ""
description: ""
toc: true
---

## 如何引用站内文章

下面的 `\\`去掉
```html
{{</* ref "blog/post.md" */>}} => https://example.com/blog/post/
{{</* ref "post.md#tldr" */>}} => https://example.com/blog/post/#tldr:caffebad
{{</* relref "post.md" */>}} => /blog/post/
{{</* relref "blog/post.md#tldr" */>}} => /blog/post/#tldr:caffebad
{{</* ref "#tldr" */>}} => #tldr:badcaffe
{{</* relref "#tldr" */>}} => #tldr:badcaffe
```

## 新窗口打开链接

根据文档：[https://gohugo.io/getting-started/configuration-markup#blackfriday](https://gohugo.io/getting-started/configuration-markup#blackfriday)

以前默认的markdown解析引擎是 `blackfriday`,现在换成了 `GoldMark`。
但是没发现 `GlodMark`有配置 `a`标签 `blank`的属性，所以暂时用 `blackfriday`。

```toml
[markup]
defaultMarkdownHandler = 'blackfriday'
[markup.blackFriday]
hrefTargetBlank = true
```

会在命令行收到弃用的警告：

```shell
markup.defaultMarkdownHandler=blackfriday is deprecated and will be removed in a future release. See https://gohugo.io//content-management/formats/#list-of-content-formats
```

## 如何左对齐、右对齐内容

通过自定义[shortcodes](https://gohugo.io/content-management/shortcodes/) 来实现。
参考文章：[https://guanqr.com/tech/website/hugo-shortcodes-customization/](https://guanqr.com/tech/website/hugo-shortcodes-customization/)

```html
<!-- 文件位置：~/layouts/shortcodes/align.html -->
<div style="text-align:{{ index .Params 0 }}">{{ markdownify .Inner }}</div>
```

然后在文章里使用：

```markdown
{{</* align left */>}}
内容
{{</* /align */>}}
```

## 如何阻止Hugo渲染Shortcodes

需要在 `{{</*` 和 `*/>}}`之间加上注释 `/* ... */`
```markdown
{{</*/* align left */*/>}}
```

