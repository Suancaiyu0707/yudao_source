title: 精尽 Dubbo 源码分析 —— 服务引用（二）之远程引用（Dubbo）
date: 2018-05-04
tags:
categories: Dubbo
permalink: Dubbo/reference-refer-dubbo

-------

摘要: 原创出处 http://www.iocoder.cn/Dubbo/reference-refer-dubbo/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Dubbo/reference-refer-dubbo/)
- [2. 远程引用](http://www.iocoder.cn/Dubbo/reference-refer-dubbo/)
- [3. Protocol](http://www.iocoder.cn/Dubbo/reference-refer-dubbo/)
  - [3.1 ProtocolFilterWrapper](http://www.iocoder.cn/Dubbo/reference-refer-dubbo/)
  - [3.2 RegistryProtocol](http://www.iocoder.cn/Dubbo/reference-refer-dubbo/)
  - [3.3 DubboProtocol](http://www.iocoder.cn/Dubbo/reference-refer-dubbo/)
- [4. Invoker](http://www.iocoder.cn/Dubbo/reference-refer-dubbo/)
  - [4.1 DubboInvoker](http://www.iocoder.cn/Dubbo/reference-refer-dubbo/)
- [666. 彩蛋](http://www.iocoder.cn/Dubbo/reference-refer-dubbo/)

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

在 [《精尽 Dubbo 源码分析 —— 服务引用（一）之本地引用（Injvm）》](http://www.iocoder.cn/Dubbo/reference-refer-local/?self) 一文中，我们已经分享了**本地引用服务**。在本文中，我们来分享**远程引用服务**。在 Dubbo 中提供多种协议( Protocol ) 的实现，大体流程一致，本文以 [Dubbo Protocol](https://dubbo.gitbooks.io/dubbo-user-book/references/protocol/dubbo.html) 为例子，这也是 Dubbo 的**默认**协议。

如果不熟悉该协议的同学，可以先看看 [《Dubbo 使用指南 —— dubbo://》](https://dubbo.gitbooks.io/dubbo-user-book/references/protocol/dubbo.html) ，简单了解即可。

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

![远程引用顺序图](http://www.iocoder.cn/images/Dubbo/2018_05_04/02.png)

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

**服务引用与暴露的 Protocol 很多类似点**，本文就不重复叙述了。

建议不熟悉的胖友，请点击 [《精尽 Dubbo 源码分析 —— 服务暴露（一）之本地暴露（Injvm）》「3. Protocol」](http://www.iocoder.cn/Dubbo/service-export-local/?self) 查看。

本文涉及的 Protocol 类图如下：

![Protocol 类图](http://www.iocoder.cn/images/Dubbo/2018_05_04/03.png)

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
 15:     // 分组聚合，参见文档 https://dubbo.gitbooks.io/dubbo-user-book/demos/group-merger.html
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

* 第 3 行：获得**真实**的注册中心的 URL 。该过程是我们在 [《精尽 Dubbo 源码分析 —— 服务暴露（一）之本地暴露（Injvm）》「2.1 loadRegistries」](#) 的那张图的反向流程，即**红线部分** ：![getRegistryUrl](http://www.iocoder.cn/images/Dubbo/2018_03_10/01.png)
* 第 5 行：获得注册中心 Registry 对象。
* 第 7至 9 行：【TODO 8018】RegistryService.class
* 第 13 行：获得服务引用配置参数集合 `qs` 。
* 第 16 至 22 行：分组聚合，参见 [《Dubbo 用户指南 —— 分组聚合》](https://dubbo.gitbooks.io/dubbo-user-book/demos/group-merger.html) 文档。
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
 20:     // 向注册中心注册自己（服务消费者）
 21:     if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
 22:             && url.getParameter(Constants.REGISTER_KEY, true)) {
 23:         registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
 24:                 Constants.CHECK_KEY, String.valueOf(false)));
 25:     }
 26:     // 向注册中心订阅服务提供者
 27:     directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
 28:             Constants.PROVIDERS_CATEGORY
 29:                     + "," + Constants.CONFIGURATORS_CATEGORY
 30:                     + "," + Constants.ROUTERS_CATEGORY));
 31: 
 32:     // 创建 Invoker 对象，【TODO 8015】集群容错
 33:     Invoker invoker = cluster.join(directory);
 34:     // 向本地注册表，注册消费者
 35:     ProviderConsumerRegTable.registerConsuemr(invoker, url, subscribeUrl, directory);
 36:     return invoker;
 37: }
```

* 第 12 至 15 行，创建 RegistryDirectory 对象，并设置注册中心到它的属性。
* 第 18 行：获得服务引用配置集合 `parameters` 。**注意**，`url` 传入 RegistryDirectory 后，经过处理并重新创建，所以 `url != directory.url` ，所以获得的是服务引用配置集合。如下图所示：![parameters](http://www.iocoder.cn/images/Dubbo/2018_05_04/01.png)
* 第 19 行：创建订阅 URL 对象。
* 第 20 至 25 行：调用 `RegistryService#register(url)` 方法，向注册中心注册**自己**（服务消费者）。
    * 在 [《精尽 Dubbo 源码分析 —— 注册中心（一）之抽象 API》「3. RegistryService」 ](http://www.iocoder.cn/Dubbo/registry-api/?self) ，有详细解析。
* 第 26 终 30 行：调用 `Directory#subscribe(url)` 方法，向注册中心订阅服务提供者 + 路由规则 + 配置规则。
    * 在该方法中，会循环获得到的服务体用这列表，调用 `Protocol#refer(type, url)` 方法，创建每个调用服务的 Invoker 对象。
* 第 33 行：创建 Invoker 对象，【TODO 8015】集群容错
* 第 35 行：调用 `ProviderConsumerRegTable#registerConsuemr(invoker, url, subscribeUrl, directory)` 方法，向本地注册表，注册消费者。
    * 在 [《精尽 Dubbo 源码分析 —— 注册中心（一）之抽象 API》「5. ProviderConsumerRegTable」 ](http://www.iocoder.cn/Dubbo/registry-api/?self) ，有详细解析。
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
  2:     // 初始化序列化优化器
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
* 第 3 行：调用 `#optimizeSerialization(url)` 方法，初始化序列化优化器。在 [《精尽 Dubbo 源码分析 —— 序列化（一）之总体实现》](http://www.iocoder.cn/Dubbo/serialize-1-all?self) 中，详细解析。
* 第 7 行：调用 `#getClients(url)` 方法，创建远程通信客户端数组。
* 第 7 行：创建 DubboInvoker 对象。
* 第 9 行：添加到 `invokers` 。
* 第 10 行：返回 Invoker 对象。

### 3.3.2 getClients

> 友情提示，涉及 Client 的内容，胖友先看过 [《精尽 Dubbo 源码分析 —— NIO 服务器》](http://www.iocoder.cn/Dubbo/remoting-api-interface/?self)  所有的文章。

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
 18:     // 创建连接服务提供者的 ExchangeClient 对象数组
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
* `connections` 配置项。
    * 默认 0 。即，对同一个远程服务器，**共用**同一个连接。
    * 大于 0 。即，每个服务引用，**独立**每一个连接。
    * [《Dubbo 用户指南 —— 连接控制》](https://dubbo.gitbooks.io/dubbo-user-book/demos/config-connections.html)
    * [《Dubbo 用户指南 —— dubbo:reference》](https://dubbo.gitbooks.io/dubbo-user-book/references/xml/dubbo-reference.html)

### 3.3.3 getSharedClient

`#getClients(url)` 方法，获得连接服务提供者的远程通信客户端数组。代码如下：

```Java
/**
 * 通信客户端集合
 *
 * key: 服务器地址。格式为：host:port
 */
private final Map<String, ReferenceCountExchangeClient> referenceClientMap = new ConcurrentHashMap<String, ReferenceCountExchangeClient>(); // <host:port,Exchanger>
/**
 * TODO 8030 ，这个是什么用途啊。
 *
 * key: 服务器地址。格式为：host:port 。和 {@link #referenceClientMap} Key ，是一致的。
 */
private final ConcurrentMap<String, LazyConnectExchangeClient> ghostClientMap = new ConcurrentHashMap<String, LazyConnectExchangeClient>();

  1: private ExchangeClient getSharedClient(URL url) {
  2:     // 从集合中，查找 ReferenceCountExchangeClient 对象
  3:     String key = url.getAddress();
  4:     ReferenceCountExchangeClient client = referenceClientMap.get(key);
  5:     if (client != null) {
  6:         // 若未关闭，增加指向该 Client 的数量，并返回它
  7:         if (!client.isClosed()) {
  8:             client.incrementAndGetCount();
  9:             return client;
 10:         // 若已关闭，移除
 11:         } else {
 12:             referenceClientMap.remove(key);
 13:         }
 14:     }
 15:     // 同步，创建 ExchangeClient 对象。
 16:     synchronized (key.intern()) {
 17:         // 创建 ExchangeClient 对象
 18:         ExchangeClient exchangeClient = initClient(url);
 19:         // 将 `exchangeClient` 包装，创建 ReferenceCountExchangeClient 对象
 20:         client = new ReferenceCountExchangeClient(exchangeClient, ghostClientMap);
 21:         // 添加到集合
 22:         referenceClientMap.put(key, client);
 23:         // 添加到 `ghostClientMap`
 24:         ghostClientMap.remove(key);
 25:         return client;
 26:     }
 27: }
```

* `referenceClientMap` 属性，通信客户端集合。在我们创建好 Client 对象，“**连接**”服务器后，会添加到这个集合中，用于后续的 Client 的**共享**。
    *  ReferenceCountExchangeClient ，顾名思义，带有指向数量计数的 Client 封装。
    *  “**连接**” ，打引号的原因，因为有 LazyConnectExchangeClient ，还是顾名思义，延迟连接的 Client 封装。
    *  🙂 ReferenceCountExchangeClient 和 LazyConnectExchangeClient 的具体实现，在 [「5. Client」](#) 详细解析。
* `ghostClientMap` 属性，幽灵客户端集合。TODO 8030 ，这个是什么用途啊。
    * 【添加】每次 ReferenceCountExchangeClient **彻底**关闭( 指向归零 ) ，其内部的 `client` 会替换成**重新创建**的 LazyConnectExchangeClient 对象，此时叫这个对象为**幽灵客户端**，添加到 `ghostClientMap` 中。
    * 【移除】当幽灵客户端，对应的 URL 的服务器被重新连接上后，会被移除。
    * **注意**，在幽灵客户端**被移除之前**，`referenceClientMap` 中，依然保留着对应的 URL 的 ReferenceCountExchangeClient 对象。所以，`ghostClientMap` 相当于标记 `referenceClientMap` 中，哪些 LazyConnectExchangeClient 对象，是**幽灵**状态。👻
* 第 2 至 4 行：从集合 `referenceClientMap` 中，查找 ReferenceCountExchangeClient 对象。
* 第 5 至 14 行：查找到客户端。
    * 第 6 至 9 行：若**未关闭**，调用 `ReferenceCountExchangeClient#incrementAndGetCount()`  方法，增加指向该客户端的数量，并返回。
    * 第 11 至 13 行：若**已关闭**，适用于**幽灵**状态的 ReferenceCountExchangeClient 对象，从 `referenceClientMap` 中移除，准备下面的代码，创建**新的** ReferenceCountExchangeClient 对象。
* 第 15 至 26 行：**同步**( `synchronized` ) ，创建新的 ReferenceCountExchangeClient 对象。
    * 第 18 行：调用 `#initClient(url)`  方法，创建 ExchangeClient 对象。
    * 第 20 行：将 ExchangeClient 对象，封装创建成 ReferenceCountExchangeClient 独享。
    * 第 22 行：添加到集合 `referenceClientMap` 。
    * 第 24 行：移除出集合 `ghostClientMap` ，因为不再是**幽灵**状态啦。

### 3.3.4 initClient

`#initClient(url)` 方法，创建 ExchangeClient 对象，"连接"服务器。

```Java
  1: private ExchangeClient initClient(URL url) {
  2:     // 校验 Client 的 Dubbo SPI 拓展是否存在
  3:     // client type setting.
  4:     String str = url.getParameter(Constants.CLIENT_KEY, url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_CLIENT));
  5:     // BIO is not allowed since it has severe performance issue.
  6:     if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
  7:         throw new RpcException("Unsupported client type: " + str + "," +
  8:                 " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
  9:     }
 10: 
 11:     // 设置编解码器为 Dubbo ，即 DubboCountCodec
 12:     url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
 13: 
 14:     // 默认开启 heartbeat
 15:     // enable heartbeat by default
 16:     url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
 17: 
 18:     // 连接服务器，创建客户端
 19:     ExchangeClient client;
 20:     try {
 21:         // 懒连接，创建 LazyConnectExchangeClient 对象
 22:         // connection should be lazy
 23:         if (url.getParameter(Constants.LAZY_CONNECT_KEY, false)) {
 24:             client = new LazyConnectExchangeClient(url, requestHandler);
 25:         // 直接连接，创建 HeaderExchangeClient 对象
 26:         } else {
 27:             client = Exchangers.connect(url, requestHandler);
 28:         }
 29:     } catch (RemotingException e) {
 30:         throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
 31:     }
 32:     return client;
 33: }
```

* 第 2 至 9 行：校验配置的 Client 的 Dubbo SPI 拓展是否存在。若不存在，抛出 RpcException 异常。 
* 第 12 行：设置编解码器为 `"Dubbo"` 协议，即 DubboCountCodec 。
* 第 16 行：默认开启**心跳**功能。
* 第 19 至 31 行：连接服务器，创建客户端。
    * 第 21 至 24 行：**懒加载**，创建 LazyConnectExchangeClient 对象。
    * 第 25 至 28 行：**直接连接**，创建 HeaderExchangeClient 对象。

# 4. Invoker

本文涉及的 Invoker 类图如下：

![Invoker 类图](http://www.iocoder.cn/images/Dubbo/2018_05_04/04.png)

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

# 5. Client

> 友情提示，涉及 Client 的内容，胖友先看过 [《精尽 Dubbo 源码分析 —— NIO 服务器》](http://www.iocoder.cn/Dubbo/remoting-api-interface/?self)  所有的文章。

## 5.1 ReferenceCountExchangeClient

[`com.alibaba.dubbo.rpc.protocol.dubbo.ReferenceCountExchangeClient`](https://github.com/YunaiV/dubbo/blob/master/dubbo-rpc/dubbo-rpc-default/src/main/java/com/alibaba/dubbo/rpc/protocol/dubbo/ReferenceCountExchangeClient.java)  ，实现 ExchangeClient 接口，**支持指向计数**的信息交换客户端实现类。

**构造方法**

```Java
  1: /**
  2:  * URL
  3:  */
  4: private final URL url;
  5: /**
  6:  * 指向数量
  7:  */
  8: private final AtomicInteger refenceCount = new AtomicInteger(0);
  9: /**
 10:  * 幽灵客户端集合
 11:  */
 12: private final ConcurrentMap<String, LazyConnectExchangeClient> ghostClientMap;
 13: /**
 14:  * 客户端
 15:  */
 16: private ExchangeClient client;
 17: 
 18: public ReferenceCountExchangeClient(ExchangeClient client, ConcurrentMap<String, LazyConnectExchangeClient> ghostClientMap) {
 19:     this.client = client;
 20:     // 指向加一
 21:     refenceCount.incrementAndGet();
 22:     this.url = client.getUrl();
 23:     if (ghostClientMap == null) {
 24:         throw new IllegalStateException("ghostClientMap can not be null, url: " + url);
 25:     }
 26:     this.ghostClientMap = ghostClientMap;
 27: }
```

* `refenceCount` 属性，指向计数。
    * 【初始】构造方法，【第 21 行】，计数加一。
    * 【引用】每次引用，计数加一。
* `ghostClientMap` 属性，幽灵客户端集合，和 `Protocol.ghostClientMap` 参数，一致。
* `client` 属性，客户端。
    * 【创建】构造方法，传入 `client` 属性，指向它。
    * 【关闭】关闭方法，创建 LazyConnectExchangeClient 对象，指向该幽灵客户端。

**装饰器模式**

基于**装饰器模式**，所以，每个实现方法，都是调用 `client` 的对应的方法。例如：

```Java
@Override
public void send(Object message) throws RemotingException {
    client.send(message);
}
```

**计数**

```Java
public void incrementAndGetCount() {
    refenceCount.incrementAndGet();
}
```

**关闭**

```Java
  1: @Override
  2: public void close(int timeout) {
  3:     if (refenceCount.decrementAndGet() <= 0) {
  4:         // 关闭 `client`
  5:         if (timeout == 0) {
  6:             client.close();
  7:         } else {
  8:             client.close(timeout);
  9:         }
 10:         // 替换 `client` 为 LazyConnectExchangeClient 对象。
 11:         client = replaceWithLazyClient();
 12:     }
 13: }
```

* 第 3 行：计数**减一**。若无指向，进行真正的关闭。
* 第 4 至 9 行：调用 `client` 的关闭方法，进行关闭。
* 第 11 行：调用 `#replaceWithLazyClient()` 方法，替换 `client` 为 LazyConnectExchangeClient 对象。代码如下：

    ```Java
      1: private LazyConnectExchangeClient replaceWithLazyClient() {
      2:     // this is a defensive operation to avoid client is closed by accident, the initial state of the client is false
      3:     URL lazyUrl = url.addParameter(Constants.LAZY_CONNECT_INITIAL_STATE_KEY, Boolean.FALSE)
      4:             .addParameter(Constants.RECONNECT_KEY, Boolean.FALSE) // 不重连
      5:             .addParameter(Constants.SEND_RECONNECT_KEY, Boolean.TRUE.toString())
      6:             .addParameter("warning", Boolean.TRUE.toString())
      7:             .addParameter(LazyConnectExchangeClient.REQUEST_WITH_WARNING_KEY, true)
      8:             .addParameter("_client_memo", "referencecounthandler.replacewithlazyclient"); // 备注
      9: 
     10:     // 创建 LazyConnectExchangeClient 对象，若不存在。
     11:     String key = url.getAddress();
     12:     // in worst case there's only one ghost connection.
     13:     LazyConnectExchangeClient gclient = ghostClientMap.get(key);
     14:     if (gclient == null || gclient.isClosed()) {
     15:         gclient = new LazyConnectExchangeClient(lazyUrl, client.getExchangeHandler());
     16:         ghostClientMap.put(key, gclient);
     17:     }
     18:     return gclient;
     19: }
    ```
    * 第 3 至 8 行：基于 `url` ，创建 LazyConnectExchangeClient 的 URL 链接。设置的一些参数，结合 [「5.2 LazyConnectExchangeClient」](#) 一起看。
    * 第 10 至 17 行：创建 LazyConnectExchangeClient 对象，若不存在。

## 5.2 LazyConnectExchangeClient

[`com.alibaba.dubbo.rpc.protocol.dubbo.LazyConnectExchangeClient`](https://github.com/YunaiV/dubbo/blob/master/dubbo-rpc/dubbo-rpc-default/src/main/java/com/alibaba/dubbo/rpc/protocol/dubbo/LazyConnectExchangeClient.java)  ，实现 ExchangeClient 接口，**支持懒连接服务器**的信息交换客户端实现类。

**构造方法**

```Java
  1: static final String REQUEST_WITH_WARNING_KEY = "lazyclient_request_with_warning";
  2: 
  3: /**
  4:  * URL
  5:  */
  6: private final URL url;
  7: /**
  8:  * 通道处理器
  9:  */
 10: private final ExchangeHandler requestHandler;
 11: /**
 12:  * 连接锁
 13:  */
 14: private final Lock connectLock = new ReentrantLock();
 15: /**
 16:  * lazy connect 如果没有初始化时的连接状态
 17:  */
 18: // lazy connect, initial state for connection
 19: private final boolean initialState;
 20: /**
 21:  * 通信客户端
 22:  */
 23: private volatile ExchangeClient client;
 24: /**
 25:  * 请求时，是否检查告警
 26:  */
 27: protected final boolean requestWithWarning;
 28: /**
 29:  * 警告计数器。每超过一定次数，打印告警日志。参见 {@link #warning(Object)}
 30:  */
 31: private AtomicLong warningcount = new AtomicLong(0);
 32: 
 33: public LazyConnectExchangeClient(URL url, ExchangeHandler requestHandler) {
 34:     // lazy connect, need set send.reconnect = true, to avoid channel bad status.
 35:     this.url = url.addParameter(Constants.SEND_RECONNECT_KEY, Boolean.TRUE.toString());
 36:     this.requestHandler = requestHandler;
 37:     this.initialState = url.getParameter(Constants.LAZY_CONNECT_INITIAL_STATE_KEY, Constants.DEFAULT_LAZY_CONNECT_INITIAL_STATE);
 38:     this.requestWithWarning = url.getParameter(REQUEST_WITH_WARNING_KEY, false);
 39: }
```

* `initialState` 属性，如果没有初始化客户端时的链接状态。有点绕，看 `#isConnected()` 方法，代码如下：

    ```Java
    @Override
    public boolean isConnected() {
        if (client == null) { // 客户端未初始化
            return initialState;
        } else {
            return client.isConnected();
        }
    }
    ```
    * 所以，我们可以看到 ReferenceCountExchangeClient 关闭创建的 LazyConnectExchangeClient  对象的 `initialState = false` ，未连接。
    * **默认值**，`DEFAULT_LAZY_CONNECT_INITIAL_STATE = true` 。

* `requestWithWarning` 属性，请求时，是否检查告警。
    * 所以，我们可以看到 ReferenceCountExchangeClient 关闭创建的 LazyConnectExchangeClient  对象的 `initialState = false` ，未连接。
    * **默认值**，`false` 。
* `warningcount` 属性，警告计数器。每超过一定次数，打印告警日志。每次发送请求时，会调用 `#warning(request)` 方法，根据情况，打印告警日志。代码如下：

    ```Java
    private void warning(Object request) {
        if (requestWithWarning) { // 开启
            if (warningcount.get() % 5000 == 0) { // 5000 次
                logger.warn(new IllegalStateException("safe guard client , should not be called ,must have a bug."));
            }
            warningcount.incrementAndGet(); // 增加计数
        }
    }
    ```
    * 理论来说，不会被调用。如果被调用，那么就是一个 BUG 咯。

**装饰器模式**

基于**装饰器模式**，所以，每个实现方法，都是调用 `client` 的对应的方法。例如：

```Java
@Override
@Override
public void close(int timeout) {
    if (client != null)
        client.close(timeout);
}
```

**初始化客户端**

```Java
private void initClient() throws RemotingException {
    // 已初始化，跳过
    if (client != null) {
        return;
    }
    if (logger.isInfoEnabled()) {
        logger.info("Lazy connect to " + url);
    }
    // 获得锁
    connectLock.lock();
    try {
        // 已初始化，跳过
        if (client != null) {
            return;
        }
        // 创建 Client ，连接服务器
        this.client = Exchangers.connect(url, requestHandler);
    } finally {
        // 释放锁
        connectLock.unlock();
    }
}
```

* 发送消息/请求前，都会调用该方法，保证客户端已经初始化。代码如下：

```Java
public void send(Object message, boolean sent) throws RemotingException {
    initClient();
    client.send(message, sent);
}

@Override
public ResponseFuture request(Object request, int timeout) throws RemotingException {
    warning(request);
    initClient();
    return client.request(request, timeout);
}
```

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

写的有点迷糊，主要是集群和注册中心，看的不是特别细致。

不过咋说呢？

读源码的过程，就像剥洋葱，一层一层拨开它的心。


