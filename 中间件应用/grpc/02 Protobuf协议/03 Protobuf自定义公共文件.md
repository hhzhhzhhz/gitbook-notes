# 前言

当gRPC新增服务的服务逐渐变多，慢慢发现在很多Proto文件中，重复的消息类型与字段就明显增多。这就给我们增加了维护的成本，比如：分页参数，通用数据类型等。对于项目复杂而且依赖较多的，对于自定义公共的Proto文件，迫在眉睫。

# 制作自定义选项

## 定义选项

直接举例说明，以下是定义好的common.proto文件：

```protobuf
syntax = "proto3";
 
package gd.protobuf;
 
option java_package = "com.example.grpc.common.protobuf";
 
//分页请求
message Page {
 
    //每页记录数
    int32 pageSize = 21;
 
    //页码
    int32 pageNum = 22;
}
 
//排序请求
message Sort {
    //属性
    string fieldName = 21;
    //方向
    Direction direction = 22;
    //枚举消息类型
    enum Direction {
        //倒序
        DESC = 0;
        //正序
        ASC = 1;
    }
}
```


**选项说明：**

- syntax = "proto3";
  - 使用 proto3 语法。
- package gd.protobuf;
  - 包名
  - 必须有，表示作用范围
- option java_package = "com.gaoding.grpc.common.protobuf";
  - java代码生成后的路径
  - 必须要有
- message Page {}
  - 定义一个消息内容，其他proto文件，可以引用
  - 非必须，根据具体业务情况自定义

## 生成公共代码桩

配置生成插件，代码构建时，根据插件生成对应的代码桩。

```xml
<build>
    <resources>
        <resource>
            <directory>src/main/proto</directory>
            <includes>
                <include>**/*.proto</include>
            </includes>
        </resource>
    </resources>
 
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>${os.plugin.version}</version>
        </extension>
    </extensions>
 
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.5.0</version>
            <configuration>
                <!--
                 ~ Embed generated validate.proto classes for all Protobuf 3.x versions. This is safe because
                 ~ Protobuf guarantees API stability for all 3.x version.
                 ~ https://github.com/protocolbuffers/protobuf/blob/17cc42a45aeab0f78a137fa6be6c36095db32e58/CHANGES.txt#L107-L122
                -->
                <protocArtifact>com.google.protobuf:protoc:3.10.0:exe:${os.detected.classifier}</protocArtifact>
            </configuration>
            <executions>
                <execution>
                    <id>protoc-java-common-protobuf</id>
                    <goals>
                        <goal>compile</goal>
                    </goals>
                    <configuration>
                        <protoSourceRoot>${project.basedir}/src/main/proto</protoSourceRoot>
                        <includes>
                            <include>gd/protobuf/*.proto</include>
                        </includes>
                    </configuration>
                </execution>
            </executions>
        </plugin>
 
    </plugins>
 
</build>
```

# 如何在 proto 文件中使用自定义选项

如，定义hello.proto文件

```protobuf
syntax = "proto3";
 
option java_package = "com.example.grpc.api.lib.greeter";
 
import "gd/protobuf/common.proto";
 
 
message MessageRequest {
    string message_id = 1;
    string message_type = 2 [(validate.rules).string.const = "example"];
    // 分页信息
    gd.protobuf.Page pagination = 3;
}
```