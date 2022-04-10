---
title: "Hugo使用技巧"
date: 2022-04-10T08:20:32+08:00
tags: [ "hugo" ]
featured_image: ""
description: ""
---

## 如何引用站内文章

```html
{{\< ref "blog/post.md" >}} => https://example.com/blog/post/
{{\< ref "post.md#tldr" >}} => https://example.com/blog/post/#tldr:caffebad
{{\< relref "post.md" >}} => /blog/post/
{{\< relref "blog/post.md#tldr" >}} => /blog/post/#tldr:caffebad
{{\< ref "#tldr" >}} => #tldr:badcaffe
{{\< relref "#tldr" >}} => #tldr:badcaffe
```




