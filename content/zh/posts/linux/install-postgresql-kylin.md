---
title: "在麒麟linux上安装Postgresql12.5"
date: 2022-04-14T19:17:12+08:00
tags: []
featured_image: ""
description: "install postgresql 12.5 on kylin v10 linux"
toc: true
---

本文主要实践在麒麟V10版本上通过源码编译安装PostgreSQL12.5，因为是源码编译，所以对于其他版本也具有参考性。

## 麒麟版本

V10
```shell
$ uname -a
Linux kylin-v10-02 4.19.90-24.4.v2101.ky10.aarch64 #1 SMP Mon May 24 14:45:37 CST 2021 aarch64 aarch64 aarch64 GNU/Linux
```

## 下载PostgreSQL12.5源码

下载地址为官网：[https://www.postgresql.org/ftp/source/v12.5/](https://www.postgresql.org/ftp/source/v12.5/)

上传解压到服务器。

## 编译源码过程

[官方文档](http://www.postgres.cn/docs/12/install-procedure.html).

运行
```shell
./configure
```
得到如下报错：
```shell
error: readline library not found
```
需要安装`readline`
```shell
yum install readline-devel -y
```

然后再运行 `./configure`

没有报错后，再运行
```shell
make
```
编译完成后得到成功提示

![image](https://user-images.githubusercontent.com/17684996/163381674-d74bd018-f281-4fc2-9c3f-8ca1380761e1.png)

然后开始安装, 运行
```shell
make install
```

安装成功后可以看到

![image](https://user-images.githubusercontent.com/17684996/163381744-5fd5f7ee-5f07-4776-99d2-f702bfbe2c29.png)

默认安装路径为 `/usr/local/pgsql`，我们需要将权限改为`postgres`
```shell
[root@kylin-v10-02 local]# useradd postgres
[root@kylin-v10-02 local]# chown -R postgres:postgres /usr/local/pgsql/
```

![image](https://user-images.githubusercontent.com/17684996/163381830-09b55013-d3ca-4831-bb51-b3ab2bf07f4e.png)

## 配置环境变量
安装完成后，还要做一些配置操作.

`vim /etc/profile`加入下面的环境变量配置
```shell
export PATH=/usr/local/pgsql/bin:$PATH

LD_LIBRARY_PATH=/usr/local/pgsql/lib
export LD_LIBRARY_PATH
```
然后刷新配置：
```shell
source /etc/profile
```

## 启动服务

在启动PG之前，需要准备一个数据目录，注意修改权限：
```shell
mkdir -p /export/pgdata
chown -R postgres:postgres /export/pgdata
```

初始化数据库，注意，要切换到postgres用户
```shell
# sudo su - postgres
initdb -D /export/pgdata/
```
完整输出如下：
```shell
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "zh_CN.UTF-8".
The default database encoding has accordingly been set to "UTF8".
initdb: could not find suitable text search configuration for locale "zh_CN.UTF-8"
The default text search configuration will be set to "simple".

Data page checksums are disabled.

fixing permissions on existing directory /export/pgdata ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Asia/Shanghai
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

initdb: warning: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    pg_ctl -D /export/pgdata/ -l logfile start

启动PG
```

那我们就启动服务：
```shell
[postgres@kylin-v10-02 ~]$ pg_ctl -D /export/pgdata/ -l logfile start
waiting for server to start.... done
server started
```

## 测试

```shell
[postgres@kylin-v10-02 ~]$ psql
psql (12.5)
Type "help" for help.

postgres=# \l
List of databases
Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
postgres  | postgres | UTF8     | zh_CN.UTF-8 | zh_CN.UTF-8 |
template0 | postgres | UTF8     | zh_CN.UTF-8 | zh_CN.UTF-8 | =c/postgres          +
|          |          |             |             | postgres=CTc/postgres
template1 | postgres | UTF8     | zh_CN.UTF-8 | zh_CN.UTF-8 | =c/postgres          +
|          |          |             |             | postgres=CTc/postgres
(3 rows)
```

测试ok，不过要提供给其他机器用，还需要一些监听和安全配置。

## 修改密码
```sql
postgres=# alter user postgres password 'postgres';
ALTER ROLE
```
## 修改监听地址
默认情况是在localhost上监听
`vim /export/pgdata/postgresql.conf` ，修改监听地址
```shell
listen_addresses = '*'
```
还要允许密码访问，`vim /export/pgdata/pg_hba.conf`，在最后加入：
```shell
host  all  all 0.0.0.0/0 md5
```
然后重启服务
```shell
pg_ctl -D /export/pgdata/ -l logfile restart
```

下一篇，我们还会接着安装 `PostGIS`: [在麒麟Linux安装Postgis]({{< relref "install-postgis-kylin" >}}) .

