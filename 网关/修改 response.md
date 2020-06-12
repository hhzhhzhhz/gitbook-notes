# cloud gateway 修改、记录返回的response 、headers

### 一、修改 response headers

https://github.com/spring-cloud/spring-cloud-gateway/issues/1092



### 二、记录 response （不读取body内容,记录长度）

```
chain.filter(exchange).doOnSuccess(v->RecordHanlel(exchange)) 
```



### 三、修改response

```
这种修改会导致请求不会继续,已通过wireshark 抓包验证
@Bean
@Qualifier(BeanNameConstants.REDIS_BEAN)
public RouteLocator test(RouteLocatorBuilder builder) throws UnsupportedEncodingException {
    return builder.routes().route(r -> r.path("/menu/**").filters(f -> f.filter( ( exchange, chain ) -> {
                ServerHttpResponse response = exchange.getResponse();
                response.setStatusCode(HttpStatus.OK);
                DataBufferFactory dataBufferFactory = response.bufferFactory();
                DataBuffer wrap = dataBufferFactory.wrap(
                        "{\"value\": \"intercepted\"}".getBytes(StandardCharsets.UTF_8));
                return response.writeWith(Mono.just(wrap))
                        .doOnSuccess(v -> response.setComplete());
    }))
                    .uri("http://127.0.0.1:9002")
            ).build();
}
```