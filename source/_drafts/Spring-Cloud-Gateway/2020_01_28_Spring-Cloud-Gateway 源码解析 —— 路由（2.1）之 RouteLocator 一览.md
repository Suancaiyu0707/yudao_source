title: Spring-Cloud-Gateway 源码解析 —— 路由（2.1）之 RouteLocator 一览
date: 2020-01-28
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/route-locator-intro

---

# 1. 概述

本文主要对 **路由定位器 RouteLocator 做整体的认识**。

在 [《Spring-Cloud-Gateway 源码解析 —— 路由（1.1）之 RouteDefinitionLocator 一览》](http://www.iocoder.cn/Spring-Cloud-Gateway/route-definition-locator-intro/?self) 中，我们对 RouteLocator 对了简单的介绍 ：

* [](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_10/01.png)
* RouteLocator 可以直接**自定义**路由( `org.springframework.cloud.gateway.route.Route` ) ，也可以通过 RouteDefinitionRouteLocator 获取 RouteDefinition ，并转换成 Route 。 
* RoutePredicateHandlerMapping 使用 RouteLocator 获得 Route 信息。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。

# 2. Route

`org.springframework.cloud.gateway.route.Route` ，路由。代码如下 ：

```Java
public class Route implements Ordered {

    /**
     * 路由编号
     */
	private final String id;
    /**
     * 路由向的 URI
     */
	private final URI uri;
    /**
     * 顺序
     */
	private final int order;
    /**
     * 断言数组
     */
	private final Predicate<ServerWebExchange> predicate;
    /**
     * 过滤器数组
     */
	private final List<GatewayFilter> gatewayFilters;
}
```

* `id` 属性，ID 编号，**唯一**。
* `predicates` 属性，断言数组。**请求**通过 `predicates` 判断是否**匹配**。
* `filters` 属性，过滤器数组。
* `uri` 属性，路由向的 URI 。
* `order` 属性，顺序。当请求匹配到多个路由时，使用顺序**小**的。
* [](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_10/03.png)

-------

Route 内置 Builder 类，点击 [链接](https://github.com/YunaiV/spring-cloud-gateway/blob/382a4cd98fbb8ac53a83a5559bacb0f885838074/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/Route.java#L67) 查看。

 -------

Route 提供 `routeDefinition` RouteDefinition 创建对象，代码如下 ：

```Java
public static Builder builder(RouteDefinition routeDefinition) {
	return new Builder()
			.id(routeDefinition.getId())
			.uri(routeDefinition.getUri())
			.order(routeDefinition.getOrder());
}
```

* `predicate` / `gatewayFilters` 属性，需要调用 Builder 相关方法进行设置。

# 3. RouteLocator

`org.springframework.cloud.gateway.route.RouteLocator` ，路由定位器**接口**，定义获得路由数组的方法。代码如下 ：

```Java
public interface RouteLocator {

	Flux<Route> getRoutes();

}
```

* 对 Reactor Flux 暂时不熟悉的同学，可以阅读完本文 Google 进行学习。随着 Spring 5 对响应式编程的推广，厉害如你一定要去掌握。

在上文中，我们也看到了 RouteLocator 的多个实现类，类图如下 ：

[](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_28/01.png)

* 本文只解析 CompositeRouteLocator / CachingRouteLocator 的源码实现。其他的实现类会在后面文章详细解析。
* 自定义的 RouteLocator ，通过**内部类**实现，类图暂时不好体现。

# 4. CompositeRouteLocator

`org.springframework.cloud.gateway.route.CompositeRouteLocator` ，组合**多种** RouteLocator 的实现类，为 RoutePredicateHandlerMapping 提供**统一**入口访问路由。代码如下 ：

```Java
public class CompositeRouteLocator implements RouteLocator {

	private final Flux<RouteLocator> delegates;

	public CompositeRouteLocator(Flux<RouteLocator> delegates) {
		this.delegates = delegates;
	}

	@Override
	public Flux<Route> getRoutes() {
		return this.delegates.flatMap(RouteLocator::getRoutes);
	}
}
```

* `#getRoutes()` 方法，提供**统一**方法，将组合的 `delegates` 的路由**全部**返回。

# 5. CachingRouteLocator

`org.springframework.cloud.gateway.route.CachingRouteLocator` ，**缓存**路由的 RouteLocator 实现类。RoutePredicateHandlerMapping 调用 CachingRouteLocator 的 `RouteLocator#getRoutes()` 方法，获取路由。


CachingRouteLocator 代码如下 ：

```Java
public class CachingRouteLocator implements RouteLocator {

	private final RouteLocator delegate;
    /**
     * 路由缓存
     */
	private final AtomicReference<List<Route>> cachedRoutes = new AtomicReference<>();

	public CachingRouteLocator(RouteLocator delegate) {
		this.delegate = delegate;
		this.cachedRoutes.compareAndSet(null, collectRoutes());
	}

	@Override
	public Flux<Route> getRoutes() {
		return Flux.fromIterable(this.cachedRoutes.get());
	}

	/**
	 * Sets the new routes
	 * @return old routes
	 */
	public Flux<Route> refresh() {
		return Flux.fromIterable(this.cachedRoutes.getAndUpdate(
				routes -> CachingRouteLocator.this.collectRoutes()));
	}

	private List<Route> collectRoutes() {
		List<Route> routes = this.delegate.getRoutes().collectList().block();
		// 排序
		AnnotationAwareOrderComparator.sort(routes);
		return routes;
	}

	@EventListener(RefreshRoutesEvent.class)
    /* for testing */ void handleRefresh() {
        refresh();
    }
}
```

* `cachedRoutes` 属性，路由**缓存**。
* CachingRouteLocator **构造**方法，调用 `#collectRoutes()` 方法获得路由，并缓存到 `cachedRoutes` 属性。
* `#collectRoutes()` 方法，从 `delegate` 获取路由数组。
* `#getRoutes()` 方法，返回路由**缓存**。
* `#refresh()` 方法，刷新**缓存** `cachedRoutes` 属性。
* `#handleRefresh()` 方法，监听 [`org.springframework.context.ApplicationEvent.RefreshRoutesEvent`](https://github.com/YunaiV/spring-cloud-gateway/blob/382a4cd98fbb8ac53a83a5559bacb0f885838074/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/RefreshRoutesEvent.java) 事件，刷新**缓存**。

-------

GatewayWebfluxEndpoint 有**一个** HTTP API 调用了 ApplicationEventPublisher ，发布 RefreshRoutesEvent 事件。代码如下 ：

```Java
@RestController
@RequestMapping("${management.context-path:/application}/gateway")
public class GatewayWebfluxEndpoint implements ApplicationEventPublisherAware {

    // ... 省略其他代码
    
    /**
     * 应用事件发布器
     */
	private ApplicationEventPublisher publisher;
	
	@PostMapping("/refresh")
	public Mono<Void> refresh() {
	    this.publisher.publishEvent(new RefreshRoutesEvent(this));
		return Mono.empty();
	}

}
```

* `POST "/refresh` ，发布 RefreshRoutesEvent 事件。CachingRouteLocator 监听到该事件，刷新缓存。

# 666. 彩蛋

又是比较干爽( 水更 )的一篇文章。

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_28/02.png)

胖友，分享一波朋友圈可好！


