---
title: "必会Shell命令"
date: 2022-04-11T14:38:12+08:00
toc: true
---

这些命令并不是先学再用，而是用到再学。一般是从一个问题出发，比如：如何通过名称查询一个进程？

## 如何查询一个进程命令的目录（工作空间）？

### 方法1：pwdx pid
命令：pwdx得到的是启动一个命令的路径。

```shell
pwdx ${PID}
```

案例：知道nginx进程ID，需要知道nginx安装在哪？

```shell
# pwdx 1498563
1498563: /export/server/openresty/nginx/conf/conf.d
```

### 方法2：lsof -p pid

该命令可得到更完整的进程信息。

```shell
# lsof -p 1498563 | grep cwd
nginx   1498563 root  cwd    DIR               8,32       37  5371079203 /export/server/openresty/nginx/conf/conf.d
```

### 方法3：/proc/pid/cwd

```shell
# readlink /proc/1498563/cwd
/export/server/openresty/nginx/conf/conf.d
```

