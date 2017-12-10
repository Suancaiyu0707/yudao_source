title: SkyWalking 源码分析 —— Agent 初始化
date: 2020-07-05
tags:
categories: SkyWalking
permalink: SkyWalking/agent-init

---

# 1. 概述

本文主要分享 **SkyWalking Agent 启动初始化的过程**。

SkyWalking Agent 基于 **JavaAgent** 机制，实现应用**透明**接入 SkyWalking 。关于 JavaAgent 机制，笔者推荐如下两篇文章 ：

* [《Instrumentation 新功能》](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html)
* [《JVM源码分析之javaagent原理完全解读》](http://www.infoq.com/cn/articles/javaagent-illustrated)

> 友情提示 ：建议自己手撸一个简单的 JavaAgent ，更容易理解 SkyWalking Agent 。 
>  
> 笔者练手的 JavaAgent 项目地址 ：https://github.com/YunaiV/learning/tree/master/javaagent01

# 2. SkyWalkingAgent

`org.skywalking.apm.agent.SkyWalkingAgent` ，在 `apm-sniffer/apm-agent` Maven 模块项目里，SkyWalking Agent **启动入口**。为什么说它是启动入口呢？在 `apm-sniffer/apm-agent` 的 [`pom.xml`](https://github.com/OpenSkywalking/skywalking/blob/23133f7d97d17b471f69e7214a01885ebcd2e882/apm-sniffer/apm-agent/pom.xml#L53) 文件的【第 73 行】，我们可以看到 SkyWalkingAgent 被配置成 JavaAgent 的 **PremainClass** 。

[`#premain(...)`](https://github.com/YunaiV/skywalking/blob/c51dbc997348111674dbeedb71d22b0414936cdb/apm-sniffer/apm-agent/src/main/java/org/skywalking/apm/agent/SkyWalkingAgent.java#L54) 方法，代码如下 ：

* 第 58 行 ：调用 `SnifferConfigInitializer#initialize()` 方法，初始化 Agent 配置。
* 第 61 行 ：调用 `PluginBootstrap#loadPlugins()` 方法，加载 Agent 插件们。而后，创建 PluginFinder 。 
* 第 64 行 ：调用 `ServiceManager#boot()` 方法，初始化 Agent 服务管理。在这过程中，Agent 服务们会被初始化。
* 第 79 至 133 行 ：基于 [byte-buddy](https://github.com/raphw/byte-buddy) ，初始化 Instrumentation 的 [`java.lang.instrument.ClassFileTransformer`](https://docs.oracle.com/javase/7/docs/api/java/lang/instrument/ClassFileTransformer.html) 。

# 3. SnifferConfigInitializer

`org.skywalking.apm.agent.core.conf.SnifferConfigInitializer` ，Agent 配置初始化器。

在看具体代码实现之前，我们先看下 `org.skywalking.apm.agent.core.conf` 包的大体结构 ：

![](../../../images/SkyWalking/2020_07_05/02.png)

配置类有 Config 和 RemoteDownstreamConfig 两种。从命名上可以看出 ：

* Config 为 Agent **本地**配置类，使用 SnifferConfigInitializer 进行初始化。
* RemoteDownstreamConfig 为 Agent **远程**配置类，从 [Collector Server](https://github.com/OpenSkywalking/skywalking/wiki/3.2.3-%E9%83%A8%E7%BD%B2Collector) 读取。 

-------

[`#initialize()`](https://github.com/YunaiV/skywalking/blob/cea46a7e93437bbb16db2bfe0fae5c6fcf733fc2/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/conf/SnifferConfigInitializer.java#L56) 方法，初始化 Agent 本地配置，代码如下 ：

* 第 59 至 67 行 ：从配置文件( `agent.config` ) 加载配置。配置文件所在**固定**路径为 `${AGENT_PACKAGE_PATH}/config/agent.config` ，其中 `${AGENT_PACKAGE_PATH}` 通过 [`org.skywalking.apm.agent.core.boot.AgentPackagePath`](https://github.com/YunaiV/skywalking/blob/c51dbc997348111674dbeedb71d22b0414936cdb/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/boot/AgentPackagePath.java#L31) 初始化。Agent 整理目录如下图 ：![](../../../images/SkyWalking/2020_07_05/01.png)
* 第 70 至 74 行 ：从环境变量**覆盖**配置。环境变量 **Key** 需以 `"skywalking."` 开头。例如，`Config.Agent.APPLICATION_CODE` 在 `agent.config` 为 `agent.application_code` ，环境变量为 `skywalking.agent.application_code` 。另外，环境变量包括 JVM 进程的和系统的。
* 第 77 至 82 行 ：校验配置是否正确加载。

## 3.1 Config

`org.skywalking.apm.agent.core.conf.Config` ，Agent 本地配置类。

打开 [Config](https://github.com/YunaiV/skywalking/blob/cea46a7e93437bbb16db2bfe0fae5c6fcf733fc2/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/conf/Config.java#L32) ，我们会看到拆分了 Agent / Collector / Jvm / Buffer / Dictionary / Logging / Plugin 七个小类。如下图 ：

![](../../../images/SkyWalking/2020_07_05/03.png)

本文暂不对配置项详细解析，胖友可以看下每个属性的英文注释。

## 3.2 RemoteDownstreamConfig

`org.skywalking.apm.agent.core.conf.RemoteDownstreamConfig` ，Agent 远程配置类。

打开 [RemoteDownstreamConfig](https://github.com/YunaiV/skywalking/blob/cea46a7e93437bbb16db2bfe0fae5c6fcf733fc2/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/conf/RemoteDownstreamConfig.java) ，我们会看到拆分了 Agent / Collector 两小类。如下图 ：

![](../../../images/SkyWalking/2020_07_05/04.png)

本文暂不对配置项详细解析，胖友可以看下每个属性的英文注释。

# 4. Plugin

SkyWalking Agent 提供了多种插件，实现不同框架的**透明**接入 SkyWalking 。在 [《官方文档 —— supported list》](https://github.com/OpenSkywalking/skywalking/wiki/3.2.3-supported-list) 里，有目前的插件列表。

另外，在 `apm-sniffer/apm-sdk-plugin` 目录下，有插件的实现代码 ：![](../../../images/SkyWalking/2020_07_05/05.png)

本小节会分享的较为简单，在 [《TODO 【4000】》]() 详细解析。

## 4.1 PluginBootstrap

`org.skywalking.apm.agent.core.plugin.PluginBootstrap` ，插件引导程序类，创建需要加载的插件对象数组。

[`#loadPlugins()`](https://github.com/YunaiV/skywalking/blob/130f0a5a3438663b393e53ba2cca02a8d13c258a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginBootstrap.java#L45) 方法，代码如下 ：

* 第 47 行 ：初始化 AgentClassLoader 。
* 第 50 至 56 行 ：获得插件**路径**数组。
* 第 59 至 66 行 ：获得插件**定义**( [`org.skywalking.apm.agent.core.plugin.PluginDefine`](https://github.com/OpenSkywalking/skywalking/blob/b16d23c1484bec941367d6b36fa932b8ace40971/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginDefine.java) )数组。
* 第 69 至 82 行 ：创建**类增强插件定义**( [`org.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine`](https://github.com/OpenSkywalking/skywalking/blob/b16d23c1484bec941367d6b36fa932b8ace40971/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/AbstractClassEnhancePluginDefine.java) )对象数组。不同插件通过实现 AbstractClassEnhancePluginDefine **抽象类**，定义不同框架的**切面**，**记录调用链路**。以 Spring 插件为例子，如下是相关类图 ：![](../../../images/SkyWalking/2020_07_05/06.png)

## 4.2 PluginFinder

`org.skywalking.apm.agent.core.plugin.PluginFinder` ，插件发现者。其提供 [`#find(...)`](https://github.com/YunaiV/skywalking/blob/09c654af33081e56547cb8b3b9e0c8525ddce32f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginFinder.java#L80) 方法，获得**类增强插件定义**( [`org.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine`](https://github.com/OpenSkywalking/skywalking/blob/b16d23c1484bec941367d6b36fa932b8ace40971/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/AbstractClassEnhancePluginDefine.java) )对象。

PluginFinder **[构造方法](https://github.com/YunaiV/skywalking/blob/09c654af33081e56547cb8b3b9e0c8525ddce32f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginFinder.java#L56)**，代码如下 ：

* 第 57 至 77 行 ：循环 AbstractClassEnhancePluginDefine 对象数组，添加到 `nameMatchDefine` / `signatureMatchDefine` 属性，方便 `#find(...)` 方法查找 AbstractClassEnhancePluginDefine 对象。
    * 第 65 至 72 行 ：处理 NameMatch 为匹配的 AbstractClassEnhancePluginDefine 对象，添加到 `nameMatchDefine`  属性。
    * 第 74 至 76 行 ：处理**非** NameMatch 为匹配的 AbstractClassEnhancePluginDefine 对象，添加到 `signatureMatchDefine` 属性。

# 5. ServiceManager

`org.skywalking.apm.agent.core.boot.ServiceManager` ，[BootService](https://github.com/YunaiV/skywalking/blob/09c654af33081e56547cb8b3b9e0c8525ddce32f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/boot/BootService.java) 管理器。负责管理、初始化 BootService 实例们。

[`#boot()`](https://github.com/YunaiV/skywalking/blob/3de8a6c15d07aa3b2c3b4e732e6654fc87c4e70e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/boot/ServiceManager.java#L45) 方法，代码如下 ：

* 第 47 行 ：调用 [`#loadAllServices()`](https://github.com/YunaiV/skywalking/blob/3de8a6c15d07aa3b2c3b4e732e6654fc87c4e70e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/boot/ServiceManager.java#L72) 方法，加载所有 BootService 实现类的实例数组。ServiceManager 基于 SPI (Service Provider Interface) 机制，在 [/resources/META-INF.services/org.skywalking.apm.agent.core.boot.BootService](https://github.com/OpenSkywalking/skywalking/blob/b16d23c1484bec941367d6b36fa932b8ace40971/apm-sniffer/apm-agent-core/src/main/resources/META-INF/services/org.skywalking.apm.agent.core.boot.BootService) 文件里，定义了所有 BootService 的实现类。如果胖友对 SPI 机制不熟悉，可以看下如下文章 ：
    * [《SPI 和 ServiceLoader》](http://www.jianshu.com/p/32d3e108f30a) 
    * [《跟我学Dubbo系列之Java SPI机制简介》](http://www.jianshu.com/p/46aa69643c97) 

* 第 50 行 ：调用 [`#beforeBoot()`](https://github.com/YunaiV/skywalking/blob/3de8a6c15d07aa3b2c3b4e732e6654fc87c4e70e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/boot/ServiceManager.java#L82) 方法，调用每个 `BootService#beforeBoot()` 方法。
* 第 52 行 ：调用 [`#startup()`](https://github.com/YunaiV/skywalking/blob/3de8a6c15d07aa3b2c3b4e732e6654fc87c4e70e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/boot/ServiceManager.java#L92) 方法，调用每个 `BootService#boot()` 方法。
* 第 54 行 ：调用 [`#afterBoot()`](https://github.com/YunaiV/skywalking/blob/3de8a6c15d07aa3b2c3b4e732e6654fc87c4e70e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/boot/ServiceManager.java#L102) 方法，调用每个 `BootService#afterBoot()` 方法。

## 5.1 BootService

`org.skywalking.apm.agent.core.boot.BootService` ，Agent 启动服务**接口**，定义了 `#beforeBoot()` / `#boot()` / `#afterBoot()` / `#shutdown()` 接口方法。

BootService 目前有**七个**实现类，在后续的文章，我们会解析相关实现。

![](../../../images/SkyWalking/2020_07_05/07.png)

# 666. 彩蛋

每次写初始化相关的文章，写少了，怕太水；写多了，又怕太复杂。

嗯，送一发妹子。

![](../../../images/SkyWalking/2020_07_05/08.png)

胖友，分享个朋友圈可好？


