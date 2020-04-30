# 							spring cloud gateway 动态路由

过滤器若要实现在动态路由中添加修改需要把过滤器添加到过滤器工厂中

```
org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory 继承过滤器工厂类且bean交给spring 管理
```

```
org.springframework.cloud.gateway.route.RouteDefinitionRouteLocator         
类中 gatewayFilterFactories 可以查看可以动态路由可添加的过滤器
```

### 一、过滤器工厂实现

```
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.core.Ordered;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;
// 此抽象类为了使 过滤器工厂跟普通过滤器规则一致
public abstract class AbsFilter<Config> extends AbstractGatewayFilterFactory<Config> implements GatewayFilter, Ordered{


    @Override
    public GatewayFilter apply(Config config) {
        return ((exchange, chain) -> filter(exchange,chain) );
    };


    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return verify(exchange,chain);
    }


    public abstract Mono<Void> verify(ServerWebExchange exchange, GatewayFilterChain chain);


    public abstract int getOrder();

}
```

### 二、动态路由实现

定义接口:

```
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.context.ApplicationEventPublisherAware;
import java.util.List;

public interface RouteService<T extends RouteDefinition> extends ApplicationEventPublisherAware {

    public void add(T t);

    public void update(T t);

    public void delete(String id);

    public <M> M get();

}
```

实现

```
import com.nascent.devops.gateway.constants.BeanNameConstants;
import com.nascent.devops.gateway.enums.ErrorCodeEnums;
import com.nascent.devops.gateway.exception.BusinessException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.gateway.event.RefreshRoutesEvent;
import org.springframework.cloud.gateway.route.*;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;
import java.util.List;

@Service(BeanNameConstants.ROUTE_SERVICE_IMPL)
public class RouteServiceImpl<T extends RouteDefinition> implements RouteService<T>{

    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;

    private ApplicationEventPublisher publisher;

    @Autowired
    private RouteDefinitionLocator routeDefinitionLocator;

    @Autowired
    private RouteLocator routeLocator;

    public RouteServiceImpl() {
        super();
    }

    @Override
    public void add(T t) {
        routeDefinitionWriter.save(Mono.just(t)).subscribe();
        this.publisher.publishEvent(new RefreshRoutesEvent(this));
    }

    @Override
    public void update(T t) {
        delete(t.getId());
        try {
            routeDefinitionWriter.save(Mono.just(t)).subscribe();
            this.publisher.publishEvent(new RefreshRoutesEvent(this));
        } catch (Exception e) {
            e.printStackTrace();
            throw new BusinessException(ErrorCodeEnums.ROUTE_SAVE, e.getMessage());
        }
    }

    @Override
    public void delete(String id) {
        try {
            this.routeDefinitionWriter.delete(Mono.just(id));
        } catch (Exception e) {
            e.printStackTrace();
            throw new BusinessException(ErrorCodeEnums.ROUTE_DELETE,e.getMessage());
        }

    }

    // todo 获取路由信息,未生效
    @Override
    public <M> M get() {
        Mono<List<Route>> routes = this.routeLocator.getRoutes().collectList();
        return null;
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.publisher = applicationEventPublisher;
    }

}
```



### 三、实现动态路由添加

根据动态路由实现，定制对应的实体添加即可,其中 添加名称为过滤器类名

```
// todo 动态路由添加的filter  会按照添加的顺序执行责任链，不会根据order 原因未知，需要执行顺序的需注意！！
app.addFilter(authFilter);
app.addFilter(getRewritePath(app));
app.addFilter(signal);
GatewayRouteDefinition gatewayRouteDefinition = new GatewayRouteDefinition(app).transform();
routeService.add(gatewayRouteDefinition);
```