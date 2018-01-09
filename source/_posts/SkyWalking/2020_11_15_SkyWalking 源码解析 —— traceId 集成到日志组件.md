title: SkyWalking 源码分析 —— traceId 集成到日志组件
date: 2020-11-10
tags:
categories: SkyWalking
permalink: SkyWalking/trace-id-integrate-into-logs

-------

# 1. 概述

本文主要分享 **traceId 集成到日志组件**，例如 log4j 、log4j2 、logback 等等。

我们首先看看**集成**的使用例子，再看看**集成**的实现代码。涉及代码如下：

* [](http://www.iocoder.cn/images/SkyWalking/2020_11_15/01.png)
* [](http://www.iocoder.cn/images/SkyWalking/2020_11_15/02.png)

本文以 **logback 1.x** 为例子。

# 2. 使用例子

1、**无需**引入相应的工具包，只需启动参数带上 `-javaagent:/Users/yunai/Java/skywalking/packages/skywalking-agent/skywalking-agent.jar` 。

2、在 `logback.xml` 配置 `%tid` 占位符：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_15/03.png)

3、使用 `logger.info(...)` ，会打印日志如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_15/04.png)

**注意**，traceId 打印到每条日志里，最终需要经过例如 ELK ，收集到日志中心。

# 3. 实现代码

## 3.1 TraceIdPatternLogbackLayout

[`org.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout`]() ，实现 `ch.qos.logback.classic.PatternLayout` 类，实现支持 `%tid` 的占位符。代码如下：

* 第 33 行：添加 `tid` 的转换器为 [`org.skywalking.apm.toolkit.log.logback.v1.x.LogbackPatternConverter`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-application-toolkit/apm-toolkit-logback-1.x/src/main/java/org/skywalking/apm/toolkit/log/logback/v1/x/LogbackPatternConverter.java) 类。

## 3.2 LogbackPatternConverterActivation

[`org.skywalking.apm.toolkit.activation.log.logback.v1.x.LogbackPatternConverterActivation`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-logback-1.x-activation/src/main/java/org/skywalking/apm/toolkit/activation/log/logback/v1/x/LogbackPatternConverterActivation.java) ，实现 ClassInstanceMethodsEnhancePluginDefine 抽象类，定义了方法切面，代码如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_15/05.png)

-------

[`org.skywalking.apm.toolkit.activation.log.logback.v1.x.PrintTraceIdInterceptor`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-logback-1.x-activation/src/main/java/org/skywalking/apm/toolkit/activation/log/logback/v1/x/PrintTraceIdInterceptor.java) ，实现 InstanceMethodsAroundInterceptor 接口，LogbackPatternConverterActivation 的拦截器。代码如下：

* [`#afterMethod(...)`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-logback-1.x-activation/src/main/java/org/skywalking/apm/toolkit/activation/log/logback/v1/x/PrintTraceIdInterceptor.java#L37) 方法，调用 `ContextManager#getGlobalTraceId()` 方法，使用全局链路追踪编号，而不是原有结果。

# 666. 彩蛋

简单一文一篇 Again 。

[](http://www.iocoder.cn/images/SkyWalking/2020_11_15/06.png)

胖友，分享个朋友圈可好？


