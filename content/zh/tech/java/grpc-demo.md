---
title: "Grpc练习"
date: 2022-05-26T22:48:27+08:00
tags: [ "grpc", "protobuf", "java" ]
featured_image: ""
description: ""
toc: true
---

练习下Grpc使用。

## proto和Service定义

`src/main/proto/AgentModel.proto`

模型定义2个实体，参数和返回值。
```shell
syntax = "proto3";
option java_package = "com.jimo.grpc";

message AgentInfo {
  string name = 1;
  sint32 index = 2;
}

message ReportResponse {
  bool ok = 1;
  string msg = 2;
}
```

`src/main/proto/AgentService.proto`

在服务这边就定义一个report方法。
```shell
syntax = "proto3";

option java_package = "com.jimo.grpc";

import "AgentModel.proto";

service Agent {
  rpc report(AgentInfo) returns (ReportResponse) {}
}
```

## 编译

加入maven插件，同时编译proto文件和grpc。
```xml
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>1.6.2</version>
            </extension>
        </extensions>
        <plugins>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.6.1</version>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:${protoc.version}:exe:${os.detected.classifier}
                    </protocArtifact>
                    <pluginId>grpc-java</pluginId>
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}
                    </pluginArtifact>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>compile-custom</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
```

假如要用离线的proto执行文件，可以换成本地路径：

```xml
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.6.1</version>
                <configuration>
                    <protocExecutable>D:\software\protoc.exe</protocExecutable>
                    <pluginId>grpc-java</pluginId>
                    <pluginExecutable>D:\software\protoc-gen-grpc-java.exe</pluginExecutable>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>compile-custom</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```

在命令行运行 `mvn compile` 可编译出protobuf和grpc的类，结构如下：

```shell
└─target
    ├─generated-sources
    │  ├─annotations
    │  └─protobuf
    │      ├─grpc-java
    │      │  └─com
    │      │      └─jimo
    │      │          └─grpc
    │      │                  AgentGrpc.java
    │      │
    │      └─java
    │          └─com
    │              └─jimo
    │                  └─grpc
    │                          AgentModel.java
    │                          AgentService.java
```

## 服务端实现

先实现服务的处理逻辑： `AgentServiceImpl` 继承 GRPC生成的抽象类。

```java
import com.jimo.grpc.AgentGrpc;
import com.jimo.grpc.AgentModel;
import io.grpc.stub.StreamObserver;

public class AgentServiceImpl extends AgentGrpc.AgentImplBase {

    @Override
    public void report(AgentModel.AgentInfo request, StreamObserver<AgentModel.ReportResponse> responseObserver) {
        AgentModel.ReportResponse response = AgentModel.ReportResponse.newBuilder().setOk(true).setMsg(request.getName()).build();
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}
```

然后起一个服务：

```java
public static void main(String[] args) throws IOException, InterruptedException {
    Server server = ServerBuilder.forPort(8000)
            .addService(new AgentServiceImpl())
            .build().start();
    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
        try {
            server.shutdown().awaitTermination(10, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }));
    System.out.println("Listening on 8000...");
    server.awaitTermination();
}
```

## 客户端实现

客户端一般共用一个 `Channel`,所以传进来.然后通过 `Stub`调用。

```java
import com.jimo.grpc.AgentGrpc;
import com.jimo.grpc.AgentModel;
import io.grpc.Channel;

public class AgentClient {

    private final AgentGrpc.AgentBlockingStub stub;

    public AgentClient(Channel channel) {
        stub = AgentGrpc.newBlockingStub(channel);
    }

    public void report() {
        AgentModel.ReportResponse res = stub.report(AgentModel.AgentInfo.newBuilder().setName("app01").setIndex(98).build());
        System.out.println("收到回复：" + res);
    }
}
```
客户端共用一个 `Channel`的创建和调用：
```java
public static void main(String[] args) throws InterruptedException {

    // channel最好创建一个，可以共用，是线程安全的
    ManagedChannel channel = ManagedChannelBuilder
            .forAddress("localhost", 8000)
            // 默认是SSL/TLS，这里不用
            .usePlaintext()
            .build();

    AgentClient agentClient = new AgentClient(channel);
    agentClient.report();
    agentClient.report();

    // channel默认会等待一段时间关闭，如果不用了最好及时关闭，否则会占用TCP连接
    channel.shutdownNow().awaitTermination(5, TimeUnit.SECONDS);
}
```

## 更多问题

1、我们需要对请求做拦截，做一些AOP的事情怎么弄？----用 `ClientInterceptor`

2、在上面的客户端使用了 `AgentGrpc.newBlockingStub(channel)`,还有好几种Stub，有什么区别，怎么使用？

3、客户端和服务端通信的网络和协议是如何实现的？为什么客户端的Channel会自动关闭？

 