---
title: "如何实现一个数据库"
date: 2022-04-26T17:28:19+08:00
tags: []
featured_image: ""
description: ""
toc: true
draft: false
---

## 一个数据库系统有哪些部分组成？

* SQL语法定义（如果和标准SQL有区别或扩展）
* SQL解析器
* SQL优化器
* 执行器：分析和查询的区分
* 存储层

实现数据库常用的技术有哪些呢？

* undo log
* WAL
* 关系模型
* 索引

## 用C构建一个sqlite数据库

如果你搜索从零构建数据库，一定会得到这个老哥的博客：[Let's Build a Simple Database](https://cstack.github.io/db_tutorial/)

他用C语言实现了一个sqlite数据库，教程也挺完整，包括B树的实现，非常推荐。


## 关系模型理解

这位老哥的文章：[How to Build a Relational Database From Scratch](https://medium.com/swlh/how-to-build-a-relational-database-from-scratch-e208061027c7)

然后他用[python代码](https://github.com/cosmic-cortex/relational-databases-from-scratch)简单描述了下，可以用来学习关系模型的交并差。

## 用Go语言实现简单内存数据库

[https://github.com/eatonphil/gosql](https://github.com/eatonphil/gosql)

这哥们实现了Insert、Create、Select的语法解析，纯手写。查询只支持Filter。
存储层只实现了内存版本。

* Writing a SQL database from scratch in Go
* Binary expressions and WHERE filters
* Indexes
* A database/sql driver

其目标是向PG前进，现在没有SQL优化器，这个进度算是5%吧。

## 用Go实现内存版MySQL数据库 

[https://github.com/dolthub/go-mysql-server](https://github.com/dolthub/go-mysql-server)

除了存储引擎是内存，其他部分基本上都包括，遵循MySQL的连接协议，还有查询计划优化。

## Go实现的PG内存版测试数据库

[https://github.com/proullon/ramsql](https://github.com/proullon/ramsql)

看起来很简单，也没有SQL优化部分。

## Cockroach DB

[https://github.com/cockroachdb/cockroach](https://github.com/cockroachdb/cockroach)

小强DB就是个完整的KV分布式数据库了，基于Go实现，已经可以用于生产。

看记录从16年就开始开源了。

## 用Go实现一个嵌入式时序数据库

其博客地址：[Write a time-series database engine from scratch](https://nakabonne.dev/posts/write-tsdb-from-scratch/)

[其Github地址](https://github.com/nakabonne/tstorage)

他还没有实现SQL接口，只实现了代码层面的访问，还是一个存储层。


