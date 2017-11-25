title: Spring-Cloud-Gateway 源码解析 —— 路由（1.2）之 PropertiesRouteDefinitionLocator 配置文件
date: 2020-01-15
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/route-definition-locator-properties

---

# 1. 概述

本文主要对 **PropertiesRouteDefinitionLocator 的源码实现**。

[](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_15/01.jpeg)

* **蓝色**部分 ：PropertiesRouteDefinitionLocator 。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。

# 2. PropertiesRouteDefinitionLocator

`org.springframework.cloud.gateway.config.PropertiesRouteDefinitionLocator` ，从**配置文件**( 例如，YML / Properties 等 ) 读取路由配置。代码如下 ：

```Java
public class PropertiesRouteDefinitionLocator implements RouteDefinitionLocator {

	private final GatewayProperties properties;

	public PropertiesRouteDefinitionLocator(GatewayProperties properties) {
		this.properties = properties;
	}

	@Override
	public Flux<RouteDefinition> getRouteDefinitions() {
		return Flux.fromIterable(this.properties.getRoutes());
	}
}
```

* `#getRouteDefinitions()` 方法，从 **GatewayProperties** 获取路由配置数组。

# 3. GatewayProperties

`org.springframework.cloud.gateway.config.GatewayProperties` ，从配置文件**读取** ：

* 路由配置
* **默认**过滤器配置。当 RouteDefinition => Route 时，会将过滤器配置添加到**每个** Route 。

GatewayProperties 代码如下 ：

```Java
@ConfigurationProperties("spring.cloud.gateway")
@Validated
public class GatewayProperties {

	/**
	 * List of Routes
	 */
	@NotNull
	@Valid
	private List<RouteDefinition> routes = new ArrayList<>();

	/**
	 * List of filter definitions that are applied to every route.
	 */
	private List<FilterDefinition> defaultFilters = loadDefaults();

	private ArrayList<FilterDefinition> loadDefaults() {
		ArrayList<FilterDefinition> defaults = new ArrayList<>();
		FilterDefinition definition = new FilterDefinition();
		definition.setName(normalizeFilterName(RemoveNonProxyHeadersGatewayFilterFactory.class));
		defaults.add(definition);
		return defaults;
	}
}
```

* `routes` 属性，路由配置。通过 `spring.cloud.gateway.routes` 配置。以 YAML 配置文件举例子 ： 

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - host_example_to_httpbin=${test.uri}, Host=**.example.org
    
          # =====================================
          - id: host_foo_path_headers_to_httpbin
            uri: ${test.uri}
            predicates:
            - Host=**.foo.org
            - Path=/headers
            - Method=GET
            - Header=X-Request-Id, \d+
            - Query=foo, ba.
            - Query=baz
            - Cookie=chocolate, ch.p
            - After=1900-01-20T17:42:47.789-07:00[America/Denver]
            filters:
            - AddResponseHeader=X-Response-Foo, Bar
    
          # =====================================
          - id: add_request_header_test
            uri: ${test.uri}
            predicates:
            - Host=**.addrequestheader.org
            - Path=/headers
            filters:
            - AddRequestHeader=X-Request-Foo, Bar
    ```
    * 更多例子，点击 [application.yml](https://github.com/YunaiV/spring-cloud-gateway/blob/382a4cd98fbb8ac53a83a5559bacb0f885838074/spring-cloud-gateway-core/src/test/resources/application.yml#L15) 查看。

* `defaultFilters` 属性，**默认**过滤器配置。通过 `spring.cloud.gateway.default-filters` 配置。以 YAML 配置文件举例子 ：

    ```YAML
    spring:
      cloud:
        gateway:
          default-filters:
          - AddResponseHeader=X-Response-Default-Foo, Default-Bar
          - PrefixPath=/httpbin
    ```
    * 更多例子，点击 [application.yml](https://github.com/YunaiV/spring-cloud-gateway/blob/382a4cd98fbb8ac53a83a5559bacb0f885838074/spring-cloud-gateway-core/src/test/resources/application.yml#L10) 查看。

# 666. 彩蛋

TODO 【3017】 与 Spring Cloud Config 结合

[](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_15/02.png)


