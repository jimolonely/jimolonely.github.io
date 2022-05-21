---
title: "ShardingSphere开发者编译指南"
date: 2022-05-21T09:54:43+08:00
tags: [ "shardingsphere" ]
featured_image: ""
description: ""
toc: true
---

需要对接ShardingSphere（后简称SS）做一些开发，所以要编译这个庞大的项目。

## 环境

[文档](https://shardingsphere.apache.org/community/en/contribute/establish-project/), 注意一点就是开启git的长文件名,不然clone到一半发现失败了。
```shell
git config --global core.longpaths true
```

## 编译

[文档](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-scaling/build/)

因为SS使用了Antlr，需要全局编译才能触发Antlr插件，否则这部分代码编译不到。使用 `release`的profile。
```shell
mvn install -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true -Drat.skip=true -Djacoco.skip=true -DskipITs -DskipTests -Prelease
```

![image](https://user-images.githubusercontent.com/17684996/169630753-8cab1f02-ec34-4377-90c9-7f6ddfdc4618.png)

## 运行调试

SS的代码有200多个模块，分得非常细，我这边主要对接 proxy端，所以可以一眼发现 proxy模块。

对应的启动类位于 `shardingsphere-proxy-bootstrap`模块的 `Bootstrap` 类。

要调试的话，将 `resources/conf/server.yaml` 里面修改一些配置:

因为我只是本地调试其他功能，所以不需要集群模式，关于完整模式请看[文档](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-jdbc/yaml-config/mode/).

下面是配置SS的用户名和密码，这里配了2个用户，root和sharding。

```yaml
mode:
  type: Memory

rules:
  - !AUTHORITY
    users:
      - root@%:root
      - sharding@:sharding
    provider:
      type: ALL_PERMITTED
```

然后是数据源的配置，修改 `conf/config-sharding.yaml`:

因为这里不需要分片，所以只配了一个mysql实例，逻辑数据库名称是db，mysql里的数据库是 ds01.
```yaml
databaseName: db

dataSources:
  ds01:
    url: jdbc:mysql://localhost:3306/ds01
    username: root
    password: 123456
```
因为需要用到mysql，这里用docker在本地起一个：
```shell
docker run --name mysql-ds01 -d -e MYSQL_DATABASE=ds01 -e MYSQL_ROOT_PASSWORD=123456 -p 3306:3306 mysql:5.7
```

现在就可以启动SS proxy了，运行 `org.apache.shardingsphere.proxy.Bootstrap`的main方法，成功可以看到日志：

```shell
[INFO ] 2022-05-21 14:30:50.607 [main] o.a.s.d.p.s.r.s.RuleAlteredContextManagerLifecycleListener - mode type is not Cluster, mode type='Memory', ignore
[INFO ] 2022-05-21 14:30:50.710 [main] o.a.s.p.v.ShardingSphereProxyVersion - Database name is `MySQL`, version is `5.7.38`, database name is `db`
[INFO ] 2022-05-21 14:30:52.037 [main] o.a.s.p.frontend.ShardingSphereProxy - ShardingSphere-Proxy Memory mode started successfully
```

然后写一个测试代码:

注意，这里连接的是SS的逻辑数据库 db, 端口也是SS的3307.

```java
@Test
public void test() throws SQLException {

    try (Connection conn = DriverManager.getConnection("jdbc:mysql://127.0.0.1:3307/db", "root", "root")) {
        final Statement stmt = conn.createStatement();

        stmt.executeUpdate("create table t1(a int, b varchar(10))");

        stmt.executeUpdate("insert into t1 values(1,'hello'),(2,'jimo')");

        final ResultSet rs = stmt.executeQuery("select count(1) from t1");
        rs.next();
        assertEquals(2, rs.getInt(1));
    }
}
```

接下来就可以在本地debug了。



