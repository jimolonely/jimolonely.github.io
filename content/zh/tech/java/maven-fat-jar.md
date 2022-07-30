---
title: "Maven打Fat Jar的一些问题"
date: 2022-05-14T07:01:22+08:00
tags: [ "maven","shade" ]
featured_image: ""
description: ""
---

总是忘记maven-shade插件的配置，这里记录一下。

见[文档](https://maven.apache.org/plugins/maven-shade-plugin/examples/executable-jar.html).

```xml
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.3.0</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <transformers>
                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                  <mainClass>com.jimo.Main</mainClass>
                </transformer>
              </transformers>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
```

##  Invalid signature file digest for Manifest main attributes

有时我们在打完jar包运行会报错：

```shell
Error: A JNI error has occurred, please check your installation and try again
Exception in thread "main" java.lang.SecurityException: Invalid signature file digest for Manifest main attributes
        at sun.security.util.SignatureFileVerifier.processImpl(Unknown Source)
        at sun.security.util.SignatureFileVerifier.process(Unknown Source)
        at java.util.jar.JarVerifier.processEntry(Unknown Source)
        at java.util.jar.JarVerifier.update(Unknown Source)
        at java.util.jar.JarFile.initializeVerifier(Unknown Source)
        at java.util.jar.JarFile.ensureInitialization(Unknown Source)
        at java.util.jar.JavaUtilJarAccessImpl.ensureInitialization(Unknown Source)
        at sun.misc.URLClassPath$JarLoader$2.getManifest(Unknown Source)
        at java.net.URLClassLoader.defineClass(Unknown Source)
        at java.net.URLClassLoader.access$100(Unknown Source)
        at java.net.URLClassLoader$1.run(Unknown Source)
        at java.net.URLClassLoader$1.run(Unknown Source)
        at java.security.AccessController.doPrivileged(Native Method)
        at java.net.URLClassLoader.findClass(Unknown Source)
        at java.lang.ClassLoader.loadClass(Unknown Source)
        at sun.misc.Launcher$AppClassLoader.loadClass(Unknown Source)
        at java.lang.ClassLoader.loadClass(Unknown Source)
        at sun.launcher.LauncherHelper.checkAndLoadMain(Unknown Source)
```

这是由于混在一起后，很多其他文件在 `META-INF`下导致的，比如下面的 ECLIPSE.RSA, ECLIPSE.SF.

![image](https://user-images.githubusercontent.com/17684996/168400053-8ef6f909-597f-4095-8db2-744fd15858b8.png)

解决办法很简单，要么手动删除，要么通过插件配置过滤：

```xml
    <configuration>
        <transformers>
            <transformer
                    implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                <mainClass>org.urbcomp.start.db.test.MiniHBaseCluster</mainClass>
            </transformer>
        </transformers>
        <filters>
            <filter>
                <artifact>*:*</artifact>
                <excludes>
                    <exclude>META-INF/*.SF</exclude>
                    <exclude>META-INF/*.DSA</exclude>
                    <exclude>META-INF/*.RSA</exclude>
                </excludes>
            </filter>
        </filters>
    </configuration>
```

