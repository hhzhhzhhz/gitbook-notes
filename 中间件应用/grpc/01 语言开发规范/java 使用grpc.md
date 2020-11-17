# java 使用grpc

下载代码桩工具

```
https://github.com/protocolbuffers/protobuf/releases
https://repo1.maven.org/maven2/io/grpc/
```

定义proto文件

```
syntax = "proto3";


// The greeting service definition.
service Greeter{
  // Sends a greeting
  rpc Hello (Hrequest) returns (Hresponse) {
  }
}

// The request message containing the user's name.
message Hrequest {
  string name = 1;
}

// The response message containing the greetings
message Hresponse {
  string message = 1;
}
```

生成protobuf文件

```
D:\soft\protoc-3.13.0-win64\bin\protoc.exe --plugin=protoc-gen-grpc-java=D:\soft\protoc-3.13.0-win64\bin\protoc-gen-grpc-java-1.32.2-windows-x86_64.exe --java_out=./ file
```

生成代码桩

```
D:\soft\protoc-3.13.0-win64\bin\protoc.exe --plugin=protoc-gen-grpc-java=D:\soft\protoc-3.13.0-win64\bin\protoc-gen-grpc-java-1.32.2-windows-x86_64.exe --grpc-java_out=./ file
```

服务端实现代码桩生成的方法即可

#### grpc在spring 中应用

```
使用 gRPC Spring Boot Starter 
- 在 spring boot 应用中，通过 @GrpcService 自动配置并运行一个嵌入式的 gRPC 服务。
- 使用 @GdGrpcClient 自动创建和管理您的 gRPC Channels 和 stubs
- 支持全局和自定义的 gRPC 服务端/客户端拦截器
提供了拦截器、全局异常处理等功能
```



#### 参考资料

```
https://gitee.com/mirrors/grpc-spring-boot-starter
https://www.cnblogs.com/areful/p/10404506.html
```

