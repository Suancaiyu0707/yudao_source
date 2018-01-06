# 1. 概述

本文主要分享 **SkyWalking JVM 指标的收集与存储**。大体流程如下：

* Agent 每秒定时收集 JVM 指标到缓冲队列。
* Agent 每秒定时将缓冲队列的 JVM 指标发送到 Collector 。
* Collector 接收到 JVM 指标，异步批量存储到存储器( 例如，ES )。

目前 JVM 指标包括**四个维度**：

* CPU
* Memory
* MemoryPool
* GC

SkyWalking UI 界面如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_10_20/02.png)

# 2. Agent 收集 JVM 指标

## 2.1 JVMService

[`org.skywalking.apm.agent.core.jvm.JVMService`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/jvm/JVMService.java) ，实现 BootService 、Runnable 接口，JVM 指标服务，负责将 JVM 指标收集并发送给 Collector 。代码如下：

* `queue` 属性，收集指标队列。
* `collectMetricFuture` 属性，收集指标定时任务。
* `sendMetricFuture` 属性，发送指标定时任务。
* `sender` 属性，发送器。

[`#beforeBoot()`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/jvm/JVMService.java#L79) 方法，初始化 `queue` ，`sender` 属性，并将自己添加到 GRPCChannelManager ，从而监听与 Collector 的连接状态。

[`boot`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/jvm/JVMService.java#L86) 方法，创建两个定时任务：

* 第 88 至 90 行：创建收集指标定时任务，0 秒延迟，1 秒间隔，调用 `JVMService#run()` 方法。
* 第 92 至 94 行：创建发送指标定时任务，0 秒延迟，1 秒间隔，调用 `Sender#run()` 方法。

### 2.1.1 定时收集

[`JVMService#run()`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/jvm/JVMService.java#L109) 方法，代码如下：

* 第 110 至 111 行：应用实例注册后，才收集 JVM 指标。
* 第 116 至 122 行：创建 JVMMetric 对象。
    * 第 118 行：调用 `CPUProvider#getCpuMetric()` 方法，获得 GC 指标。
    * 第 119 行：调用 `MemoryProvider#getMemoryMetricList()` 方法，获得 Memory 指标。
    * 第 120 行：调用 `MemoryPoolProvider#getMemoryPoolMetricList()` 方法，获得 MemoryPool 指标。
    * 第 121 行：调用 `GCProvider#getGCList()` 方法，获得 GC 指标。
* 第 125 至 128 行：提交 JVMMetric 对象到收集指标队列。

### 2.2.2 定时发送

[`JVMService.Sender`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/jvm/JVMService.java#L135) ，实现 Runnable 、GRPCChannelListener 接口，JVM 指标发送器。代码如下：

* `status` 属性，连接状态。
* `stub` 属性，**阻塞** Stub 。
* [`#statusChanged(GRPCChannelStatus)`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/jvm/JVMService.java#L171) 方法，当连接成功时，创建**阻塞** Stub 。
* [`#run()`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/jvm/JVMService.java#L147) 方法，代码如下：
    * 第 148 至 151 行：应用实例注册后，并且连接中，才发送 JVM 指标。
    * 第 153 至 155 行：调用 `#drainTo(Collection)` 方法，从队列移除所有 JVMMetric 到 `buffer` 数组。
    * 第 157 至 162 行：使用 Stub ，**批量**发送到 Collector 。

## 2.2 CPU

## 2.3 Memory

## 2.4 MemoryPool

## 2.5 GC 

# 3. Collector 存储 JVM 指标

## 3.1 JVMMetricsServiceHandler

我们先来看看 API 的定义，[`JVMMetricsService.proto`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-network/src/main/proto/JVMMetricsService.proto#L8) ，如下图所示：

[](http://www.iocoder.cn/images/SkyWalking/2020_10_20/01.png)

[`JVMMetricsServiceHandler#collect(JVMMetrics, StreamObserver<Downstream>)`](https://github.com/YunaiV/skywalking/blob/c15cf5e1356c7b44a23f2146b8209ab78c2009ac/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/TraceSegmentServiceHandler.java#L47), 代码如下：

* 第 60 行：**循环**接收到的 JVMMetric **数组**。
* 第 62 行：调用 [`#sendToInstanceHeartBeatService(...)`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/JVMMetricsServiceHandler.java#L77) 方法，发送心跳，记录应用实例的最后心跳时间。因为目前 SkyWaling 主要用于 JVM 平台，通过每秒的 JVM 指标收集的同时，记录应用实例的最后心跳时间。
* 第 62 行：调用 [`#sendToCpuMetricService(...)`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/JVMMetricsServiceHandler.java#L91) 方法，处理 CPU 数据。
* 第 64 行：调用 [`#sendToMemoryMetricService(...)`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/JVMMetricsServiceHandler.java#L81) 方法，处理 Memory 数据。
* 第 66 行：调用 [`#sendToMemoryPoolMetricService(...)`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/JVMMetricsServiceHandler.java#L85) 方法，处理 Memory Pool 数据。
* 第 68 行：调用 [`#sendToGCMetricService(...)`](https://github.com/YunaiV/skywalking/blob/4b7d7083ca9cd89437bcca6d0c5f67f3832d60dd/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/JVMMetricsServiceHandler.java#L95) 方法，处理 GC 数据。
* 第 73 至 74 行：全部处理完成，返回成功。

上述的 `#sendToXXX()` 方法，内部每个对应调用一个如下图 Service 提供的方法：[](http://www.iocoder.cn/images/SkyWalking/2020_10_20/04.png)

* 每个 Service 的实现，对应一个数据实体和一个 Graph 对象，通过流式处理，最终存储到存储器( 例如 ES ) ，流程如下图：
* 具体的实现代码，我们放在下面的数据实体一起分享。

## 3.2 CPU

## 3.3 Memory

## 3.4 MemoryPool

## 3.5 GC 

