title: 精尽 Dubbo 源码分析 —— 服务调用（四）之远程调用（Hessian）
date: 2018-10-13
tags:
categories: Dubbo
permalink: Dubbo/rpc-hessian

-------

摘要: 原创出处 http://www.iocoder.cn/Dubbo/rpc-hessian/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Dubbo/rpc-hessian/)
- [2. HttpClientConnection](http://www.iocoder.cn/Dubbo/rpc-hessian/)
  - [2.1 HttpClientConnectionFactory](http://www.iocoder.cn/Dubbo/rpc-hessian/)
- [3. HessianProtocol](http://www.iocoder.cn/Dubbo/rpc-hessian/)
  - [3.1 构造方法](http://www.iocoder.cn/Dubbo/rpc-hessian/)
  - [3.2 doExport](http://www.iocoder.cn/Dubbo/rpc-hessian/)
  - [3.3 doRefer](http://www.iocoder.cn/Dubbo/rpc-hessian/)
  - [3.4 destroy](http://www.iocoder.cn/Dubbo/rpc-hessian/)
- [666. 彩蛋](http://www.iocoder.cn/Dubbo/rpc-hessian/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

本文，我们分享 `hessian://` 协议的远程调用，主要分成**三个部分**：

* 服务暴露
* 服务引用
* 服务调用

对应项目为 `dubbo-rpc-hessian` 。

对应文档为 [《Dubbo 用户指南 —— hessian://》](https://dubbo.gitbooks.io/dubbo-user-book/references/protocol/hessian.html) 。定义如下：

> Hessian 协议用于集成 Hessian 的服务，Hessian 底层采用 Http 通讯，采用 Servlet 暴露服务，Dubbo 缺省内嵌 Jetty 作为服务器实现。

> Dubbo 的 Hessian 协议可以和原生 Hessian 服务互操作，即：
> 
> * 提供者用 Dubbo 的 Hessian 协议暴露服务，消费者直接用标准 Hessian 接口调用
> * 或者提供方用标准 Hessian 暴露服务，消费方用 Dubbo 的 Hessian 协议调用。

本文涉及类图（红圈部分）如下：

![类图](http://www.iocoder.cn/images/Dubbo/2018_10_13/01.png)

> 旁白君：整体实现和 `dubbo-rpc-http` 一致，所以内容上和 [《精尽 Dubbo 源码分析 —— 服务调用（三）之远程调用（HTTP）》](http://www.iocoder.cn/Dubbo/rpc-http/?self) 差不多。

# 2. HttpClientConnection

`com.alibaba.dubbo.rpc.protocol.hessian.HttpClientConnection` ，实现 HessianConnection 接口，HttpClient 连接器实现类。

```Java
public class HttpClientConnection implements HessianConnection {

    /**
     * Apache HttpClient
     */
    private final HttpClient httpClient;

    private final ByteArrayOutputStream output;

    private final HttpPost request;

    private volatile HttpResponse response;

    public HttpClientConnection(HttpClient httpClient, URL url) {
        this.httpClient = httpClient;
        this.output = new ByteArrayOutputStream();
        this.request = new HttpPost(url.toString());
    }

    @Override
    public void addHeader(String key, String value) {
        request.addHeader(new BasicHeader(key, value));
    }

    @Override
    public OutputStream getOutputStream() throws IOException {
        return output;
    }

    @Override
    public void sendRequest() throws IOException {
        request.setEntity(new ByteArrayEntity(output.toByteArray()));
        this.response = httpClient.execute(request);
    }

    @Override
    public int getStatusCode() {
        return response == null || response.getStatusLine() == null ? 0 : response.getStatusLine().getStatusCode();
    }

    @Override
    public String getStatusMessage() {
        return response == null || response.getStatusLine() == null ? null : response.getStatusLine().getReasonPhrase();
    }

    @Override
    public String getContentEncoding() {
        return (response == null || response.getEntity() == null || response.getEntity().getContentEncoding() == null) ? null : response.getEntity().getContentEncoding().getValue();
    }

    @Override
    public InputStream getInputStream() throws IOException {
        return response == null || response.getEntity() == null ? null : response.getEntity().getContent();
    }

    @Override
    public void close() throws IOException {
        HttpPost request = this.request;
        if (request != null) {
            request.abort();
        }
    }

    @Override
    public void destroy() throws IOException {
    }

}
```

* 基于 **Apache HttpClient** 封装。

## 2.1 HttpClientConnectionFactory

`com.alibaba.dubbo.rpc.protocol.hessian.HttpClientConnectionFactory` ，实现 HessianConnectionFactory 接口，创建 HttpClientConnection 的工厂。代码如下：

```Java
public class HttpClientConnectionFactory implements HessianConnectionFactory {

    /**
     * Apache HttpClient
     */
    private final HttpClient httpClient = new DefaultHttpClient();

    @Override
    public void setHessianProxyFactory(HessianProxyFactory factory) {
        HttpConnectionParams.setConnectionTimeout(httpClient.getParams(), (int) factory.getConnectTimeout());
        HttpConnectionParams.setSoTimeout(httpClient.getParams(), (int) factory.getReadTimeout());
    }

    @Override
    public HessianConnection open(URL url) {
        return new HttpClientConnection(httpClient, url); // HttpClientConnection
    }

}
```

# 3. HessianProtocol

[`com.alibaba.dubbo.rpc.protocol.hessian.HessianProtocol`](https://github.com/YunaiV/dubbo/blob/master/dubbo-rpc/dubbo-rpc-hessian/src/main/java/com/alibaba/dubbo/rpc/protocol/hessian/HessianProtocol.java) ，实现 AbstractProxyProtocol 抽象类，`hessian://` 协议实现类。

## 3.1 构造方法

```Java
/**
 * Http 服务器集合
 *
 * key：ip:port
 */
private final Map<String, HttpServer> serverMap = new ConcurrentHashMap<String, HttpServer>();
/**
 * Spring HttpInvokerServiceExporter 集合
 *
 * key：path 服务名
 */
private final Map<String, HessianSkeleton> skeletonMap = new ConcurrentHashMap<String, HessianSkeleton>();
/**
 * HttpBinder$Adaptive 对象
 */
private HttpBinder httpBinder;

public HessianProtocol() {
    super(HessianException.class);
}

public void setHttpBinder(HttpBinder httpBinder) {
    this.httpBinder = httpBinder;
}
```

* `serverMap` 属性，HttpServer 集合。键为 `ip:port` ，通过 `#getAddr(url)` 方法，计算。代码如下：

    ```Java
    // AbstractProxyProtocol.java
    protected String getAddr(URL url) {
        String bindIp = url.getParameter(Constants.BIND_IP_KEY, url.getHost());
        if (url.getParameter(Constants.ANYHOST_KEY, false)) {
            bindIp = Constants.ANYHOST_VALUE;
        }
        return NetUtils.getIpByHost(bindIp) + ":" + url.getParameter(Constants.BIND_PORT_KEY, url.getPort());
    }
    ```

* `skeletonMap` 属性，`com.caucho.hessian.server.HessianSkeleton` 集合。请求处理过程为 `HttpServer => DispatcherServlet => HessianHandler => HessianSkeleton` 。 
* `httpBinder` 属性，HttpBinder$Adaptive 对象，通过 `#setHttpBinder(httpBinder)` 方法，Dubbo SPI 调用设置。
* `rpcExceptions = HessianException.class` 。

## 3.2 doExport

```Java
  1: @Override
  2: protected <T> Runnable doExport(T impl, Class<T> type, URL url) throws RpcException {
  3:     // 获得服务器地址
  4:     String addr = getAddr(url);
  5:     // 获得 HttpServer 对象。若不存在，进行创建。
  6:     HttpServer server = serverMap.get(addr);
  7:     if (server == null) {
  8:         server = httpBinder.bind(url, new HessianHandler()); // HessianHandler
  9:         serverMap.put(addr, server);
 10:     }
 11:     // 添加到 skeletonMap 中
 12:     final String path = url.getAbsolutePath();
 13:     HessianSkeleton skeleton = new HessianSkeleton(impl, type);
 14:     skeletonMap.put(path, skeleton);
 15:     // 返回取消暴露的回调 Runnable
 16:     return new Runnable() {
 17:         public void run() {
 18:             skeletonMap.remove(path);
 19:         }
 20:     };
 21: }
```

* 基于 `dubbo-remoting-http` 项目，作为**通信服务器**。
* 第 4 行：调用 `#getAddr(url)` 方法，获得服务器地址。
* 第 5 至 10 行：从 `serverMap` 中，获得 HttpServer 对象。若不存在，调用 `HttpBinder#bind(url, handler)` 方法，创建 HttpServer 对象。此处使用的 HessianHandler ，下文详细解析。
* 第 11 至 14 行：创建 HessianSkeleton 对象，添加到 `skeletonMap` 集合中。
* 第 15 至 20 行：返回取消暴露的回调 Runnable 对象。

### 3.2.1 HessianHandler

```Java
private class HessianHandler implements HttpHandler {

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response)
            throws IOException, ServletException {
        String uri = request.getRequestURI();
        // 获得 HessianSkeleton 对象
        HessianSkeleton skeleton = skeletonMap.get(uri);
        // 必须是 POST 请求
        if (!request.getMethod().equalsIgnoreCase("POST")) {
            response.setStatus(500);
        // 执行调用
        } else {
            RpcContext.getContext().setRemoteAddress(request.getRemoteAddr(), request.getRemotePort());
            try {
                skeleton.invoke(request.getInputStream(), response.getOutputStream());
            } catch (Throwable e) {
                throw new ServletException(e);
            }
        }
    }

}
```

## 3.3 doRefer

```Java
  1: @Override
  2: @SuppressWarnings("unchecked")
  3: protected <T> T doRefer(Class<T> serviceType, URL url) throws RpcException {
  4:     // 创建 HessianProxyFactory 对象
  5:     HessianProxyFactory hessianProxyFactory = new HessianProxyFactory();
  6:     // 创建连接器工厂为 HttpClientConnectionFactory 对象，即 Apache HttpClient
  7:     String client = url.getParameter(Constants.CLIENT_KEY, Constants.DEFAULT_HTTP_CLIENT);
  8:     if ("httpclient".equals(client)) {
  9:         hessianProxyFactory.setConnectionFactory(new HttpClientConnectionFactory());
 10:     } else if (client != null && client.length() > 0 && !Constants.DEFAULT_HTTP_CLIENT.equals(client)) {
 11:         throw new IllegalStateException("Unsupported http protocol client=\"" + client + "\"!");
 12:     }
 13:     // 设置超时时间
 14:     int timeout = url.getParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
 15:     hessianProxyFactory.setConnectTimeout(timeout);
 16:     hessianProxyFactory.setReadTimeout(timeout);
 17:     // 创建 Service Proxy 对象
 18:     return (T) hessianProxyFactory.create(serviceType, url.setProtocol("http").toJavaURL(), Thread.currentThread().getContextClassLoader());
 19: }
```

* 基于 HttpClient ，作为**通信客户端**。
* 第 5 行：创建 `com.caucho.hessian.client.HessianProxyFactory` 对象。
* 第 6 至 12 行：创建**连接器工厂**为 `com.alibaba.dubbo.rpc.protocol.hessian.HttpClientConnectionFactory` 。
* 第 13 至 16 行：设置超时时间。
* 第 18 行：调用 `HessianProxyFactory#create(Class<?> api, URL url, ClassLoader loader)` 方法，生成 Service Proxy 对象。

### 3.3.1 getErrorCode

```Java
@Override
protected int getErrorCode(Throwable e) {
    if (e instanceof HessianConnectionException) {
        if (e.getCause() != null) {
            Class<?> cls = e.getCause().getClass();
            if (SocketTimeoutException.class.equals(cls)) {
                return RpcException.TIMEOUT_EXCEPTION;
            }
        }
        return RpcException.NETWORK_EXCEPTION;
    } else if (e instanceof HessianMethodSerializationException) {
        return RpcException.SERIALIZATION_EXCEPTION;
    }
    return super.getErrorCode(e);
}
```

* 将异常，翻译成 Dubbo 异常码。

## 3.4 destroy

```Java
@Override
public void destroy() {
    // 销毁
    super.destroy();
    // 销毁 HttpServer
    for (String key : new ArrayList<String>(serverMap.keySet())) {
        HttpServer server = serverMap.remove(key);
        if (server != null) {
            try {
                if (logger.isInfoEnabled()) {
                    logger.info("Close hessian server " + server.getUrl());
                }
                server.close();
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    }
}
```

* 这部分是 `dubbo-rpc-http` 所缺失的。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

写这篇文章的时候，脑子一直在转，如果每周的工作节奏是 9 7 5 ，那么每周，我们究竟会花在学习上，多少小时？？？

