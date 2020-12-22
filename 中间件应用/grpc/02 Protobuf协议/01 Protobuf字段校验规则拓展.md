# 前言

proto3相较于proto2支持更多语言但在语法上更为简洁。去除了一些复杂的语法和特性，更强调约定而弱化语法。经过研究发现google已经意识到这个问题，采取了一些补救方法，提供wrappers包。

目前开源社区已经围绕Protobuf和gRPC开发出众多扩展，形成了庞大的生态。接下来，我们主要围绕 Validators 验证器规则来讲。

# 开源验证器

protoc-gen-validate 是一个protoc插件，用于生成多语言消息验证器。尽管Protobuf有效地保证了结构化数据的类型，但它们不能为值强制执行语义规则。该插件为协议生成的代码添加了支持，以验证此类约束。

# 支持语言

- go
- c++ 
- java
- python

对应不同语言之间的支持情况，可以查看[constraint rule comparison matrix](https://github.com/envoyproxy/protoc-gen-validate/blob/master/rule_comparison.md) .

# 举个例子

我们可以引入 protoc-gen-validate 拓展插件，在protobuf文件中的消息和字段上使用约束规则，以下为一个完整的例子：

```protobuf
syntax = "proto3";
option java_package = "com.gaoding.grpc.lib";
 
import "validate/validate.proto";
 
message Person {
  uint64 id    = 1 [(validate.rules).uint64.gt    = 999];
  string email = 2 [(validate.rules).string.email = true];
  string name  = 3 [(validate.rules).string = {
                      pattern:   "^[^[0-9]A-Za-z]+( [^[0-9]A-Za-z]+)*$",
                      max_bytes: 256,
                   }];
  Location home = 4 [(validate.rules).message.required = true];
 
  message Location {
    double lat = 1 [(validate.rules).double = { gte: -90,  lte: 90 }];
    double lng = 2 [(validate.rules).double = { gte: -180, lte: 180 }];
  }
}
```

**使用说明：**

- 该插件只支持proto3语法
- 引入依赖库 import "validate/validate.proto";
- 在字段后面声明约束规则 [(validate.rules).uint64.gt = 999]；比如上面的id，需要满足id大于999；

# 约束规则

约束规则，对应不同语言之间的支持情况，可以查看[constraint rule comparison matrix](https://github.com/envoyproxy/protoc-gen-validate/blob/master/rule_comparison.md) .

## 数值

所有的数值类型（`float`, `double`, `int32`, `int64`, `uint32`, `uint64`, `sint32`, `sint64`, `fixed32`, `fixed64`, `sfixed32`, `sfixed64`），使用相同的规则。



**const:** 该字段必须是指定的值：

```protobuf
// x 必须刚好等于 1.23
float x = 1 [(validate.rules).float.const = 1.23];
```


**lt/lte/gt/gte:** 这些不等式（分别为<，<=，>，> =），表示字段可以支持的使用范围：

```protobuf
// x 必须小于 10
int32 x = 1 [(validate.rules).int32.lt = 10];
 
// x 必须等于 20
uint64 x = 1 [(validate.rules).uint64.gte = 20];
 
// x 必须在 [30, 40) 范围内
fixed32 x = 1 [(validate.rules).fixed32 = {gte:30, lt: 40}];
 
 
// x 必须在 [30, 40) 范围之外
double x = 1 [(validate.rules).double = {lt:30, gte:40}];

```

**in/not_in:** 这两个规则允许为字段值指定白名单/黑名单。

```protobuf
// x 必须在 1, 2, 或者 3 之间
uint32 x = 1 [(validate.rules).uint32 = {in: [1,2,3]}];
 
// x 不能为 0 或者 0.99
float x = 1 [(validate.rules).float = {not_in: [0, 0.99]}];
```

## 布尔

**const:** 该字段必须是指定的值：

```protobuf
// x 必须设置为 true
bool x = 1 [(validate.rules).bool.const = true];
 
// x 不能为 true
bool x = 1 [(validate.rules).bool.const = false];
```

## 字符串

**const:** 该字段必须是指定的值：

```protobuf
// x 必须设置为 "gaoding"
string x = 1 [(validate.rules).string.const = "gaoding"];
```



**len/min_len/max_len:** 这些规则限制了字段中的字符数：

```protobuf
// x 字符数大小为 5
string x = 1 [(validate.rules).string.len = 5];
 
// x 字符数最少为 3
string x = 1 [(validate.rules).string.min_len = 3];
 
// x 字符数范围在 5 到 10（包含）
string x = 1 [(validate.rules).string = {min_len: 5, max_len: 10}];
```



**min_bytes/max_bytes:** 这些规则限制了字段中的字节数：

```protobuf
// x 最大字节数 15
string x = 1 [(validate.rules).string.max_bytes = 15];
 
// x 字节数区间 128 到 1024
string x = 1 [(validate.rules).string = {min_bytes: 128, max_bytes: 1024}];
```



**pattern：**该字段必须匹配指定的符合[RE2-compliant](https://github.com/google/re2/wiki/Syntax)的正则表达式：

```protobuf
// x 必须支持如下表达式
string x = 1 [(validate.rules).string.pattern = "(?i)^[0-9a-f]+$"];
```



**prefix/suffix/contains****:** 该字段包含指定的子字符串：

```protobuf
// x 前缀为 gaoding
string x = 1 [(validate.rules).string.prefix = "gao"];
 
// x 后缀为 ding
string x = 1 [(validate.rules).string.suffix = "ding"];
 
// x 包含字符串 goading
string x = 1 [(validate.rules).string.contains = "goading"];
 
// x 前缀为 "gao" 且 后缀为 "ding"
string x = 1 [(validate.rules).string = {prefix: "gao", suffix: "ding"}];
 
// x 后缀为 gaoding 且 字符串长度 不超过 64
string x = 1 [(validate.rules).string = {suffix: ".proto", max_len:64}];
```



**in/not_in:** 这两个规则允许为字段值指定白名单/黑名单：

```protobuf
// x 必须包含在 "foo", "bar", or "baz"
string x = 1 [(validate.rules).string = {in: ["foo", "bar", "baz"]}];
 
// x 不能为 "fizz" 或者 "buzz"
string x = 1 [(validate.rules).string = {not_in: ["fizz", "buzz"]}];
```



**well-known formats:** 这些规则为常见的字符串模式提供了高级约束：

```protobuf
// x email格式 (via RFC 1034)
string x = 1 [(validate.rules).string.email = true];
 
// x 合法的地址 (IP or Hostname).
string x = 1 [(validate.rules).string.address = true];
 
// x hostname (via RFC 1034)
string x = 1 [(validate.rules).string.hostname = true];
 
// x IP地址 (either v4 or v6)
string x = 1 [(validate.rules).string.ip = true];
 
// x IPv4地址
// eg: "192.168.0.1"
string x = 1 [(validate.rules).string.ipv4 = true];
 
// x IPv6地址
// eg: "fe80::3"
string x = 1 [(validate.rules).string.ipv6 = true];
 
// x URI (via RFC 3986)
string x = 1 [(validate.rules).string.uri = true];
 
// x URI reference (either absolute or relative)
string x = 1 [(validate.rules).string.uri_ref = true];
 
// x UUID (via RFC 4122)
string x = 1 [(validate.rules).string.uuid = true];
```

## 枚举

以下为枚举的例子：

```protobuf
enum State {
  INACTIVE = 0;
  PENDING  = 1;
  ACTIVE   = 2;
}
```



**const:** 该字段必须是指定的值：

```protobuf
// x 必须设置为 ACTIVE (2)
State x = 1 [(validate.rules).enum.const = 2];
```



**defined_only：**该字段必须是枚举描述符中指定的值之一：

```protobuf
// x 只能在枚举定义的列表里 INACTIVE, PENDING, or ACTIVE
State x = 1 [(validate.rules).enum.defined_only = true];
```



**in/not_in****:** 这两个规则允许为字段值指定白名单/黑名单：

```protobuf
// x 必须为 INACTIVE (0) 或者 ACTIVE (2)
State x = 1 [(validate.rules).enum = {in: [0,2]}];
 
// x 不能为 PENDING (1)
State x = 1 [(validate.rules).enum = {not_in: [1]}];
```

## Messages

如果字段包含message，并且消息是使用 验证器 生成的，则将递归执行验证。

如果message不是验证器生成的，则将被跳过。



**skip:** 这个字段声明跳过校验规则

```protobuf
// x 下的字段，将不被进行校验
Person x = 1 [(validate.rules).message.skip = true];
```



required：这个规则要求字段需要复制

```protobuf
// x 需要被赋值
Person x = 1 [(validate.rules).message.required = true];
 
// x 不需要被校验，但是还是需要进行赋值
Person x = 1 [(validate.rules).message = {required: true, skip: true}];
```

## Repeated

**min_items/max_items:** 这些规则控制字段中包含多少个元素：

```protobuf
// x 最少包含 3 个元素
repeated int32 x = 1 [(validate.rules).repeated.min_items = 3];
 
// x 必须包含 5 到 10 个元素
repeated Person x = 1 [(validate.rules).repeated = {min_items: 5, max_items: 10}];
 
// x 刚好包含 7 个元素
repeated double x = 1 [(validate.rules).repeated = {min_items: 7, max_items: 7}];
```



**unique：** 此规则要求该字段中的所有元素必须唯一。：

```protobuf
// x 包含的元素不能重复
repeated int64 x = 1 [(validate.rules).repeated.unique = true];
```



**items：**此规则指定应应用于字段中每个元素的约束。 重复的消息字段也将应用其验证规则：

```protobuf
// x 包含的元素大于 0
repeated float x = 1 [(validate.rules).repeated.items.float.gt = 0];
 
// x 包含的Person消息，不进行规则验证
repeated Person x = 1 [(validate.rules).repeated.items.message.skip = true];
```

## Maps

**min_pairs/max_pairs:** 这些规则控制此字段中包含多少对KV：

```
// x 必须包含最少 3 对KV
map<string, uint64> x = 1 [(validate.rules).map.min_pairs = 3];
 
// x 必须包含 5 and 10 对KV
map<string, string> x = 1 [(validate.rules)].map = {min_pairs: 5, max_pairs: 10}];
 
// x 包含正好 7 对KV
map<string, Person> x = 1 [(validate.rules)].map = {min_pairs: 7, max_pairs: 7}];
```



**no_sparse****:** Map里的值不能设置为空：

```
// x 里的值不能设置为空
map<uint64, Person> x = 1 [(validate.rules).map.no_sparse = true];
```



**keys:** 此规则指定应用于字段中键的约束：

```
// x 的KEY都要小于 0
map<sint32, string> x = [(validate.rules).map.keys.sint32.lt = 0];
```



**values****:** 此规则指定应用于字段中每个值的约束：

```
// x 对应的值，不能少于 3 个字符
map<string, string> x = 1 [(validate.rules).map.values.string.min_len = 3];
 
// x 包含的 message 不进行验证
map<string, Person> x = 1 [(validate.rules).map.values.message.skip = true];
```

## 包装类

在 proto3 语法中，无法在标量字段的 unset 和零值之间进行区分。所以需要引入 proto3 的包装类。

```
// x 如果有值，则需要大于 3
google.protobuf.Int32Value x = 1 [(validate.rules).int32.gt = 3];
```

# 拦截器（进阶）

ValidatingClientInterceptor 是 gRPC 客户端拦截器 ClientInterceptor，用于在传输之前验证出栈 gRPC 消息。不符合要求的消息会抛出 NVALID_ARGUMENT 状态异常。

验证的规则，需要在 protobuf 文件中进行定义。具体查看上面的章节 ***约束规则***。

具体实现如下：

```java
public class ValidatingClientInterceptor implements ClientInterceptor {
     
    private final ValidatorIndex index;
 
    public ValidatingClientInterceptor(ValidatorIndex index) {
        this.index = index;
    }
 
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(MethodDescriptor<ReqT, RespT> method, CallOptions callOptions, Channel next) {
        return new ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(next.newCall(method, callOptions)) {
            @Override
            public void sendMessage(ReqT message) {
                try {
                    index.validatorFor(message.getClass()).assertValid(message);
                    super.sendMessage(message);
                } catch (ValidationException ex) {
                    throw ValidationExceptions.asStatus(ex).asRuntimeException();
                }
            }
        };
    }
}
```



参考资料：

protoc-gen-validate验证器 https://github.com/envoyproxy/protoc-gen-validate