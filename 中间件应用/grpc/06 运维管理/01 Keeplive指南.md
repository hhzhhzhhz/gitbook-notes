# 前言

Keeplive ping是一种通过transport发送HTTP2 ping来探测当前通道是否正常通讯的方法。这是进行周期性探测，如果在一定时间内没有得到远端的确认，传输连接将断开。

# 参数说明

Keeplive ping由两个重要的传输通道参数控制：

- GRPC_ARG_KEEPALIVE_TIME_MS
  这个通道参数控制了keepalive ping发送的时间间隔（毫秒）
- GRPC_ARG_KEEPALIVE_TIMEOUT_MS
  这个通道参数发送keepalive ping等待远端确认的时间（毫秒）。如果在这个时间内没有得到确认，它将断开当前连接

上面的两个参数应该能满足大部分的用户需求，但是下面的参数在某些场景也是很有用。

- GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS
  如果该参数为1（0:false, 1:true）时，尽管没有服务调用，也会发送keepalive ping，来判断连接是否可用。

- GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA
  当没有其他数据（数据帧或者数据头）需要发送当的时候，该参数控制可以发送最大的ping数。如果超过设置值，GRPC Core将不会继续发送ping数据。该参数设置为0时，在没有数据需要发送的时候运行发送ping数据。

- GRPC_ARG_HTTP2_MIN_SENT_PING_INTERVAL_WITHOUT_DATA_MS
  如果通道中没有数据帧被接收到，该参数控制GEPC Core连续两个ping数据之间的最小时间

- GRPC_ARG_HTTP2_MIN_RECV_PING_INTERVAL_WITHOUT_DATA_MS
  如果通道中没有发送数据帧，则服务端的此channel参数控制gRPC Core在接收连续ping之间期望的最短时间（毫秒）。如果连续两次ping之间的时间少于此时间，则该ping将被视为远端的不良ping。这样的ping算作“ ping strike”。在客户端，这没有任何效果。

- GRPC_ARG_HTTP2_MAX_PING_STRIKES
  该参数控制在发送HTTP2 GOAWAY帧并关闭传输之前，服务器允许的错误ping的最大数量。将其设置为0允许服务端接受任意数量的错误ping。

  

# 默认值



| 参数                                                  | 客户端             | 服务端             |
| :---------------------------------------------------- | :----------------- | :----------------- |
| 参数                                                  | 客户端             | 服务端             |
| GRPC_ARG_KEEPALIVE_TIME_MS                            | INT_MAX (disabled) | 7200000 (2 hours)  |
| GRPC_ARG_KEEPALIVE_TIMEOUT_MS                         | 20000 (20 seconds) | 20000 (20 seconds) |
| GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS               | 0 (false)          | 0 (false)          |
| GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA                 | 2                  | 2                  |
| GRPC_ARG_HTTP2_MIN_SENT_PING_INTERVAL_WITHOUT_DATA_MS | 300000 (5 minutes) | 300000 (5 minutes) |
| GRPC_ARG_HTTP2_MIN_RECV_PING_INTERVAL_WITHOUT_DATA_MS | N/A                | 300000 (5 minutes) |
| GRPC_ARG_HTTP2_MAX_PING_STRIKES                       | N/A                | 2                  |

# FAQ

1. keepalive定时器是什么时候开始的？
   transport完成连接后（握手后），将启动keepalive计时器。

2. 当keepalive触发时会发生什么？

   (1) 当keepalive计时器触发时，GRPC Core将尝试在传输中发送keepalive ping。但是以下情况可以阻止此ping的发送

   1. 该transport上没有活动调用，并且GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS为false
   2. transport中已发送的ping数目（在transport中没有其他data发送时）已超过GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA
   3. 自上次ping以来经过的时间少于GRPC_ARG_HTTP2_MIN_SENT_PING_INTERVAL_WITHOUT_DATA_MS

   (2) 如果keepalive的ping没有被blocked并在transport中发送，那么将启动keepalive watchdog计时器：如果在触发（timeout）前还未收到ping的确认，就会关闭transport

3. 为什么我能收到错误码为

   ENHANCE_YOUR_CALM的GOAWAY？

   (1) 如果客户端发送的ping存在太多不符合规则的，则服务器发送ENHANCE_YOUR_CALM的GOAWAY帧。例如

   1. 如果服务器将GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS设置为false，但客户端却在没有任何请求的transport中发送ping
   2. 如果客户端设置的GRPC_ARG_HTTP2_MIN_SENT_PING_INTERVAL_WITHOUT_DATA_MS的值低于服务器的GRPC_ARG_HTTP2_MIN_RECV_PING_INTERVAL_WITHOUT_DATA_MS的值

# 参考资料

https://github.com/grpc/grpc/blob/master/doc/keepalive.md