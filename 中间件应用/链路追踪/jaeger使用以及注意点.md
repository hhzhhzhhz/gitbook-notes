# **使用:**

```
<dependency>
    <groupId>io.jaegertracing</groupId>
    <artifactId>jaeger-client</artifactId>
    <version>1.2.0</version>
</dependency>
```

## **注意点:**

uber-trace-id 由四个字段组成 使用: 为分隔符 

```
1、 traceID 

2、spanID

3、bagaage

4、flags < 此整型数字在tracer.extract()生成context时, 会产生flags值: 计算规则该数据转化为16进制flags的值， 之后& 1 等于1 就会把需要记录的log 添加否则不添加，不添加的就没有上报,目前这个值 1，2有其他操作>
```



## 1、网关

```
import io.opentracing.Span;
import io.opentracing.SpanContext;
import io.opentracing.propagation.Format;
import io.opentracing.propagation.TextMapAdapter;
import io.opentracing.util.GlobalTracer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.cloud.gateway.filter.FilterDefinition;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.server.ServerWebExchange;
import java.io.UnsupportedEncodingException;
import java.util.*;

@Configuration
public class DevopsConfig {


    @Bean
    public void registJaegerClient(){
        io.jaegertracing.Configuration tracerConfig = new io.jaegertracing.Configuration("devops-gateway");
        io.jaegertracing.Configuration.SenderConfiguration sender = new io.jaegertracing.Configuration.SenderConfiguration();
        sender.withAgentHost(config.getJaeGerhost());
        sender.withAgentPort(config.getJaeGerpost());
        tracerConfig.withReporter(new io.jaegertracing.Configuration.ReporterConfiguration().withSender(sender).withFlushInterval(100).withLogSpans(false));
        tracerConfig.withSampler(new io.jaegertracing.Configuration.SamplerConfiguration().withType("const").withParam(1));
        io.opentracing.Tracer tracer = tracerConfig.getTracer();
        GlobalTracer.register(tracer);
    }


    @Bean
    public RouteLocator test(RouteLocatorBuilder builder) throws UnsupportedEncodingException {



        return builder.routes().route(r -> r.path("/menu/**").filters(f -> f.filter( ( exchange, chain ) -> {
                    Span parent = GlobalTracer.get().buildSpan("spring-cloud-gateway").start();
                    Map<String,String> map = new HashMap<>();
//                    tracer.activateSpan(parent);
                    SpanContext spanContext = parent.context();
//ID最后一个数字 extract 计算规则 转化为16进制flags的值， 之后& 1 等于1 就会把需要记录的log 添加否则不添加， 所以不添加的就没有上报,目前这个值 0，1，2，3，4 有其他操作
                    String id =  String.format("%s:%s:%s:%s",spanContext.toTraceId(),spanContext.toSpanId(),spanContext.toTraceId(),1);
                    map.put("uber-trace-id",id);
                    GlobalTracer.get().inject(spanContext, Format.Builtin.TEXT_MAP, new TextMapAdapter(map));
                    parent.log("经过网关-spring cloud gateway");
                    parent.log(100, id);
                    parent.finish();

            ServerWebExchange reBuildExchange = exchange.mutate().request(exchange.getRequest().mutate().header("uber-trace-id",id).build()).build();
            return chain.filter(reBuildExchange);
    
        }))
                        .uri("http://127.0.0.1:9002")
                ).build();
    }

}
```

## 2、web 应用:

```
import io.jaegertracing.Configuration;
import io.opentracing.Span;
import io.opentracing.SpanContext;
import io.opentracing.Tracer;
import io.opentracing.propagation.Format;
import io.opentracing.propagation.TextMapAdapter;
import io.opentracing.util.GlobalTracer;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.*;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/")
@SpringBootApplication
public class DemoApplication {

	@Bean
    public void registJaegerClient(){
        io.jaegertracing.Configuration tracerConfig = new io.jaegertracing.Configuration("devops-gateway");
        io.jaegertracing.Configuration.SenderConfiguration sender = new io.jaegertracing.Configuration.SenderConfiguration();
        sender.withAgentHost(config.getJaeGerhost());
        sender.withAgentPort(config.getJaeGerpost());
        tracerConfig.withReporter(new io.jaegertracing.Configuration.ReporterConfiguration().withSender(sender).withFlushInterval(100).withLogSpans(false));
        tracerConfig.withSampler(new io.jaegertracing.Configuration.SamplerConfiguration().withType("const").withParam(1));
        io.opentracing.Tracer tracer = tracerConfig.getTracer();
        GlobalTracer.register(tracer);
    }


    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    
    }
    
    @GetMapping(value = "/menu/tree/seting")
    public String po(HttpServletRequest request, HttpServletResponse response) throws InterruptedException {
        Map<String,String> map = new HashMap<>();
        String id = request.getHeader("uber-trace-id");
        map.put("uber-trace-id", id);
        Tracer tracer = GlobalTracer.get();
    
        Tracer.SpanBuilder spanBuilder = tracer.buildSpan("api-backend");
        SpanContext parentSpanCtx = GlobalTracer.get().extract(Format.Builtin.TEXT_MAP, new TextMapAdapter(map));
        spanBuilder.asChildOf(parentSpanCtx);
        Span span = spanBuilder.start();

//        tracer.activateSpan(span);
        span.log("应用接口-/menu/tree/seting");
        span.log(100, id);
        Thread.sleep(1000);
        span.finish();
        return request.getParameterMap().toString();

    }

}
```

