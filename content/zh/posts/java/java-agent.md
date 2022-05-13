---
title: "Java Agent如何调试"
date: 2022-05-14T06:51:46+08:00
tags: [ "java", "javaagent" ]
featured_image: ""
description: ""
toc: true
---

通过Java Agent，可以无侵入性地增强代码功能，或者拦截代码做一些统计。

比如，在监控系统里，采集应用端的各种代码执行情况就是使用 agent，典型的开源项目 [pinpoint](https://pinpoint-apm.gitbook.io/pinpoint/).

其最基本使用是通过命令行参数`-javaagent:xxx-agent.jar`指定:

```shell
$ java -jar -javaagent:pinpoint-agent.jar testapp.jar
```

而在开发过程中，用打日志的方式显然不好调试，所以测试了下在IDEA里，这种agent也是可以debug的。

# 准备测试项目

TODO

# 在IDEA里如何调试

## 代码在同一个项目里

## 安装agent

## debug

## 修改代码后


