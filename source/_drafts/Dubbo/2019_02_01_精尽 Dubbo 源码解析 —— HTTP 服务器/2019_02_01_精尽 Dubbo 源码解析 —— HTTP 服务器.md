title: 精尽 Dubbo 源码分析 —— HTTP 服务器
date: 2019-02-01
tags:
categories: Dubbo
permalink: Dubbo/remoting-http-api-and-impl

-------

摘要: 原创出处 http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
- [2. 原理](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
- [3. API](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
  - [3.1 HttpServer](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
  - [3.2 HttpHandler](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
  - [3.3 HttpBinder](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
  - [3.4 DispatcherServlet](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
  - [3.5 ServletManager](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
- [4. Tomcat 实现](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
  - [4.1 TomcatHttpServer](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
  - [4.2 TomcatHttpBinder](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
- [5. Jetty 实现](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
  - [5.1 JettyHttpServer](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
  - [5.2 JettyHttpBinder](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
- [6. Servlet Bridge 实现](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
  - [6.1 ServletHttpServer](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
  - [6.2 ServletHttpBinder](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
  - [6.3 BootstrapListener](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)
- [666. 彩蛋](http://www.iocoder.cn/Dubbo/remoting-http-api-and-impl/)

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

本文，我们来分享 Dubbo 的 HTTP 服务器，在 `dubbo-remoting-http` 模块中实现，使用在 [`http://`](https://dubbo.gitbooks.io/dubbo-user-book/references/protocol/http.html)、 [`rest://`](https://dubbo.gitbooks.io/dubbo-user-book/references/protocol/rest.html)、[`hessian://`](https://dubbo.gitbooks.io/dubbo-user-book/references/protocol/hessian.html)、
[`webservice://`](https://dubbo.gitbooks.io/dubbo-user-book/references/protocol/webservice.html)
 协议上。

`dubbo-remoting-http` **只提供 Server 部分**，不同于前面分享的 Dubbo 的 NIO 服务器( `dubbo-remoting-api` )，提供 Client 和 Server 。代码结构如下：![代码结构](http://www.iocoder.cn/images/Dubbo/2019_02_01/02.png)

* API 层：
    * 最外层：API 定义。
    * `support` 包： 公用实现。
* 实现层：
    * `jetty` 包：基于**内嵌的** Jetty 实现，版本为 `6.x` 。
    * `tomcat` 包：基于**内嵌的** Tomcat 实现，版本为 `8.x` 。
    * `servlet` 包：基于 Servlet Bridge Server 实现。简单的说，使用 `war` 包，部署在**外部的** Tomcat 、Jetty 等 Servlet 容器。

在 [《Dubbo 用户指南 —— http://》](https://dubbo.gitbooks.io/dubbo-user-book/references/protocol/http.html) 文档中，分享了具体的配置方式。这块的文档，写的比较简略，如果看不太明白的胖友，可以看看 [《Dubbo 组成原理 - http服务消费端如何调用》](https://blog.csdn.net/hdu09075340/article/details/71636972) 作为补充。  

另外，文档中推荐使用 Servlet Bridge Server 的部署方式，可能是文档写的比较早，现在主流是的 **Fat Jar** 的方式，所以实际使用时，`jetty` 或 `tomcat` 方式更为适合。  

当然，能够方便的通过配置的方式，切换具体的 HTTP 服务的拓展，依托于 Dubbo SPI 的机制。

# 2. 原理

`dubbo-remoting-http` 模块，**类图**如下：

![代码结构](http://www.iocoder.cn/images/Dubbo/2019_02_01/01.jpeg)

* HttpBinder ，负责创建对应的 HttpServer 对象。
* 不同的 Protocol ，实现各自的 HttpHandler 类。并且，暴露服务时，启动 HttpServer 的同时，创建对应的 HttpHandler 对象，以 **port** 为键，注册到 DispatcherServlet 上。
* DispatcherServlet ，**核心**，调度请求，到对应的 HttpHandler 中。

**整体流程**如下：![流程](http://www.iocoder.cn/images/Dubbo/2019_02_01/03.png)

# 3. API

## 3.1 HttpServer

[`com.alibaba.dubbo.remoting.http.HttpServer`](https://github.com/YunaiV/dubbo/blob/master/dubbo-remoting/dubbo-remoting-http/src/main/java/com/alibaba/dubbo/remoting/http/HttpServer.java) ，实现 Resetable 接口，HTTP **服务器**接口。方法如下：

```Java
// 处理器
HttpHandler getHttpHandler();

// 【属性相关】
URL getUrl();
InetSocketAddress getLocalAddress();

// 【状态相关】
boolean isBound();

void close();
void close(int timeout);
boolean isClosed();
```

### 3.1.1 AbstractHttpServer

 [`com.alibaba.dubbo.remoting.http.AbstractHttpServer`](https://github.com/YunaiV/dubbo/blob/master/dubbo-remoting/dubbo-remoting-http/src/main/java/com/alibaba/dubbo/remoting/http/AbstractHttpServer.java) ，实现 HttpServer 接口，HTTP 服务器抽象类。代码如下：
 
 ```Java
 public abstract class AbstractHttpServer implements HttpServer {

    /**
     * URL 对象
     */
    private final URL url;
    /**
     * 处理器
     */
    private final HttpHandler handler;
    /**
     * 是否关闭
     */
    private volatile boolean closed;

    public AbstractHttpServer(URL url, HttpHandler handler) {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        this.url = url;
        this.handler = handler;
    }

    @Override
    public HttpHandler getHttpHandler() {
        return handler;
    }

    @Override
    public URL getUrl() {
        return url;
    }

    @Override
    public void reset(URL url) {
    }

    @Override
    public boolean isBound() {
        return true;
    }

    @Override
    public InetSocketAddress getLocalAddress() {
        return url.toInetSocketAddress();
    }

    @Override
    public void close() {
        closed = true;
    }

    @Override
    public void close(int timeout) {
        close();
    }

    @Override
    public boolean isClosed() {
        return closed;
    }

}
 ```

## 3.2 HttpHandler

[`com.alibaba.dubbo.remoting.http.HttpHandler`](https://github.com/YunaiV/dubbo/blob/master/dubbo-remoting/dubbo-remoting-http/src/main/java/com/alibaba/dubbo/remoting/http/HttpHandler.java) ，HTTP **处理器**接口。方法如下：

```Java
/**
 * invoke.
 *
 * 处理器请求
 *
 * @param request  request. 请求
 * @param response response. 响应
 * @throws IOException 当 IO 发生异常
 * @throws ServletException 当 Servlet 发生异常
 */
void handle(HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException;
```

## 3.3 HttpBinder

[`com.alibaba.dubbo.remoting.http.HttpBinder`](https://github.com/YunaiV/dubbo/blob/master/dubbo-remoting/dubbo-remoting-http/src/main/java/com/alibaba/dubbo/remoting/http/HttpBinder.java) ，HTTP **绑定器**接口。方法如下：

```Java
@SPI("jetty")
public interface HttpBinder {

    /**
     * bind the server.
     *
     * @param url server url.
     * @return server.
     */
    @Adaptive({Constants.SERVER_KEY})
    HttpServer bind(URL url, HttpHandler handler);

}
```

* `@SPI("jetty")` 注解，Dubbo SPI **拓展点**，默认为 `"jetty"` ，即未配置情况下，使用 Jetty Server 。
* `@Adaptive({Constants.SERVER_KEY})` 注解，基于 Dubbo SPI Adaptive 机制，加载对应的 Server 实现，使用 `URL.server` 属性。

## 3.4 DispatcherServlet

[`com.alibaba.dubbo.remoting.http.serlvet.DispatcherServlet`](https://github.com/YunaiV/dubbo/blob/master/dubbo-remoting/dubbo-remoting-http/src/main/java/com/alibaba/dubbo/remoting/http/servlet/DispatcherServlet.java) ，实现 `javax.servlet.http.HttpServlet` 接口，服务请求**调度** Servlet。代码如下：

```Java
  1: public class DispatcherServlet extends HttpServlet {
  2: 
  3:     /**
  4:      * 处理器集合
  5:      *
  6:      * key：服务器端口
  7:      */
  8:     private static final Map<Integer, HttpHandler> handlers = new ConcurrentHashMap<Integer, HttpHandler>();
  9:     /**
 10:      * 单例
 11:      */
 12:     private static DispatcherServlet INSTANCE;
 13: 
 14:     public DispatcherServlet() {
 15:         DispatcherServlet.INSTANCE = this;
 16:     }
 17: 
 18:     public static DispatcherServlet getInstance() {
 19:         return INSTANCE;
 20:     }
 21: 
 22:     /**
 23:      * 添加处理器
 24:      *
 25:      * @param port 服务器端口
 26:      * @param processor 处理器
 27:      */
 28:     public static void addHttpHandler(int port, HttpHandler processor) {
 29:         handlers.put(port, processor);
 30:     }
 31: 
 32:     /**
 33:      * 移除处理器
 34:      *
 35:      * @param port 服务器端口
 36:      */
 37:     public static void removeHttpHandler(int port) {
 38:         handlers.remove(port);
 39:     }
 40: 
 41:     @Override
 42:     protected void service(HttpServletRequest request, HttpServletResponse response)
 43:             throws ServletException, IOException {
 44:         // 获得处理器
 45:         HttpHandler handler = handlers.get(request.getLocalPort());
 46:         // 处理器不存在，报错
 47:         if (handler == null) {// service not found.
 48:             response.sendError(HttpServletResponse.SC_NOT_FOUND, "Service not found.");
 49:         // 处理请求
 50:         } else {
 51:             handler.handle(request, response);
 52:         }
 53:     }
 54: 
 55: }
```

* `handlers` **静态**属性，处理器集合。
    * `#addHttpHandler(port, HttpHandler)`  方法，注册处理器。
    * `#removeHttpHandler(port)` 方法，取消处理器。
* `#service(request, response)` **实现**方法，调度请求。
    * 第 45 行：基于端口，获得处理器。
    * 第 46 至 48 行：处理器不存在，返回 500 报错。
    * 第 49 至 52 行：调用 `HttpHandler#handle(request, response)` 方法，处理请求，从而调度到 Service 的对应的方法。

## 3.5 ServletManager

[`com.alibaba.dubbo.remoting.http.serlvet.ServletManager`](https://github.com/YunaiV/dubbo/blob/master/dubbo-remoting/dubbo-remoting-http/src/main/java/com/alibaba/dubbo/remoting/http/servlet/ServletManager.java) ，Servlet 管理器，负责管理 ServletContext ，目前仅有 `dubbo-rpc-rest` 模块，需要使用到这个类。代码如下：

```Java
public class ServletManager {

    /**
     * 外部服务器端口，用于 `servlet` 的服务器端口
     */
    public static final int EXTERNAL_SERVER_PORT = -1234;

    /**
     * 单例
     */
    private static final ServletManager instance = new ServletManager();
    /**
     * ServletContext 集合
     */
    private final Map<Integer, ServletContext> contextMap = new ConcurrentHashMap<Integer, ServletContext>();

    public static ServletManager getInstance() {
        return instance;
    }

    /**
     * 添加 ServletContext 对象
     *
     * @param port 服务器端口
     * @param servletContext ServletContext 对象
     */
    public void addServletContext(int port, ServletContext servletContext) {
        contextMap.put(port, servletContext);
    }

    /**
     * 移除 ServletContext 对象
     *
     * @param port 服务器端口
     */
    public void removeServletContext(int port) {
        contextMap.remove(port);
    }

    /**
     * 获得 ServletContext 对象
     *
     * @param port 服务器端口
     * @return ServletContext 对象
     */
    public ServletContext getServletContext(int port) {
        return contextMap.get(port);
    }

}
```

* `EXTERNAL_SERVER_PORT` **静态**属性，**外部**服务器端口，用于 `servlet` 的服务器端口。
* `contextMap` **静态**属性，ServletContext 集合。
    * `#addServletContext(port, ServletContext)`  方法，添加。
    * `#removeServletContext(port)` 方法，移除。
    * `#getServletContext(port)` 方法，查询。

# 4. Tomcat 实现

## 4.1 TomcatHttpServer

[`com.alibaba.dubbo.remoting.http.tomcat.TomcatHttpServer`](https://github.com/YunaiV/dubbo/blob/master/dubbo-remoting/dubbo-remoting-http/src/main/java/com/alibaba/dubbo/remoting/http/tomcat/TomcatHttpServer.java) ，实现 AbstractHttpServer 抽象类，基于 Tomcat 的 HTTP 服务器实现类。

### 4.1.1 构造方法

```Java
  1: /**
  2:  * 内嵌的 Tomcat 对象
  3:  */
  4: private final Tomcat tomcat;
  5: /**
  6:  * URL 对象
  7:  */
  8: private final URL url;
  9: 
 10: public TomcatHttpServer(URL url, final HttpHandler handler) {
 11:     super(url, handler);
 12:     this.url = url;
 13: 
 14:     // 注册 HttpHandler 到 DispatcherServlet 中
 15:     DispatcherServlet.addHttpHandler(url.getPort(), handler);
 16: 
 17:     // 创建内嵌的 Tomcat 对象
 18:     String baseDir = new File(System.getProperty("java.io.tmpdir")).getAbsolutePath();
 19:     tomcat = new Tomcat();
 20:     tomcat.setBaseDir(baseDir);
 21:     tomcat.setPort(url.getPort());
 22:     tomcat.getConnector().setProperty("maxThreads", String.valueOf(url.getParameter(Constants.THREADS_KEY, Constants.DEFAULT_THREADS))); // 最大线程数
 23: //    tomcat.getConnector().setProperty(
 24: //            "minSpareThreads", String.valueOf(url.getParameter(Constants.THREADS_KEY, Constants.DEFAULT_THREADS)));
 25:     tomcat.getConnector().setProperty("maxConnections", String.valueOf(url.getParameter(Constants.ACCEPTS_KEY, -1))); // 最大连接池
 26:     tomcat.getConnector().setProperty("URIEncoding", "UTF-8"); // 编码为 UTF-8
 27:     tomcat.getConnector().setProperty("connectionTimeout", "60000"); // 连接超时，60 秒
 28:     tomcat.getConnector().setProperty("maxKeepAliveRequests", "-1");
 29:     tomcat.getConnector().setProtocol("org.apache.coyote.http11.Http11NioProtocol");
 30: 
 31:     // 添加 DispatcherServlet 到 Tomcat 中
 32:     Context context = tomcat.addContext("/", baseDir);
 33:     Tomcat.addServlet(context, "dispatcher", new DispatcherServlet());
 34:     context.addServletMapping("/*", "dispatcher");
 35: 
 36:     // 添加 ServletContext 对象，到 ServletManager 中
 37:     ServletManager.getInstance().addServletContext(url.getPort(), context.getServletContext());
 38: 
 39:     // 启动 Tomcat
 40:     try {
 41:         tomcat.start();
 42:     } catch (LifecycleException e) {
 43:         throw new IllegalStateException("Failed to start tomcat server at " + url.getAddress(), e);
 44:     }
 45: }
```

* 第 15 行：调用 `DispatcherServlet#addHttpHandler(port, handler)` 方法，注册 HttpHandler 到 DispatcherServlet 中。
* 第 17 至 29 行：**创建**内嵌的 Tomcat 对象。
* 第 31 至 34 行：**创建**并添加 DispatcherServlet 对象，到 Tomcat 中。
* 第 37 行：调用 `ServletManager#addServletContext(port, ServletContext)` 方法，添加 DispatcherServlet 对象，到 ServletManager 中。
* 第 39 至 44 行：调用 `Tomcat#start()` 方法，**启动** Tomcat 。

### 4.1.2 关闭

```Java
@Override
public void close() {
    // 标记关闭
    super.close();

    // 移除 ServletContext 对象
    ServletManager.getInstance().removeServletContext(url.getPort());

    // 关闭 Tomcat
    try {
        tomcat.stop();
    } catch (Exception e) {
        logger.warn(e.getMessage(), e);
    }
}
```

* **缺少**，调用 `DispacherServlet#remove(port)` 方法，将 HttpHandler 对象，移除出 DispatcherServlet 。

## 4.2 TomcatHttpBinder

[`com.alibaba.dubbo.remoting.http.tomcat.TomcatHttpBinder`](https://github.com/YunaiV/dubbo/blob/master/dubbo-remoting/dubbo-remoting-http/src/main/java/com/alibaba/dubbo/remoting/http/tomcat/TomcatHttpBinder.java) ，TomcatHttpServer 绑定器实现类。代码如下：

```Java
public class TomcatHttpBinder implements HttpBinder {

    @Override
    public HttpServer bind(URL url, HttpHandler handler) {
        return new TomcatHttpServer(url, handler);
    }

}
```

# 5. Jetty 实现

> `jetty` 和 `tomcat` 包的实现，差不多，主要差异在 Tomcat 和 Jetty 的 API 不同。
> 
> 所以，下面我们就贴贴代码啦，当然，还是有中文详细注释的。

## 5.1 JettyHttpServer

```Java
public class JettyHttpServer extends AbstractHttpServer {

    private static final Logger logger = LoggerFactory.getLogger(JettyHttpServer.class);

    /**
     * 内嵌的 Jetty 服务器
     */
    private Server server;
    /**
     * URL 对象
     */
    private URL url;

    public JettyHttpServer(URL url, final HttpHandler handler) {
        super(url, handler);
        this.url = url;

        // 设置日志的配置
        // TODO we should leave this setting to slf4j
        // we must disable the debug logging for production use
        Log.setLog(new StdErrLog());
        Log.getLog().setDebugEnabled(false);

        // 注册 HttpHandler 到 DispatcherServlet 中
        DispatcherServlet.addHttpHandler(url.getParameter(Constants.BIND_PORT_KEY, url.getPort()), handler);

        // 创建线程池
        int threads = url.getParameter(Constants.THREADS_KEY, Constants.DEFAULT_THREADS);
        QueuedThreadPool threadPool = new QueuedThreadPool();
        threadPool.setDaemon(true);
        threadPool.setMaxThreads(threads);
        threadPool.setMinThreads(threads);

        // 创建 Jetty Connector 对象
        SelectChannelConnector connector = new SelectChannelConnector();
        String bindIp = url.getParameter(Constants.BIND_IP_KEY, url.getHost());
        if (!url.isAnyHost() && NetUtils.isValidLocalHost(bindIp)) {
            connector.setHost(bindIp);
        }
        connector.setPort(url.getParameter(Constants.BIND_PORT_KEY, url.getPort()));

        // 创建内嵌的 Jetty 对象
        server = new Server();
        server.setThreadPool(threadPool);
        server.addConnector(connector);

        // 添加 DispatcherServlet 到 Jetty 中
        ServletHandler servletHandler = new ServletHandler();
        ServletHolder servletHolder = servletHandler.addServletWithMapping(DispatcherServlet.class, "/*");
        servletHolder.setInitOrder(2);

        // 添加 ServletContext 对象，到 ServletManager 中
        // dubbo's original impl can't support the use of ServletContext
//        server.addHandler(servletHandler);
        // TODO Context.SESSIONS is the best option here?
        Context context = new Context(server, "/", Context.SESSIONS);
        context.setServletHandler(servletHandler);
        ServletManager.getInstance().addServletContext(url.getParameter(Constants.BIND_PORT_KEY, url.getPort()), context.getServletContext());

        // 启动 Jetty
        try {
            server.start();
        } catch (Exception e) {
            throw new IllegalStateException("Failed to start jetty server on " + url.getParameter(Constants.BIND_IP_KEY) + ":" + url.getParameter(Constants.BIND_PORT_KEY) + ", cause: "
                    + e.getMessage(), e);
        }
    }

    @Override
    public void close() {
        // 标记关闭
        super.close();

        // 移除 ServletContext 对象
        ServletManager.getInstance().removeServletContext(url.getParameter(Constants.BIND_PORT_KEY, url.getPort()));

        // 关闭 Jetty
        if (server != null) {
            try {
                server.stop();
            } catch (Exception e) {
                logger.warn(e.getMessage(), e);
            }
        }
    }

}
```

## 5.2 JettyHttpBinder

```Java
public class JettyHttpBinder implements HttpBinder {

    @Override
    public HttpServer bind(URL url, HttpHandler handler) {
        return new JettyHttpServer(url, handler);
    }

}
```

# 6. Servlet Bridge 实现

## 6.1 ServletHttpServer

[`com.alibaba.dubbo.remoting.http.servlet.ServletHttpServer`](https://github.com/YunaiV/dubbo/blob/master/dubbo-remoting/dubbo-remoting-http/src/main/java/com/alibaba/dubbo/remoting/http/servlet/ServletHttpServer.java) ，实现 AbstractHttpServer 抽象类， 基于 Servlet 的服务器实现类。代码如下：

```Java
public class ServletHttpServer extends AbstractHttpServer {

    public ServletHttpServer(URL url, HttpHandler handler) {
        super(url, handler);

        // 注册 HttpHandler 到 DispatcherServlet 中
        DispatcherServlet.addHttpHandler(url.getParameter(Constants.BIND_PORT_KEY, 8080), handler);
    }

}
```

* **注意**，在 `<dubbo:protocol />` 配置的**端口**，和外部的 Servlet 容器的**端口**，**保持一致**。
* 需要配置 DispatcherServlet 到 `web.xml` 中。通过这样的方式，让外部的 Servlet 容器，可以进行转发。

## 6.2 ServletHttpBinder

```Java
public class ServletHttpBinder implements HttpBinder {

    @Adaptive()
    public HttpServer bind(URL url, HttpHandler handler) {
        return new ServletHttpServer(url, handler);
    }

}
```

## 6.3 BootstrapListener

[`com.alibaba.dubbo.remoting.http.servlet.BootstrapListener`](https://github.com/YunaiV/dubbo/blob/master/dubbo-remoting/dubbo-remoting-http/src/main/java/com/alibaba/dubbo/remoting/http/servlet/BootstrapListener.java) ，实现 ServletContextListener 接口， 启动监听器。代码如下：

```Java
public class BootstrapListener implements ServletContextListener {

    @Override
    public void contextInitialized(ServletContextEvent servletContextEvent) {
        ServletManager.getInstance().addServletContext(ServletManager.EXTERNAL_SERVER_PORT, servletContextEvent.getServletContext());
    }

    @Override
    public void contextDestroyed(ServletContextEvent servletContextEvent) {
        ServletManager.getInstance().removeServletContext(ServletManager.EXTERNAL_SERVER_PORT);
    }

}
```

* 需要配置 BootstrapListener 到 `web.xml` 中。通过这样的方式，让外部的 ServletContext 对象，添加到 ServletManager 中。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

清明节，扫代码第四波。

又开阔了下思路，美滋滋。另外，艿艿配置了 `http://` 协议的例子，使用 Tomcat 内嵌服务器。地址如下：

* [`dubbo-http-demo-provider`](https://github.com/YunaiV/dubbo/tree/408eeb2af44f11dcd466a976add1db258a111ef0/dubbo-demo/dubbo-http-demo-provider)
* [`dubbo-http-demo-consumer`](https://github.com/YunaiV/dubbo/tree/408eeb2af44f11dcd466a976add1db258a111ef0/dubbo-demo/dubbo-http-demo-consumer)


