# 1. 概述

本文主要分享 **GatewayFilterFactory 的实现类**。

GatewayFilterFactory 实现类较多，根据用途整理如下脑图 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_03_05/01.png)

下面我们开始逐块解析源码实现。

# 2. Header 

本小节分享 Header 相关的 GatewayFilterFactory 实现类。

## 2.1 AddRequestHeaderGatewayFilterFactory

* 用途 ：添加指定请求 Header 为指定值。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: add_request_header_route
            uri: http://example.org
            filters:
            - AddRequestHeader=X-Request-Foo, Bar
    ```

* 代码 ：

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
     14:             // 创建新的 ServerHttpRequest
     15: 			ServerHttpRequest request = exchange.getRequest().mutate()
     16: 					.header(name, value)
     17: 					.build();
     18: 
     19: 			// 创建新的 ServerWebExchange ，提交过滤器链继续过滤
     20: 			return chain.filter(exchange.mutate().request(request).build());
     21: 		};
     22: 	}
     23: }
    ```
    
    * Tuple 参数 ：`name` / `value` 。
    * 第 14 至 17 行 ：创建**新**的 ServerHttpRequest 。
    * 第 19 至 20 行 ：创建**新**的 ServerWebExchange ，提交**过滤器链**继续过滤。

## 2.2 RemoveRequestHeaderGatewayFilterFactory 

