title: SkyWalking 源码分析 —— Collector 初始化
date: 2020-07-15
tags:
categories: SkyWalking
permalink: SkyWalking/collector-init

-------

# 1. 概述

本文主要分享 **SkyWalking Collector 启动初始化的过程**。在分享的过程中，我们会**简单**介绍 Collector 每个模块及其用途。

ps ：Collector 是 SkyWalking 的 Server 端。整体如下图 ：

> FROM https://github.com/apache/incubating-skywalking  
> [](../../../images/SkyWalking/2020_07_15/01.png)

# 2. CollectorBootStartUp

`org.skywalking.apm.collector.boot.CollectorBootStartUp` ，在 `apm-sniffer/apm-agent` Maven 模块项目里，SkyWalking Collector **启动入口**。 

[`#main(args)`](https://github.com/YunaiV/skywalking/blob/c633c1f0e143d1df2457926ab239350a642f7be2/apm-collector/apm-collector-boot/src/main/java/org/skywalking/apm/collector/boot/CollectorBootStartUp.java#L38) 方法，启动 Collector ，代码如下 ：

* 第 45 行 ：调用 `ApplicationConfiguration#load()` 方法，加载 Collector **配置**。
* 第 47 行 ：调用 `ModuleManager#init(...)` 方法，初始化 Collector **组件**们。
* 第 60 行 ：调用 `Thread#sleep(60000)` 方法，等待 Collector 内嵌的 Jetty Server 启动完成。

# 2. ApplicationConfigLoader

`org.skywalking.apm.collector.boot.config.ApplicationConfigLoader` ，实现 [`org.skywalking.apm.collector.boot.config.ConfigLoader`](https://github.com/YunaiV/skywalking/blob/c633c1f0e143d1df2457926ab239350a642f7be2/apm-collector/apm-collector-boot/src/main/java/org/skywalking/apm/collector/boot/config/ConfigLoader.java#L24) 接口，Collector 配置( [`org.skywalking.apm.collector.core.module.ApplicationConfiguration`](https://github.com/YunaiV/skywalking/blob/c633c1f0e143d1df2457926ab239350a642f7be2/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ApplicationConfiguration.java) )加载器。

在看具体代码实现之前，我们先了解下 ApplicationConfiguration 整体类结构。如下图所示 ：

[](../../../images/SkyWalking/2020_07_15/02.png)

* Collector 使用组件管理器( ModuleManager )，管理**多个**组件( [Module](https://github.com/YunaiV/skywalking/blob/c633c1f0e143d1df2457926ab239350a642f7be2/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java) )。
    * 一个组件有多种组件服务提供者( [ModuleProvider](https://github.com/YunaiV/skywalking/blob/c633c1f0e143d1df2457926ab239350a642f7be2/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java) )，**同时**一个组件只允许使用**一个**组件服务提供者。这块下面会有代码解析说明。
* Collector 使用一个应用配置类( [ApplicationConfiguration](https://github.com/YunaiV/skywalking/blob/c633c1f0e143d1df2457926ab239350a642f7be2/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ApplicationConfiguration.java) )。
    * 一个应用配置类包含多个组件配置类( [ModuleConfiguration](https://github.com/YunaiV/skywalking/blob/c633c1f0e143d1df2457926ab239350a642f7be2/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ApplicationConfiguration.java#L62) )。每个组件对应一个组件配置类。
    * 一个组件配置类包含多个组件服务提供者配置( [ProviderConfiguration](https://github.com/YunaiV/skywalking/blob/c633c1f0e143d1df2457926ab239350a642f7be2/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ApplicationConfiguration.java#L93) )。每个组件服务提供者对应一个组件配置类。**注意**：因为一个组件只允许**同时**使用**一个**组件服务提供者，所以一个组件配置类**只设置**一个组件服务提供者配置。
* 整个配置文件，对应应用配置类。**绿框**部分，对应一个组件配置类。**红框**部分，对应一个组件服务提供者配置类。

下面，我们来看看 [`ApplicationConfigLoader#load()`](https://github.com/YunaiV/skywalking/blob/c633c1f0e143d1df2457926ab239350a642f7be2/apm-collector/apm-collector-boot/src/main/java/org/skywalking/apm/collector/boot/config/ApplicationConfigLoader.java#L43) 方法，代码如下 ：

* 第 47 行 ：调用 `#loadConfig()` 方法，从 `apm-collector-core` 的 [`application.yml`](https://github.com/YunaiV/skywalking/blob/c633c1f0e143d1df2457926ab239350a642f7be2/apm-collector/apm-collector-core/src/main/resources/application-default.yml) 加载自定义配置。
* 第 49 行 ：调用 `#loadDefaultConfig()` 方法，从 `apm-collector-core` 的 [`application-default.yml`](https://github.com/YunaiV/skywalking/blob/c633c1f0e143d1df2457926ab239350a642f7be2/apm-collector/apm-collector-core/src/main/resources/application-default.yml) 加载默认配置。
* 两个方法逻辑基本一致，已经添加代码注释，胖友自己阅读理解。

# 3. ModuleManager

`org.skywalking.apm.collector.core.module.ModuleManager` ，组件管理器，负责组件的管理与初始化。

[`#init()`](https://github.com/YunaiV/skywalking/blob/ff055ee52da855ef6cc8bfdfae7c2758ae3c61cd/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleManager.java#L49) 方法，初始化组件们，代码如下 ：

* 第 51 至 53 行 ：调用 `java.util.ServiceLoader#load(Module.class)` 方法，加载所有 Module 实现类的实例数组。ServiceManager 基于 SPI (Service Provider Interface) 机制，在每个 `apm-collector-xxx-define` 项目的 `/resources/META-INF.services/org.skywalking.apm.collector.core.module.Module` 文件里，定义了该项目 Module 的实现类。如果胖友对 SPI 机制不熟悉，可以看下如下文章 ：
    * [《SPI 和 ServiceLoader》](http://www.jianshu.com/p/32d3e108f30a) 
    * [《跟我学Dubbo系列之Java SPI机制简介》](http://www.jianshu.com/p/46aa69643c97) 

* 第 55 至 75 行 ：遍历所有 Module 实现类的实例数组，创建**在配置中**的 Module 实现类的实例，并执行 Module 准备阶段的逻辑，后添加到加载的组件实例的映射( `loadedModules` )。
    * 第 59 至 67 行 ：创建 Module 对象。
    * 第 69 行 ：调用 `Module#prepare(...)` 方法，执行 Module 准备阶段的逻辑。在改方法内部，会创建 Module 对应的 ModuleProvider 。在 [「3.1 Module」](#) 详细解析。
    * 第 71 行 ：添加到 `loadedModules` 。
* 第 77 至 80 行 ：校验**在配置中**的 Module 实现类的实例都创建了，否则抛出异常。
* 第 84 行 ：调用 `BootstrapFlow#start(...)` 方法，执行 Module 启动逻辑。[「3.4 BootstrapFlow」](#) 详细解析。
* 第 86 行 ：调用 `BootstrapFlow#notifyAfterCompleted()` 方法，执行 Module 启动完成，通知 ModuleProvider 。[「3.4 BootstrapFlow」](#) 详细解析。
* 总的来说，Module 初始化的过程，可以理解成三个阶段，如下图所示 ：[](../../../images/SkyWalking/2020_07_15/03.png)

## 3.1 Module

`org.skywalking.apm.collector.core.module.Module` ，组件**抽象类**。通过实现 Module 抽象类，实现不同功能的组件。目前 Collector 的 Module 实现类如下图 ：[](../../../images/SkyWalking/2020_07_15/04.png)

[`#name()`](https://github.com/YunaiV/skywalking/blob/204a9e658dd95cc8ee5d4e65d7ca1ed58f3a71da/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java#L50) **抽象**方法，获得组件名。目前组件名有 ：[](../../../images/SkyWalking/2020_07_15/05.png)

[`#providers()`](https://github.com/YunaiV/skywalking/blob/204a9e658dd95cc8ee5d4e65d7ca1ed58f3a71da/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java#L110) 方法，获得 ModuleProvider 数组。实际上，一个 Module **同时**只能有一个 ModuleProvider ，参见 [`#provider()`](https://github.com/YunaiV/skywalking/blob/204a9e658dd95cc8ee5d4e65d7ca1ed58f3a71da/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java#L114) 方法。

[`#services()`](https://github.com/YunaiV/skywalking/blob/204a9e658dd95cc8ee5d4e65d7ca1ed58f3a71da/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java#L55) **抽象**方法，获得 Service **类**数组。具体 Service **对象**，在 ModuleProvider 对象里获取，参见 [`#getService(serviceType)`](https://github.com/YunaiV/skywalking/blob/204a9e658dd95cc8ee5d4e65d7ca1ed58f3a71da/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java#L122) 方法。

[`#prepare(...)`](https://github.com/YunaiV/skywalking/blob/204a9e658dd95cc8ee5d4e65d7ca1ed58f3a71da/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java#L66) 方法，执行 Module **准备阶段**的逻辑，代码如下 ：

* 第 69 行 ：调用 `java.util.ServiceLoader#load(ModuleProvider.class)` 方法，加载所有 ModuleProvider 实现类的实例数组。ServiceManager 基于 SPI (Service Provider Interface) 机制，在每个 `apm-collector-xxx-yyy-provider` 项目的 `/resources/META-INF.services/org.skywalking.apm.collector.core.module.ModuleProvider` 文件里，定义了该项目 ModuleProvider 的实现类。
* 第 72 至 93 行 ：遍历所有 ModuleProvider 实现类的实例数组，创建**在配置中**的 ModuleProvider 实现类的实例，后添加到加载的组件服务提供者实例的映射( `loadedProviders` )。
* 第 95 至 98 行 ：校验有 ModuleProvider 初始化，否则抛出异常。
* 第 100 至 104 行 ：调用 `ModuleProvider#prepare(...)`  方法，执行 ModuleProvider 准备阶段的逻辑。在改方法内部，会创建 ModuleProvider 对应的 Service 。在 [「3.2 ModuleProvider」](#) 详细解析。

## 3.2 ModuleProvider

`org.skywalking.apm.collector.core.module.ModuleProvider` ，组件服务提供者**抽象类**。通过实现 ModuleProvider 抽象类，实现不同功能的组件服务提供者。目前 Collector 的 ModuleProvider 实现类如下图 ：[](../../../images/SkyWalking/2020_07_15/06.png)

[`#name()`](https://github.com/YunaiV/skywalking/blob/3fb837104a111ff94abef9c871c814fb60c18340/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java#L64) **抽象**方法，获得组件服务提供者名。目前组件服务提供者名有 ：[](../../../images/SkyWalking/2020_07_15/07.png)

[`#module()`](https://github.com/YunaiV/skywalking/blob/3fb837104a111ff94abef9c871c814fb60c18340/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java#L69) **抽象**方法，获得 ModuleProvider 对应的 Module **类**。注意，ModuleProvider 的名字可以重复，例如上图的 `jetty` ，通过对应的 Module **类**来区分。

[`#requiredModules()`](https://github.com/YunaiV/skywalking/blob/3fb837104a111ff94abef9c871c814fb60c18340/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java#L95) **抽象**方法，获得 ModuleProvider 依赖的 Module **名字**数组。

---------- Service 相关方法 Begin ----------

[`#registerServiceImplementation(Class<? extends Service>, Service)`](https://github.com/YunaiV/skywalking/blob/3fb837104a111ff94abef9c871c814fb60c18340/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java#L103) 方法，注册 Service 对象。一个 ModuleProvider 可以有 0 到 N 个 Service 对象。

[`#getService(Class<T>)`](https://github.com/YunaiV/skywalking/blob/3fb837104a111ff94abef9c871c814fb60c18340/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java#L133) 方法，获得 Service 对象。

[`#requiredCheck(...)`](https://github.com/YunaiV/skywalking/blob/3fb837104a111ff94abef9c871c814fb60c18340/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java#L118) 方法，**校验** ModuleProvider 包含的 Service 们都创建成功。

* **方法参数**，从 `Module#services()` 方法获得。
* 该方法会被 `BootstrapFlow#start()` 方法调用，在 [「3.4 BootstrapFlow」](#) 详细解析。

---------- Service 相关方法 End ----------

[`#prepare(Properties)`](https://github.com/YunaiV/skywalking/blob/3fb837104a111ff94abef9c871c814fb60c18340/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java#L76) **抽象**方法，执行 ModuleProvider 准备阶段的逻辑：Service 的创建，私有变量的创建等等。例如，[`StorageModuleH2Provider#prepare(Properties)`](https://github.com/YunaiV/skywalking/blob/3fb837104a111ff94abef9c871c814fb60c18340/apm-collector/apm-collector-storage/collector-storage-h2-provider/src/main/java/org/skywalking/apm/collector/storage/h2/StorageModuleH2Provider.java#L123) 。

[`#start(Properties)`](https://github.com/YunaiV/skywalking/blob/3fb837104a111ff94abef9c871c814fb60c18340/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java#L83) **抽象**方法，执行 ModuleProvider 启动阶段的逻辑：私有变量的初始化等等。例如，[`StorageModuleH2Provider#start(Properties)`](https://github.com/YunaiV/skywalking/blob/3fb837104a111ff94abef9c871c814fb60c18340/apm-collector/apm-collector-storage/collector-storage-h2-provider/src/main/java/org/skywalking/apm/collector/storage/h2/StorageModuleH2Provider.java#L136) 。

* 该方法会被 `BootstrapFlow#start()` 方法调用，在 [「3.4 BootstrapFlow」](#) 详细解析。

[`#notifyAfterCompleted()`](https://github.com/YunaiV/skywalking/blob/3fb837104a111ff94abef9c871c814fb60c18340/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java#L90) **抽象**方法，执行 ModuleProvider 启动完成阶段的逻辑：私有变量的初始化等等。例如，[`StorageModuleEsProvider#notifyAfterCompleted(Properties)`](https://github.com/YunaiV/skywalking/blob/3fb837104a111ff94abef9c871c814fb60c18340/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/StorageModuleEsProvider.java#L170) 。

* 该方法会被 `BootstrapFlow#notifyAfterCompleted()` 方法调用，在 [「3.4 BootstrapFlow」](#) 详细解析。

## 3.3 Service

`org.skywalking.apm.collector.core.module.Service` ，服务**接口**。通过实现 Service 接口，实现不同功能的服务。目前 Collector 的 Service 实现类如下图 ：[](../../../images/SkyWalking/2020_07_15/08.png)

这里有一点要注意下，实际上 Module 是与 Service **"直接"** 一对多的关系。中间 有一层 ModuleProvider 存在的原因是，相同 Module 可以有多种 ModuleProvider 实现，而 ModuleProvider 提供提供相同功能的 Service ，但是实现不同。

以 `apm-collector-storage` 举例子，如下图所示 ：

[](../../../images/SkyWalking/2020_07_15/09.png)

* StorageModuleEsProvider / StorageModuleH2Provider 分别基于 ES / H2 实现，其提供存储相同数据的不同实现。例如 ：[](../../../images/SkyWalking/2020_07_15/10.png)

这也是为什么有 `Module#services()` 和 `#requiredCheck(Class<? extends Service>[])` 这样的方法涉及的原因。

另外，如下是 Service 接口的解释：

> The <code>Service</code> implementation is a service provided by its own modules.  
> 
> And every {@link ModuleProvider} must provide all the given services of the {@link Module}.

## 3.4 BootstrapFlow

[`org.skywalking.apm.collector.core.module.BootstrapFlow`](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/BootstrapFlow.java)，组件启动流程。

BootstrapFlow [**构造方法**](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/BootstrapFlow.java#L42)，调用 [`#makeSequence()`](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/BootstrapFlow.java#L86) 方法，获得 ModuleProvider 启动顺序，这个是该类的**重点**。

[`#start()`](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/BootstrapFlow.java#L51) 方法，执行 Module 启动逻辑。

* 第 54 至 63 行 ：校验**依赖** Module 已经都存在。
* 第 67 行 ：校验 ModuleProvider 包含的 Service 们都**创建成功**。
* 第 70 行 ：调用 `ModuleProvider#start(...)` 方法，执行 ModuleProvider 启动阶段逻辑。

[`#notifyAfterCompleted()`](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/BootstrapFlow.java#L74) 方法，调用 `ModuleProvider#notifyAfterCompleted()` 方法，执行 ModuleProvider 启动完成阶段的逻辑。

# 4. Module 实现类简介

![](https://camo.githubusercontent.com/2a00cb347f6a7d7afb8faef8d8b0f2a0d3215d9d/68747470733a2f2f736b7977616c6b696e67746573742e6769746875622e696f2f706167652d7265736f75726365732f332e322e352532625f6172636869746563747572652e6a7067)

TODO 【4001】Module 实现文章链接 


