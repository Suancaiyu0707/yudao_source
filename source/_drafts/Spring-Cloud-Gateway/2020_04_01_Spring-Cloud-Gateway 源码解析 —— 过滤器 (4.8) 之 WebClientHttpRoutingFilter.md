title: Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.8) 之 WebClientHttpRoutingFilter
date: 2020-04-01
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/filter-web-client-http-routing

-------

# 1. 概述

本文主要分享 **WebClientHttpRoutingFilter 的代码实现**。

WebClientHttpRoutingFilter ，Http **路由**网关过滤器。其根据 `http://` 或 `https://` 前缀( Scheme )过滤处理，使用基于 `org.springframework.cloud.gateway.filter.WebClient` 实现的 HttpClient 请求后端 Http 服务。

WebClientWriteResponseFilter ，与 WebClientHttpRoutingFilter **成对使用**的网关过滤器。其将 WebClientWriteResponseFilter 请求后端 Http 服务的**响应**写回客户端。

大体流程如下 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_04_01/01.png)

# 2. 环境配置

目前 WebClientHttpRoutingFilter / WebClientWriteResponseFilter 处于**实验**阶段，建议等正式发布在使用。

OK，下面我们来看看怎么配置环境。

第一步，在 NettyConfiguration 注释掉 `#routingFilter(...)` 和 `#nettyWriteResponseFilter()` 两个 Bean 方法。

第二步，在 GatewayAutoConfiguration 打开 `#webClientHttpRoutingFilter()` 和 `#webClientWriteResponseFilter()` 两个 Bean 方法。

第三步，配置完成，启动 Spring Cloud Gateway 。

# 3. WebClientHttpRoutingFilter

`org.springframework.cloud.gateway.filter.WebClientHttpRoutingFilter` ，Http **路由**网关过滤器。

**构造方法**，代码如下 ：

```Java
public class WebClientHttpRoutingFilter implements GlobalFilter, Ordered {

	private final WebClient webClient;

	public WebClientHttpRoutingFilter(WebClient webClient) {
		this.webClient = webClient;
	}}
```

* `webClient` 属性，默认情况下，使用 `org.springframework.web.reactive.function.client.DefaultWebClient` 实现类。通过该属性，**请求后端的 Http 服务**。

-------

`#getOrder()` 方法，代码如下 ：

```Java
@Override
public int getOrder() {
    return Ordered.LOWEST_PRECEDENCE;
}
```
* 返回顺序为 `Integer.MAX_VALUE` 。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.1) 之 GatewayFilter 一览》「3. GlobalFilter」](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/?self) ，我们列举了所有 GlobalFilter 的顺序。

-------

`#filter(ServerWebExchange, GatewayFilterChain)` 方法，代码如下 ：

```Java
  1: @Override
  2: public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
  3: 	// 获得 requestUrl
  4: 	URI requestUrl = exchange.getRequiredAttribute(GATEWAY_REQUEST_URL_ATTR);
  5: 
  6: 	// 判断是否能够处理
  7: 	String scheme = requestUrl.getScheme();
  8: 	if (isAlreadyRouted(exchange) || (!scheme.equals("http") && !scheme.equals("https"))) {
  9: 		return chain.filter(exchange);
 10: 	}
 11: 
 12: 	// 设置已经路由
 13: 	setAlreadyRouted(exchange);
 14: 
 15: 	ServerHttpRequest request = exchange.getRequest();
 16: 
 17: 	//TODO: support forms
 18: 	// Request Method
 19: 	HttpMethod method = request.getMethod();
 20: 
 21: 	// Request
 22: 	RequestBodySpec bodySpec = this.webClient.method(method)
 23: 			.uri(requestUrl)
 24: 			.headers(httpHeaders -> {
 25: 				httpHeaders.addAll(request.getHeaders());
 26: 				httpHeaders.remove(HttpHeaders.HOST);
 27: 			});
 28: 
 29: 	// Request Body
 30: 	RequestHeadersSpec<?> headersSpec;
 31: 	if (requiresBody(method)) {
 32: 		headersSpec = bodySpec.body(BodyInserters.fromDataBuffers(request.getBody()));
 33: 	} else {
 34: 		headersSpec = bodySpec;
 35: 	}
 36: 
 37: 	return headersSpec.exchange()
 38: 			// .log("webClient route")
 39: 			.flatMap(res -> {
 40: 				ServerHttpResponse response = exchange.getResponse();
 41: 
 42: 				// Response Header
 43: 				response.getHeaders().putAll(res.headers().asHttpHeaders());
 44: 
 45: 				// Response Status
 46: 				response.setStatusCode(res.statusCode());
 47: 
 48: 				// 设置 Response 到 CLIENT_RESPONSE_ATTR
 49: 				// Defer committing the response until all route filters have run
 50: 				// Put client response as ServerWebExchange attribute and write response later NettyWriteResponseFilter
 51: 				exchange.getAttributes().put(CLIENT_RESPONSE_ATTR, res);
 52: 				return chain.filter(exchange);
 53: 			});
 54: }
```

