# **1. 简述**

gRPC 是一个高性能、开源和通用的 RPC 框架，面向移动和 HTTP/2 设计。目前提供 C、Java 和 Go 语言版本，分别是：grpc, grpc-java, grpc-go. 其中 C 版本支持 C, C++, Node.js, Python, Ruby, Objective-C, PHP 和 C# 支持。

gRPC 基于 HTTP/2 标准设计，带来诸如双向流、流控、头部压缩、单 TCP 连接上的多复用请求等特性。这些特性使得其在移动设备上表现更好，更省电和节省空间占用。

# **2. 特点**

1. 语言中立，支持多语言。
2. 基于IDL文件定义服务，使用proto工具生成数据结构、服务端接口以及客户端代码桩，客户端与服务端同一份IDL。
3. 通讯协议基于标准HTTP/2设计
   -> 支持双向流传输；
   -> 消息头压缩；
   -> 单TCP的多路复用；
   -> 服务端推送。
4. 移动网络支持
5. 系列化支持（Protocol buffer），PB是一个种语言无关的高性能序列化框架
   -> 语言无关、平台无关。可拓展的系列化结构数据的方法，可用于应用层数据通信协议、数据存储等；
   -> 灵活，高效。自动化机制的结构数据序列化方法。比XML更小（3～10倍），更快（20-100倍）；
   -> 扩展性、兼容性好。你可以定义数据的结构，然后使用特殊生成的源代码轻松的在各种数据流中使用各种语言进行编写和读取结构数据。

# **3. 数据传输协议**

### 编写proto文件，Greeter.proto

