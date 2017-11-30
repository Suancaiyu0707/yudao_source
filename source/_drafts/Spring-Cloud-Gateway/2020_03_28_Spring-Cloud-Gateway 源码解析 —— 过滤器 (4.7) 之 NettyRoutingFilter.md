title: Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.7) 之 NettyRoutingFilter
date: 2020-03-28
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/filter-netty-routing

-------

# 1. 概述

本文主要分享 **NettyRoutingFilter 的代码实现**。

NettyRoutingFilter ，Netty **路由**网关过滤器。其根据 `http://` 或 `https://` 前缀( Scheme )过滤处理，使用基于 Netty 实现的 HttpClient 请求后端 Http 服务。

NettyWriteResponseFilter ，与 NettyRoutingFilter **成对使用**的网关过滤器。其将 NettyRoutingFilter 请求后端 Http 服务的**响应**写回客户端。

大体流程如下 ：

[](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_03_28/01.png)

另外，Spring Cloud Gateway 实现了 WebClientHttpRoutingFilter / WebClientWriteResponseFilter ，功能上和 NettyRoutingFilter / NettyWriteResponseFilter **相同**，差别在于基于 `org.springframework.cloud.gateway.filter.WebClient` 实现的 HttpClient 请求后端 Http 服务。在 [TODO 【3023】]() ，我们会详细解析。

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。

# 2. NettyRoutingFilter

`org.springframework.cloud.gateway.filter.NettyRoutingFilter` ，Netty **路由**网关过滤器。

**构造方法**，代码如下 ：

```
public class NettyRoutingFilter implements GlobalFilter, Ordered {

	private final HttpClient httpClient;

	public NettyRoutingFilter(HttpClient httpClient) {
		this.httpClient = httpClient;
	}}
```

* `httpClient` 属性，基于 **Netty** 实现的 HttpClient 。通过该属性，**请求后端的 Http 服务**。

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
 17: 	// Request Method
 18: 	final HttpMethod method = HttpMethod.valueOf(request.getMethod().toString());
 19: 
 20: 	// 获得 url
 21: 	final String url = requestUrl.toString();
 22: 
 23: 	// Request Header
 24: 	final DefaultHttpHeaders httpHeaders = new DefaultHttpHeaders();
 25: 	request.getHeaders().forEach(httpHeaders::set);
 26: 
 27: 	// 请求
 28: 	return this.httpClient.request(method, url, req -> {
 29: 		final HttpClientRequest proxyRequest = req.options(NettyPipeline.SendOptions::flushOnEach)
 30: 				.failOnClientError(false) // // 是否请求失败，抛出异常
 31: 				.headers(httpHeaders);
 32: 
 33: 		// Request Form
 34: 		if (MediaType.APPLICATION_FORM_URLENCODED.includes(request.getHeaders().getContentType())) {
 35: 			return exchange.getFormData()
 36: 					.flatMap(map -> proxyRequest.sendForm(form -> {
 37: 						for (Map.Entry<String, List<String>> entry: map.entrySet()) {
 38: 							for (String value : entry.getValue()) {
 39: 								form.attr(entry.getKey(), value);
 40: 							}
 41: 						}
 42: 					}).then())
 43: 					.then(chain.filter(exchange));
 44: 		}
 45: 
 46: 		// Request Body
 47: 		return proxyRequest.sendHeaders() //I shouldn't need this
 48: 				.send(request.getBody()
 49: 						.map(DataBuffer::asByteBuffer) // Flux<DataBuffer> => ByteBuffer
 50: 						.map(Unpooled::wrappedBuffer)); // ByteBuffer => Flux<DataBuffer>
 51: 	}).doOnNext(res -> {
 52: 		ServerHttpResponse response = exchange.getResponse();
 53: 		// Response Header
 54: 		// put headers and status so filters can modify the response
 55: 		HttpHeaders headers = new HttpHeaders();
 56: 		res.responseHeaders().forEach(entry -> headers.add(entry.getKey(), entry.getValue()));
 57: 		response.getHeaders().putAll(headers);
 58: 
 59: 		// Response Status
 60: 		response.setStatusCode(HttpStatus.valueOf(res.status().code()));
 61: 
 62: 		// 设置 Response 到 CLIENT_RESPONSE_ATTR
 63: 		// Defer committing the response until all route filters have run
 64: 		// Put client response as ServerWebExchange attribute and write response later NettyWriteResponseFilter
 65: 		exchange.getAttributes().put(CLIENT_RESPONSE_ATTR, res);
 66: 	}).then(chain.filter(exchange));
 67: }
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

