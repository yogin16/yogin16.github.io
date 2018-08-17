---
layout:     post
title:      "Microservice in Java: gRPC server implementation example with Spring and Gradle"
date:       2018-08-18 12:23:58 +0530
comments:   true
---

Notes on Microservice implementation in Java. Implementing example gRPC hello server with Spring and Gradle.

### gRPC
The official website for [grpc](https://grpc.io/docs/guides/) really gives nice detailed introduction to what it is. With microservice architecture, the services will need to have communication with each other. And in distributed system gRPC enables to have seamless and faster integration with polyglot microservices. We just have to write _proto_ files and the language specific stubs for client and server implementations gets _generated_ automatically. Also gRPC uses HTTP/2 which is amazingly fast.
Assuming we already know what is the proto file following shows example of how we can use Gradle and Spring to quickly setup gRPC server in Java in minutes. (If needed to learn about protocol buffers: [google link](https://developers.google.com/protocol-buffers/docs/proto))

### hello.proto

The proto used in the example server:
```proto
syntax = "proto3";

option java_package = "com.example.hello";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

### setting up server

Setup working directory:

```bash
$ mkdir grpc-spring-gradle-example
$ cd grpc-spring-gradle-example/
$ mkdir -p src/main/proto
```

Have to put `hello.proto` in above proto directory:

```bash
$ cd src/main/proto/
$ vim hello.proto     # add above file content and save
```

### build.gradle

Uses gradle plugin and spring & grpc dependencies:
```groovy
apply plugin: 'com.google.protobuf'
apply plugin: 'application'

apply plugin: 'java'

repositories {
    mavenLocal()
    mavenCentral()
}

group 'com.example'
version '0.1'

buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath "com.google.protobuf:protobuf-gradle-plugin:0.8.1"
    }
}

dependencies {
    compile group: 'io.grpc', name: 'grpc-netty', version: '1.1.2'
    compile group: 'io.grpc', name: 'grpc-protobuf', version: '1.1.2'
    compile group: 'io.grpc', name: 'grpc-stub', version: '1.1.2'
    compile group: 'io.grpc', name: 'grpc-services', version: '1.1.2'

    compile group: 'org.springframework', name: 'spring-aop', version: '4.3.5.RELEASE'
    compile group: 'org.springframework', name: 'spring-aspects', version: '4.3.5.RELEASE'
    compile group: 'org.springframework', name: 'spring-beans', version: '4.3.5.RELEASE'
    compile group: 'org.springframework', name: 'spring-context', version: '4.3.5.RELEASE'
    compile group: 'org.springframework', name: 'spring-context-support', version: '4.3.5.RELEASE'
    compile group: 'org.springframework', name: 'spring-core', version: '4.3.5.RELEASE'
    compile group: 'org.springframework', name: 'spring-expression', version: '4.3.5.RELEASE'
    compile group: 'org.springframework', name: 'spring-jdbc', version: '4.3.5.RELEASE'
    compile group: 'org.springframework', name: 'spring-jms', version: '4.3.5.RELEASE'
    compile group: 'org.springframework', name: 'spring-tx', version: '4.3.5.RELEASE'
    compile group: 'org.springframework', name: 'spring-web', version: '4.3.5.RELEASE'
    compile group: 'org.springframework', name: 'spring-test', version: '4.3.5.RELEASE'
}

subprojects {
    repositories {
        mavenLocal()
        mavenCentral()
    }
    tasks.withType(JavaCompile) {
        sourceCompatibility = "1.8"
        targetCompatibility = "1.8"
    }
}

sourceSets {
    main {
        proto {
            srcDir 'src/main/proto'
        }
        java {
            // include self written and generated code
            srcDirs 'src/main/java', 'generated-sources/main/java', 'generated-sources/main/grpc'
        }
    }
}

mainClassName = "com.example.HelloMain"

protobuf {
    // Configure the protoc executable
    protoc {
        // Download from repositories
        artifact = 'com.google.protobuf:protoc:3.0.2'
    }

    // Configure the codegen plugins
    plugins {
        grpc {
            artifact = 'io.grpc:protoc-gen-grpc-java:1.1.2'
        }
    }

    generateProtoTasks.generatedFilesBaseDir = 'generated-sources'

    generateProtoTasks {
        all()*.plugins {
            grpc {}
        }
    }
}
```


Add `build.gradle` in the root directory:

```bash
$ cd ../../..
$ vim build.gradle     # add above file content and save
```

That it! upon `gradle build` it would add the generated stub at the `generated-sources` as described in the `build.gradle`:

```bash
$ gradle build
```

This would run protoc as we have added the `generateProtoTasks` in the gradle file.

After the build is success it would show up as:
```bash
├── build.gradle
├── generated-sources
│   └── main
│       ├── grpc
│       │   └── com
│       │       └── example
│       │           └── hello
│       │               └── GreeterGrpc.java
│       └── java
│           └── com
│               └── example
│                   └── hello
│                       └── Hello.java
└── src
    └── main
        ├── proto
            └── hello.proto
```

### adding server with spring

Add spring config files and the server start up files:

```bash
└── src
    └── main
        ├── java
        │   └── com
        │       └── example
        │           ├── GreeterServiceImpl.java
        │           ├── GrpcService.java
        │           ├── HelloMain.java
        │           └── HelloServer.java
        ├── proto
        │   └── hello.proto
        └── resources
            └── spring
                └── spring-hello.xml
```

- `spring-hello.xml`: describes the application context component scan
- `GreeterServiceImpl.java`: the implementation of grpc service stub, generated from proto. (GreeterGrpc.GreeterImplBase)
- `GrpcService.java`: the marker interface to identify all spring beans which are grpc service stub implementation
- `HelloServer.java`: the grpc server wrapper for all grpc services hosted for this application; as of now we only have one service for the app: GreeterServiceImpl.java
- `HelloMain.java`: the main method entry point for the java application, which initialises the spring context, starts grpc server

### java application

#### resources/spring/spring-hello.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/aop
	    http://www.springframework.org/schema/aop/spring-aop-4.3.xsd ">

    <aop:aspectj-autoproxy/>

    <context:annotation-config/>
    <context:component-scan base-package="com.example">
    </context:component-scan>

</beans>
```


#### com.example.GreeterServiceImpl

```java
import com.example.hello.GreeterGrpc;
import com.example.hello.Hello;
import io.grpc.stub.StreamObserver;
import org.springframework.stereotype.Service;

@Service
public class GreeterServiceImpl extends GreeterGrpc.GreeterImplBase implements GrpcService {

    @Override
    public void sayHello(Hello.HelloRequest req, StreamObserver<Hello.HelloReply> responseObserver) {
        Hello.HelloReply reply = Hello.HelloReply.newBuilder().setMessage("Hello " + req.getName()).build();
        responseObserver.onNext(reply);
        responseObserver.onCompleted();
    }
}
```


#### com.example.GrpcService

```java
/**
 * Marker interface; has to be added for those BindableService which is supposed to be added in matcher server
 *
 */
public interface GrpcService {
}
```


#### com.example.HelloServer

```java
package com.example;

import io.grpc.BindableService;
import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.protobuf.service.ProtoReflectionService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.List;

@Service
public class HelloServer {

    @Autowired
    private List<GrpcService> services;

    private Server server;

    public void start(int port) throws IOException, InterruptedException {

        ServerBuilder<?> serverBuilder = ServerBuilder.forPort(port);
        for (GrpcService grpcService : services) {
            if (!(grpcService instanceof BindableService)) {
                throw new RuntimeException("GrpcService should only used for grpc BindableService; found wrong usage of GrpcService for service: " + grpcService.getClass().getSimpleName());
            }
            serverBuilder.addService((BindableService) grpcService);
        }

        this.server = ((ServerBuilder<?>) serverBuilder)
                .addService(ProtoReflectionService.getInstance())
                .build()
                .start();

        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            // Use stderr here since the logger may have been reset by its JVM shutdown hook.
            System.err.println("*** shutting down gRPC server since JVM is shutting down");
            stop();
            System.err.println("*** server shut down");
        }));

        blockUntilShutdown();
    }

    public void stop() {
        if (this.server != null) {
            this.server.shutdown();
        }
    }

    /**
     * Wait for main method. the gprc services uses daemon threads
     * @throws InterruptedException
     */
    public void blockUntilShutdown() throws InterruptedException {
        if (this.server != null) {
            this.server.awaitTermination();
        }
    }
}
```

#### com.example.HelloMain

```java
package com.example;

import org.springframework.context.support.ClassPathXmlApplicationContext;

import java.io.IOException;
import java.util.TimeZone;

public class HelloMain {

    private static final int DEFAULT_PORT = 10000;

    public static void main(String[] args) throws IOException, InterruptedException {

        TimeZone.setDefault(TimeZone.getTimeZone("UTC"));

        //now that we have the configuration files, let the spring context read it and start the message listener container(s).
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("classpath*:/spring/spring*.xml");
        context.registerShutdownHook();

        int port = getPort(args);
        HelloServer server = context.getAutowireCapableBeanFactory().createBean(HelloServer.class);
        server.start(port);
    }

    private static int getPort(String[] args) {
        int port = DEFAULT_PORT;
        if (args.length > 0) {
            try {
                String portNum = args[0];
                int parsedPort = Integer.parseInt(portNum);
                if (parsedPort > 0) {
                    port = parsedPort;
                }
            } catch (NumberFormatException ignored) {
            }
        }
        return port;
    }
}
```

Thats it. Running above main method should start the server for greeter service. Above setup is added at [github](https://github.com/yogin16/grpc-spring-gradle-example) .

### References:
1. https://grpc.io/docs/guides/
1. https://github.com/grpc/grpc-java/
1. https://github.com/google/protobuf-gradle-plugin
1. https://spring.io/blog/2015/03/22/using-google-protocol-buffers-with-spring-mvc-based-rest-services
1. https://blog.bugsnag.com/grpc-and-microservices-architecture/
1. https://www.baeldung.com/grpc-introduction