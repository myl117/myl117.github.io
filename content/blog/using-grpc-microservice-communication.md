+++
title = "Using gRPC for Microservice Communication"
date = "2026-01-12"
tags = [
    "java",
    "springboot",
    "typescript"
]
+++

In traditional microservice architectures, communication between services is done via REST or sometimes for event driven communication, protocols like MQTT are employed. In a RESTful architecture, resources are identified by URIs (Uniform Resource Identifiers) and operations on those resources are performed using standard HTTP methods. Data Transfer Objects (DTOs) are represented as JSON or XML which are both lightweight and human readable formats.

gRPC on the other hand is a relatively newer framework which facilitates efficient communication using RPCs (Remote Procedure Calls) enabling services to call functions on other machines as if they were local. It is built on the use of a mechanism called Protobufs (Protocol Buffers) which supports strongly typed service contracts for DTOs and binary data encoding as opposed to JSON. Protobuf contracts are defined in .proto files which are used to generate code that can be used by a microservice akin to a library or SDK.

Data encoded in a binary form is up to 5 times more space efficient and faster to serialise and deserialise than JSON/XML with the obvious caveat of not being human readable. Unless you are a living Mentat of course ;)

## Building a Simple Library System

Let's create a simple library system that will demonstrate a simple microservices architecture (Node.js client and Java server) with gRPC communication and touch upon the fundamentals of the RPC protocol. It will allow us to view a list of books or get information about a specific one.

Let's first create a Protobuf contract that will define the methods for our service.

```c
syntax = "proto3";

option java_package = "com.myl117.java.library.grpc.service";
option java_outer_classname = "LibraryProto";

service LibraryService {
  // Get information about a book by ID
  rpc GetBook (BookRequest) returns (BookResponse);

  // List all books
  rpc ListBooks (Empty) returns (BookList);
}

message Empty {}

message BookRequest {
  int32 id = 1;
}

message BookResponse {
  int32 id = 1;
  string title = 2;
  string author = 3;
  bool available = 4;
}

message BookList {
  repeated BookResponse books = 1;
}
```

Next, let's create a Java Spring Project with Spring Initializer. We will need a bunch of dependencies and plugins so that we can use gPRC and compile Protobufs in our project. Make sure to include your proto file in a separate folder called proto. I'll be using Maven for this project so here is the Project Object Model (POM).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project
	xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>4.0.1</version>
		<relativePath/>
		<!-- lookup parent from repository -->
	</parent>
	<groupId>com.myl117.java.library.grpc.service</groupId>
	<artifactId>java-grpc-library-service</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>java-grpc-library-service</name>
	<description>Demo project for Spring Boot</description>
	<url/>
	<licenses>
		<license/>
	</licenses>
	<developers>
		<developer/>
	</developers>
	<scm>
		<connection/>
		<developerConnection/>
		<tag/>
		<url/>
	</scm>
	<properties>
		<java.version>17</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-webmvc</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<scope>runtime</scope>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-webmvc-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>io.grpc</groupId>
			<artifactId>grpc-netty-shaded</artifactId>
			<version>1.57.2</version>
		</dependency>
		<dependency>
			<groupId>io.grpc</groupId>
			<artifactId>grpc-protobuf</artifactId>
			<version>1.57.2</version>
		</dependency>
		<dependency>
			<groupId>io.grpc</groupId>
			<artifactId>grpc-stub</artifactId>
			<version>1.57.2</version>
		</dependency>
		<dependency>
			<groupId>com.google.protobuf</groupId>
			<artifactId>protobuf-java</artifactId>
			<version>3.24.3</version>
		</dependency>
		<dependency>
			<groupId>jakarta.annotation</groupId>
			<artifactId>jakarta.annotation-api</artifactId>
			<version>2.1.1</version>
		</dependency>
		<dependency>
			<groupId>javax.annotation</groupId>
			<artifactId>javax.annotation-api</artifactId>
			<version>1.3.2</version>
		</dependency>
	</dependencies>
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
			<plugin>
				<groupId>org.xolstice.maven.plugins</groupId>
				<artifactId>protobuf-maven-plugin</artifactId>
				<version>0.6.1</version>
				<configuration>
					<protocArtifact>com.google.protobuf:protoc:3.24.3:exe:${os.detected.classifier}</protocArtifact>
					<pluginId>grpc-java</pluginId>
					<pluginArtifact>io.grpc:protoc-gen-grpc-java:1.57.2:exe:${os.detected.classifier}</pluginArtifact>
					<!-- Remove outputOptions, it’s not supported in this version -->
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
		<extensions>
			<extension>
				<groupId>kr.motd.maven</groupId>
				<artifactId>os-maven-plugin</artifactId>
				<version>1.7.0</version>
			</extension>
		</extensions>
	</build>
