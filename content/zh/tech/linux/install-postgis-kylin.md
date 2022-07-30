---
title: "在麒麟Linux安装Postgis"
date: 2022-04-14T20:33:15+08:00
tags: []
featured_image: ""
description: "install postgis 3.0.5 on kylin v10 linux"
toc: true
---

接着上一篇[在麒麟linux上安装Postgresql12.5]({{< relref "install-postgresql-kylin" >}}) ,我们来安装 `PostGIS`插件。

## 方案

因为 `PostgreSQL`不是通过 rpm包安装的，所以即便 `PostGIS`有现成的rpm包，也无法使用（需要引用 `PG`的包）。

所以，我们还是采用源码编译的方式。

## 下载PostGIS源码

我们选择的版本是`3.0.5`, 如果是不同版本，那么后面他所依赖的东西可能略有差别。

下载地址： [http://postgis.net/stuff/postgis-3.0.5.tar.gz](http://postgis.net/stuff/postgis-3.0.5.tar.gz)

## 编译过程

运行 
```shell
./configure
```
报错
```shell
configure: error: could not find geos-config within the current path. 
You may need to try re-running configure with a --with-geosconfig parameter.
```
报差错原因是缺少 `geos`依赖， 那就安装`geos`
```shell
yum install geos geos-devel
```
再运行`./configure`发现报错
```shell
configure: error: could not find proj.h or proj_api.h - 
you may need to specify the directory of a PROJ installation using --with-projdir
```
报差错原因是缺少 `proj`依赖,那就安装
```shell
yum install proj proj-devel
```
再次运行，还报错：
```shell
configure: error: gdal-config not found. 
Use --without-raster or try --with-gdalconfig=<path to gdal-config>
```

这个报错是`PostGIS`需要`gdal`，关于`gdal`，`PostGIS`用来操作栅格数据.
在[文档](https://postgis.net/docs/postgis_installation.html#install_requirements) 中，`gdal`需要的版本是`2+`.

> GDAL, version 2+ is required 3+ is preferred. This is required for raster support. http://trac.osgeo.org/gdal/wiki/DownloadSource.

### gdal源码安装

首先肯定想到有没有现成的 `gdal`安装包，可惜没找到合适的，那还是源码安装吧。

经过我几次尝试，发现 `2.0`版本根本无法编译，而`2.3`,`2.4`版本需要 `proj 6`以上的版本，而我们上面安装的是`proj 4`版本，最终确定`2.1.4`版本是可以编译的。

下面是编译过程。

下载源码：[http://download.osgeo.org/gdal/2.1.4/gdal214.zip](http://download.osgeo.org/gdal/2.1.4/gdal214.zip)

解压到某个路径下。

编译gdal源码，运行
```shell
make
```
这个过程持续了挺长时间，大约20分钟，最终成功后输出.

![image](https://user-images.githubusercontent.com/17684996/163394414-58ed0022-e9a1-415b-a55f-09689c139777.png)


然后安装`gdal`，运行
```shell
make install
```
成功后输出如下，提示安装在 `/usr/local/lib`

![image](https://user-images.githubusercontent.com/17684996/163394489-eca51e1f-db21-4ca4-a6a2-13a8fdaae7b3.png)


## 接着PostGIS编译 

再运行`PostGIS`的`configure`

运行成功，得到下面的信息：

```shell
PostGIS is now configured for aarch64-unknown-linux-gnu

-------------- Compiler Info -------------
C compiler:           gcc -std=gnu99 -g -O2 -fno-math-errno -fno-signed-zeros
CPPFLAGS:              -I/usr/include   -I/usr/include/libxml2    
SQL preprocessor:     /usr/bin/cpp -traditional-cpp -w -P

-------------- Additional Info -------------
Interrupt Tests:   DISABLED use: --with-interrupt-tests to enable

-------------- Dependencies --------------
GEOS config:          /usr/bin/geos-config
GEOS version:         3.6.1
GDAL config:          /usr/local/bin/gdal-config
GDAL version:         2.1.4
PostgreSQL config:    /usr/local/pgsql/bin/pg_config
PostgreSQL version:   PostgreSQL 12.5
PROJ4 version:        49
Libxml2 config:       /usr/bin/xml2-config
Libxml2 version:      2.9.10
JSON-C support:       no
protobuf support:     no
PCRE support:         yes
Perl:                 /usr/bin/perl
Wagyu:                no

--------------- Extensions ---------------
PostGIS Raster:                     enabled
PostGIS Topology:                   enabled
SFCGAL support:                     disabled
Address Standardizer support:       enabled

-------- Documentation Generation --------
xsltproc:             /usr/bin/xsltproc
xsl style sheets:     /usr/share/sgml/docbook/xsl-stylesheets
dblatex:              
convert:              
mathml2.dtd:          http://www.w3.org/Math/DTD/mathml2/mathml2.dtd

configure: WARNING:  --------- GEOS VERSION WARNING ------------
configure: WARNING:   You are building against GEOS 3.6.1.
configure: WARNING:
configure: WARNING:   To take advantage of _all_ the features of
configure: WARNING:   PostGIS, GEOS 3.7.0 or higher is required.
configure: WARNING:
configure: WARNING:   For _most_ features, GEOS 3.6.0 is enough.
configure: WARNING:
configure: WARNING:   We recommend GEOS 3.7.0 or higher
configure: WARNING:
configure: WARNING:   You can download the latest versions from
configure: WARNING:   http://geos.osgeo.org/
configure: WARNING:
```

发现检查没问题后，就基本上离成功不远了。

开始编译postgis源码，运行
```shell
make
```
几分钟后，编译成功，输出

![image](https://user-images.githubusercontent.com/17684996/163394635-d5dff6bf-95a0-4043-b678-2fca0c58007a.png)

然后运行安装命令
```shell
make install
```

安装很快。

## 验证

结束后，我们来验证下。

查看下版本，没问题。

![image](https://user-images.githubusercontent.com/17684996/163394710-db713416-2aa5-463c-83c7-d8369e6b8a7c.png)

建张表，做个简单的空间查询。

```sql
postgres=# create table testg ( id int, geom geometry );
CREATE TABLE
postgres=# insert into testg values (1, ST_GeomFromText('point(116 39)'));
INSERT 0 1
postgres=# select st_astext(geom) from testg where ST_Intersects(ST_GeomFromText('POLYGON((116 39, 116.1 39, 116.1 39.1, 116 39.1, 116 39))'), geom);
st_astext
---------------
POINT(116 39)
(1 row)
```



