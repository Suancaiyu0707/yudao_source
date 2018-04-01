title: 精尽 Dubbo 源码分析 —— NIO 服务器（二）之 Transport 层
date: 2018-12-04
tags:
categories: Dubbo
permalink: Dubbo/remoting-api-transport

-------

# 1. 概述

本文接 [《精尽 Dubbo 源码分析 —— NIO 服务器（三）之 NIO 服务器（三）之 Telnet 层》](http://www.iocoder.cn/Dubbo/remoting-api-telnet//?self) 一文，分享 `dubbo-remoting-api` 模块， `exchange` 包，**信息交换层**。

> **exchange** 信息交换层：封装请求响应模式，同步转异步，以 Request, Response 为中心，扩展接口为 Exchanger, ExchangeChannel, ExchangeClient, ExchangeServer。  

在一次 RPC 调用，每个**请求**( Request )，是关注对应的**响应**( Response )。那么 **transport 层** 提供的**网络传输** 功能，是无法满足 RPC 的诉求的。因此，**exchange 层**，在其 **Message** 之上，构造了**Request-Response** 的模型。

实现上，也非常简单，将 Message 分成 Request 和 Response 两种类型，并增加**编号**属性，将 Request 和 Response 能够**一一映射**。

实际上，RPC 调用，会有更多特性的需求：1）**异步**处理返回结果；2）内置事件；3）等等。因此，Request 和 Response 上会有类似**编号**的**系统字段**。

一条消息，我们分成两段：

* 协议头( Header ) ： 系统字段，例如编号等。
* 内容( Body )  ：具体请求的参数和响应的结果等。

胖友在看下面这张图，是否就亲切多了 🙂 ：

[类图](http://www.iocoder.cn/images/Dubbo/2018_12_10/01.png)

所以，`exchange` 包，很多的代码，是在 Header 的处理。OK ，下面我们来看下这个包的**类图**：

[类图](http://www.iocoder.cn/images/Dubbo/2018_12_10/02.png)

* 白色部分，为通用接口和 `transport` 包下的类。
* 蓝色部分，为 `exchange` 包下的类。

在 [《精尽 Dubbo 源码分析 —— NIO 服务器（二）之 Transport 层》](http://www.iocoder.cn/Dubbo/remoting-api-transport/?self)  中，我们提到，**装饰器设计模式**，是 `dubbo-remoting` 项目，最核心的实现方式，所以，`exchange` 其实是在 `transport` 上的**装饰**，提供给 `dubbo-rpc` 项目使用。

下面，我们来看具体代码实现。

# 2. ExchangeChannel

`com.alibaba.dubbo.remoting.exchange.ExchangeChannel` ，继承 Channel 接口，**信息交换通道**接口。方法如下：

```Java
// 发送请求
ResponseFuture request(Object request) throws RemotingException;
ResponseFuture request(Object request, int timeout) throws RemotingException;

// 获得信息交换处理器
ExchangeHandler getExchangeHandler();

// 优雅关闭
void close(int timeout);
```

## 2.1 HeaderExchangeChannel

`com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeChannel` ，实现 ExchangeChannel 接口，基于**消息头部( Header )**的信息交换通道实现类。

**构造方法**

```Java
private static final String CHANNEL_KEY = HeaderExchangeChannel.class.getName() + ".CHANNEL";

/**
 * 通道
 */
private final Channel channel;
/**
 * 是否关闭
 */
private volatile boolean closed = false;

HeaderExchangeChannel(Channel channel) {
    if (channel == null) {
        throw new IllegalArgumentException("channel == null");
    }
    this.channel = channel;
}
```

* `channel` 属性，通道。HeaderExchangeChannel 是传入 `channel` 属性的**装饰器**，每个实现的方法，都会调用 `channel` 。如下是该属性的一个例子：[`channel`](http://www.iocoder.cn/images/Dubbo/2018_12_10/03.png)
* `#getOrAddChannel(Channel)` **静态**方法，创建 HeaderExchangeChannel 对象。代码如下：

    ```Java
    static HeaderExchangeChannel getOrAddChannel(Channel ch) {
        if (ch == null) {
            return null;
        }
        HeaderExchangeChannel ret = (HeaderExchangeChannel) ch.getAttribute(CHANNEL_KEY);
        if (ret == null) {
            ret = new HeaderExchangeChannel(ch);
            if (ch.isConnected()) { // 已连接
                ch.setAttribute(CHANNEL_KEY, ret);
            }
        }
        return ret;
    }
    ```
    * 传入的 `ch` 属性，实际就是 `HeaderExchangeChanel.channel` 属性。
    * 通过 `ch.attribute` 的 `CHANNEL_KEY` 键值，保证有且仅有为 `ch` 属性，创建唯一的 HeaderExchangeChannel 对象。
    * 要求**已连接**。
* `#removeChannelIfDisconnected(ch)` **静态方法**，移除 HeaderExchangeChannel 对象。代码如下：

    ```Java
    static void removeChannelIfDisconnected(Channel ch) {
        if (ch != null && !ch.isConnected()) { // 未连接
            ch.removeAttribute(CHANNEL_KEY);
        }
    }
    ```

**发送请求**

```Java
  1: @Override
  2: public ResponseFuture request(Object request, int timeout) throws RemotingException {
  3:     if (closed) {
  4:         throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
  5:     }
  6:     // create request. 创建请求
  7:     Request req = new Request();
  8:     req.setVersion("2.0.0");
  9:     req.setTwoWay(true); // 需要响应
 10:     req.setData(request);
 11:     // 创建 DefaultFuture 对象
 12:     DefaultFuture future = new DefaultFuture(channel, req, timeout);
 13:     try {
 14:         // 发送请求
 15:         channel.send(req);
 16:     } catch (RemotingException e) { // 发生异常，取消 DefaultFuture
 17:         future.cancel();
 18:         throw e;
 19:     }
 20:     // 返回 DefaultFuture 对象
 21:     return future;
 22: }
```

* 第 3 至 5 行：若已经关闭，不再允许发起新的请求。
* 第 6 至 10 行：创建 Request 对象。其中，`twoWay = true` 需要响应；`data = request` 具体数据。
* 第 12 行：创建 DefaultFuture 对象。
* 第 13 至 15 行：调用 `Channel#send(req)` 方法，发送请求。
* 第 16 至 19 行：发生 RemotingException 异常，调用 `DefaultFuture#cancel()` 方法，取消。
* 第 21 行：返回 DefaultFuture 对象。从代码的形式上来说，有点类似线程池提交任务，返回 Future 对象。🙂 看到 DefaultFuture 的具体代码，我们就会更加理解了。

**优雅关闭**

```Java
  1: @Override
  2: public void close(int timeout) {
  3:     if (closed) {
  4:         return;
  5:     }
  6:     closed = true;
  7:     // 等待请求完成
  8:     if (timeout > 0) {
  9:         long start = System.currentTimeMillis();
 10:         while (DefaultFuture.hasFuture(channel) && System.currentTimeMillis() - start < timeout) {
 11:             try {
 12:                 Thread.sleep(10);
 13:             } catch (InterruptedException e) {
 14:                 logger.warn(e.getMessage(), e);
 15:             }
 16:         }
 17:     }
 18:     // 关闭通道
 19:     close();
 20: }
```

* 第 3 至 6 行：标记 `closed = true` ，避免发起**新**的请求。
* 第 7 至 17 行：调用 `DefaultFuture#hasFuture(channel)` 方法，判断已发起的已经是否已经都响应了。若否，等待完成或超时。
* 第 19 行：关闭**通道**。

**其它方法**

其它**实现**方法，主要是直接调用 `channel` 的方法，点击 [传送门](https://github.com/YunaiV/dubbo/blob/2a0484941defceb9a600c7f7914ada335e3186af/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/exchange/support/header/HeaderExchangeChannel.java) 查看代码。

# 3. ExchangeClient

`com.alibaba.dubbo.remoting.exchange.ExchangeClient` ，实现 Client ，ExchangeChannel 接口，**信息交换客户端**接口。

无自定义方法。

## 3.1 HeaderExchangeClient

`com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeClient` ，实现 ExchangeClient 接口，基于**消息头部( Header )**的信息交换客户端实现类。

**构造方法**

```Java
  1: /**
  2:  * 定时器线程池
  3:  */
  4: private static final ScheduledThreadPoolExecutor scheduled = new ScheduledThreadPoolExecutor(2, new NamedThreadFactory("dubbo-remoting-client-heartbeat", true));
  5: /**
  6:  * 客户端
  7:  */
  8: private final Client client;
  9: /**
 10:  * 信息交换通道
 11:  */
 12: private final ExchangeChannel channel;
 13: // heartbeat timer
 14: /**
 15:  * 心跳定时器
 16:  */
 17: private ScheduledFuture<?> heartbeatTimer;
 18: /**
 19:  * 是否心跳
 20:  */
 21: private int heartbeat;
 22: // heartbeat timeout (ms), default value is 0 , won't execute a heartbeat.
 23: /**
 24:  * 心跳间隔，单位：毫秒
 25:  */
 26: private int heartbeatTimeout;
 27: 
 28: public HeaderExchangeClient(Client client, boolean needHeartbeat) {
 29:     if (client == null) {
 30:         throw new IllegalArgumentException("client == null");
 31:     }
 32:     this.client = client;
 33:     // 创建 HeaderExchangeChannel 对象
 34:     this.channel = new HeaderExchangeChannel(client);
 35:     // 读取心跳相关配置
 36:     String dubbo = client.getUrl().getParameter(Constants.DUBBO_VERSION_KEY);
 37:     this.heartbeat = client.getUrl().getParameter(Constants.HEARTBEAT_KEY, dubbo != null && dubbo.startsWith("1.0.") ? Constants.DEFAULT_HEARTBEAT : 0);
 38:     this.heartbeatTimeout = client.getUrl().getParameter(Constants.HEARTBEAT_TIMEOUT_KEY, heartbeat * 3);
 39:     if (heartbeatTimeout < heartbeat * 2) { // 避免间隔太短
 40:         throw new IllegalStateException("heartbeatTimeout < heartbeatInterval * 2");
 41:     }
 42:     // 发起心跳定时器
 43:     if (needHeartbeat) {
 44:         startHeatbeatTimer();
 45:     }
 46: }
```

* `client` 属性，客户端。如下是该属性的一个例子：[`client`](http://www.iocoder.cn/images/Dubbo/2018_12_10/04.png)
* 第 34 行：使用传入的 `client` 属性，创建  HeaderExchangeChannel 对象。
* 第 35 至 41 行：读取心跳相关配置。**默认，开启心跳功能**。为什么需要有心跳功能呢？

    > FROM [《Dubbo 用户指南 —— dubbo:protocol》](https://dubbo.gitbooks.io/dubbo-user-book/references/xml/dubbo-protocol.html)
    > 
    > 心跳间隔，对于长连接，当物理层断开时，比如拔网线，TCP的FIN消息来不及发送，对方收不到断开事件，此时需要心跳来帮助检查连接是否已断开

* 第 42 至 45 行：调用 `#startHeatbeatTimer()` 方法，发起心跳定时器。

**发起心跳定时器**

```Java
  1: private void startHeatbeatTimer() {
  2:     // 停止原有定时任务
  3:     stopHeartbeatTimer();
  4:     // 发起新的定时任务
  5:     if (heartbeat > 0) {
  6:         heartbeatTimer = scheduled.scheduleWithFixedDelay(
  7:                 new HeartBeatTask(new HeartBeatTask.ChannelProvider() {
  8:                     public Collection<Channel> getChannels() {
  9:                         return Collections.<Channel>singletonList(HeaderExchangeClient.this);
 10:                     }
 11:                 }, heartbeat, heartbeatTimeout),
 12:                 heartbeat, heartbeat, TimeUnit.MILLISECONDS);
 13:     }
 14: }
```

* 第 3 行：调用 [`#stopHeartbeatTimer()`](https://github.com/YunaiV/dubbo/blob/0f933100ad0ea81d3760d42169318904f91a45bb/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/exchange/support/header/HeaderExchangeClient.java#L176-L188) 方法，停止原有定时任务。
* 第 5 至 13 行：发起新的定时任务。
    * 第 7 至 11 行：创建定时任务 HeartBeatTask 对象。具体实现见下文。

**其它方法**

其它**实现**方法，主要是直接调用 `channel` 或 `client` 的方法，点击 [传送门](https://github.com/YunaiV/dubbo/blob/0f933100ad0ea81d3760d42169318904f91a45bb/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/exchange/support/header/HeaderExchangeClient.java) 查看代码。

# 4. ExchangeServer

`com.alibaba.dubbo.remoting.exchange.ExchangeServer` ，继承 Server 接口，**信息交换服务器**接口。方法如下：

```Java
// 获得通道数组
Collection<ExchangeChannel> getExchangeChannels();
ExchangeChannel getExchangeChannel(InetSocketAddress remoteAddress);
```

## 4.1 HeaderExchangeServer

`com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeServer` ，实现 ExchangeServer 接口，基于**消息头部( Header )**的信息交换服务器实现类。

> 代码实现上，和 HeaderExchangeChannel + HeaderExchangeClient 的综合。

**构造方法**

> 代码实现上，和 HeaderExchangeClient 的类似。

```Java
/**
 * 定时器线程池
 */
private final ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(1, new NamedThreadFactory("dubbo-remoting-server-heartbeat", true));
/**
 * 服务器
 */
private final Server server;
// heartbeat timer
/**
 * 心跳定时器
 */
private ScheduledFuture<?> heatbeatTimer;
/**
 * 是否心跳
 */
// heartbeat timeout (ms), default value is 0 , won't execute a heartbeat.
private int heartbeat;
/**
 * 心跳间隔，单位：毫秒
 */
private int heartbeatTimeout;
/**
 * 是否关闭
 */
private AtomicBoolean closed = new AtomicBoolean(false);

public HeaderExchangeServer(Server server) {
    if (server == null) {
        throw new IllegalArgumentException("server == null");
    }
    // 读取心跳相关配置
    this.server = server;
    this.heartbeat = server.getUrl().getParameter(Constants.HEARTBEAT_KEY, 0);
    this.heartbeatTimeout = server.getUrl().getParameter(Constants.HEARTBEAT_TIMEOUT_KEY, heartbeat * 3);
    if (heartbeatTimeout < heartbeat * 2) {
        throw new IllegalStateException("heartbeatTimeout < heartbeatInterval * 2");
    }
    // 发起心跳定时器
    startHeatbeatTimer();
}
```

**发起心跳定时器**

> 代码实现上，和 HeaderExchangeClient 的类似。

```Java
private void startHeatbeatTimer() {
    // 停止原有定时任务
    stopHeartbeatTimer();
    // 发起新的定时任务
    if (heartbeat > 0) {
        heatbeatTimer = scheduled.scheduleWithFixedDelay(
                new HeartBeatTask(new HeartBeatTask.ChannelProvider() {
                    public Collection<Channel> getChannels() {
                        return Collections.unmodifiableCollection(HeaderExchangeServer.this.getChannels());
                    }
                }, heartbeat, heartbeatTimeout),
                heartbeat, heartbeat, TimeUnit.MILLISECONDS);
    }
}
```

* 差异，Server 持有**多条** Client 连接的 Channel ，所以通过 ChannelProvider 返回的是**多条**。

**重置属性**

```Java
@Override
public void reset(URL url) {
    // 重置服务器
    server.reset(url);
    try {
        if (url.hasParameter(Constants.HEARTBEAT_KEY)
                || url.hasParameter(Constants.HEARTBEAT_TIMEOUT_KEY)) {
            int h = url.getParameter(Constants.HEARTBEAT_KEY, heartbeat);
            int t = url.getParameter(Constants.HEARTBEAT_TIMEOUT_KEY, h * 3);
            if (t < h * 2) {
                throw new IllegalStateException("heartbeatTimeout < heartbeatInterval * 2");
            }
            // 重置定时任务
            if (h != heartbeat || t != heartbeatTimeout) {
                heartbeat = h;
                heartbeatTimeout = t;
                startHeatbeatTimer();
            }
        }
    } catch (Throwable t) {
        logger.error(t.getMessage(), t);
    }
}
```

**优雅关闭**

> 代码实现上，和 HeaderExchangeChannel 的类似，且复杂一些。

```Java
  1: @Override
  2: public void close(final int timeout) {
  3:     // 关闭
  4:     startClose();
  5:     if (timeout > 0) {
  6:         final long max = (long) timeout;
  7:         final long start = System.currentTimeMillis();
  8:         // 发送 READONLY 事件给所有 Client ，表示 Server 不可读了。
  9:         if (getUrl().getParameter(Constants.CHANNEL_SEND_READONLYEVENT_KEY, true)) {
 10:             sendChannelReadOnlyEvent();
 11:         }
 12:         // 等待请求完成
 13:         while (HeaderExchangeServer.this.isRunning() && System.currentTimeMillis() - start < max) {
 14:             try {
 15:                 Thread.sleep(10);
 16:             } catch (InterruptedException e) {
 17:                 logger.warn(e.getMessage(), e);
 18:             }
 19:         }
 20:     }
 21:     // 关闭心跳定时器
 22:     doClose();
 23:     // 关闭服务器
 24:     server.close(timeout);
 25: }
```

* Server 关闭的过程，分成**两个阶段**：正在关闭和已经关闭。
* 第 4 行：调用 `#startClose()` 方法，标记正在关闭。代码如下：

    ```Java
    @Override
    public void startClose() {
        server.startClose();
    }
    
    // AbstractPeer.java
    @Override
    public void startClose() {
        if (isClosed()) {
            return;
        }
        closing = true;
    }
    ```

* 第 8 至 11 行：发送 **READONLY** 事件给所有 Client ，表示 Server 不再接收新的消息，避免不断有**新的消息**接收到。杂实现的呢？以 DubboInvoker 举例子，`#isAvailable()` 方法，代码如下：

    ```Java
    @Override
    public boolean isAvailable() {
        if (!super.isAvailable())
            return false;
        for (ExchangeClient client : clients) {
            if (client.isConnected() && !client.hasAttribute(Constants.CHANNEL_ATTRIBUTE_READONLY_KEY)) { // 只读判断
                //cannot write == not Available ?
                return true;
            }
        }
        return false;
    }
    ```
    * 即使 `client` 处于**连接中**，但是 Server 处于**正在关闭中**，也算**不可用**，不进行发送请求( 消息 )。
* `#sendChannelReadOnlyEvent()` 方法，广播客户端，READONLY_EVENT 事件。代码如下：

    ```Java
    private void sendChannelReadOnlyEvent() {
        // 创建 READONLY_EVENT 请求
        Request request = new Request();
        request.setEvent(Request.READONLY_EVENT);
        request.setTwoWay(false); // 无需响应
        request.setVersion(Version.getVersion());
    
        // 发送给所有 Client
        Collection<Channel> channels = getChannels();
        for (Channel channel : channels) {
            try {
                if (channel.isConnected())
                    channel.send(request, getUrl().getParameter(Constants.CHANNEL_READONLYEVENT_SENT_KEY, true));
            } catch (RemotingException e) {
                logger.warn("send connot write messge error.", e);
            }
        }
    }
    ```

* 第 22 行：调用 `#oClose()` 方法，关闭心跳定时器。代码如下：

    ```Java
    private void doClose() {
        if (!closed.compareAndSet(false, true)) {
            return;
        }
        stopHeartbeatTimer();
        try {
            scheduled.shutdown();
        } catch (Throwable t) {
            logger.warn(t.getMessage(), t);
        }
    }
    ```

* 第 24 行：**真正**关闭服务器。

## 4.2 ExchangeServerDelegate

[`com.alibaba.dubbo.remoting.exchange.support.ExchangeServerDelegate`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/exchange/support/ExchangeServerDelegate.java) ，实现 ExchangeServer 接口，信息交换服务器装饰者。在每个实现的方法里，直接调用被装饰的 [`server`](https://github.com/YunaiV/dubbo/blob/31b3f1e868ed2d62c97a26b5cd233a921ce2205a/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/exchange/support/ExchangeServerDelegate.java#L34) 属性的方法。

目前 `dubbo-remoting-p2p` 模块中，ExchangeServerPeer 会继承该类，后续再看。

# 5. 请求/响应模型

## 5.1 Request

## 5.2 Response

## 5.3 ResponseFuture

# 6. 

# 7. Handler

在文初的

## 6.1 ExchangeHandler