* 第 4 行 ：获得 `requestUrl` 。
* 第 7 至 10 行 ：判断 ForwardRoutingFilter 是否能够处理该请求，需要满足两个条件 ：
    * `http://` 或者 `https://` 前缀( Scheme ) 。
    * 调用 `ServerWebExchangeUtils#isAlreadyRouted(ServerWebExchange)` 方法，判断该请求暂未被其他 Routing 网关处理。代码如下 ：

        ```Java
        public static boolean isAlreadyRouted(ServerWebExchange exchange) {
            return exchange.getAttributeOrDefault(GATEWAY_ALREADY_ROUTED_ATTR, false);
        }
        ```
        * x
* 第 13 行 ：设置该请求已经被处理。代码如下 ：

    ```Java
    public static void setAlreadyRouted(ServerWebExchange exchange) {
        exchange.getAttributes().put(GATEWAY_ALREADY_ROUTED_ATTR, true);
    }
    ```

* 第 17 行 ：TODO 【3025】 目前暂不支持 forms 参数
* 第 22 至 35 行 ：**创建**向后端服务的请求。
    * 第 22 行 ：设置 Method 属性。
    * 第 24 至 27 行 ：设置 Header 属性。
    * 第 30 至 35 行 ：设置 Body 属性。
* 第 37 行 ：**发起**向后端服务的请求。
* 第 40 至 53 行 ：**处理**返回自后端服务的相应。
    * 第 43 行 ：设置 `response` 的 Header 属性。
    * 第 46 行 ：设置 `response` 的 Status 属性。
    * 第 51 行 ：设置 `res` 到 `CLIENT_RESPONSE_ATTR` 。后续 WebClientWriteResponseFilter 将响应**写回**给客户端。
    * 第 52 行 ：提交过滤器链继续过滤。

# 4. WebClientWriteResponseFilter

`org.springframework.cloud.gateway.filter.WebClientWriteResponseFilter` ，Http 回写**响应**网关过滤器。

`#getOrder()` 方法，代码如下 ：

```Java
public static final int WRITE_RESPONSE_FILTER_ORDER = -1;
    
@Override
public int getOrder() {
    return WRITE_RESPONSE_FILTER_ORDER;
}
```
* 返回顺序为 `-1` 。在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.1) 之 GatewayFilter 一览》「3. GlobalFilter」](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-intro/?self) ，我们列举了所有 GlobalFilter 的顺序。

-------

`#filter(ServerWebExchange, GatewayFilterChain)` 方法，代码如下 ：

```Java
  1: @Override
  2: public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
  3: 	// NOTICE: nothing in "pre" filter stage as CLIENT_RESPONSE_ATTR is not added
  4: 	// until the WebHandler is run
  5: 	return chain.filter(exchange).then(Mono.defer(() -> {
  6: 	    // 获得 Response
  7: 		ClientResponse clientResponse = exchange.getAttribute(CLIENT_RESPONSE_ATTR);
  8: 		if (clientResponse == null) {
  9: 			return Mono.empty();
 10: 		}
 11: 		log.trace("WebClientWriteResponseFilter start");
 12: 		ServerHttpResponse response = exchange.getResponse();
 13: 
 14: 		return response.writeWith(clientResponse.body(BodyExtractors.toDataBuffers())).log("webClient response");
 15: 	}));
 16: }
```

* 第 5 行 ：调用 `#then(Mono)` 方法，实现 **After Filter** 逻辑。
* 第 7 至 11 行 ：从 `CLIENT_RESPONSE_ATTR` 中，获得 ClientResponse 。
* 第 14 行 ：将 ClientResponse 写回给客户端。

# 5. 和 NettyRoutingFilter 对比

在 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.7) 之 NettyRoutingFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-netty-routing/?self) 中，我们知道 NettyRoutingFilter / NettyWriteResponseFilter 和 WebClientHttpRoutingFilter / WebClientHttpRoutingFilter 实现**一样**的功能。

那么为什么要再实现一次呢？

TODO 【3001】

# 666. 彩蛋

呼呼，主要的过滤器已经写完，后面熔断、限流过滤器的实现。

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_04_01/02.png)

胖友，分享一波朋友圈可好！