* 第 18 行 ：创建 **Netty Request Method** 对象。`request#getMethod()` 返回的不是 `io.netty.handler.codec.http.HttpMethod` ，所以需要进行转换。
* 第 21 行 ：获得 `url` 。
* 第 24 至 25 行 ：创建  **Netty Request Header** 对象( `io.netty.handler.codec.http.DefaultHttpHeaders` )，将请求的 Header 设置给它。
* ---------- 第 28 至 50 行 ：调用 `HttpClient#request(HttpMethod, String, Function)` 方法，请求后端 Http 服务。
* 第 29 至 31 行 ：创建 **Netty Request** 对象( `reactor.ipc.netty.http.client.HttpClientRequest` )。
    * 第 29 行 ：TODO 【3024】 NettyPipeline.SendOptions::flushOnEach
    * 第 30 行 ：设置请求失败( 后端服务返回响应状体码 `>= 400 ` )时，不抛出异常。相关代码如下 ：

        ```Java
        // HttpClientOperations#checkResponseCode(HttpResponse response)
            
        // ... 省略无关代码
        
        if (code >= 400) {
        	if (clientError) {
        		if (log.isDebugEnabled()) {
        			log.debug("{} Received Request Error, stop reading: {}",
        					channel(),
        					response.toString());
        		}
        		Exception ex = new HttpClientException(uri(), response);
        		parentContext().fireContextError(ex);
        		receive().subscribe();
        		return false;
        	}
        	return true;
        }
        
        ```    
        * 通过设置 `clientError = false` ，第 51 行可以调用 `Mono#doNext(Consumer)` 方法，**统一订阅处理**返回的 `reactor.ipc.netty.http.client.HttpClientResponse` 对象。
    * 第 31 行 ：设置 **Netty Request** 对象的 Header 。
* 第 34 至 44 行 ：【TODO 3025】目前是一个 BUG ，在 2.0.x 版本修复。见 [FormIntegrationTests#formUrlencodedWorks()](FormIntegrationTests) 单元测试的注释说明。
* 第 47 至 50 行 ：请求后端的 Http 服务。
    * 第 47 行 ：发送请求 Header 。
    * 第 48 至 50 行 ：发送请求 Body 。其中中间的 `#map(...)` 的过程为 `Flux<DataBuffer> => ByteBuffer => Flux<DataBuffer>` 。

* ---------- 第 51 至 65 行 ：请求后端 Http 服务**完成**，将 **Netty Response** 赋值给响应 `response` 。
* 第 53 至 57 行 ：创建 `org.springframework.http.HttpHeaders` 对象，将 **Netty Response Header** 设置给它，而后设置回给响应 `response` 。
* 第 60 行 ：设置响应 `response` 的状态码。
* 第 65 行 ：设置 **Netty Response** 到 `CLIENT_RESPONSE_ATTR` 。后续 NettyWriteResponseFilter 将 **Netty Response** 写回给客户端。
* ---------- 第 66 行 ：提交过滤器链继续过滤。

# 3. NettyWriteResponseFilter

`org.springframework.cloud.gateway.filter.NettyWriteResponseFilter` ，Netty 回写**响应**网关过滤器。

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
  7: 		HttpClientResponse clientResponse = exchange.getAttribute(CLIENT_RESPONSE_ATTR);
  8: 		// HttpClientResponse clientResponse = getAttribute(exchange, CLIENT_RESPONSE_ATTR, HttpClientResponse.class);
  9: 		if (clientResponse == null) {
 10: 			return Mono.empty();
 11: 		}
 12: 		log.trace("NettyWriteResponseFilter start");
 13: 		ServerHttpResponse response = exchange.getResponse();
 14: 
 15: 		// 将 Netty Response 写回给客户端。
 16: 		NettyDataBufferFactory factory = (NettyDataBufferFactory) response.bufferFactory();
 17: 		//TODO: what if it's not netty
 18: 		final Flux<NettyDataBuffer> body = clientResponse.receive()
 19: 				.retain() // ByteBufFlux => ByteBufFlux
 20: 				.map(factory::wrap); // ByteBufFlux  => Flux<NettyDataBuffer>
 21: 		return response.writeWith(body);
 22: 	}));
 23: }
```

* 第 5 行 ：调用 `#then(Mono)` 方法，实现 **After Filter** 逻辑。
* 第 7 至 11 行 ：从 `CLIENT_RESPONSE_ATTR` 中，获得 **Netty Response** 。
* 第 15 至 21 行 ：将 **Netty Response** 写回给客户端。因为 `org.springframework.http.server.reactive#writeWith(Publisher<? extends DataBuffer>)` 需要的参数类型是 `Publisher<? extends DataBuffer>` ，所以【第 18 至 20 行】的转换过程是 `ByteBufFlux => Flux<NettyDataBuffer>` 。
    * 第 19 行 ：TODO 【3024】ByteBufFlux#retain() 

# 666. 彩蛋

下一篇 [《Spring-Cloud-Gateway 源码解析 —— 过滤器 (4.8) 之 WebClientHttpRoutingFilter》](http://www.iocoder.cn/Spring-Cloud-Gateway/filter-web-client-http-routing) 走起！




