---
title: "Hugo递归遍历目录生成文件列表"
date: 2022-07-30T18:07:01+08:00
tags: [ "hugo" ]
featured_image: ""
description: ""
---

建一个partials/walk-dir.html，里面递归调用

```json
{{ $path     := . }}
<ul>
  {{range (readDir $path) }}
  <li> {{.Name}}
    {{if .IsDir}}
    {{ partial "walk-dir.html" (path.Join $path .Name) }}
    {{end}}
  </li>
  {{end}}
</ul>
```

再建一个shortcodes/list-dir.html 来调用

```json
{{ $path     := .Get "path" }}
{{ partial "walk-dir.html" $path }}
```

最后，使用

```json
{{< list-dir path= "content/zh/read" >}}
```

效果