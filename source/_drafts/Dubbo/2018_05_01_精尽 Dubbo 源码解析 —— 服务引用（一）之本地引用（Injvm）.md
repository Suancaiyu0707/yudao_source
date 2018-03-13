# 1. 概述

Dubbo 服务引用，**和 Dubbo 服务暴露一样**，**也**有两种方式：

* 本地引用，JVM 本地调用。配置如下：

    ```XML
    // 推荐
    <dubbo:service scope="local" />
    // 不推荐使用，准备废弃
    <dubbo:service injvm="true" />
    ```

* 远程暴露，网络远程通信。配置如下：

    ```XML
    <dubbo:service scope="remote" />
    ```

我们知道 Dubbo 提供了多种协议( Protocol )实现。

* **本文**仅分享本地引用，该方式仅使用 Injvm 协议实现，具体代码在 `dubbo-rpc-injvm` 模块中。
* **下几篇**会分享远程引用，该方式有多种协议实现，例如 Dubbo ( 默认协议 )、Hessian 、Rest 等等。我们会每个协议对应一篇文章，进行分享。

# 2. createProxy

远程暴露服务的顺序图如下：

TODO

在 [《精尽 Dubbo 源码分析 —— API 配置（三）之服务消费者》](http://www.iocoder.cn/Dubbo/configuration-api-3/?self) 一文中，我们看到 `ReferenceConfig#init()` 方法中，会在配置初始化完成后，调用顺序图的**起点** `#createProxy(map)` 方法，开始引用服务。代码如下：

```Java
/**
 * 自适应 Protocol 实现对象
 */
private static final Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
/**
 * 自适应 ProxyFactory 实现对象
 */
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
    
/**
 * 直连服务提供者地址
 */
// url for peer-to-peer invocation
private String url;

  1: private T createProxy(Map<String, String> map) {
  2:     URL tmpUrl = new URL("temp", "localhost", 0, map);
  3:     // 是否本地引用
  4:     final boolean isJvmRefer;
  5:     // injvm 属性为空，不通过该属性判断
  6:     if (isInjvm() == null) {
  7:         // 直连服务提供者，参见文档《直连提供者》https://dubbo.gitbooks.io/dubbo-user-book/demos/explicit-target.html
  8:         if (url != null && url.length() > 0) { // if a url is specified, don't do local reference
  9:             isJvmRefer = false;
 10:         // 通过 `tmpUrl` 判断，是否需要本地引用
 11:         } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
 12:             // by default, reference local service if there is
 13:             isJvmRefer = true;
 14:         // 默认不是
 15:         } else {
 16:             isJvmRefer = false;
 17:         }
 18:     // 通过 injvm 属性。
 19:     } else {
 20:         isJvmRefer = isInjvm();
 21:     }
 22: 
 23:     // 本地引用
 24:     if (isJvmRefer) {
 25:         // 创建服务引用 URL 对象
 26:         URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
 27:         // 引用服务，返回 Invoker 对象
 28:         invoker = refprotocol.refer(interfaceClass, url);
 29:         if (logger.isInfoEnabled()) {
 30:             logger.info("Using injvm service " + interfaceClass.getName());
 31:         }
 32:     // 正常流程，一般为远程引用
 33:     } else {
 34:         // ... 省略本文暂时不分享的服务远程引用 
 35:         }
 36:     }
 37: 
 38:     // 启动时检查
 39:     Boolean c = check;
 40:     if (c == null && consumer != null) {
 41:         c = consumer.isCheck();
 42:     }
 43:     if (c == null) {
 44:         c = true; // default true
 45:     }
 46:     if (c && !invoker.isAvailable()) {
 47:         throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
 48:     }
 49:     if (logger.isInfoEnabled()) {
 50:         logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
 51:     }
 52: 
 53:     // 创建 Service 代理对象
 54:     // create service proxy
 55:     return (T) proxyFactory.getProxy(invoker);
 56: }
```

* `map` 方法参数，URL 参数集合，包含服务引用配置对象的配置项。
* ============ 分割线 ============
* 第 2 行：创建 URL 对象，重点在**第四个参数**，传入的是 `map` ，仅用于第 11 行，是否本地引用。
    * `protocol = temp` 的原因是，在第 11 行，已经直接使用了 InjvmProtocol ，而不需要通过该值去获取。
* 第 4 行：是否本地引用变量 `isJvmRefer` 。
* 第 19 行 至 20 行：调用 `#isInjvm()` 方法，返回**非空**，说明配置了 `injvm` 配置项，直接使用配置项。
* 第 8 至 9 行：配置了 `url` 配置项，说明使用直连服务提供者的功能，则不使用本地使用。
    * [《Dubbo 用户指南 —— 直连提供者》](https://dubbo.gitbooks.io/dubbo-user-book/demos/explicit-target.html) 
* 第 11 至 13 行：调用 `InjvmProtocol#isInjvmRefer(url)` 方法，通过 `tmpUrl` 判断，是否需要本地引用。使用 `tmpUrl` ，相当于使用服务引用配置对象的配置项。该方法代码如下：

    ```Java
      1: /**
      2:  * 是否本地引用
      3:  *
      4:  * @param url URL
      5:  * @return 是否
      6:  */
      7: public boolean isInjvmRefer(URL url) {
      8:     final boolean isJvmRefer;
      9:     String scope = url.getParameter(Constants.SCOPE_KEY);
     10:     // Since injvm protocol is configured explicitly, we don't need to set any extra flag, use normal refer process.
     11:     // 当 `protocol = injvm` 时，本身已经是 jvm 协议了，走正常流程就是了。
     12:     if (Constants.LOCAL_PROTOCOL.toString().equals(url.getProtocol())) {
     13:         isJvmRefer = false;
     14:     // 当 `scope = local` 或者 `injvm = true` 时，本地引用
     15:     } else if (Constants.SCOPE_LOCAL.equals(scope) || (url.getParameter("injvm", false))) {
     16:         // if it's declared as local reference
     17:         // 'scope=local' is equivalent to 'injvm=true', injvm will be deprecated in the future release
     18:         isJvmRefer = true;
     19:     // 当 `scope = remote` 时，远程引用
     20:     } else if (Constants.SCOPE_REMOTE.equals(scope)) {
     21:         // it's declared as remote reference
     22:         isJvmRefer = false;
     23:     // 当 `generic = true` 时，即使用泛化调用，远程引用。
     24:     } else if (url.getParameter(Constants.GENERIC_KEY, false)) {
     25:         // generic invocation is not local reference
     26:         isJvmRefer = false;
     27:     // 当本地已经有该 Exporter 时，本地引用
     28:     } else if (getExporter(exporterMap, url) != null) {
     29:         // by default, go through local reference if there's the service exposed locally
     30:         isJvmRefer = true;
     31:     // 默认，远程引用
     32:     } else {
     33:         isJvmRefer = false;
     34:     }
     35:     return isJvmRefer;
     36: }
    ```
    
    * ============ 本地引用 ============
    * 第 15 至 18 行：当 `scope = local` 或 `injvm = true` 时，本地引用。
    * 第 27 至 30 行：调用 `#getExporter(url)` 方法，判断当本地已经有 `url` 对应的 InjvmExporter 时，**直接**引用。🙂 本地已有的服务，不必要使用远程服务，减少网络开销，提升性能。
        * 🙂 代码比较简单，已经添加中文注释，胖友点击链接查看。
        * [`InjvmProtocol#getExporter(url)`](https://github.com/YunaiV/dubbo/blob/6f366fae76b4fc5fc4fb0352737b6e847a3a2b0b/dubbo-rpc/dubbo-rpc-injvm/src/main/java/com/alibaba/dubbo/rpc/protocol/injvm/InjvmProtocol.java#L63-L94)
        * [`UrlUtils#isServiceKeyMatch(pattern, value)`](https://github.com/YunaiV/dubbo/blob/6f366fae76b4fc5fc4fb0352737b6e847a3a2b0b/dubbo-common/src/main/java/com/alibaba/dubbo/common/utils/UrlUtils.java#L462-L491)
    * ============ 远程引用 ============
    * 第 10 至 13 行：当 `protocol = injvm` 时，本身已经是 Injvm 协议了，走正常流程即可。**这是最特殊的，下面会更好的理解**。另外，因为 `#isInjvmRefer(url)` 方法，仅有在 `#createProxy(map)` 方法中调用，因此实际也不会触发该逻辑。
    * 第 19 至 22 行：当 `scope = remote` 时，远程引用。
    * 第 23 至 26 行：当 `generic = true` 时，即使用泛化调用，远程引用。
        * [《Dubbo 用户指南 —— 泛化调用》](https://dubbo.gitbooks.io/dubbo-user-book/demos/generic-reference.html) 
    * 第 31 至 34 行：默认，远程引用。

* 第 23 至 31 行：**本地引用**。
    * 第 26 行：创建本地服务引用 URL 对象。 
    * 第 28 行：调用 `Protocol#refer(interface, url)` 方法，引用服务，返回 Invoker 对象。
        * 此处 Dubbo SPI **自适应**的特性的**好处**就出来了，可以**自动**根据 URL 参数，获得对应的拓展实现。例如，`invoker` 传入后，根据 `invoker.url` 自动获得对应 Protocol 拓展实现为 InjvmProtocol 。
        * 实际上，Protocol 有两个 Wrapper 拓展实现类： ProtocolFilterWrapper、ProtocolListenerWrapper 。所以，`#refer(...)` 方法的调用顺序是：**Protocol$Adaptive => ProtocolFilterWrapper => ProtocolListenerWrapper => InjvmProtocol** 。
        * 🙂 详细的调用，在 [「3. Protocol」](#) 在解析。
* 第 32 至 36 行：正常流程，一般为**远程引用**。为什么是**一般**呢？如果我们配置 `protocol = injvm` ，实际走的是**本地引用**。例如：

    ```XML
    <dubbo:reference protocol="injvm" >
    </dubbo:reference>
    ```
    * 🌞 当然，笔者建议，如果真的是需要本地应用，建议配置 `scope = local` 。这样，会更加明确和清晰。

* 第 38 至 51 行：若配置 `check = true` 配置项时，调用 `Invoker#isAvailable()` 方法，启动时检查。
    * 🙂 该方法在 [TODO]() ，详细分享。
    * 🙂 [《Dubbo 用户指南 —— 启动时检查》](https://dubbo.gitbooks.io/dubbo-user-book/demos/preflight-check.html)
* 第 55 行：调用 `ProxyFactory#getProxy(invoker)` 方法，创建 Service 代理对象。该 Service 代理对象的内部，会调用 `Invoker#invoke(Invocation)` 方法，进行 Dubbo 服务的调用。
    * 🙂 详细的实现，后面单独写文章分享。

# 3. Protocol

# 4. Invoker

