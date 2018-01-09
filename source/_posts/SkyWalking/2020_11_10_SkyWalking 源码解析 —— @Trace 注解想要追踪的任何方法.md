title: SkyWalking 源码分析 —— @Trace 注解想要追踪的任何方法
date: 2020-11-10
tags:
categories: SkyWalking
permalink: SkyWalking/@trace-for-any-methods

-------

# 1. 概述

本文主要分享 **@Trace 注解想要追踪的任何方法**。

我们首先看看 [`@Trace`](https://github.com/apache/incubator-skywalking/blob/af2c1b979fe025603dc65d7e2a2dbdea8005ede8/apm-application-toolkit/apm-toolkit-trace/src/main/java/org/apache/skywalking/apm/toolkit/trace/Trace.java) 的使用例子，再看看 `@Trace` 的实现代码。涉及代码如下：

* [](http://www.iocoder.cn/images/SkyWalking/2020_11_10/01.png)
* [](http://www.iocoder.cn/images/SkyWalking/2020_11_10/02.png)

# 2. 使用例子

> 本节参考官方文档：[Application-toolkit-trace-CN.md](https://github.com/apache/incubator-skywalking/blob/master/docs/cn/Application-toolkit-trace-CN.md)

1、使用 Maven 引入相应的工具包

```XML
<dependency>
    <groupId>org.skywalking</groupId>
    <artifactId>apm-toolkit-trace</artifactId>
    <version>${skywalking.version}</version> 
</dependency>
```

2、在**任何想要追踪**的方法上添加 `@Trace` 注解，以 SpringMVC 为例子：

```Java
@Trace
@GetMapping("/log")
public String log() {
    ActiveSpan.tag("mp", "芋道源码");
    System.out.println("traceId：" + TraceContext.traceId());
    return "log";
}
```

* `@Trace` 注解的方法，会创建一个 **LocalSpan** 。
* [`ActiveSpan#tag(key, value)`](https://github.com/apache/incubator-skywalking/blob/af2c1b979fe025603dc65d7e2a2dbdea8005ede8/apm-application-toolkit/apm-toolkit-trace/src/main/java/org/apache/skywalking/apm/toolkit/trace/ActiveSpan.java#L32) 方法，在 LocalSpan 上添加标签键值对。
* [`TraceContext.traceId#traceId`]() 方法，获得全局链路追踪编号。

3、执行后，我们看来看看 SkyWalking WEBUI 的展示。

[](http://www.iocoder.cn/images/SkyWalking/2020_11_10/03.png)

# 3. 实现代码

> 友情提示：本小节需要胖友阅读过 [《SkyWalking 源码分析 —— Agent 插件体系》](http://www.iocoder.cn/SkyWalking/agent-plugin-system/?self)  。

## 3.1 TraceAnnotationActivation

[`org.skywalking.apm.toolkit.activation.trace.TraceAnnotationActivation`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-trace-activation/src/main/java/org/skywalking/apm/toolkit/activation/trace/TraceAnnotationActivation.java) ，实现 ClassInstanceMethodsEnhancePluginDefine 抽象类，定义了方法切面，代码如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_10/04.png)

-------

[`org.skywalking.apm.toolkit.activation.trace.TraceAnnotationMethodInterceptor`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-trace-activation/src/main/java/org/skywalking/apm/toolkit/activation/trace/TraceAnnotationMethodInterceptor.java) ，实现 InstanceMethodsAroundInterceptor 接口，TraceAnnotationActivation 的拦截器。代码如下：

* [`#beforeMethod(...)`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-trace-activation/src/main/java/org/skywalking/apm/toolkit/activation/trace/TraceAnnotationMethodInterceptor.java#L39) 方法，创建 LocalSpan 对象。代码如下：
    * 第 42 至 46 行：获得操作名。若 [`@Trace#operationName()`](https://github.com/apache/incubator-skywalking/blob/af2c1b979fe025603dc65d7e2a2dbdea8005ede8/apm-application-toolkit/apm-toolkit-trace/src/main/java/org/apache/skywalking/apm/toolkit/trace/Trace.java#L40) 非空，作为操作名。否则，调用 [`#generateOperationName(Method)`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-trace-activation/src/main/java/org/skywalking/apm/toolkit/activation/trace/TraceAnnotationMethodInterceptor.java#L58) 方法，使用方法签名。
    * 第 49 行：调用 `ContextManager#createLocalSpan(operationName)` 方法，创建 LocalSpan 对象。
* [`#afterMethod(...)`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-trace-activation/src/main/java/org/skywalking/apm/toolkit/activation/trace/TraceAnnotationMethodInterceptor.java#L72) 方法，调用 `ContextManager#stopSpan()` 方法，完成 LocalSpan 对象。
* [`#handleMethodException(...)`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-trace-activation/src/main/java/org/skywalking/apm/toolkit/activation/trace/TraceAnnotationMethodInterceptor.java#L79) 方法，发生异常时，打印错误日志。

## 3.2 ActiveSpanTagActivation

[`org.skywalking.apm.toolkit.activation.trace.ActiveSpanTagActivation`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-trace-activation/src/main/java/org/skywalking/apm/toolkit/activation/trace/ActiveSpanTagActivation.java) ，实现 ClassStaticMethodsEnhancePluginDefine 抽象类，定义了方法切面，代码如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_10/05.png)

-------

[`org.skywalking.apm.toolkit.activation.trace.TraceAnnotationMethodInterceptor`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-trace-activation/src/main/java/org/skywalking/apm/toolkit/activation/trace/TraceAnnotationMethodInterceptor.java) ，实现 StaticMethodsAroundInterceptor 接口，ActiveSpanTag 的拦截器。代码如下：

* [`#beforeMethod(...)`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-trace-activation/src/main/java/org/skywalking/apm/toolkit/activation/trace/ActiveSpanTagInterceptor.java#L30) 方法，添加 Span 的**标签键值对**。**注意**，可以不依赖 `@Trace` 注解。

## 3.3 TraceContextActivation

[`org.skywalking.apm.toolkit.activation.trace.TraceContextActivation`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-trace-activation/src/main/java/org/skywalking/apm/toolkit/activation/trace/TraceContextActivation.java) ，实现 ClassStaticMethodsEnhancePluginDefine 抽象类，定义了方法切面，代码如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_10/06.png)

-------

[`org.skywalking.apm.toolkit.activation.trace.TraceAnnotationMethodInterceptor`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-trace-activation/src/main/java/org/skywalking/apm/toolkit/activation/trace/TraceContextInterceptor.java) ，实现 StaticMethodsAroundInterceptor 接口，TraceContextActivation 的拦截器。代码如下：

* [`#afterMethod(...)`](https://github.com/YunaiV/skywalking/blob/5106601937af942dabcad917b90d8c92886a2e4d/apm-sniffer/apm-toolkit-activation/apm-toolkit-trace-activation/src/main/java/org/skywalking/apm/toolkit/activation/trace/TraceContextInterceptor.java#L39) 方法，调用 `ContextManager#getGlobalTraceId()` 方法，使用全局链路追踪编号，而不是原有结果。

# 666. 彩蛋

简单一文一篇。

胖友，分享个朋友圈可好？

