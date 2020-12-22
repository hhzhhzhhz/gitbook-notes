# NGINX 配置

在nginx的配置文件中配置GRPC代理即可。

## 代理配置

```sh
server{
    #nginx 监听8888接口
    listen  8888    http2;
    location / {
        #转发请求至实际服务侧
        grpc_pass grpc://localhost:9999;
    }
}
```

## 代理多个路由配置&负载均衡

```sh
server{
    #nginx 监听8888接口
    listen  8888    http2;
    location / {
        #转发请求至实际服务侧
        grpc_pass grpc://grpcservers;
    }
  
    location /helloworld.decodeService {
        grpc_pass grpc://grpcservers;
    }
  
    location /detect.DetectService {
        grpc_pass grpc://grpcservers;
    }
}
  
upstream grpcservers {
    server localhost:9999;
    server localhost:10000;
    server localhost:10001;
}
```

# 负载原理

首先GRPC是建立在HTTP2.0的基础上进行数据传输，HTTP2.0的优势就不用多说了。因此NGINX负载均衡同样也是建立在HTTP2.0的基础之上。并且NGINX新增了GRPC模块。用于负载GRPC请求。
需要注意以下几个NGINX的参数配置：

## Nginx_http2模块

http2_max_requests：在一个tcp连接上默认通过的最大数据包数，默认1000个
http2_max_concurrent_streams：在一个tcp连接上默认最大并发流，默认128个

## Nginx_grpc模块

grpc_send_timeout：将请求传输到gRPC服务器的超时时间，如果超过这个时间，NGINX会断开连接。
grpc_read_timeout：接收gRPC服务器数据的超时时间，如果超过这个时间，NGINX会断开连接。
grpc_socket_keepalive：NGINX与gRPC服务器长连接配置设置。



参考资料

https://www.jianshu.com/p/3e51914beaed