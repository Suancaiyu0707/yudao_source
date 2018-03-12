title: 精尽 Dubbo 源码分析 —— 服务暴露（二）之远程暴露（Dubbo）
date: 2018-03-10
tags:
categories: Dubbo
permalink: Dubbo/service-export-remote-dubbo

-------

# 1. 概述

在 [《精尽 Dubbo 源码分析 —— 服务暴露（一）之本地暴露（Injvm）》](http://www.iocoder.cn/Dubbo/service-export-local/?self) 一文中，我们已经分享了**本地暴露服务**。在本文中，我们来分享**远程暴露服务**。在 Dubbo 中提供多种协议( Protocol ) 的实现，大体流程一致，本文以 [Dubbo Protocol](http://dubbo.io/books/dubbo-user-book/references/protocol/dubbo.html) 为例子，这也是 Dubbo 的**默认**协议。

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

相比**本地暴露**，**远程暴露**会多做如下几件事情：

* 启动本地服务器，绑定服务端口，提供远程调用。
* 向注册中心注册服务提供者，提供服务消费者从注册中心发现服务。

# 2. 远程暴露

远程暴露服务的顺序图如下：

TODO 芋艿，此处有一图。

在 [`#doExportUrlsFor1Protocol(protocolConfig, registryURLs)`](https://github.com/YunaiV/dubbo/blob/c635dd1990a1803643194048f408db310f06175b/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ServiceConfig.java#L621-L648) 方法中，涉及**远程暴露服务**的代码如下：

```Java
  1: // 服务远程暴露
  2: // export to remote if the config is not local (export to local only when config is local)
  3: if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
  4:     if (logger.isInfoEnabled()) {
  5:         logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
  6:     }
  7:     if (registryURLs != null && !registryURLs.isEmpty()) {
  8:         for (URL registryURL : registryURLs) {
  9:             // "dynamic" ：服务是否动态注册，如果设为false，注册后将显示后disable状态，需人工启用，并且服务提供者停止时，也不会自动取消册，需人工禁用。
 10:             url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
 11:             // 获得监控中心 URL
 12:             URL monitorUrl = loadMonitor(registryURL); // TODO 芋艿，监控
 13:             if (monitorUrl != null) {
 14:                 url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
 15:             }
 16:             if (logger.isInfoEnabled()) {
 17:                 logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
 18:             }
 19:             // 使用 ProxyFactory 创建 Invoker 对象
 20:             Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
 21: 
 22:             // 创建 DelegateProviderMetaDataInvoker 对象
 23:             DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
 24: 
 25:             // 使用 Protocol 暴露 Invoker 对象
 26:             Exporter<?> exporter = protocol.export(wrapperInvoker);
 27:             // 添加到 `exporters`
 28:             exporters.add(exporter);
 29:         }
 30:     } else { // 用于被服务消费者直连服务提供者，参见文档 http://dubbo.io/books/dubbo-user-book/demos/explicit-target.html 。主要用于开发测试环境使用。
 31:         // 使用 ProxyFactory 创建 Invoker 对象
 32:         Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
 33: 
 34:         // 创建 DelegateProviderMetaDataInvoker 对象
 35:         DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
 36: 
 37:         // 使用 Protocol 暴露 Invoker 对象
 38:         Exporter<?> exporter = protocol.export(wrapperInvoker);
 39:         // 添加到 `exporters`
 40:         exporters.add(exporter);
 41:     }
 42: }
```

* 第 30 至 41 行：**大体和【第 7 至 29 行】逻辑相同**。差别在于，当配置注册中心为 `"N/A"` 时，表示即使远程暴露服务，也不向注册中心注册。这种方式用于被服务消费者直连服务提供者，参见 [《Dubbo 用户指南 —— 直连提供者》](http://dubbo.io/books/dubbo-user-book/demos/explicit-target.html) 文档。

    > 在**开发及测试环境下**，经常需要绕过注册中心，只测试指定服务提供者，这时候可能需要点对点直连，点对点直联方式，将以服务接口为单位，忽略注册中心的提供者列表，A 接口配置点对点，不影响 B 接口从注册中心获取列表。

* 第 8 行：循环祖册中心 URL 数组 `registryURLs` 。
* 第 10 行：`"dynamic"` 配置项，服务是否动态注册。如果设为 **false** ，注册后将显示后 **disable** 状态，需人工启用，并且服务提供者停止时，也不会自动取消册，需人工禁用。
* 第 12 行：调用 `#loadMonitor(registryURL)` 方法，获得监控中心 URL 。
    * 🙂 在 [「2.1 loadMonitor」](#) 小节，详细解析。
* 第 13 至 15 行：调用 [`URL#addParameterAndEncoded(key, value)`](https://github.com/YunaiV/dubbo/blob/c635dd1990a1803643194048f408db310f06175b/dubbo-common/src/main/java/com/alibaba/dubbo/common/URL.java#L891-L896) 方法，将监控中心的 URL 作为 `"monitor"` 参数添加到服务提供者的 URL 中，**并且需要编码**。通过这样的方式，服务提供者的 URL 中，**包含了监控中心的配置**。
* 第 20 行：调用 [`URL#addParameterAndEncoded(key, value)`](https://github.com/YunaiV/dubbo/blob/c635dd1990a1803643194048f408db310f06175b/dubbo-common/src/main/java/com/alibaba/dubbo/common/URL.java#L891-L896) 方法，将服务体用这的 URL 作为 `"export"` 参数添加到注册中心的 URL 中。通过这样的方式，注册中心的 URL 中，**包含了服务提供者的配置**。
* 第 20 行：调用 `ProxyFactory#getInvoker(proxy, type, url)` 方法，创建 Invoker 对象。该 Invoker 对象，执行 `#invoke(invocation)` 方法时，内部会调用 Service 对象( `ref` )对应的调用方法。
    * 🙂 详细的实现，后面单独写文章分享。
    * 😈 为什么传递的是**注册中心的 URL** 呢？下文会详细解析。
* 第 23 行：创建 [`com.alibaba.dubbo.config.invoker.DelegateProviderMetaDataInvoker`](https://github.com/YunaiV/dubbo/blob/c635dd1990a1803643194048f408db310f06175b/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/invoker/DelegateProviderMetaDataInvoker.java) 对象。该对象在 Invoker 对象的基础上，增加了当前服务提供者 ServiceConfig 对象。
* 第 26 行：调用 `Protocol#export(invoker)` 方法，暴露服务。
    * 此处 Dubbo SPI **自适应**的特性的**好处**就出来了，可以**自动**根据 URL 参数，获得对应的拓展实现。例如，`invoker` 传入后，根据 `invoker.url` 自动获得对应 Protocol 拓展实现为 InjvmProtocol 。
    * 实际上，Protocol 有两个 Wrapper 拓展实现类： ProtocolFilterWrapper、ProtocolListenerWrapper 。所以，`#export(...)` 方法的调用顺序是：
        * **Protocol$Adaptive => ProtocolFilterWrapper => ProtocolListenerWrapper => RegistryProtocol**
        * => 
        * **Protocol$Adaptive => ProtocolFilterWrapper => ProtocolListenerWrapper => DubboProtocol**
        * 也就是说，**这一条大的调用链，包含两条小的调用链**。原因是：
            * 首先，传入的是注册中心的 URL ，通过 Protocol$Adaptive 获取到的是 RegistryProtocol 对象。
            * 其次，RegistryProtocol 会在其 `#export(...)` 方法中，使用服务提供者的 URL ( 即注册中心的 URL 的 `export` 参数值)，再次调用 Protocol$Adaptive 获取到的是 DubboProtocol 对象，进行服务暴露。
        * **为什么是这样的顺序**？通过这样的顺序，可以实现类似 **AOP** 的效果，在本地服务器启动完成后，再向注册中心注册。伪代码如下：

            ```Java
            RegistryProtocol#export(...) {
                
                // 1. 启动本地服务器
                DubboProtocol#export(...);
                
                // 2. 向注册中心注册。 
            }
            ```
            * 这也是为什么上文提到的 “为什么传递的是**注册中心的 URL** 呢？” 的原因。
    * 🙂 如果无法理解，在 [「3. Protocol」](#) 在解析代码，进一步理顺。
* 第 28 行：添加到 `exporters` 集合中。

## 2.1 loadMonitor

> 友情提示，监控中心不是本文的重点，简单分享下该方法的逻辑。  
> 可直接跳过。

`#loadMonitor(registryURL)` 方法，加载监控中心 `com.alibaba.dubbo.common.URL` 数组。代码如下：

```Java
  1: /**
  2:  * 加载监控中心 URL
  3:  *
  4:  * @param registryURL 注册中心 URL
  5:  * @return 监控中心 URL
  6:  */
  7: protected URL loadMonitor(URL registryURL) {
  8:     // 从 属性配置 中加载配置到 MonitorConfig 对象。
  9:     if (monitor == null) {
 10:         String monitorAddress = ConfigUtils.getProperty("dubbo.monitor.address");
 11:         String monitorProtocol = ConfigUtils.getProperty("dubbo.monitor.protocol");
 12:         if ((monitorAddress == null || monitorAddress.length() == 0) && (monitorProtocol == null || monitorProtocol.length() == 0)) {
 13:             return null;
 14:         }
 15: 
 16:         monitor = new MonitorConfig();
 17:         if (monitorAddress != null && monitorAddress.length() > 0) {
 18:             monitor.setAddress(monitorAddress);
 19:         }
 20:         if (monitorProtocol != null && monitorProtocol.length() > 0) {
 21:             monitor.setProtocol(monitorProtocol);
 22:         }
 23:     }
 24:     appendProperties(monitor);
 25:     // 添加 `interface` `dubbo` `timestamp` `pid` 到 `map` 集合中
 26:     Map<String, String> map = new HashMap<String, String>();
 27:     map.put(Constants.INTERFACE_KEY, MonitorService.class.getName());
 28:     map.put("dubbo", Version.getVersion());
 29:     map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
 30:     if (ConfigUtils.getPid() > 0) {
 31:         map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
 32:     }
 33:     // 将 MonitorConfig ，添加到 `map` 集合中。
 34:     appendParameters(map, monitor);
 35:     // 获得地址
 36:     String address = monitor.getAddress();
 37:     String sysaddress = System.getProperty("dubbo.monitor.address");
 38:     if (sysaddress != null && sysaddress.length() > 0) {
 39:         address = sysaddress;
 40:     }
 41:     // 直连监控中心服务器地址
 42:     if (ConfigUtils.isNotEmpty(address)) {
 43:         // 若不存在 `protocol` 参数，默认 "dubbo" 添加到 `map` 集合中。
 44:         if (!map.containsKey(Constants.PROTOCOL_KEY)) {
 45:             if (ExtensionLoader.getExtensionLoader(MonitorFactory.class).hasExtension("logstat")) {
 46:                 map.put(Constants.PROTOCOL_KEY, "logstat");
 47:             } else {
 48:                 map.put(Constants.PROTOCOL_KEY, "dubbo");
 49:             }
 50:         }
 51:         // 解析地址，创建 Dubbo URL 对象。
 52:         return UrlUtils.parseURL(address, map);
 53:     // 从注册中心发现监控中心地址
 54:     } else if (Constants.REGISTRY_PROTOCOL.equals(monitor.getProtocol()) && registryURL != null) {
 55:         return registryURL.setProtocol("dubbo").addParameter(Constants.PROTOCOL_KEY, "registry").addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map));
 56:     }
 57:     return null;
 58: }
```

* 第 8 至 24 行：从**属性配置**中，加载配置到 MonitorConfig 对象。
* 第 25 至 32 行：添加 `interface` `dubbo` `timestamp` `pid` 到 `map` 集合中。
* 第 34 行：调用 `#appendParameters(map, config)` 方法，将 MonitorConfig ，添加到 `map` 集合中。
* 第 35 至 40 行：获得**监控中心**的地址。
* 第 42 行：当 `address` 非空时，**直连监控中心服务器地址的情况**。
    * 第 43 至 50 行：若不存在 `protocol` 参数，缺省默认为 "dubbo" ，并添加到 `map` 集合中。
        * 第 44 至 55 行：**可以忽略**。因为，`logstat` 这个拓展实现已经不存在。
    * 第 52 行：调用 [`UrlUtils#parseURL(address, map)`](https://github.com/YunaiV/dubbo/blob/8de6d56d06965a38712c46a0220f4e59213db72f/dubbo-common/src/main/java/com/alibaba/dubbo/common/utils/UrlUtils.java#L30-L145) 方法，解析 `address` ，创建 Dubbo URL 对象。
        * 🙂 已经添加了代码注释，胖友点击链接查看。 
* 第 54 至 56 行：当 `protocol = registry` 时，并且注册中心 URL 非空时，**从注册中心发现监控中心地址**。以 `registryURL` 为基础，创建 URL ：
    * `protocol = dubbo`
    * `parameters.protocol = registry` 
    * `parameters.refer = map`
* 第 57 行：**无注册中心**，返回空。
* ps ：后续会有文章，详细分享。

# 3. Protocol

本文涉及的 Protocol 类图如下：

TODO [Protocol 类图](http://www.iocoder.cn/images/Dubbo/2018_03_07/04.png)

## 3.1 ProtocolFilterWrapper

## 3.2 RegistryProtocol

## 3.3 DubboProtocol

# 4. Exporter

# 5. 

# 666. 彩蛋

