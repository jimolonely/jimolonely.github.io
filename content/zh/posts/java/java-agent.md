---
title: "Java Agent如何调试"
date: 2022-05-14T06:51:46+08:00
tags: [ "java", "javaagent" ]
featured_image: ""
description: ""
toc: true
---

做agent开发必备调试技巧：如何在IDEA里debug agent代码。

我们知道，agent一般是增强代码功能，常用的方式就是提供一个agent的jar包，应用通过 `-javaagent:xxx-agent.jar`这样启动。

```shell
java -javaagent:D:\agent\aweson-agent.jar -jar app.jar
```

使用起来简单，但开发起来就会遇到一个问题：如何调试？

总不能靠输入输出吧，当然不需要，比较IDEA还是很强大的。

## 先写一个简单的agent

建一个子模块 `method-agent`，这很重要。

我们以一个例子来演示：agent拦截类的方法，前后加入一句开始结束输出。

通过maven插件生成 `MANIFEST.MF` 的定义：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.0.0</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <transformers>
                    <transformer
                            implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <manifestEntries>
                            <Premain-Class>com.jimo.MethodAgent</Premain-Class>
                            <!-- <Agent-Class>com.jimo.MethodAgent</Agent-Class>-->
                            <Can-Redefine-Classes>true</Can-Redefine-Classes>
                        </manifestEntries>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

我们只拦截自己写的类, 这里用到了 `javassist` 来增强类。

```xml
<dependency>
    <groupId>org.javassist</groupId>
    <artifactId>javassist</artifactId>
    <version>3.29.0-GA</version>
</dependency>
```

```java
import javassist.*;

import java.io.IOException;
import java.lang.instrument.Instrumentation;

public class MethodAgent {
    public static void premain(String agentArgs, Instrumentation inst) {
        System.out.println("premain");
        inst.addTransformer((loader, className, classBeingRedefined, protectionDomain, classfileBuffer) -> {

            String classPkgName = className.replace('/', '.');
            if (!classPkgName.startsWith("com.jimo")) {
                return classfileBuffer;
            }

            ClassPool pool = ClassPool.getDefault();
            try {
                CtClass ctClass = pool.get(classPkgName);
                CtMethod[] methods = ctClass.getDeclaredMethods();
                for (CtMethod method : methods) {
                    method.insertBefore("System.out.println(\"方法" + method.getName() + "开始\");");
                    method.insertAfter("System.out.println(\"方法" + method.getName() + "结束\");");
                }
                return ctClass.toBytecode();
            } catch (NotFoundException | CannotCompileException | IOException e) {
                e.printStackTrace();
            }
            return classfileBuffer;
        });
    }
}
```

然后打包，记录 `method-agent.jar`的完整路径。

## 应用端使用agent

建一个新的子模块`app`，和 `agent`模块在一个项目里,这很重要，只有这样才能debug。

里面就一个Main类：

```java
package com.jimo.app;

public class Main {

    public static class A {
        public void sayHello() {
            System.out.println("sayHello呀！");
        }
    }

    public static void main(String[] args) {
        System.out.println(add(122, 345));

        new A().sayHello();
    }

    public static int add(int a, int b) {
        return a + b;
    }
}
```

运行的时候指定 `agent.jar`的路径：

![image](https://user-images.githubusercontent.com/17684996/170643818-c4736347-68cd-442c-8dd4-4a3a8a9cdf1c.png)

运行可以看到如下结果：

```shell
premain
方法main开始
方法add开始
方法add结束
467
方法sayHello开始
sayHello呀！
方法sayHello结束
方法main结束
```

## 如何debug

很简单，只需要在agent的代码里打断点就行，如下图：

![image](https://user-images.githubusercontent.com/17684996/170643994-6049ce8e-4b53-4295-b80d-3a360ac1ea75.png)

### 为什么可以？

因为在同一个项目里，IDEA可以找到对应的源码。

### 如果修改了代码怎么办？

需要重新打包，才能生效。  