</project>
```

Now that we have all the dependencies for our project setup, we can now focus on the gRPC server implementation. It is important this service sits in a new package, the same one defined in your proto file. Before writing the service code, compile your Maven project to generate code from your proto file.

```java
package com.myl117.java.library.grpc.service.java_grpc_library_service.service;

import com.myl117.java.library.grpc.service.LibraryProto.BookRequest;
import com.myl117.java.library.grpc.service.LibraryProto.BookResponse;
import com.myl117.java.library.grpc.service.LibraryProto.BookList;
import com.myl117.java.library.grpc.service.LibraryServiceGrpc;
import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.stub.StreamObserver;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

@Component
public class GrpcServer {

  private Server server;

  // In-memory list of books
  private final List <BookResponse> books = new ArrayList <>();

  @PostConstruct
  public void start() throws IOException {
    // Add sample books
    books.add(BookResponse.newBuilder()
      .setId(1)
      .setTitle("1984")
      .setAuthor("George Orwell")
      .setAvailable(true)
      .build());
    books.add(BookResponse.newBuilder()
      .setId(2)
      .setTitle("The Hobbit")
      .setAuthor("J.R.R. Tolkien")
      .setAvailable(false)
      .build());
      books.add(BookResponse.newBuilder()
      .setId(3)
      .setTitle("Sunrise on the Reaping")
      .setAuthor("Suzanne Collins")
      .setAvailable(false)
      .build());

    // Start gRPC server on port 9090
    server = ServerBuilder.forPort(9090)
      .addService(new LibraryServiceImpl())
      .build()
      .start();

    System.out.println("gRPC server started on port 9090");
  }

  @PreDestroy
  public void stop() {
    if (server != null) {
      server.shutdown();
      System.out.println("gRPC server stopped");
    }
  }

  // Implementation of the gRPC service
  private class LibraryServiceImpl extends LibraryServiceGrpc.LibraryServiceImplBase {

    @Override
    public void getBook(BookRequest request, StreamObserver <BookResponse> responseObserver) {
      Optional <BookResponse> book = books.stream()
        .filter(b -> b.getId() == request.getId())
        .findFirst();

      responseObserver.onNext(book.orElse(
        BookResponse.newBuilder()
        .setId(0)
        .setTitle("Not Found")
        .setAuthor("")
        .setAvailable(false)
        .build()
      ));
      responseObserver.onCompleted();
    }

    @Override
    public void listBooks(com.myl117.java.library.grpc.service.LibraryProto.Empty Empty, StreamObserver <BookList> responseObserver) {
      BookList.Builder builder = BookList.newBuilder();
      builder.addAllBooks(books);
      responseObserver.onNext(builder.build());
      responseObserver.onCompleted();
    }
  }
}
```

As you can see in the code, we import methods generated by the protocol buffer compiler. The code itself is fairly straightforward to understand but there are more powerful constructs we can work with as well as features like streaming and deadlines. But I'll let you, the reader, expand on this project and make those implementations.

We now have a gRPC server but one that is very lonely and would like a client to be able to communicate with it. So let's create a client. Let's create a new Typescript file project, install the necessary modules/types and copy the same proto file used in the java server.

```bash
npm install @grpc/grpc-js @grpc/proto-loader
npm install -D typescript ts-node @types/node
```

In our client.ts, we'll load the proto definition and create a client. Now that the library is set up, we can make procedure calls on our Java gRPC server as if they were local procedures!

```ts
import * as grpc from "@grpc/grpc-js";
import * as protoLoader from "@grpc/proto-loader";
import path from "path";

// Load proto definition
const packageDef = protoLoader.loadSync(
  path.join(__dirname, "/proto/library.proto"),
  {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true,
  }
);

// Convert proto into gRPC object
const grpcObject = grpc.loadPackageDefinition(packageDef) as any;

// Create client
const client = new grpcObject.LibraryService(
  "localhost:9090",
  grpc.credentials.createInsecure()
);

client.getBook({ id: 1 }, (err: grpc.ServiceError | null, response: any) => {
  if (err) {
    console.error(err.message);
    return;
  }

  console.log("Book:", response);
});

client.listBooks({}, (err: grpc.ServiceError | null, response: any) => {
  if (err) {
    console.error(err.message);
    return;
  }

  console.log("Books:", response.books);
});
```

And there you have it! A fully working client-server demo which if you've followed all the steps correctly, should display the books we defined on our server in the terminal. Although we've covered the fundamentals of gRPC, this is only the tip of the iceberg and I would like to encourage you to further explore more advanced concepts such as streaming, authentication, deadlines and more. You can find [the code for this project here.](https://github.com/myl117/grpc-library)

Thanks for reading!

## References

[gRPC vs. REST](https://blog.postman.com/grpc-vs-rest/)

[gRPC in 5 minutes | Eric Anderson & Ivy Zhuang, Google](https://www.youtube.com/watch?v=njC24ts24Pg)

[Protocol Buffers - Google's data interchange format](https://github.com/protocolbuffers/protobuf)
