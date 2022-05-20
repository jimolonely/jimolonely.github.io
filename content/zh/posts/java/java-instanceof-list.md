---
title: "Java instanceof List<T>"
date: 2022-05-20T22:51:28+08:00
tags: [ "java" ]
featured_image: ""
description: ""
---

其实这是一个很简单的问题，在Java中，通常我们要判断一个对象是不是某种类型，会用 `instanceof` 关键字。

但遇到带有泛型的 `List<T>` 是不能直接使用，这时候怎么办呢？

我会写出下面的代码：

```java
if(a instanceof A){
    A a1 = (A)a;    
}

if(a instanceof List && !((List<?>)a).isEmpty() && ((List<?>)a).get(0) instanceof A){
    List<A> list = (List<A>)a;    
}
```

没问题，就是会有警告，目前还没想到更好的办法，欢迎补充。

