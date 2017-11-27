# 1. 概述

本文主要分享 **RoutePredicateFactory 路由谓语工厂**。

RoutePredicateFactory 涉及到的类在 `org.springframework.cloud.gateway.handler.predicate` 包下，如下图 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_15/01.png)

Spring Cloud Gateway 创建 Route 对象时，使用 RoutePredicateFactory 创建 Predicate 对象。Predicate 对象可以赋值给 [`Route.predicate`](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/route/Route.java#L51) 属性，用于匹配**请求**对应的 Route 。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。

# 2. RoutePredicateFactory

`org.springframework.cloud.gateway.handler.predicate.RoutePredicateFactory` ， 路由谓语工厂**接口**。代码如下 ：

```Java
@FunctionalInterface
public interface RoutePredicateFactory extends ArgumentHints {

    String PATTERN_KEY = "pattern";

	Predicate<ServerWebExchange> apply(Tuple args);

	default String name() {
		return NameUtils.normalizePredicateName(getClass());
	}

}
```

* `#name()` **默认**方法，调用 `NameUtils#normalizePredicateName(Class)` 方法，获得 RoutePredicateFactory 的名字。该方法截取类名**前半段**，例如 QueryRoutePredicateFactory 的结果为 `Query` 。点击 [链接](https://github.com/YunaiV/spring-cloud-gateway/blob/6bb8d6f93c289fd3a84c802ada60dd2bb57e1fb7/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/support/NameUtils.java#L33) 查看该方法。
* `#apply()` **接口**方法，创建 Predicate 。
* 继承 `org.springframework.cloud.gateway.support.ArgumentHints` **接口** ，在 [《Spring-Cloud-Gateway 源码解析 —— 路由（2.2）之 RouteDefinitionRouteLocator 路由配置》「2.4 获得 Tuple」](http://www.iocoder.cn/Spring-Cloud-Gateway/route-locator-route-definition/?self) 有使用到它的代码。

-------

RoutePredicateFactory **实现类**如下图 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_02_15/02.png)

下面我们一个一个 RoutePredicateFactory 实现类理解。

# 3. AfterRoutePredicateFactory

* Route 匹配 ：请求**时间**满足在配置时间**之后**。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: after_route
            uri: http://example.org
            predicates:
            - After=2017-01-20T17:42:47.789-07:00[America/Denver]
    ```

* 代码 ：

    ```Java
      1: public class AfterRoutePredicateFactory implements RoutePredicateFactory {
      2: 
      3: 	public static final String DATETIME_KEY = "datetime";
      4: 
      5: 	@Override
      6: 	public List<String> argNames() {
      7: 		return Collections.singletonList(DATETIME_KEY);
      8: 	}
      9: 
     10: 	@Override
     11: 	public Predicate<ServerWebExchange> apply(Tuple args) {
     12: 		Object value = args.getValue(DATETIME_KEY);
     13: 		final ZonedDateTime dateTime = BetweenRoutePredicateFactory.getZonedDateTime(value);
     14: 
     15: 		return exchange -> {
     16: 			final ZonedDateTime now = ZonedDateTime.now();
     17: 			return now.isAfter(dateTime);
     18: 		};
     19: 	}
     20: 
     21: }
    ```
    * 第 13 行 ：调用 `BetweenRoutePredicateFactory#getZonedDateTime(value)` 方法，解析配置的时间值，在 [「5. BetweenRoutePredicateFactory」](#) 详细解析。

# 4. BeforeRoutePredicateFactory

* Route 匹配 ：请求**时间**满足在配置时间**之前**。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: before_route
            uri: http://example.org
            predicates:
            - Before=2017-01-20T17:42:47.789-07:00[America/Denver]
    ```

* 代码 ：

    ```Java
      1: public class BeforeRoutePredicateFactory implements RoutePredicateFactory {
      2: 
      3: 	public static final String DATETIME_KEY = "datetime";
      4: 
      5: 	@Override
      6: 	public List<String> argNames() {
      7: 		return Collections.singletonList(DATETIME_KEY);
      8: 	}
      9: 
     10: 	@Override
     11: 	public Predicate<ServerWebExchange> apply(Tuple args) {
     12: 		Object value = args.getValue(DATETIME_KEY);
     13: 		final ZonedDateTime dateTime = BetweenRoutePredicateFactory.getZonedDateTime(value);
     14: 
     15: 		return exchange -> {
     16: 			final ZonedDateTime now = ZonedDateTime.now();
     17: 			return now.isBefore(dateTime);
     18: 		};
     19: 	}
     20: 
     21: }
    ```
    * 第 13 行 ：调用 `BetweenRoutePredicateFactory#getZonedDateTime(value)` 方法，解析配置的时间值，在 [「5. BetweenRoutePredicateFactory」](#) 详细解析。

# 5. BetweenRoutePredicateFactory

Route 匹配 ：请求**时间**满足在配置时间**之间**。

