title: 精尽 Dubbo 源码分析 —— 服务引用（二）之远程引用（Dubbo）
date: 2018-05-04
tags:
categories: Dubbo
permalink: Dubbo/reference-refer-dubbo

-------

# 1. 概述

在 [《精尽 Dubbo 源码分析 —— 服务引用（一）之本地引用（Injvm）》](http://www.iocoder.cn/Dubbo/reference-refer-local/?self) 一文中，我们已经分享了**本地引用服务**。在本文中，我们来分享**远程引用服务**。在 Dubbo 中提供多种协议( Protocol ) 的实现，大体流程一致，本文以 [Dubbo Protocol](http://dubbo.io/books/dubbo-user-book/references/protocol/dubbo.html) 为例子，这也是 Dubbo 的**默认**协议。

如果不熟悉该协议的同学，可以先看看 [《Dubbo 使用指南 —— dubbo://》](http://dubbo.io/books/dubbo-user-book/references/protocol/dubbo.html) ，简单了解即可。

> **特性**
> 
> 缺省协议，使用基于 mina `1.1.7` 和 hessian `3.2.1` 的 remoting 交互。
> 
> * 连接个数：单连接
> * 连接方式：长连接
> * 传输协议：TCP
> * 传输方式：NIO 异步传输
> * 序列化：Hessian 二进制序列化
> * 适用范围：传入传出参数数据包较小（建议小于100K），消费者比提供者个数多，单一消费者无法压满提供者，尽量不要用 dubbo 协议传输大文件或超大字符串。
> * 适用场景：常规远程服务方法调用

相比**本地引用**，**远程引用**会多做如下几件事情：

* 向注册中心**订阅**，从而**发现**服务提供者列表。
* 启动通信客户端，通过它进行**远程调用**。

# 2. 远程引用

远程暴露服务的顺序图如下：

TODO [远程引用顺序图](http://www.iocoder.cn/images/Dubbo/2018_03_10/02.png)

在 [`#createProxy(map)`](https://github.com/YunaiV/dubbo/blob/c635dd1990a1803643194048f408db310f06175b/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L621-L648) 方法中，涉及**远程引用服务**的代码如下：

```Java
/**
 * 服务引用 URL 数组
 */
private final List<URL> urls = new ArrayList<URL>();
/**
 * 直连服务地址
 *
 * 1. 可以是注册中心，也可以是服务提供者
 * 2. 可配置多个，使用 ; 分隔
 */
// url for peer-to-peer invocation
private String url;

  1: /**
  2:  * 创建 Service 代理对象
  3:  *
  4:  * @param map 集合
  5:  * @return 代理对象
  6:  */
  7: @SuppressWarnings({"unchecked", "rawtypes", "deprecation"})
  8: private T createProxy(Map<String, String> map) {
  9:     URL tmpUrl = new URL("temp", "localhost", 0, map);
 10:     // 【省略代码】是否本地引用
 11:     final boolean isJvmRefer;
 12: 
 13:     // 【省略代码】本地引用
 14:     if (isJvmRefer) {
 15:     // 正常流程，一般为远程引用
 16:     } else {
 17:         // 定义直连地址，可以是服务提供者的地址，也可以是注册中心的地址
 18:         if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
 19:             // 拆分地址成数组，使用 ";" 分隔。
 20:             String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
 21:             // 循环数组，添加到 `url` 中。
 22:             if (us != null && us.length > 0) {
 23:                 for (String u : us) {
 24:                     // 创建 URL 对象
 25:                     URL url = URL.valueOf(u);
 26:                     // 设置默认路径
 27:                     if (url.getPath() == null || url.getPath().length() == 0) {
 28:                         url = url.setPath(interfaceName);
 29:                     }
 30:                     // 注册中心的地址，带上服务引用的配置参数
 31:                     if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
 32:                         urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
 33:                     // 服务提供者的地址
 34:                     } else {
 35:                         urls.add(ClusterUtils.mergeUrl(url, map));
 36:                     }
 37:                 }
 38:             }
 39:         // 注册中心
 40:         } else { // assemble URL from register center's configuration
 41:             // 加载注册中心 URL 数组
 42:             List<URL> us = loadRegistries(false);
 43:             // 循环数组，添加到 `url` 中。
 44:             if (us != null && !us.isEmpty()) {
 45:                 for (URL u : us) {
 46:                     // 加载监控中心 URL
 47:                     URL monitorUrl = loadMonitor(u);
 48:                     // 服务引用配置对象 `map`，带上监控中心的 URL
 49:                     if (monitorUrl != null) {
 50:                         map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
 51:                     }
 52:                     // 注册中心的地址，带上服务引用的配置参数
 53:                     urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map))); // 注册中心，带上服务引用的配置参数
 54:                 }
 55:             }
 56:             if (urls == null || urls.isEmpty()) {
 57:                 throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
 58:             }
 59:         }
 60: 
 61:         // 单 `urls` 时，引用服务，返回 Invoker 对象
 62:         if (urls.size() == 1) {
 63:             // 引用服务
 64:             invoker = refprotocol.refer(interfaceClass, urls.get(0));
 65:         } else {
 66:             // 循环 `urls` ，引用服务，返回 Invoker 对象
 67:             List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
 68:             URL registryURL = null;
 69:             for (URL url : urls) {
 70:                 // 引用服务
 71:                 invokers.add(refprotocol.refer(interfaceClass, url));
 72:                 // 使用最后一个注册中心的 URL
 73:                 if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
 74:                     registryURL = url; // use last registry url
 75:                 }
 76:             }
 77:             // 有注册中心
 78:             if (registryURL != null) { // registry url is available
 79:                 // 对有注册中心的 Cluster 只用 AvailableCluster
 80:                 // use AvailableCluster only when register's cluster is available
 81:                 URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
 82:                 // TODO 芋艿
 83:                 invoker = cluster.join(new StaticDirectory(u, invokers));
 84:             // 无注册中心
 85:             } else { // not a registry url
 86:                 // TODO 芋艿
 87:                 invoker = cluster.join(new StaticDirectory(invokers));
 88:             }
 89:         }
 90:     }
 91: 
 92:     // 【省略代码】启动时检查
 93: 
 94:     // 创建 Service 代理对象
 95:     // create service proxy
 96:     return (T) proxyFactory.getProxy(invoker);
 97: }
```

* 第 11 行：省略**是否本地引用**的代码，在 [《精尽 Dubbo 源码分析 —— 服务引用（一）之本地引用（Injvm）》](http://www.iocoder.cn/Dubbo/reference-refer-local/?self) 已经有分享。
* 第 13 至 15 行：省略**本地引用**的代码，在 [《精尽 Dubbo 源码分析 —— 服务引用（一）之本地引用（Injvm）》](http://www.iocoder.cn/Dubbo/reference-refer-local/?self) 已经有分享。
* 第 16 至 90 行：正常流程，一般为远程引用。
* 第 18 至 38 行：`url` 配置项，**定义直连地址**，可以是服务提供者的地址，也可以是注册中心的地址。
    * 第 20 行：拆分地址成数组，使用 ";" 分隔。
    * 第 22 至 23 行：循环数组 `us` ，创建 URL 对象后，添加到 `urls` 中。
    * 第 25 行：创建 URL 对象。
    * 第 26 至 29 行：路径属性 `url.path` 未设置时，缺省使用接口全名 `interfaceName` 。
    * 第 30 至 32 行：若 `url.protocol = registry` 时，**注册中心的地址**，在参数 `url.parameters.refer` 上，设置上服务引用的配置参数集合 `map` 。
    * 第 33 至 36 行：**服务提供者的地址**。
        * 从逻辑上类似【第 53 行】的代码。
        * 一般情况下，不建议这样在 `url` 配置注册中心，而是在 `registry` 配置。如果要配置，格式为 `registry://host:port?registry=` ，例如 `registry://127.0.0.1?registry=zookeeper` 。
        * TODO ClusterUtils.mergeUrl
* 第 39 至 59 行：`protocol` 配置项，**注册中心**。
    * 第 42 行：调用 `#loadRegistries(provider)` 方法，加载注册中心的 com.alibaba.dubbo.common.URL` 数组。
        * 🙂 在 [《精尽 Dubbo 源码分析 —— 服务暴露（一）之本地暴露（Injvm）》「2.1 loadRegistries」](http://www.iocoder.cn/Dubbo/service-export-local/?self) 详细解析。 
    * 第 43 至 58 行：循环数组 `us` ，创建 URL 对象后，添加到 `urls` 中。
        * 第 47 行：调用 `#loadMonitor(registryURL)` 方法，获得监控中心 URL 。
            * 🙂 在 [《精尽 Dubbo 源码分析 —— 服务暴露（二）之远程暴露（Dubbo）》「2.1 loadRegistries」](#) 小节，详细解析。
        * 第 49 至 51 行：服务引用配置对象 `map`，带上监控中心的 URL 。具体用途，我们在后面分享监控中心会看到。
        * 第 53 行：调用 [`URL#addParameterAndEncoded(key, value)`](https://github.com/YunaiV/dubbo/blob/c635dd1990a1803643194048f408db310f06175b/dubbo-common/src/main/java/com/alibaba/dubbo/common/URL.java#L891-L896) 方法，将服务引用配置对象参数集合 `map` ，作为 `"refer"` 参数添加到注册中心的 URL 中，**并且需要编码**。通过这样的方式，注册中心的 URL 中，**包含了服务引用的配置**。
* 第 61 至 64 行：单 `urls` 时，**直接调用** `Protocol#refer(type, url)` 方法，引用服务，返回 Invoker 对象。
    * 此处 Dubbo SPI **自适应**的特性的**好处**就出来了，可以**自动**根据 URL 参数，获得对应的拓展实现。例如，`invoker` 传入后，根据 `invoker.url` 自动获得对应 Protocol 拓展实现为 DubboProtocol 。
    * 实际上，Protocol 有两个 Wrapper 拓展实现类： ProtocolFilterWrapper、ProtocolListenerWrapper 。所以，`#export(...)` 方法的调用顺序是：
        * **Protocol$Adaptive => ProtocolFilterWrapper => ProtocolListenerWrapper => RegistryProtocol**
        * => 
        * **Protocol$Adaptive => ProtocolFilterWrapper => ProtocolListenerWrapper => DubboProtocol**
        * 也就是说，**这一条大的调用链，包含两条小的调用链**。原因是：
            * 首先，传入的是注册中心的 URL ，通过 Protocol$Adaptive 获取到的是 RegistryProtocol 对象。
            * 其次，RegistryProtocol 会在其 `#refer(...)` 方法中，使用服务提供者的 URL ( 即注册中心的 URL 的 `refer` 参数值)，再次调用 Protocol$Adaptive 获取到的是 DubboProtocol 对象，进行服务暴露。
        * **为什么是这样的顺序**？通过这样的顺序，可以实现类似 **AOP** 的效果，在获取服务提供者列表后，再创建连接服务提供者的客户端。伪代码如下：

            ```Java
            RegistryProtocol#refer(...) {
                
                // 1. 获取服务提供者列表 【并且订阅】
                
                // 2. 创建调用连接服务提供者的客户端 
                DubboProtocol#refer(...);
                
                // ps：实际这个过程中，还有别的代码，详细见下文。
            }
            ```
            * x

* 第 65 至 89 行：多 `urls` 时，**循环调用** `Protocol#refer(type, url)` 方法，引用服务，返回 Invoker 对象。此时，会有多个 Invoker 对象，需要进行合并。
    * 什么时候会出现多个 `urls` 呢？例如：[《Dubbo 用户指南 —— 多注册中心注册》](https://dubbo.gitbooks.io/dubbo-user-book/demos/multi-registry.html) 。
    * 第 66 至 76 行：循环 `urls` ，引用服务。
        * 第 71 行：调用 `Protocol#refer(type, url)` 方法，引用服务，返回 Invoker 对象。然后，添加到 `invokers` 中。
        * 第 72 会 75 行：使用最后一个注册中心的 URL ，赋值到 `registryURL` 。
    * 第 77 至 88 行：【TODO 8017】集群容错
* 第 92 行：省略**启动时检查**的代码，在 [《精尽 Dubbo 源码分析 —— 服务引用（一）之本地引用（Injvm）》](http://www.iocoder.cn/Dubbo/reference-refer-local/?self) 已经有分享。
* 第 96 行：省略**创建 Service 代理对象**的代码，在 [《精尽 Dubbo 源码分析 —— 服务引用（一）之本地引用（Injvm）》](http://www.iocoder.cn/Dubbo/reference-refer-local/?self) 已经有分享。

# 3. Protocol

## 3.1 ProtocolFilterWrapper

接 [《精尽 Dubbo 源码分析 —— 服务引用（一）之本地引用（Injvm）》「 3.1 ProtocolFilterWrapper」](http://www.iocoder.cn/Dubbo/service-reference-local/?self) 小节。

本文涉及的 `#refer(type, url)` 方法，代码如下：

```Java
  1: public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
  2:     // 注册中心
  3:     if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
  4:         return protocol.refer(type, url);
  5:     }
  6:     // 引用服务，返回 Invoker 对象
  7:     // 给改 Invoker 对象，包装成带有 Filter 过滤链的 Invoker 对象
  8:     return buildInvokerChain(protocol.refer(type, url), Constants.REFERENCE_FILTER_KEY, Constants.CONSUMER);
  9: }
```

* 第 2 至 5 行：当 `invoker.url.protocl = registry` ，**注册中心的 URL** ，无需创建 Filter 过滤链。 
* 第 8 行：调用 `protocol#refer(type, url)` 方法，继续引用服务，最终返回 Invoker 。
* 第 8 行：在引用服务完成后，调用 `#buildInvokerChain(invoker, key, group)` 方法，创建带有 Filter 过滤链的 Invoker 对象。

## 3.2 RegistryProtocol

### 3.2.1 refer

本文涉及的 `#refer(type, url)` 方法，代码如下：

```Java
/**
 * Cluster 自适应拓展实现类对象
 */
private Cluster cluster;

  1: public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
  2:     // 获得真实的注册中心的 URL
  3:     url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
  4:     // 获得注册中心
  5:     Registry registry = registryFactory.getRegistry(url);
  6:     // TODO 芋艿
  7:     if (RegistryService.class.equals(type)) {
  8:         return proxyFactory.getInvoker((T) registry, type, url);
  9:     }
 10: 
 11:     // 获得服务引用配置参数集合
 12:     // group="a,b" or group="*"
 13:     Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
 14:     String group = qs.get(Constants.GROUP_KEY);
 15:     // 分组聚合，参见文档 http://dubbo.io/books/dubbo-user-book/demos/group-merger.html
 16:     if (group != null && group.length() > 0) {
 17:         if ((Constants.COMMA_SPLIT_PATTERN.split(group)).length > 1
 18:                 || "*".equals(group)) {
 19:             // 执行服务引用
 20:             return doRefer(getMergeableCluster(), registry, type, url);
 21:         }
 22:     }
 23:     // 执行服务引用
 24:     return doRefer(cluster, registry, type, url);
 25: }
```

* 第 3 行：获得**真实**的注册中心的 URL 。该过程是我们在 [《精尽 Dubbo 源码分析 —— 服务暴露（一）之本地暴露（Injvm）》「2.1 loadRegistries」](#) 的那张图的反向流程，即**红线部分** ：[getRegistryUrl](http://www.iocoder.cn/images/Dubbo/2018_03_10/01.png)
* 第 5 行：获得注册中心 Registry 对象。
* 第 7至 9 行：【TODO 8018】RegistryService.class
* 第 13 行：获得服务引用配置参数集合 `qs` 。
* 第 16 至 22 行：分组聚合，参见 [《Dubbo 用户指南 —— 分组聚合》](http://dubbo.io/books/dubbo-user-book/demos/group-merger.html) 文档。
* 第 24 行：调用 `#doRefer(cluster, registry, type, url)` 方法，执行服务引用。不同于【第 20 行】的代码，后者调用 `#getMergeableCluster()` 方法，获得**可合并的** Cluster 对象，代码如下：

    ```Java
    private Cluster getMergeableCluster() {
        return ExtensionLoader.getExtensionLoader(Cluster.class).getExtension("mergeable");
    }
    ```

### 3.2.2 doRefer

`#doRefer(cluster, registry, type, url)` 方法，执行服务引用的逻辑。代码如下：

```Java
  1: /**
  2:  * 执行服务引用，返回 Invoker 对象
  3:  *
  4:  * @param cluster Cluster 对象
  5:  * @param registry 注册中心对象
  6:  * @param type 服务接口类型
  7:  * @param url 注册中心 URL
  8:  * @param <T> 泛型
  9:  * @return Invoker 对象
 10:  */
 11: private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
 12:     // 创建 RegistryDirectory 对象，并设置注册中心
 13:     RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
 14:     directory.setRegistry(registry);
 15:     directory.setProtocol(protocol);
 16:     // 创建订阅 URL
 17:     // all attributes of REFER_KEY
 18:     Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters()); // 服务引用配置集合
 19:     URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);
 20:     // 向注册中心注册自己（服务消费者） 【TODO 8014】注册中心
 21:     if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
 22:             && url.getParameter(Constants.REGISTER_KEY, true)) {
 23:         registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
 24:                 Constants.CHECK_KEY, String.valueOf(false)));
 25:     }
 26:     // 向注册中心订阅服务提供者 【TODO 8014】注册中心
 27:     directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
 28:             Constants.PROVIDERS_CATEGORY
 29:                     + "," + Constants.CONFIGURATORS_CATEGORY
 30:                     + "," + Constants.ROUTERS_CATEGORY));
 31: 
 32:     // 创建 Invoker 对象，【TODO 8015】集群容错
 33:     Invoker invoker = cluster.join(directory);
 34:     // 【TODO 8014】注册中心
 35:     ProviderConsumerRegTable.registerConsuemr(invoker, url, subscribeUrl, directory);
 36:     return invoker;
 37: }
```

* 第 12 至 15 行，创建 RegistryDirectory 对象，并设置注册中心到它的属性。
* 第 18 行：获得服务引用配置集合 `parameters` 。**注意**，`url` 传入 RegistryDirectory 后，经过处理并重新创建，所以 `url != directory.url` ，所以获得的是服务引用配置集合。如下图所示：[parameters](http://www.iocoder.cn/images/Dubbo/2018_05_04/01.png)
* 第 19 行：创建订阅 URL 对象。
* 第 20 至 25 行：向注册中心注册**自己**（服务消费者）【TODO 8014】注册中心
* 第 26 终 30 行：向注册中心订阅服务提供者列表 TODO 8014】注册中心
    * 在该方法中，会循环获得到的服务体用这列表，调用 `Protocol#refer(type, url)` 方法，创建每个调用服务的 Invoker 对象。
* 第 33 行：创建 Invoker 对象，【TODO 8015】集群容错
* 第 35 行：【TODO 8014】注册中心
* 第 36 行：返回 Invoker 对象。

## 3.3 DubboProtocol

### 3.3.1 refer

本文涉及的 `#refer(type, url)` 方法，代码如下：

```Java
// AbstractProtocol.java 父类
/**
 * Invoker 集合
 */
//TODO SOFEREFENCE
protected final Set<Invoker<?>> invokers = new ConcurrentHashSet<Invoker<?>>();

// DubboProtocol.java

  1: public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
  2:     // TODO 【8013 】kryo fst
  3:     optimizeSerialization(url);
  4:     // 获得远程通信客户端数组
  5:     // 创建 DubboInvoker 对象
  6:     // create rpc invoker.
  7:     DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
  8:     // 添加到 `invokers`
  9:     invokers.add(invoker);
 10:     return invoker;
 11: }
```

* `invokers` 属性，Invoker 集合。
* 第 3 行：TODO 【8013 】kryo fst
* 第 7 行：调用 `#getClients(url)` 方法，创建远程通信客户端数组。
* 第 7 行：创建 DubboInvoker 对象。
* 第 9 行：添加到 `invokers` 。
* 第 10 行：返回 Invoker 对象。

### 3.3.2 getClients

`#getClients(url)` 方法，获得连接服务提供者的远程通信客户端数组。代码如下：

```Java
  1: /**
  2:  * 获得连接服务提供者的远程通信客户端数组
  3:  *
  4:  * @param url 服务提供者 URL
  5:  * @return 远程通信客户端
  6:  */
  7: private ExchangeClient[] getClients(URL url) {
  8:     // 是否共享连接
  9:     // whether to share connection
 10:     boolean service_share_connect = false;
 11:     int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
 12:     // if not configured, connection is shared, otherwise, one connection for one service
 13:     if (connections == 0) { // 未配置时，默认共享
 14:         service_share_connect = true;
 15:         connections = 1;
 16:     }
 17: 
 18:     // 创建连接服务提供者的 ExchangeClient 对象数组 【TODO 8016】
 19:     ExchangeClient[] clients = new ExchangeClient[connections];
 20:     for (int i = 0; i < clients.length; i++) {
 21:         if (service_share_connect) { // 共享
 22:             clients[i] = getSharedClient(url);
 23:         } else { // 不共享
 24:             clients[i] = initClient(url);
 25:         }
 26:     }
 27:     return clients;
 28: }
```

* 第 8 至 16 行：是否共享连接。
* 第 18 至 26 行：创建连接服务提供者的 ExchangeClient 对象数组。
    * **注意**，若开启共享连接，基于 URL 为维度共享。
    * 第 21 至 22 行：共享连接，调用 `#getSharedClient(url)` 方法，获得 ExchangeClient 对象。
    * 第 23 至 25 行：不共享连接，调用 `#initClient(url)` 方法，直接创建 ExchangeClient 对象。

### 3.3.3 getClient

【TODO 8016】

### 3.3.4 getSharedClient

【TODO 8016】

# 4. Invoker

## 4.1 DubboInvoker

[`com.alibaba.dubbo.rpc.protocol.dubbo.DubboInvoker`](https://github.com/YunaiV/dubbo/blob/8de6d56d06965a38712c46a0220f4e59213db72f/dubbo-rpc/dubbo-rpc-default/src/main/java/com/alibaba/dubbo/rpc/protocol/dubbo/DubboInvoker.java) ，实现 AbstractExporter 抽象类，Dubbo Invoker 实现类。代码如下：

```Java
  1: /**
  2:  * 远程通信客户端数组
  3:  */
  4: private final ExchangeClient[] clients;
  5: /**
  6:  * 使用的 {@link #clients} 的位置
  7:  */
  8: private final AtomicPositiveInteger index = new AtomicPositiveInteger();
  9: /**
 10:  * 版本
 11:  */
 12: private final String version;
 13: /**
 14:  * 销毁锁
 15:  *
 16:  * 在 {@link #destroy()} 中使用
 17:  */
 18: private final ReentrantLock destroyLock = new ReentrantLock();
 19: /**
 20:  * Invoker 集合，从 {@link DubboProtocol#invokers} 获取
 21:  */
 22: private final Set<Invoker<?>> invokers;
 23: 
 24: public DubboInvoker(Class<T> serviceType, URL url, ExchangeClient[] clients) {
 25:     this(serviceType, url, clients, null);
 26: }
 27: 
 28: public DubboInvoker(Class<T> serviceType, URL url, ExchangeClient[] clients, Set<Invoker<?>> invokers) {
 29:     super(serviceType, url, new String[]{Constants.INTERFACE_KEY, Constants.GROUP_KEY, Constants.TOKEN_KEY, Constants.TIMEOUT_KEY});
 30:     this.clients = clients;
 31:     // get version.
 32:     this.version = url.getParameter(Constants.VERSION_KEY, "0.0.0");
 33:     this.invokers = invokers;
 34: }
```

* 胖友，请看属性上的代码注释。
* 第 29 行：调用父类构造方法。该方法中，会将 `interface` `group` `version` `token` `timeout` 添加到公用的隐式传参 `AbstractInvoker.attachment` 属性。
    * 🙂 代码比较简单，胖友请自己阅读。 

# 666. 彩蛋