类似 AddRequestHeaderGatewayFilterFactory ，不重复分享，点击 [《Spring Cloud Gateway —— RemoveRequestHeader GatewayFilter Factory》](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/docs/src/main/asciidoc/spring-cloud-gateway.adoc#removerequestheader-gatewayfilter-factory) 查看官方文档。

## 2.3 AddResponseHeaderGatewayFilterFactory

类似 AddRequestHeaderGatewayFilterFactory ，不重复分享，点击 [《Spring Cloud Gateway —— AddResponseHeader GatewayFilter Factory》](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/docs/src/main/asciidoc/spring-cloud-gateway.adoc#addresponseheader-gatewayfilter-factory) 查看官方文档。

## 2.4 RemoveResponseHeaderGatewayFilterFactory

类似 AddRequestHeaderGatewayFilterFactory ，不重复分享，点击 [《Spring Cloud Gateway —— RemoveResponseHeader GatewayFilter Factory》](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/docs/src/main/asciidoc/spring-cloud-gateway.adoc#removeresponseheader-gatewayfilter-factory) 查看官方文档。

## 2.5 SetResponseHeaderGatewayFilterFactory

类似 AddRequestHeaderGatewayFilterFactory ，不重复分享，点击 [《Spring Cloud Gateway —— SetStatus GatewayFilter Factory》](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/docs/src/main/asciidoc/spring-cloud-gateway.adoc#removeresponseheader-gatewayfilter-factory) 查看官方文档。

## 2.6 RemoveNonProxyHeadersGatewayFilterFactory

* 用途 ：移除请求 **Proxy** 相关的 Header 。默认值为 `[ "Connection", "Keep-Alive", "Proxy-Authenticate", "Proxy-Authorization", "TE", "Trailer", "Transfer-Encoding", "Upgrade" ]` ( 参考自 ：[https://tools.ietf.org/html/draft-ietf-httpbis-p1-messaging-14#section-7.1.3](https://tools.ietf.org/html/draft-ietf-httpbis-p1-messaging-14#section-7.1.3) ) ，可以通过 `spring.cloud.gateway.filter.remove-non-proxy-headers` 配置。
* 代码 ：

    ```Java
     1: @ConfigurationProperties("spring.cloud.gateway.filter.remove-non-proxy-headers")
      2: public class RemoveNonProxyHeadersGatewayFilterFactory implements GatewayFilterFactory {
      3: 
      4:     /**
      5:      * 默认
      6:      */
      7: 	public static final String[] DEFAULT_HEADERS_TO_REMOVE = new String[] {"Connection", "Keep-Alive",
      8: 			"Proxy-Authenticate", "Proxy-Authorization", "TE", "Trailer", "Transfer-Encoding", "Upgrade"};
      9: 
     10: 	private List<String> headers = Arrays.asList(DEFAULT_HEADERS_TO_REMOVE);
     11: 
     12: 	public List<String> getHeaders() {
     13: 		return headers;
     14: 	}
     15: 
     16: 	public void setHeaders(List<String> headers) {
     17: 		this.headers = headers;
     18: 	}
     19: 
     20: 	@Override
     21: 	public GatewayFilter apply(Tuple args) {
     22: 		//TODO: support filter args
     23: 
     24: 		return (exchange, chain) -> {
     25: 			// 创建新的 ServerHttpRequest
     26: 			ServerHttpRequest request = exchange.getRequest().mutate()
     27: 					.headers(httpHeaders -> {
     28: 						for (String header : this.headers) {
     29: 							httpHeaders.remove(header); // 移除
     30: 						}
     31: 					})
     32: 					.build();
     33: 
     34:             // 创建新的 ServerWebExchange ，提交过滤器链继续过滤
     35: 			return chain.filter(exchange.mutate().request(request).build());
     36: 		};
     37: 	}
     38: 
    ```

## 2.7 SecureHeadersGatewayFilterFactory

* 用途 ：添加响应 Secure 相关的 Header 。默认值在 [`org.springframework.cloud.gateway.filter.factory.SecureHeadersProperties`](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/filter/factory/SecureHeadersProperties.java#L27) ，可以通过 `spring.cloud.gateway.filter.secure-headers` 配置。
* 推荐文章 ：
    * [《Everything you need to know about HTTP security headers》](https://blog.appcanary.com/2017/http-security-headers.html)
    * [《4个常用的HTTP安全头部》](http://blog.jobbole.com/60143/) 
* 代码 ：

    ```Java
      1: public class SecureHeadersGatewayFilterFactory implements GatewayFilterFactory {
      2: 
      3: 	public static final String X_XSS_PROTECTION_HEADER = "X-Xss-Protection";
      4: 	public static final String STRICT_TRANSPORT_SECURITY_HEADER = "Strict-Transport-Security";
      5: 	public static final String X_FRAME_OPTIONS_HEADER = "X-Frame-Options";
      6: 	public static final String X_CONTENT_TYPE_OPTIONS_HEADER = "X-Content-Type-Options";
      7: 	public static final String REFERRER_POLICY_HEADER = "Referrer-Policy";
      8: 	public static final String CONTENT_SECURITY_POLICY_HEADER = "Content-Security-Policy";
      9: 	public static final String X_DOWNLOAD_OPTIONS_HEADER = "X-Download-Options";
     10: 	public static final String X_PERMITTED_CROSS_DOMAIN_POLICIES_HEADER = "X-Permitted-Cross-Domain-Policies";
     11: 
     12: 	private final SecureHeadersProperties properties;
     13: 
     14: 	public SecureHeadersGatewayFilterFactory(SecureHeadersProperties properties) {
     15: 		this.properties = properties;
     16: 	}
     17: 
     18: 	@Override
     19: 	public GatewayFilter apply(Tuple args) {
     20: 		//TODO: allow args to override properties
     21: 
     22: 		return (exchange, chain) -> {
     23: 			HttpHeaders headers = exchange.getResponse().getHeaders();
     24: 
     25: 			//TODO: allow header to be disabled
     26: 			headers.add(X_XSS_PROTECTION_HEADER, properties.getXssProtectionHeader());
     27: 			headers.add(STRICT_TRANSPORT_SECURITY_HEADER, properties.getStrictTransportSecurity());
     28: 			headers.add(X_FRAME_OPTIONS_HEADER, properties.getFrameOptions());
     29: 			headers.add(X_CONTENT_TYPE_OPTIONS_HEADER, properties.getContentTypeOptions());
     30: 			headers.add(REFERRER_POLICY_HEADER, properties.getReferrerPolicy());
     31: 			headers.add(CONTENT_SECURITY_POLICY_HEADER, properties.getContentSecurityPolicy());
     32: 			headers.add(X_DOWNLOAD_OPTIONS_HEADER, properties.getDownloadOptions());
     33: 			headers.add(X_PERMITTED_CROSS_DOMAIN_POLICIES_HEADER, properties.getPermittedCrossDomainPolicies());
     34: 
     35: 			return chain.filter(exchange);
     36: 		};
     37: 	}
     38: }
    ```
    
    * 第 26 至 33 行 ：添加响应 Secure 相关的 Header 。 

# 3. Parameter

本小节分享 Parameter 相关的 GatewayFilterFactory 实现类。

## 3.1 AddRequestParameterGatewayFilterFactory

类似 AddRequestHeaderGatewayFilterFactory ，不重复分享，点击 [《Spring Cloud Gateway —— SetStatus GatewayFilter Factory》](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/docs/src/main/asciidoc/spring-cloud-gateway.adoc#addrequestheader-gatewayfilter-factory) 查看官方文档。

# 4. Path

本小节分享 Path 相关的 GatewayFilterFactory 实现类。

## 4.1 RewritePathGatewayFilterFactory

* 用途 ：根据配置的正则表达式 `regexp` ，使用配置的 `replacement` 重写请求 Path 。从功能目的上类似 [《Module ngx_http_rewrite_module》](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html) 。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: rewritepath_route
            uri: http://example.org
            predicates:
            - Path=/foo/**
            filters:
            - RewritePath=/foo/(?<segment>.*), /$\{segment}
    ```
    * 注意，`$\` 用于替代 `$` ，避免和 YAML 语法冲突。

* 代码 ：

    ```Java
      1: public class RewritePathGatewayFilterFactory implements GatewayFilterFactory {
      2: 
      3: 	public static final String REGEXP_KEY = "regexp";
      4: 	public static final String REPLACEMENT_KEY = "replacement";
      5: 
      6: 	@Override
      7: 	public List<String> argNames() {
      8: 		return Arrays.asList(REGEXP_KEY, REPLACEMENT_KEY);
      9: 	}
     10: 
     11: 	@Override
     12: 	public GatewayFilter apply(Tuple args) {
     13: 		final String regex = args.getString(REGEXP_KEY);
     14: 		// `$\` 用于替代 `$` ，避免和 YAML 语法冲突。
     15: 		String replacement = args.getString(REPLACEMENT_KEY).replace("$\\", "$");
     16: 
     17: 		return (exchange, chain) -> {
     18: 			ServerHttpRequest req = exchange.getRequest();
     19: 			// 添加 原始请求URI 到 GATEWAY_ORIGINAL_REQUEST_URL_ATTR
     20: 			addOriginalRequestUrl(exchange, req.getURI());
     21: 			// 重写 Path
     22: 			String path = req.getURI().getPath();
     23: 			String newPath = path.replaceAll(regex, replacement);
     24: 
     25: 			// 创建新的 ServerHttpRequest
     26: 			ServerHttpRequest request = req.mutate()
     27: 					.path(newPath) // 设置 Path
     28: 					.build();
     29: 
     30: 			// 添加 请求URI 到 GATEWAY_REQUEST_URL_ATTR
     31: 			exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, request.getURI());
     32: 
     33: 			// 创建新的 ServerWebExchange ，提交过滤器链继续过滤
     34: 			return chain.filter(exchange.mutate().request(request).build());
     35: 		};
     36: 	}
     37: }
    ```
    * Tuple 参数 ：`regexp` / `replacement` 。
    * 第 15 行 ：`$\` 用于替代 `$` ，避免和 YAML 语法冲突。
    * 第 20 行 ：调用 `ServerWebExchangeUtils#addOriginalRequestUrl(...)` 添加原始请求 URI 到 `GATEWAY_ORIGINAL_REQUEST_URL_ATTR` 。代码如下 ：

        ```Java
        public static void addOriginalRequestUrl(ServerWebExchange exchange, URI url) {
        		exchange.getAttributes().computeIfAbsent(GATEWAY_ORIGINAL_REQUEST_URL_ATTR, s -> new LinkedHashSet<>()); // 数组，考虑多次重写
            LinkedHashSet<URI> uris = exchange.getRequiredAttribute(GATEWAY_ORIGINAL_REQUEST_URL_ATTR);
            uris.add(url);
        }
        ```
        * 为什么使用 LinkedHashSet ？因为可以使用 RewritePathGatewayFilterFactory / PrefixPathGatewayFilterFactory 多次重写。
    
    * 第 21 至 23 行 ：**重写**请求 Path 。
    * 第 26 至 28 行 ：创建**新**的 ServerHttpRequest 。
    * 第 31 行 ：添加请求 URI 到 `GATEWAY_REQUEST_URL_ATTR` 。
    * 第 34 行 ：创建**新**的 ServerWebExchange ，提交过滤器链继续过滤。

## 4.2 PrefixPathGatewayFilterFactory

类似 RewritePathGatewayFilterFactory ，不重复分享，点击 [《Spring Cloud Gateway —— PrefixPath GatewayFilter Factory》](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/docs/src/main/asciidoc/spring-cloud-gateway.adoc#setpath-gatewayfilter-factory) 查看官方文档。

## 4.3 SetPathGatewayFilterFactory

类似 RewritePathGatewayFilterFactory ，不重复分享，点击 [《Spring Cloud Gateway —— SetPath GatewayFilter Factory》](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/docs/src/main/asciidoc/spring-cloud-gateway.adoc#prefixpath-gatewayfilter-factory) 查看官方文档。

# 5. Status

本小节分享 Status 相关的 GatewayFilterFactory 实现类。

## 5.1 SetStatusGatewayFilterFactory

类似 RedirectToGatewayFilterFactory ，不重复分享，点击 [《Spring Cloud Gateway —— SetStatus GatewayFilter Factory》](https://github.com/spring-cloud/spring-cloud-gateway/blob/9ffb0f18678460fda9b25c572c12f9054a62ca52/docs/src/main/asciidoc/spring-cloud-gateway.adoc#setstatus-gatewayfilter-factory) 查看官方文档。

# 6. Redirect

本小节分享 Redirect 相关的 GatewayFilterFactory 实现类。

## 6.1 RedirectToGatewayFilterFactory

* 用途 ：将响应重定向到指定 URL ，并设置响应状态码为指定 Status 。**注意**，Status 必须为 3XX 重定向状态码。
* 配置 ：

    ```YAML
    spring:
      cloud:
        gateway:
          routes:
          # =====================================
          - id: prefixpath_route
            uri: http://example.org
            filters:
            - RedirectTo=302, http://www.iocoder.cn
    ```
    
* 代码 ：

```Java
  1: public class RedirectToGatewayFilterFactory implements GatewayFilterFactory {
  2: 
  3: 	public static final String STATUS_KEY = "status";
  4: 	public static final String URL_KEY = "url";
  5: 
  6: 	@Override
  7: 	public List<String> argNames() {
  8: 		return Arrays.asList(STATUS_KEY, URL_KEY);
  9: 	}
 10: 
 11: 	@Override
 12: 	public GatewayFilter apply(Tuple args) {
 13: 		String statusString = args.getRawString(STATUS_KEY);
 14: 		String urlString = args.getString(URL_KEY);
 15: 
 16: 		// 解析 status ，并判断是否是 3XX 重定向状态
 17: 		final HttpStatus httpStatus = parse(statusString);
 18: 		Assert.isTrue(httpStatus.is3xxRedirection(), "status must be a 3xx code, but was " + statusString);
 19: 		// 创建 URL
 20: 		final URL url;
 21: 		try {
 22: 			url = URI.create(urlString).toURL();
 23: 		} catch (MalformedURLException e) {
 24: 			throw new IllegalArgumentException("Invalid url " + urlString, e);
 25: 		}
 26: 
 27: 		return (exchange, chain) ->
 28: 			chain.filter(exchange).then(Mono.defer(() -> { // After Filter
 29: 				if (!exchange.getResponse().isCommitted()) {
 30: 				    // 设置响应 Status
 31: 					setResponseStatus(exchange, httpStatus);
 32: 
 33: 					// 设置响应 Header
 34: 					final ServerHttpResponse response = exchange.getResponse();
 35: 					response.getHeaders().set(HttpHeaders.LOCATION, url.toString());
 36: 					return response.setComplete();
 37: 				}
 38: 				return Mono.empty();
 39: 			}));
 40: 	}
 41: 
 42: }
```
* 第 16 至 18 行 ：解析配置的 `statusString` ，并判断是否是 3XX 重定向状态码。
* 第 19 至 25 行 ：解析配置的 `urlString` ，创建 URL 。
* 第 28 行 ：调用 `#then(Mono)` 方法，实现 **After Filter** 逻辑。这里和 AddRequestHeaderGatewayFilterFactory 实现的 **Before Filter** 【方式】**不同**。
* 第 29 至 37 行 ：**若响应未提交**，设置响应的状态码、响应的 Header ( `Location` ) 。
* 第 38 行 ：**设置响应已提交**。

# 7. Hystrix

熔断相关 GatewayFilter，我们在 [《TODO【3021】》]() 详细解析。

# 8. RateLimiter

限流相关 GatewayFilter，我们在 [《TODO【3002】》]() 详细解析。

# 666. 彩蛋


