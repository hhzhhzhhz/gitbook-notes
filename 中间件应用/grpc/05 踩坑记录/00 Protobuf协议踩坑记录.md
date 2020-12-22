## 默认值

protobuf3 删除了 protobuf2 中用来设置默认值的 default 关键字，取而代之的是protobuf3为各类型定义的默认值，也就是约定的默认值。

如下表所示：

| 类型     | 默认值                                                       |
| :------- | :----------------------------------------------------------- |
| bool     | false                                                        |
| 整型     | 0                                                            |
| string   | 空字符串""                                                   |
| 枚举enum | 第一个枚举元素的值，因为Protobuf3强制要求第一个枚举元素的值必须是0，所以枚举的默认值就是0； |
| message  | 不是null，而是DEFAULT_INSTANCE                               |

## 枚举类型

### 重复定义枚举常量名

不支持一个proto文件中，多个枚举中定义相同的枚举常量名。

如下的两个枚举，定义在同一个proto文件中：

```protobuf
enum Enum1 {
    IDLE = 0;
    RUNNING = 1;
}
 
enum Enum2 {
    IDLE = 5;
    RUNNING = 6;
}
```

### 枚举第一个常量的值

枚举第一个常量的值必须是0，实际项目中使用的枚举常量值经常是从0开始的，这样项目需求与protobuf3有冲突。

Google API Guider建议：

建议枚举的第一个值为 ENUN_TYPE_UNSPECIFIED，即：枚举名_UNSPECIFIED。这样，让默认的枚举值与业务的相分离。

举个例子：

```protobuf
enum FooBar {
    // 第一个值代表着默认值，必须为0.
    FOO_BAR_UNSPECIFIED = 0;
    FIRST_VALUE = 1;
    SECOD_VALUE = 2;
}
```

## 使用 wrapper 类型

Protobuf 3 为每个字段都提供默认值，原始数据类型保证不出现 null。不过这也引入了新的问题，“没设置”跟默认 0 / 0.0 / false / "" 等价，这也给我们带来了疑惑，是没有设置值呢，还是值就是这些内容。

不过Google还是采取了一些补救方法——提供wrappers包。该包位于google/protobuf/wrappers.proto。proto文件包含了以下类型：

- DoubleValue
- FloatValue
- Int64Value
- UInt64Value
- Int32Value
- UInt32Value
- BoolValue
- StringValue
- BytesValue

以Int32Value为例，其包装方法如下：

```protobuf
// Wrapper message for `int32`.
//
// The JSON representation for `Int32Value` is JSON number.
message Int32Value {
  // The int32 value.
  int32 value = 1;
}
```


使用事例：

```protobuf
import "google/protobuf/wrappers.proto";
 
message Test {
    google.protobuf.Int32Value a = 1;
}
```

采用 wrapper 类型后，就可以用 hasXxx() 判断是否设置过了。

## message类型

Java中，message类型的默认值是DEFAULT_INSTANCE，其值相当于空的message，即XXX.newBuilder().build()。

这样对message类型的判空操作不能与java原始的判空方式一样。

举个例子：

```protobuf
// protobuf message.
message HelloReply {
    string message = 1;
    string nameClose = 2;
}
 
 
// java 判空操作
if (helloReply == null || helloReply == GreeterOuterClass.HelloReply.getDefaultInstance()) {
    log.info("response is empty.");
}
```

## 参数和返回值

rpc接口的参数和返回值必须是message类型，即使接口不需要参数或返回值，也得指定一个message。

这也导致不能不传参数，也不能不返回结果，也不能使用基本类型作为参数和返回结果（因为基本类型不是message）。

那么不想传参和返回结果时怎么办呢？

解决方案：

使用protobuf提供的Empty，这个message没有属性，不需要设置Empty message，而且客户端在调用gRPC接口时，不需要构造Empty对象，直接传null即可，减少gRPC接口的这一规定对客户端的影响。

```protobuf
syntax = "proto3";
 
import "google/protobuf/empty.proto";
 
option java_package = "com.gaoding.grpc";
option java_outer_classname = "UserProto";
 
message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
}
message UserList {
    repeated User user = 1;
}
 
service UserSevice {
    rpc list(google.protobuf.Empty) returns (UserList);
}
```

## 空数组

gRPC返回的空数组不能直接操作。调用Arrays.asList()生产的List的add、remove方法时报异常，这是由Arrays.asList() 返回的市Arrays的内部类ArrayList， 而不是java.util.ArrayList。Arrays的内部类ArrayList和java.util.ArrayList都是继承AbstractList，remove、add等方法AbstractList中是默认throw UnsupportedOperationException而且不作任何操作。java.util.ArrayList重新了这些方法而Arrays的内部类ArrayList没有重新，所以会抛出异常

```java
List arrList = new ArrayList(list);
```