```protobuf
syntax = "proto3";
 
option java_package = "com.example.grpc.lib";
 
// The greeting service definition.
service Greeter{
    // Sends a greeting
    rpc SayHello (HelloRequest) returns (HelloReply) {
    }
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

syntax="proto3":指定protobuf的版本

java_package (文件选项) :这个选项表明生成java类所在的包。

required、optional、repeated:表示该字段是否必须填充；required表示必须指定且只能指定一个；当optional表示可选，可指定也可不指定，但不可超过一个不指定值的时候会采用空值，如string类型的字段会用字符串表示；repeated表示可以重复，类似与编程语言中的list

message HelloRequest：在一个message体内定义一个HelloRequest结构体，对应一个字段

数据类型： int32、int64、uint32、uint64、sint32、sint64、double、float、 string、bool、bytes、enum、message等等

**提示：Service下的方法名称，首字母使用大写字母。**

e.g. Service Greeter下的SayHello，S使用大写字母。

# **4. 调用模型**

![image2020-3-26_10-3-38](D:/code/note/gitbook-notes-master/images/01 带你入门 gRPC/image2020-3-26_10-3-38.png)

**gPRC调用流程**

1. 服务消费方以本地调用的方式调用程序。

2. client stub本地代码桩接收到调用后，负责把方法，参数等组装成能够进行网络传输的消息体。

3. client stub找到服务地址，并将消息发送到服务端。

4. server stub接收到消息后，进行解码。

5. server stub根据解码结果调用本地服务。

6. 本地服务执行并将结果返回给server stub。

7. server stub将结果打包成消息题并发送给消费方。

8. client stub接收到消息进行解码 9. 消费方得到最终结果。

*gRPC的作用：就是要将2～8这些布骤都封装起来，让用户对这些细节透明。*

# 5. 客户端与服务端交互细节

对于gRPC调用有交互细节能有较为深入的印象，较为简单的方式就是对Client调用Server端进行抓包刨析，能够较快的了解整个过程做了什么事情。如下图：

![image2020-3-24_14-38-4](D:/code/note/gitbook-notes-master/images/01 带你入门 gRPC/image2020-3-24_14-38-4.png)

**1. 行为分析**

Magic：Magic 帧的主要作用是建立 HTTP/2 请求的前言。在 HTTP/2 中，要求两端都要发送一个连接前言，作为对所使用协议的最终确认，并确定 HTTP/2 连接的初始设置，客户端和服务端各自发送不同的连接前言。

SETTINGS：SETTINGS 帧的主要作用是设置这一个连接的参数，作用域是整个连接而并非单一的流。

HEADERS：HEADERS 帧的主要作用是存储和传播 HTTP 的标头信息。

DATA：DATA 帧的主要作用是装填主体信息（e.g. 请求参数），是数据帧。

WINDOW_UPDATE：主要作用是管理和流的窗口控制。通常情况下打开一个连接后，服务器和客户端会立即交换 SETTINGS 帧来确定流控制窗口的大小。默认情况下，该大小设置为约 65 KB，但可通过发出一个 WINDOW_UPDATE 帧为流控制设置不同的大小。

PING/PONG：主要作用是判断当前连接是否仍然可用，也常用于计算RTT。



**2. 交互流程**

- 在建立连接之前，客户端/服务端都会发送连接前言（Magic+SETTINGS），确立协议和配置项。
- 在传输数据时，是会涉及滑动窗口（WINDOW_UPDATE）等流控策略的。
- 传播 gRPC 附加信息时，是基于 HEADERS 帧进行传播和设置；而具体的请求/响应数据是存储的 DATA 帧中的。
- 客户端发起 PING，服务端就会回应 PONG，反之亦可。



# **6. 异常处理流程**

gRPC异常处理流程，如下所示：

是通过 Status 和 Metadata 来传递异常信息，服务器端抛出异常到客户端的过程，实际是对 异常 和 Status/Metadata 的传递和转换的过程：

- 服务器端抛出异常
- 异常转为 Status/Metadata
- Status/Metadata 传递到客户端
- 客户端将 Status/Metadata 转回异常

Status/Metadata 在服务器端和客户端的传输中，是通过 HTTP header 来实现的。

![](file://D:/code/note/gitbook-notes-master/images/01%20%E5%B8%A6%E4%BD%A0%E5%85%A5%E9%97%A8%20gRPC/%E5%BC%82%E5%B8%B8%E6%B5%81%E7%A8%8B-2300843.png?lastModify=1608619515)

# **7. 超时机制**

详细细节，参考该专题内容

[g](https://doc.huanleguang.com/pages/viewpage.action?pageId=82845192)[RPC超时机制](https://doc.huanleguang.com/pages/viewpage.action?pageId=82845192)

# **8. 客户端与服务端交互分析**

分析Client端与Server端的心跳机制，以及Client端多路复用的情况，最简单的就是对 Client 端调用 Server 端进行抓包去剖析，看看整个过程中它都做了些什么事。

在开始分析之前，我们先熟悉一下，HTTP/2主要帧的情况：

- Magic：Magic 帧的主要作用是建立 HTTP/2 请求的前言。在 HTTP/2 中，要求两端都要发送一个连接前言，作为对所使用协议的最终确认，并确定 HTTP/2 连接的初始设置，客户端和服务端各自发送不同的连接前言。

- SETTINGS：SETTINGS 帧的主要作用是设置这一个连接的参数，作用域是整个连接而并非单一的流。

- HEADERS：HEADERS 帧的主要作用是存储和传播 HTTP 的标头信息。

- DATA：DATA 帧的主要作用是装填主体信息（e.g. 请求参数），是数据帧。

- WINDOW_UPDATE：主要作用是管理和流的窗口控制。通常情况下打开一个连接后，服务器和客户端会立即交换 SETTINGS 帧来确定流控制窗口的大小。默认情况下，该大小设置为约 65 KB，但可通过发出一个 WINDOW_UPDATE 帧为流控制设置不同的大小。

- PING/PONG：主要作用是判断当前连接是否仍然可用，也常用于计算RTT。

## 请求过程

首次连接与请求过程，单个连接支持多个请求，如下图

![image2020-4-1_9-57-49](D:/code/note/gitbook-notes-master/images/01 带你入门 gRPC/image2020-4-1_9-57-49.png)

过程分析如下：

- 刚开始建立连接，Client 端通过发送Magic帧，建立HTTP/2请求前言，以确定启用 HTTP/2 连接。
- 建立连接之后，Client端与Server端互相发送SETTINGS,WINDOW_UPDATE帧，设置这一个连接的参数，作用域是整个连接而并非单一的流。
- 建立连接后 ，我采用两个浏览器，连续发送3次请求（POST /Greeter/SayHello），从上面抓包的情况，连续抓取到3个请求的包，而且三个请求的中间无重新进行Magic与SETTINGS帧之间的交互，说明没有重新建立连接。
- 结论：说明在首次建立连接之后，Client端与Server端采用之前建立好的连接，单连接上支持多请求，实现了多路复用。



心跳机制，采用PING/PONG判断当前连接是否仍然可用，如下图所示

![image2020-4-1_10-29-5](D:/code/note/gitbook-notes-master/images/01 带你入门 gRPC/image2020-4-1_10-29-5.png)

过程分析如下：

- 当Client端长期无请求时，Client端会发起PING帧来探测Server端的死活情况，Server端也会通过PING帧来响应自己的状态。
- 相反地，Server端长期未接收到Client的请求，会发送PING,GOWAY帧，来断开当前的连接。
- Client接收到Server端断开连接的请求，会再次尝试发起建立HTTP/2连接的请求，双方重新发送SETTNGS,WINDOW_UPDATE帧来设置当前新的连接。
- 当Client端与Server端重新建立连接之后，我在重新发起多次请求（POST /Greeter/SayHello），多个请求同时复用新建立的连接，同时也再次验证了多路复用，在客户端复用连接的结论。
- 结论：Client端与Server端会采用PING/PONG的机制来保持心跳，并且Client端与Server端都有主动权，Server端为了减少连接长时间的未使用，会主动断开连接。

# **9. gRPC服务校验神器**

grpcurl是一个命令行工具，使用go语言编写，这样可以比较方便验证提供的服务是否可用。

grpcurl挺强大的，可以支持各种rpc接口，还可以验证流式传输的接口。

### 安装方式

在我们的mac上，安装很方便，支持HomeBrew：

```
brew install grpcurl
```

如果对go源码比较感兴趣的话，也可以直接下载源码自己编译

```
go get github.com/fullstorydev/grpcurl
go install github.com/fullstorydev/grpcurl/cmd/grpcurl
```

### 使用方式

grpc-server 将使用明文模式启动，所以grpccurl也是使用明文响应的方式访问

#### 查看远端地址提供的服务列表

```
grpcurl --plaintext localhost:9898 list
```

#### 通过服务名称去获取方法列表

```
grpcurl --plaintext localhost:9898 list Greeter
```

#### 接口访问参数传递

```
grpcurl --plaintext -d '{"name": "gaoding"}' localhost:9898 Greeter/SayHello
```



参考资料：

grpc详解 https://www.jianshu.com/p/9c947d98e192

深入 ProtoBuf - 编码 https://www.jianshu.com/p/73c9ed3a4877

protobuf语法详解 https://www.cnblogs.com/tohxyblog/p/8974763.html

protoc插件 https://www.xolstice.org/protobuf-maven-plugin/compile-mojo.html