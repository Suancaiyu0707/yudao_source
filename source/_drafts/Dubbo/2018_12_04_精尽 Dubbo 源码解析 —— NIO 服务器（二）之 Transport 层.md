# 1. 概述

本文接 [《精尽 Dubbo 源码分析 —— NIO 服务器（一）之抽象 API》](http://www.iocoder.cn/Dubbo/remoting-api-interface/?self) 一文，分享 `dubbo-remoting-api` 模块， `transport` 包，**网络传输层**。

>  **transport** 网络传输层：抽象 mina 和 netty 为统一接口，以 Message 为中心，扩展接口为 Channel, Transporter, Client, Server, Codec  

涉及的类图如下：

[类图](http://www.iocoder.cn/images/Dubbo/2018_12_04/01.png)

* 白色部分，为通用接口。
* 蓝色部分，为 `transport` 包下的类。
* 整个类图，我们分成**六个**部分：
    * Client
    * Server
    * Channel
    * ChannelHandler
    * Codec
    * Dispacher
* 从流程上来说，我们分成：
    * Server
        * 启动
        * 关闭
    * Client
        * 启动
        * 关闭 
    * ChannelHandler 
        * 处理连接
        * 处理断开
        * 发送消息
        * 接收消息
        * 处理异常

> 艿艿的旁白：涉及较多类和流程，内容不是很线性，可能分享的比较凌乱，还望胖友谅解。建议，读 2-3 遍，并且做一些调试。

# 2. AbstractPeer

[`com.alibaba.dubbo.remoting.transport.AbstractPeer`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractPeer.java) ，实现 Endpoint、ChannelHandler 接口，**Peer** 抽象类。

**构造方法**

```Java
  1: /**
  2:  * 通道处理器
  3:  */
  4: private final ChannelHandler handler;
  5: /**
  6:  * URL
  7:  */
  8: private volatile URL url;
  9: /**
 10:  * 正在关闭
 11:  *
 12:  * {@link #startClose()}
 13:  */
 14: // closing closed means the process is being closed and close is finished
 15: private volatile boolean closing;
 16: /**
 17:  * 关闭完成
 18:  *
 19:  * {@link #close()}
 20:  */
 21: private volatile boolean closed;
 22: 
 23: public AbstractPeer(URL url, ChannelHandler handler) {
 24:     if (url == null) {
 25:         throw new IllegalArgumentException("url == null");
 26:     }
 27:     if (handler == null) {
 28:         throw new IllegalArgumentException("handler == null");
 29:     }
 30:     this.url = url;
 31:     this.handler = handler;
 32: }
```

* `handler` 属性，通道处理器，通过构造方法传入。实现的 ChannelHandler 的接口方法，直接调用 `handler` 的方法，进行执行逻辑处理。
    * 参见代码：[传送门](https://github.com/YunaiV/dubbo/blob/7fad710c2dbf66356d5e7b7995e843b8f6225652/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractPeer.java#L114-L141)
    *  这种方式在设计模式中被称作 "[装饰模式](https://www.cnblogs.com/java-my-life/archive/2012/04/20/2455726.html)" 。在下文中，我们会看到**大量的**装饰模式的使用。实际上，这也是 `dubbo-remoting` 抽象 API + 实现最核心的方式之一。
*  `url` 属性，URL ，通过构造方法传入。通过该属性，传递 Dubbo 服务引用和服务暴露的**配置项**。
*  `closing` 属性，正在关闭，调用 `#startClose()` 方法，变更。
*  `close` 属性，关闭完成，调用 `#close()` 方法，变更。

**发送消息**

```Java
@Override
public void send(Object message) throws RemotingException {
    send(message, url.getParameter(Constants.SENT_KEY, false));
}
```

* TODO 芋艿，sent ？？？

**其他方法**

胖友点击 [AbstractPeer](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractPeer.java) ，再看看**所有**的方法。

## 2.1 AbstractEndpint

[`com.alibaba.dubbo.remoting.transport.AbstractPeer.AbstractEndpint`](TODO) ，实现 Resetable 接口，继承 AbstractPeer 抽象类，**端点**抽象类。

**构造方法**

```Java
  1: /**
  2:  * 编解码器
  3:  */
  4: private Codec2 codec;
  5: /**
  6:  * 超时时间
  7:  */
  8: private int timeout;
  9: /**
 10:  * 连接超时时间
 11:  */
 12: private int connectTimeout;
 13: 
 14: public AbstractEndpoint(URL url, ChannelHandler handler) {
 15:     super(url, handler);
 16:     this.codec = getChannelCodec(url);
 17:     this.timeout = url.getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
 18:     this.connectTimeout = url.getPositiveParameter(Constants.CONNECT_TIMEOUT_KEY, Constants.DEFAULT_CONNECT_TIMEOUT);
 19: }
```

* `codec` 属性，编解码器。在构造方法中，可以看到调用 `#getChannelCodec(url)` 方法，基于 `url` 参数，加载对应的 Codec 实现对象。代码如下：

    ```Java
      1: protected static Codec2 getChannelCodec(URL url) {
      2:     String codecName = url.getParameter(Constants.CODEC_KEY, "telnet");
      3:     if (ExtensionLoader.getExtensionLoader(Codec2.class).hasExtension(codecName)) { // 例如，在 DubboProtocol 中，会获得 DubboCodec
      4:         return ExtensionLoader.getExtensionLoader(Codec2.class).getExtension(codecName);
      5:     } else {
      6:         return new CodecAdapter(ExtensionLoader.getExtensionLoader(Codec.class).getExtension(codecName));
      7:     }
      8: }
    ```
    * 第 3 行：基于 Dubbo SPI 机制，加载对应的 Codec 实现对象。例如，在 DubboProtocol 中，会获得 [DubboCodec](https://github.com/apache/incubator-dubbo/blob/bb8884e04433677d6abc6f05c6ad9d39e3dcf236/dubbo-rpc/dubbo-rpc-dubbo/src/main/java/com/alibaba/dubbo/rpc/protocol/dubbo/DubboCodec.java) 对象。
    * 第 6 行：Codec 接口，已经废弃了，目前 Dubbo 项目里，也没有它的拓展实现。

**重置属性**

[`#reset(url)`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractEndpoint.java#L60-L92) **实现**方法，使用新的 `url` 属性，可重置 `codec` `timeout` `connectTimeout` 属性。🙂 已经添加了谅解，胖友点击可看。

# 3. Client

## 3.1 AbstractClient

[`com.alibaba.dubbo.remoting.transport.AbstractClient`](TODO) ，实现 Client 接口，继承 AbstractEndpoint 抽象类，**客户端**抽象类，**重点**实现了公用的重连逻辑，同时抽象了连接等模板方法，供子类实现。抽象方法如下：

```Java
protected abstract void doOpen() throws Throwable;
protected abstract void doClose() throws Throwable;

protected abstract void doConnect() throws Throwable;
protected abstract void doDisConnect() throws Throwable;

protected abstract Channel getChannel();
```

**构造方法**

```Java
  1: /**
  2:  * 重连定时任务执行器
  3:  */
  4: private static final ScheduledThreadPoolExecutor reconnectExecutorService = new ScheduledThreadPoolExecutor(2, new NamedThreadFactory("DubboClientReconnectTimer", true));
  5: /**
  6:  * 发送消息时，若断开，是否重连
  7:  */
  8: private final boolean send_reconnect;
  9: /**
 10:  * 重连 warning 的间隔.(waring多少次之后，warning一次) //for test
 11:  */
 12: // reconnect warning period. Reconnect warning interval (log warning after how many times) //for test
 13: private final int reconnect_warning_period;
 14: /**
 15:  * 关闭超时时间
 16:  */
 17: private final long shutdown_timeout;
 18: /**
 19:  * 线程池
 20:  *
 21:  * 在调用 {@link #wrapChannelHandler(URL, ChannelHandler)} 时，会调用 {@link com.alibaba.dubbo.remoting.transport.dispatcher.WrappedChannelHandler} 创建
 22:  */
 23: protected volatile ExecutorService executor;
 24: 
 25: public AbstractClient(URL url, ChannelHandler handler) throws RemotingException {
 26:     super(url, handler);
 27:     // 从 URL 中，获得重连相关配置项
 28:     send_reconnect = url.getParameter(Constants.SEND_RECONNECT_KEY, false);
 29:     shutdown_timeout = url.getParameter(Constants.SHUTDOWN_TIMEOUT_KEY, Constants.DEFAULT_SHUTDOWN_TIMEOUT);
 30:     // The default reconnection interval is 2s, 1800 means warning interval is 1 hour.
 31:     reconnect_warning_period = url.getParameter("reconnect.waring.period", 1800);
 32: 
 33:     // 初始化客户端
 34:     try {
 35:         doOpen();
 36:     } catch (Throwable t) {
 37:         close(); // 失败，则关闭
 38:         throw new RemotingException(url.toInetSocketAddress(), null,
 39:                 "Failed to start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress()
 40:                         + " connect to the server " + getRemoteAddress() + ", cause: " + t.getMessage(), t);
 41:     }
 42: 
 43:     // 连接服务器
 44:     try {
 45:         // connect.
 46:         connect();
 47:         if (logger.isInfoEnabled()) {
 48:             logger.info("Start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress() + " connect to the server " + getRemoteAddress());
 49:         }
 50:     } catch (RemotingException t) {
 51:         if (url.getParameter(Constants.CHECK_KEY, true)) {
 52:             close(); // 失败，则关闭
 53:             throw t;
 54:         } else {
 55:             logger.warn("Failed to start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress()
 56:                     + " connect to the server " + getRemoteAddress() + " (check == false, ignore and retry later!), cause: " + t.getMessage(), t);
 57:         }
 58:     } catch (Throwable t) {
 59:         close(); // 失败，则关闭
 60:         throw new RemotingException(url.toInetSocketAddress(), null,
 61:                 "Failed to start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress()
 62:                         + " connect to the server " + getRemoteAddress() + ", cause: " + t.getMessage(), t);
 63:     }
 64: 
 65:     // 获得线程池
 66:     executor = (ExecutorService) ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension()
 67:             .get(Constants.CONSUMER_SIDE, Integer.toString(url.getPort()));
 68:     ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension()
 69:             .remove(Constants.CONSUMER_SIDE, Integer.toString(url.getPort()));
 70: }
```

* `reconnectExecutorService` 属性，重连定时任务执行器。在客户端连接服务端时，会创建后台任务，定时检查连接，若断开，会进行重连。
* 第 27 至 31 行：从 URL 中，获得重连相关**配置项**。
* 第 33 至 41 行：调用 `#doOpen()` **抽象**方法，初始化客户端。若异常，调用 `#close()` 方法，进行关闭。
* 第 43 至 63 行：调用 `#connect()` **实现**方法，连接服务器。若异常，调用 `#close()` 方法，进行关闭。
    * 第 51 至 57 行：若是连接失败 RemotingException ，若开启了 [启动时检查](https://dubbo.gitbooks.io/dubbo-user-book/demos/preflight-check.html) ，则调用 `#close()` 方法，进行关闭。
* 第 66 至 69 行：从 [DataStore](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-common/src/main/java/com/alibaba/dubbo/common/store/DataStore.java) 中，获得线程池。
    * DataStore 在 `dubbo-common` 模块，[`store`](https://github.com/YunaiV/dubbo/tree/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-common/src/main/java/com/alibaba/dubbo/common/store) 包下实现。目前的实现比较简单，可以认为是 `ConcurrentMap<String, ConcurrentMap<String, Object>>`  的集合。胖友可以自己看相关实现。
    * 此处的线程池，实际就是 [《Dubbo 用户指南 —— 线程模型》](https://dubbo.gitbooks.io/dubbo-user-book/demos/thread-model.html) 中说的**线程池**。在 [「8. Dispacher」](#) 中，详细解析。

**连接服务器**

```Java
/**
 * 连接锁，用于实现发起连接和断开连接互斥，避免并发。
 */
private final Lock connectLock = new ReentrantLock();

  1: protected void connect() throws RemotingException {
  2:     // 获得锁
  3:     connectLock.lock();
  4:     try {
  5:         // 已连接，
  6:         if (isConnected()) {
  7:             return;
  8:         }
  9:         // 初始化重连线程
 10:         initConnectStatusCheckCommand();
 11:         // 执行连接
 12:         doConnect();
 13:         // 连接失败，抛出异常
 14:         if (!isConnected()) {
 15:             throw new RemotingException(this, "Failed connect to server " + getRemoteAddress() + " from " + getClass().getSimpleName() + " "
 16:                     + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion()
 17:                     + ", cause: Connect wait timeout: " + getTimeout() + "ms.");
 18:         // 连接成功，打印日志
 19:         } else {
 20:             if (logger.isInfoEnabled()) {
 21:                 logger.info("Successed connect to server " + getRemoteAddress() + " from " + getClass().getSimpleName() + " "
 22:                         + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion()
 23:                         + ", channel is " + this.getChannel());
 24:             }
 25:         }
 26:         // 设置重连次数归零
 27:         reconnect_count.set(0);
 28:         // 设置未打印过错误日志
 29:         reconnect_error_log_flag.set(false);
 30:     } catch (RemotingException e) {
 31:         throw e;
 32:     } catch (Throwable e) {
 33:         throw new RemotingException(this, "Failed connect to server " + getRemoteAddress() + " from " + getClass().getSimpleName() + " "
 34:                 + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion()
 35:                 + ", cause: " + e.getMessage(), e);
 36:     } finally {
 37:         // 释放锁
 38:         connectLock.unlock();
 39:     }
 40: }
```

* 第 3 行：获得锁。在连接和断开连接时，通过锁，避免并发冲突。
* 第 5 至 8 行：调用 `#isConnected()` 方法，判断连接状态。若已经连接，就不重复连接。代码如下：

    ```Java
    @Override
    public boolean isConnected() {
        Channel channel = getChannel();
        return channel != null && channel.isConnected();
    }
    ```
    * 该方法，是因为实现 Channel 接口( Client 实现 Channel 接口 )，所以需要实现的。我们可以看到，实际方法内部，调用的是 `channel` 对象，进行判断。其它实现 Channel 的方法，也是这么处理的，例如 [`#getAttribute(key)`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractClient.java#L194-L245) 等方法。

* 第 10 行：调用 `#initConnectStatusCheckCommand()` 方法，初始化重连**线程**。
    * 🙂 方法会复杂一些，不杂糅在这里讲。 
* 第 14 至 17 行：连接失败，抛出异常 RemotingException 。
* 第 18 至 25 行：连接成功，打印日志。
* 第 26 至 29 行：设置重连次数归零，打印过错误日志状态为否。下面，我们会看到这些状态字段的变更。
* 第 38 行：释放锁。

**初始化重连线程**

```Java
/**
 * 重连次数
 */
private final AtomicInteger reconnect_count = new AtomicInteger(0);
/**
 * 重连时，是否已经打印过错误日志。
 */
// Reconnection error log has been called before?
private final AtomicBoolean reconnect_error_log_flag = new AtomicBoolean(false);
/**
 * 重连执行任务 Future
 */
private volatile ScheduledFuture<?> reconnectExecutorFuture = null;
/**
 * 最后成功连接时间
 */
// the last successed connected time
private long lastConnectedTime = System.currentTimeMillis();
    
  1: private synchronized void initConnectStatusCheckCommand() {
  2:     //reconnect=false to close reconnect
  3:     // 获得获得重连频率，默认开启。
  4:     int reconnect = getReconnectParam(getUrl());
  5:     // 若开启重连功能，创建重连线程
  6:     if (reconnect > 0 && (reconnectExecutorFuture == null || reconnectExecutorFuture.isCancelled())) {
  7:         // 创建 Runnable 对象
  8:         Runnable connectStatusCheckCommand = new Runnable() {
  9:             public void run() {
 10:                 try {
 11:                     // 未连接，重连
 12:                     if (!isConnected()) {
 13:                         connect();
 14:                     // 已连接，记录最后连接时间
 15:                     } else {
 16:                         lastConnectedTime = System.currentTimeMillis();
 17:                     }
 18:                 } catch (Throwable t) {
 19:                     // 超过一定时间未连接上，才打印异常日志。并且，仅打印一次。默认，15 分钟。
 20:                     String errorMsg = "client reconnect to " + getUrl().getAddress() + " find error . url: " + getUrl();
 21:                     // wait registry sync provider list
 22:                     if (System.currentTimeMillis() - lastConnectedTime > shutdown_timeout) {
 23:                         if (!reconnect_error_log_flag.get()) {
 24:                             reconnect_error_log_flag.set(true);
 25:                             logger.error(errorMsg, t);
 26:                             return;
 27:                         }
 28:                     }
 29:                     // 每一定次发现未重连，才打印告警日志。默认，1800 次，1 小时。
 30:                     if (reconnect_count.getAndIncrement() % reconnect_warning_period == 0) {
 31:                         logger.warn(errorMsg, t);
 32:                     }
 33:                 }
 34:             }
 35:         };
 36:         // 发起定时任务
 37:         reconnectExecutorFuture = reconnectExecutorService.scheduleWithFixedDelay(connectStatusCheckCommand, reconnect, reconnect, TimeUnit.MILLISECONDS);
 38:     }
 39: }
```

* 第 4 行：调用 [`#getReconnectParam(url)`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractClient.java#L120-L142) 方法，获得重连频率。默认开启，2000 毫秒。
    * 🙂 代码比较简单，胖友自己点击方法查看。
* 第 6 至 38 行：若**开启**重连功能， 创建重连线程。
    * 第 8 至 35 行：创建 Runnable 对象。
        * 第 11 至 13 行：未连接时，调用 `#connect()` 方法，进行重连。
        * 第 14 至 17 行：已连接时，记录最后连接时间。
        * 第 18 至 33 行：**符合**条件时，打印**错误**或**告警**日志。为什么要符合条件才打印呢？之前也和朋友聊起来过，线上因为中间件组件，打印了太多的日志，结果整个 JVM 崩了。特别在网络场景 + 大量“无限”重试的场景，特别容易打出满屏的日志。这块，我们可以学习下。另外，Eureka 在集群同步，也有类似处理。
    * 第 36 行：发起任务，**定时**检查，是否需要重连。
* `reconnect=false to close reconnect` ，从目前代码上来看，未实现 `#reset(url)` 方法，在 URL 的 `reconnect=false` 配置项时，关闭重连线程。

**发送消息**

```Java
@Override
public void send(Object message, boolean sent) throws RemotingException {
    // 未连接时，开启重连功能，则先发起连接
    if (send_reconnect && !isConnected()) {
        connect();
    }
    // 发送消息
    Channel channel = getChannel();
    //TODO Can the value returned by getChannel() be null? need improvement.
    if (channel == null || !channel.isConnected()) {
        throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
    }
    channel.send(message, sent);
}
```

**包装通道处理器**

```Java
// Constants.java
public static final String DEFAULT_CLIENT_THREADPOOL = "cached";

protected static final String CLIENT_THREAD_POOL_NAME = "DubboClientHandler";

  1: /**
  2:  * 包装通道处理器
  3:  *
  4:  * @param url URL
  5:  * @param handler 被包装的通道处理器
  6:  * @return 包装后的通道处理器
  7:  */
  8: protected static ChannelHandler wrapChannelHandler(URL url, ChannelHandler handler) {
  9:     // 设置线程名
 10:     url = ExecutorUtil.setThreadName(url, CLIENT_THREAD_POOL_NAME);
 11:     // 设置使用的线程池类型
 12:     url = url.addParameterIfAbsent(Constants.THREADPOOL_KEY, Constants.DEFAULT_CLIENT_THREADPOOL);
 13:     // 包装通道处理器
 14:     return ChannelHandlers.wrap(handler, url);
 15: }
```

* 第 10 行：调用 `ExecutorUtil#setThreadName(url, CLIENT_THREAD_POOL_NAME)`方法，设置**线程名**，即 `URL.threadname=xxx` 。代码如下：

    ```Java
    public static URL setThreadName(URL url, String defaultName) {
        String name = url.getParameter(Constants.THREAD_NAME_KEY, defaultName);
        name = new StringBuilder(32).append(name).append("-").append(url.getAddress()).toString();
        url = url.addParameter(Constants.THREAD_NAME_KEY, name);
        return url;
    }
    ```
    * 注意，线程名中，包含 **URL 的地址信息**。

* 第 12 行：设置**线程类型**，即 `URL.threadpool=xxx` 。默认情况下，使用 `"cached"` 类型，这个和 Server 是不同的，下面我们会看到。
    * [《精尽 Dubbo 源码分析 —— 线程池》](http://www.iocoder.cn/Dubbo/thread-pool/?self)
* 第 14 行：调用 `ChannelHandlers#wrap(handler, url)`  方法，包装通道处理器。这里我们不细说，在 [「8. Dispacher」](#) 中，结合解析。
* 🙂 这是一个非常关键的方法，在例如 NettyClient 等里，都会调用该方法。

**其他方法**

如下方法比较简单，艿艿就不重复啰嗦了。

* [`#disconnect()`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractClient.java#L291-L311) 方法，断开连接。
* [`#reconnect()`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractClient.java#L313-L316) 方法，主动重连。
* [`#close()`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractClient.java#L318-L341) 方法，强制关闭。 
* [`#close(timeout)`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractClient.java#L343-L346) 方法，优雅关闭。

**子类类图**

[类图](http://www.iocoder.cn/images/Dubbo/2018_12_04/03.png)

## 3.2 ClientDelegate 

[`com.alibaba.dubbo.remoting.transport.ClientDelegate`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/ClientDelegate.java) ，实现 Client 接口，客户端委托实现类。在每个实现的方法里，直接调用被委托的 [`client`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/ClientDelegate.java#L31) 属性的方法。

目前 `dubbo-rpc-default` 模块中，[ChannelWrapper](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-rpc/dubbo-rpc-default/src/main/java/com/alibaba/dubbo/rpc/protocol/dubbo/ChannelWrappedInvoker.java#L91-L160) 继承了 ClientDelegate 类。但实际上，ChannelWrapper **重新实现了所有的方法**，并且，并未复用任何方法。所以，ClientDelegate 目前用途不大。

# 4. Server

## 4.1 AbstractServer


[`com.alibaba.dubbo.remoting.transport.AbstractServer`](TODO) ，实现 Server 接口，继承 AbstractEndpoint 抽象类，**服务器**抽象类，**重点**实现了公用的逻辑，同时抽象了开启、关闭等模板方法，供子类实现。抽象方法如下：

```Java
protected abstract void doOpen() throws Throwable;

protected abstract void doClose() throws Throwable;
```

**构造方法**

```Java
  1: /**
  2:  * 线程池
  3:  */
  4: ExecutorService executor;
  5: /**
  6:  * 服务地址
  7:  */
  8: private InetSocketAddress localAddress;
  9: /**
 10:  * 绑定地址
 11:  */
 12: private InetSocketAddress bindAddress;
 13: /**
 14:  * 服务器最大可接受连接数
 15:  */
 16: private int accepts;
 17: /**
 18:  * 空闲超时时间，单位：毫秒
 19:  */
 20: private int idleTimeout; //600 seconds
 21: 
 22: public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
 23:     super(url, handler);
 24:     // 服务地址
 25:     localAddress = getUrl().toInetSocketAddress();
 26:     // 绑定地址
 27:     String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
 28:     int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
 29:     if (url.getParameter(Constants.ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
 30:         bindIp = NetUtils.ANYHOST;
 31:     }
 32:     bindAddress = new InetSocketAddress(bindIp, bindPort);
 33:     // 服务器最大可接受连接数
 34:     this.accepts = url.getParameter(Constants.ACCEPTS_KEY, Constants.DEFAULT_ACCEPTS);
 35:     // 空闲超时时间
 36:     this.idleTimeout = url.getParameter(Constants.IDLE_TIMEOUT_KEY, Constants.DEFAULT_IDLE_TIMEOUT);
 37: 
 38:     // 开启服务器
 39:     try {
 40:         doOpen();
 41:         if (logger.isInfoEnabled()) {
 42:             logger.info("Start " + getClass().getSimpleName() + " bind " + getBindAddress() + ", export " + getLocalAddress());
 43:         }
 44:     } catch (Throwable t) {
 45:         throw new RemotingException(url.toInetSocketAddress(), null, "Failed to bind " + getClass().getSimpleName()
 46:                 + " on " + getLocalAddress() + ", cause: " + t.getMessage(), t);
 47:     }
 48: 
 49:     // 获得线程池
 50:     //fixme replace this with better method
 51:     DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension();
 52:     executor = (ExecutorService) dataStore.get(Constants.EXECUTOR_SERVICE_COMPONENT_KEY, Integer.toString(url.getPort()));
 53: }
```

* 第 24 至 36 行：从 URL 中，加载 `localAddress` `bindAddress` `accepts` `idleTimeout` 配置项。比较难理解的，可能是两个地址属性，如下是比例提供的一个例子：[例子](http://www.iocoder.cn/images/Dubbo/2018_12_04/02.png)
    * 配置项可在 [`#reset(url)`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractServer.java#L80-L129) 方法中，重置属性。 
* 第 38 至 47 行：调用 `#doOpen()` 方法，开启服务器。
* 第 49 至 52 行：从 [DataStore](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-common/src/main/java/com/alibaba/dubbo/common/store/DataStore.java) 中，获得线程池。
    * `fixme replace this with better method` ，说明**官方**在这块实现上，也不是很满意，后面会优化掉。

**被客户端连接**

```Java
@Override
public void connected(Channel ch) throws RemotingException {
    // If the server has entered the shutdown process, reject any new connection
    if (this.isClosing() || this.isClosed()) {
        logger.warn("Close new channel " + ch + ", cause: server is closing or has been closed. For example, receive a new connect request while in shutdown process.");
        ch.close();
        return;
    }

    // 超过上限，关闭新的链接
    Collection<Channel> channels = getChannels();
    if (accepts > 0 && channels.size() > accepts) {
        logger.error("Close channel " + ch + ", cause: The server " + ch.getLocalAddress() + " connections greater than max config " + accepts);
        ch.close(); // 关闭新的链接
        return;
    }
    // 连接
    super.connected(ch);
}
```

**发送消息**

```Java
@Override
public void send(Object message, boolean sent) throws RemotingException {
    // 获得所有的客户端的通道
    Collection<Channel> channels = getChannels();
    // 群发消息
    for (Channel channel : channels) {
        if (channel.isConnected()) {
            channel.send(message, sent);
        }
    }
}
```

**其他方法**

如下方法比较简单，艿艿就不重复啰嗦了。

* [`#disconnect()`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractServer.java#L196-L203) 方法，断开连接。
* [`#close()`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractServer.java#L140-L155) 方法，强制关闭。 
* [`#close(timeout)`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractServer.java#L157-L160) 方法，优雅关闭。

**子类类图**

[类图](http://www.iocoder.cn/images/Dubbo/2018_12_04/04.png)

## 4.2 ServerDelegate

[`com.alibaba.dubbo.remoting.transport.ServerDelegate`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/ServerDelegate.java) ，实现 Client 接口，客户端委托实现类。在每个实现的方法里，直接调用被委托的 [`server`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/ServerDelegate.java#L35) 属性的方法。

目前 `dubbo-remoting-p2p` 模块中，PeerServer 会继承该类，后续再看。

# 5. Channel

## 5.1 AbstractChannel

[`com.alibaba.dubbo.remoting.transport.AbstractChannel`](https://github.com/YunaiV/dubbo/blob/4fa80f25673c4e7060847a711a87ea37ed152d91/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/transport/AbstractChannel.java) ，实现 Channel 接口，实现 AbstractPeer 抽象类，**通道**抽象类。

**发送消息**

**子类类图**



# 7. ChannelHandler

# 8. Dispacher

# 9. Codec

# 666. 彩蛋

