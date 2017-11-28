# 1. 概述

本文主要对 **过滤器 GatewayFilter 做整体的认识**。

过滤器整体类图如下 ：

[](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_03_01/01.jpeg)

是不是有点疑惑 GlobalFilter 与 GatewayFilter 的关系 ？且见本文分晓。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。

# 2. GatewyFilter

`org.springframework.cloud.gateway.filter.GatewayFilter` ，网关过滤器**接口**，代码如下 ：

```Java
public interface GatewayFilter {

	/**
	 * Process the Web request and (optionally) delegate to the next
	 * {@code WebFilter} through the given {@link GatewayFilterChain}.
	 * @param exchange the current server exchange
	 * @param chain provides a way to delegate to the next filter
	 * @return {@code Mono<Void>} to indicate when request processing is complete
	 */
	Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);

}
```

* 从接口方法可以看到，和 [`javax.servlet.Filter`](https://tomcat.apache.org/tomcat-5.5-doc/servletapi/javax/servlet/Filter.html) 类似。

GatewayFilter 有三种类型的子类实现，我们下面每节介绍一种。

## 2.1 GatewayFilterFactory 内部类

在每个 GatewayFilterFactory 实现类的 `#apply(Tuple)` 方法里，都声明了一个实现 GatewayFilter 的**内部类**，以 AddRequestHeaderGatewayFilterFactory 的代码举例子 ：

```Java
  1: public class AddRequestHeaderGatewayFilterFactory implements GatewayFilterFactory {
  2: 
  3: 	@Override
  4: 	public List<String> argNames() {
  5: 		return Arrays.asList(NAME_KEY, VALUE_KEY);
  6: 	}
  7: 
  8: 	@Override
  9: 	public GatewayFilter apply(Tuple args) {
 10: 		String name = args.getString(NAME_KEY);
 11: 		String value = args.getString(VALUE_KEY);
 12: 
 13: 		return (exchange, chain) -> { // GatewayFilter  
 14: 			ServerHttpRequest request = exchange.getRequest().mutate()
 15: 					.header(name, value)
 16: 					.build();
 17: 
 18: 			return chain.filter(exchange.mutate().request(request).build());
 19: 		};
 20: 	}
 21: }
```

* 第 13 至 19 行 ：定义了一个 GatewayFilter **内部实现类**。

在 [TODO 【3020】]() ，我们会详细解析每个 GatewayFilterFactory 定义的GatewayFilter **内部实现类**。

## 2.2 OrderedGatewayFilter

`org.springframework.cloud.gateway.filter.OrderedGatewayFilter` ，**有序的**网关过滤器**实现类**。在 FilterChain 里，过滤器数组首先会按照 `order` 升序排序，按照**顺序**过滤请求。代码如下 ：

```Java
public class OrderedGatewayFilter implements GatewayFilter, Ordered {

	private final GatewayFilter delegate;
	private final int order;

	public OrderedGatewayFilter(GatewayFilter delegate, int order) {
		this.delegate = delegate;
		this.order = order;
	}

	@Override
	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
		return this.delegate.filter(exchange, chain);
	}

	@Override
	public int getOrder() {
		return this.order;
	}
}
```

* `delegate` 属性，委托的 GatewayFilter 。
* `order` 属性，顺序。
* `#filter(ServerWebExchange, GatewayFilterChain)` 方法，使用 `delegate` 过滤请求。

## 2.3 GatewayFilterAdapter

`org.springframework.cloud.gateway.handler.FilteringWebHandler.GatewayFilterAdapter` ，网关过滤器适配器。在 GatewayFilterChain 使用 GatewayFilter 过滤请求，所以通过 GatewayFilterAdapter 将 GlobalFilter 适配成 GatewayFilter 。GatewayFilterAdapter 代码如下 ：

```Java
private static class GatewayFilterAdapter implements GatewayFilter {

    private final GlobalFilter delegate;
    
    public GatewayFilterAdapter(GlobalFilter delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return this.delegate.filter(exchange, chain);
    }
}
```

* `delegate` 属性，委托的 GlobalFilter 。
* `#filter(ServerWebExchange, GatewayFilterChain)` 方法，使用 `delegate` 过滤请求。

-------

在 FilteringWebHandler 初始化时，将 GlobalFilter 委托成 GatewayFilterAdapter ，代码如下 ：

```Java
  1: public class FilteringWebHandler implements WebHandler {
  2: 
  3:     /**
  4:      * 全局过滤器
  5:      */
  6: 	private final List<GatewayFilter> globalFilters;
  7: 
  8: 	public FilteringWebHandler(List<GlobalFilter> globalFilters) {
  9: 		this.globalFilters = loadFilters(globalFilters);
 10: 	}
 11: 
 12: 	private static List<GatewayFilter> loadFilters(List<GlobalFilter> filters) {
 13: 		return filters.stream()
 14: 				.map(filter -> {
 15: 					GatewayFilterAdapter gatewayFilter = new GatewayFilterAdapter(filter);
 16: 					if (filter instanceof Ordered) {
 17: 						int order = ((Ordered) filter).getOrder();
 18: 						return new OrderedGatewayFilter(gatewayFilter, order);
 19: 					}
 20: 					return gatewayFilter;
 21: 				}).collect(Collectors.toList());
 22: 	}
 23: }
```

* 第 16 至 19 行 ：当 GlobalFilter 子类实现了 `org.springframework.core.Ordered` 接口，在委托一层 OrderedGatewayFilter 。这样 `AnnotationAwareOrderComparator#sort(List)` 方法好排序。
* 第 20 行 ：当 GlobalFilter 子类**没有**实现了 `org.springframework.core.Ordered` 接口，在 `AnnotationAwareOrderComparator#sort(List)` 排序时，顺序值为 `Integer.MAX_VALUE` 。
* 目前 GlobalFilter 都实现了 `org.springframework.core.Ordered` 接口。

# 3. GlobalFilter

`org.springframework.cloud.gateway.filter.GlobalFilter` ，全局过滤器**接口**，代码如下 ：

```Java
public interface GlobalFilter {

	/**
	 * Process the Web request and (optionally) delegate to the next
	 * {@code WebFilter} through the given {@link GatewayFilterChain}.
	 * @param exchange the current server exchange
	 * @param chain provides a way to delegate to the next filter
	 * @return {@code Mono<Void>} to indicate when request processing is complete
	 */
	Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);

}
```

* GlobalFilter 和 GatewayFilter 的 `#filter(ServerWebExchange, GatewayFilterChain)` **方法签名一致**。官方说，未来的版本将作出一些调整。

    > FROM [《Spring Cloud Gateway》](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/docs/src/main/asciidoc/spring-cloud-gateway.adoc#path-route-predicate-factory#user-content-global-filters)    
    > This interface and usage are subject to change in future milestones

* GlobalFilter 会作用到**所有的** Route 上。

GlobalFilter 实现类如下类图 ：




