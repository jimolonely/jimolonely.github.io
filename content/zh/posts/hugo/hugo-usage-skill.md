---
title: "Hugo使用技巧"
date: 2022-04-10T08:20:32+08:00
tags: [ "hugo" ]
featured_image: ""
description: ""
toc: true
---

## 如何引用站内文章

下面的 `\`去掉
```html
{{\< ref "blog/post.md" >}} => https://example.com/blog/post/
{{\< ref "post.md#tldr" >}} => https://example.com/blog/post/#tldr:caffebad
{{\< relref "post.md" >}} => /blog/post/
{{\< relref "blog/post.md#tldr" >}} => /blog/post/#tldr:caffebad
{{\< ref "#tldr" >}} => #tldr:badcaffe
{{\< relref "#tldr" >}} => #tldr:badcaffe
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
{{< align left >}}
内容
{{< /align >}}
```



