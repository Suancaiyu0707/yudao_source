title: 精尽 Dubbo 源码分析 —— 过滤器（二）之 ContextFilter
date: 2018-11-13
tags:
categories: Dubbo
permalink: Dubbo/filter-context-filter

-------

摘要: 原创出处 http://www.iocoder.cn/Dubbo/filter-context-filter/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Dubbo/filter-context-filter/)
- [2. RpcContext](http://www.iocoder.cn/Dubbo/filter-context-filter/)
- [3. ConsumerContextFilter](http://www.iocoder.cn/Dubbo/filter-context-filter/)
- [4. ContextFilter](http://www.iocoder.cn/Dubbo/filter-context-filter/)
- [5. RpcContext.values](http://www.iocoder.cn/Dubbo/filter-context-filter/)
- [666. 彩蛋](http://www.iocoder.cn/Dubbo/filter-context-filter/)

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

本文分享 RpcContext 相关过滤器，包括两个：

* ConsumerContextFilter ：在服务**消费者**中使用，负责**发起**调用时，初始化 RpcContext 。
* ContextFilter ：在服务**提供者**中使用，负责**被**调用时，初始化 RpcContext 。

# 2. RpcContext

RpcContext，上下文信息。在 [《Dubbo 用户指南 —— 上下文信息》](https://dubbo.gitbooks.io/dubbo-user-book/demos/context.html)  ，定义如下：

> 上下文中存放的是当前调用过程中所需的环境信息。所有配置信息都将转换为 URL 的参数，参见 [schema 配置参考手册](https://dubbo.gitbooks.io/dubbo-user-book/references/xml/introduction.html) 中的对应URL参数一列。
> 
> RpcContext 是一个 ThreadLocal 的临时状态记录器，当接收到 RPC 请求，或发起 RPC 请求时，RpcContext 的状态都会变化。比如：A 调 B，B 再调 C，则 B 机器上，
> 
> * 在 B 调 C 之前，RpcContext 记录的是 A 调 B 的信息，
> * 在 B 调 C 之后，RpcContext 记录的是 B 调 C 的信息。

* RpcContext 在调用时的状态变化，有点绕，下面我们看具体的 Filter 实现，就相对容易明白列。

[`com.alibaba.dubbo.rpc.RpcContext`](TODO)  ，代码如下：

```Java
/**
 * RpcContext 线程变量
 */
private static final ThreadLocal<RpcContext> LOCAL = new ThreadLocal<RpcContext>() {

    @Override
    protected RpcContext initialValue() {
        return new RpcContext();
    }

};

/**
 * 隐式参数集合
 */
private final Map<String, String> attachments = new HashMap<String, String>();
// 实际未使用
private final Map<String, Object> values = new HashMap<String, Object>();
/**
 * 异步调用 Future
 */
private Future<?> future;
/**
 * 可调用服务的 URL 对象集合
 */
private List<URL> urls;
/**
 * 调用服务的 URL 对象
 */
private URL url;
/**
 * 方法名
 */
private String methodName;
/**
 * 参数类型数组
 */
private Class<?>[] parameterTypes;
/**
 * 参数值数组
 */
private Object[] arguments;
/**
 * 服务消费者地址
 */
private InetSocketAddress localAddress;
/**
 * 服务提供者地址
 */
private InetSocketAddress remoteAddress;

@Deprecated // DUBBO-325 废弃的，使用 urls 属性替代
private List<Invoker<?>> invokers;
@Deprecated // DUBBO-325 废弃的，使用 url 属性替代
private Invoker<?> invoker;
@Deprecated // DUBBO-325 废弃的，使用 methodName、parameterTypes、arguments 属性替代
private Invocation invocation;

/**
 * 请求
 *
 * 例如，在 RestProtocol
 */
private Object request;
/**
 * 响应
 *
 * 例如，在 RestProtocol
 */
private Object response;
 
 // ... 省略一些
```

* `LOCAL` **静态**属性，RpcContext 线程变量。初始获得时，返回新的 RpcContext 对象。
* `attachments` 属性，隐式参数集合。
    * 例如，我们在 PRC 调用前，可在业务代码里添加一些想要传递给服务的参数到该属性
    * 又例如，在分布式链路追踪时，添加链路追踪**编号**到该属性种。
    * [《Dubbo 用户指南 —— 隐式参数》](https://dubbo.gitbooks.io/dubbo-user-book/demos/attachment.html)
* `future` 属性，异步调用 Future 对象，在 [《精尽 Dubbo 源码分析 —— 服务调用（三）之远程调用（Dubbo）【3】异步调用》](http://www.iocoder.cn/Dubbo/rpc-dubbo-3-async/?self) 有详细使用的代码分享。
* 【替代 `invokers` 属性】
    * `urls` 属性，**可调用**的服务的 URL 对象集合，在集群容错模块实现。
* 【替代 `invoker` 属性】
    * `url` 属性，**调用**的服务的 URL 对象。
* 【替代 `invocation` 属性】
    * `methodName` 属性，**调用**的方法名。
    * `parameterTypes` 属性，**调用**的参数类型数组。
    * `arguments` 属性，**调用**的参数值数组。
* 地址
    * `localAddress` 属性， 服务消费者地址。
    * `remoteAddress` 属性，服务提供者地址。
* `request` `response` 属性，请求和响应。例如，在 RestProtocol 中使用，代表 HTTP Request 和 Response 对象，在 RpcContextFilter 中设置，如下图所示：![RpcContextFilter](http://www.iocoder.cn/images/Dubbo/2018_11_13/01.png)
    * 我们可以看到 `request` `response` 的类型是 Object 类。通过这种形式，可以不仅仅适用于 HTTP 的场景。 

RpcContext  中，有很多方法，比较易懂，胖友自己查看噢。

# 3. ConsumerContextFilter

`com.alibaba.dubbo.rpc.filter.ConsumerContextFilter` ，实现 Filter 接口，**服务消费者**的 ContextFilter 实现类。

```Java
  1: @Activate(group = Constants.CONSUMER, order = -10000)
  2: public class ConsumerContextFilter implements Filter {
  3: 
  4:     @Override
  5:     public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
  6:         // 设置 RpcContext 对象
  7:         RpcContext.getContext()
  8:                 .setInvoker(invoker)
  9:                 .setInvocation(invocation)
 10:                 .setLocalAddress(NetUtils.getLocalHost(), 0) // 本地地址
 11:                 .setRemoteAddress(invoker.getUrl().getHost(), invoker.getUrl().getPort()); // 远程地址
 12:         // 设置 RpcInvocation 对象的 `invoker` 属性
 13:         if (invocation instanceof RpcInvocation) {
 14:             ((RpcInvocation) invocation).setInvoker(invoker);
 15:         }
 16:         // 服务调用
 17:         try {
 18:             return invoker.invoke(invocation);
 19:         } finally {
 20:             // 清理隐式参数集合
 21:             RpcContext.getContext().clearAttachments();
 22:         }
 23:     }
 24: 
 25: }
```

* 第 6 至 11 行：设置 RpcContext 对象。
* 第 12 至 15 行：设置 RpcInvocation 对象的 `invoker` 属性。该属性，目前使用在如下图的场景：![RpcInvocation](http://www.iocoder.cn/images/Dubbo/2018_11_13/02.png)
* 第 18 行：调用 `Invoker#invoke(invocation)` 方法，服务调用。
* 第 19 至 22 行：调用 `RpcContext#clearAttachments()` 方法，清理隐式参数集合。所以，**每次**（注意，每次！！！）服务调用完成，RpcContext 设置的隐式参数**都会被清理**！代码如下：

    ```Java
    public void clearAttachments() {
        this.attachments.clear();
    }
    ```
    
😈 看到此处，`RpcContext.attachments` 属性，是如何传递给被调用的服务的呢？答案在下图：![透传](http://www.iocoder.cn/images/Dubbo/2018_11_13/03.png)
    
# 4. ContextFilter

`com.alibaba.dubbo.rpc.filter.ContextFilter` ，实现 Filter 接口，**服务提供者**的 ContextFilter 实现类。

```Java
  1: @Activate(group = Constants.PROVIDER, order = -10000)
  2: public class ContextFilter implements Filter {
  3: 
  4:     @Override
  5:     public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
  6:         // 创建新的 `attachments` 集合，清理公用的隐式参数
  7:         Map<String, String> attachments = invocation.getAttachments();
  8:         if (attachments != null) {
  9:             attachments = new HashMap<String, String>(attachments);
 10:             attachments.remove(Constants.PATH_KEY);
 11:             attachments.remove(Constants.GROUP_KEY);
 12:             attachments.remove(Constants.VERSION_KEY);
 13:             attachments.remove(Constants.DUBBO_VERSION_KEY);
 14:             attachments.remove(Constants.TOKEN_KEY);
 15:             attachments.remove(Constants.TIMEOUT_KEY);
 16:             attachments.remove(Constants.ASYNC_KEY); // Remove async property to avoid being passed to the following invoke chain.
 17:                                                      // 清空消费端的异步参数
 18:         }
 19:         // 设置 RpcContext 对象
 20:         RpcContext.getContext()
 21:                 .setInvoker(invoker)
 22:                 .setInvocation(invocation)
 23: //                .setAttachments(attachments)  // merged from dubbox
 24:                 .setLocalAddress(invoker.getUrl().getHost(), invoker.getUrl().getPort());
 25:         // mreged from dubbox
 26:         // we may already added some attachments into RpcContext before this filter (e.g. in rest protocol)
 27:         // 在此过滤器(例如rest协议)之前，我们可能已经在RpcContext中添加了一些附件。
 28:         if (attachments != null) {
 29:             if (RpcContext.getContext().getAttachments() != null) {
 30:                 RpcContext.getContext().getAttachments().putAll(attachments);
 31:             } else {
 32:                 RpcContext.getContext().setAttachments(attachments);
 33:             }
 34:         }
 35:         // 设置 RpcInvocation 对象的 `invoker` 属性
 36:         if (invocation instanceof RpcInvocation) {
 37:             ((RpcInvocation) invocation).setInvoker(invoker);
 38:         }
 39:         // 服务调用
 40:         try {
 41:             return invoker.invoke(invocation);
 42:         } finally {
 43:             // 移除上下文
 44:             RpcContext.removeContext();
 45:         }
 46:     }
 47: 
 48: }
```

* 第 6 至 18 行：创建新的 `attachments` 集合，因为要清理**公用**的隐式参数。该**公用**的隐式参数，设置的地方，如下图所示：![RpcInvocation](http://www.iocoder.cn/images/Dubbo/2018_11_13/04.png)
* 第 19 至 24 行：设置 RpcContext 对象。
* 第 25 至 34 行：在此过滤器( 例如 RestProtocol 的 RpcContextFilter )之前，我们可能已经在 RpcContext 中添加了一些隐式参数。
* 第 35 至 38 行：调用 `Invoker#invoke(invocation)` 方法，服务调用。
* 第 41 行：调用 `Invoker#invoke(invocation)` 方法，服务调用。
* 第 42 至 45 行：调用 `RpcContext#removeContext()` 方法，移除上下文。代码如下：

    ```Java
    public static void removeContext() {
        LOCAL.remove();
    }
    ```

# 5. RpcContext.values

我们在回过头来看 `RpcContext.values` 属性。目前 Dubbo 中，**并未使用它**。

从代码看下来，如果我们希望有**多次** Dubbo 调用，共享参数，并且不被 ConsumerContextFilter 清理隐式参数，笔者觉得可以使用该 `values` 属性。

和 `value` 属性相关的方法如下：

```Java
public Map<String, Object> get() {
    return values;
}

public RpcContext set(String key, Object value) {
    if (value == null) {
        values.remove(key);
    } else {
        values.put(key, value);
    }
    return this;
}   

public RpcContext remove(String key) {
    values.remove(key);
    return this;
} 

public Object get(String key) {
    return values.get(key);
}
```

当然，如果同时我们希望一些**通用**的 `values` 传递给被调用的服务，可以实现一个 Filter ，简化代码如下：

```Java
RpcContext.getContext().setAttachment("key1", RpcContext.getContext().get("key2").toString());
```

恩，还是当然，在业务代码里，也可以这么调用。🙂

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

美滋滋，梳理干净了。


