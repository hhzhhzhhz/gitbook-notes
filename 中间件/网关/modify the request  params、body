# cloud gateway 修改 request  params、body

###### 参数、body 修改需要重建reqeust，若修改了body 那么http头部 **content-length** 需填入真实长度，否则不符合HTTP规范 

### 一、修改params

```
URI newUri = UriComponentsBuilder.fromUri(exchange.getRequest().getURI())
                    .replaceQuery(newParams)
                    .build(true)
                    .toUri();
            ServerWebExchange reBuildExchange = exchange.mutate().request(exchange.getRequest().mutate().uri(newUri).build()).build();
```

### 二、 修改body

可参考以下链接

https://github.com/spring-cloud/spring-cloud-gateway/issues/1649 

https://blog.csdn.net/seantdj/article/details/100546713

http post 请求头部会携带content-length 来标识body 的大小可先判断此数据 大于0在做读取

读取后根据此字段**content-type** 编码，上传的文件不读取, https 不能读取

```
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.factory.rewrite.CachedBodyOutputMessage;
import org.springframework.cloud.gateway.support.BodyInserterContext;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpRequestDecorator;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.lang.Nullable;
import org.springframework.web.reactive.function.BodyInserter;
import org.springframework.web.reactive.function.BodyInserters;
import org.springframework.web.reactive.function.server.HandlerStrategies;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import java.io.IOException;
import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;

ServerRequest serverRequest = ServerRequest.create(exchange, HandlerStrategies.withDefaults().messageReaders());
        MediaType mediaType = exchange.getRequest().getHeaders().getContentType();
        Mono<String> modifiedBody = serverRequest.bodyToMono(String.class).flatMap(body -> {
        // todo body
            logBuf.append(body);
            return Mono.just(body);
        });
        BodyInserter bodyInserter = BodyInserters.fromPublisher(modifiedBody, String.class);
        HttpHeaders headers = new HttpHeaders();
        headers.putAll(exchange.getRequest().getHeaders());
        CachedBodyOutputMessage outputMessage = new CachedBodyOutputMessage(exchange, headers);
        return bodyInserter.insert(outputMessage, new BodyInserterContext()).then(Mono.defer(() -> {
            ServerHttpRequest decorator = decorate(exchange, headers, outputMessage);
            return chain.filter(exchange.mutate().request(decorator).build());
        }));
```

```
private  static ServerHttpRequestDecorator  decorate(ServerWebExchange exchange, HttpHeaders headers, CachedBodyOutputMessage outputMessage) {
        return new ServerHttpRequestDecorator(exchange.getRequest()) {
            public HttpHeaders getHeaders() {
//                long contentLength = headers.getContentLength();
                HttpHeaders httpHeaders = new HttpHeaders();
                httpHeaders.putAll(super.getHeaders());
//                if (contentLength > 0L) {
//                    httpHeaders.setContentLength(contentLength);
//                } else {
//                    httpHeaders.set("Transfer-Encoding", "chunked");
//                }
                return httpHeaders;
            }
            public Flux<DataBuffer> getBody() {
                return outputMessage.getBody();
            }
        };
    }
```