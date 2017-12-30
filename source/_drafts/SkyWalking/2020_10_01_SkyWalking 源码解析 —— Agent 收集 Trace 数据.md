# 1. 概述

分布式链路追踪系统，链路的追踪大体流程如下：

1. Agent 收集 Trace 数据。
2. Agent 发送 Trace 数据给 Collector 。
3. Collector 存储 Trace 数据到存储器，例如，数据库。

本文主要分享 **SkyWalking Agent 收集 Trace 数据**。文章的内容顺序如下：

* Trace 的数据结构
* Context 收集 Trace 的方法

不包括插件对 Context 收集的方法的**调用**，后续单独文章专门分享，胖友也可以阅读完本文后，自己去看 [`apm-sdk-plugin`](https://github.com/YunaiV/skywalking/tree/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-sniffer/apm-sdk-plugin) 的实现代码。

本文涉及到的代码如下图：

[](http://www.iocoder.cn/images/SkyWalking/2020_10_01/01.png)

* **红框**部分：Trace 的数据结构，在 [「2. Trace」](#) 分享。
* **黄框**部分：Context 收集 Trace 的方法，在 [「3. Context」](#) 分享。

# 2. Trace

> 友情提示：胖友，请先行阅读 [《OpenTracing语义标准》](https://github.com/opentracing-contrib/opentracing-specification-zh/blob/master/specification.md) 。  
> 
> 本小节，笔者认为胖友已经对 OpenTracing 有一定的理解。

[`org.skywalking.apm.agent.core.context.trace.TraceSegment`](todo) ，是**一次**分布式链路追踪( Distributed Trace ) 的**一段**。

* **一条** TraceSegment ，用于记录所在**线程**( Thread )的链路。
* **一次**分布式链路追踪，可以包含**多条** TraceSegment ，因为存在**跨进程**( 例如，RPC 、MQ 等等)，或者垮**线程**( 例如，并发执行、异步回调等等 )。

TraceSegment 属性，如下：

* `traceSegmentId` 属性，TraceSegment 的编号，全局唯一。在 [「2.1 ID」](#) 详细解析。
* `refs` 属性，TraceSegmentRef **数组**，指向的**父** TraceSegment 数组。
    * **为什么会有多个爸爸**？下面统一讲。
    * TraceSegmentRef ，在 [「2.3 TraceSegmentRef」](#) 详细解析。
* `relatedGlobalTraces` 属性，关联的 DistributedTraceId **数组**。
    * **为什么会有多个爸爸**？下面统一讲。
    * DistributedTraceId ，在 [「2.1.2 DistributedTraceId」](#) 详细解析。
* `spans` 属性，包含的 Span **数组**。在 [「2.2 AbstractSpan」](#) 详细解析。这是 TraceSegment 的**主体**，总的来说，TraceSegment 是 Span 数组的封装。
* `ignore` 属性，是否忽略该条 TraceSegment 。在一些情况下，我们会忽略 TraceSegment ，即不收集链路追踪，在下面 [「3. Context」](#) 部分内容，我们将会看到这些情况。
* `isSizeLimited` 属性，Span 是否超过上限( [`Config.Agent.SPAN_LIMIT_PER_SEGMENT`](https://github.com/YunaiV/skywalking/blob/2961e9f539286ef91af1ff1ef7681d0a06f156b0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/conf/Config.java#L56) )。超过上限，不在记录 Span 。

**为什么会有多个爸爸**？

* 我们先来看看**一个爸爸**的情况，常见于 RPC 调用。例如，【服务 A】调用【服务 B】时，【服务 B】新建一个 TraceSegment 对象：
    * 将自己的 `refs` 指向【服务 A】的 TraceSegment 。
    * 将自己的 `relatedGlobalTraces` 设置为 【服务 A】的 DistributedTraceId 对象。
* 我们再来看看**多个爸爸**的情况，常见于 MQ / Batch 调用。例如，MQ 批量消费消息时，消息来自【多个服务】。每次批量消费时，【消费者】新建一个 TraceSegment 对象：
    * 将自己的 `refs` 指向【多个服务】的**多个** TraceSegment 。
    * 将自己的 `relatedGlobalTraces` 设置为【多个服务】的**多个** DistributedTraceId 。

> 友情提示：多个爸爸的故事，可能比较难懂，等胖友读完全文，在回过头想想。或者拿起来代码调试调试。

下面，我们来具体看看 TraceSegment 的每个元素，最后，我们会回过头，在 [「2.4 TraceSegment」](#) 详细解析它。

## 2.1 ID

[`org.skywalking.apm.agent.core.context.ids.ID`](https://github.com/YunaiV/skywalking/blob/cc27e35d69d922ba8fa38fbe4e8cc4704960f602/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/ID.java) ，编号。从类的定义上，这是一个**通用**的编号，由三段整数组成。

目前使用 GlobalIdGenerator 生成，作为**全局唯一编号**。属性如下：

* `part1` 属性，应用实例编号。
* `part2` 属性，线程编号。
* `part3` 属性，时间戳串，生成方式为 `${时间戳} * 10000 + 线程自增序列([0, 9999])` 。例如：15127007074950012 。具体生成方法的代码，在 GlobalIdGenerator 中详细解析。
* `encoding` 属性，编码后的字符串。格式为 `"${part1}.${part2}.${part3}"` 。例如，`"12.35.15127007074950000"` 。
    * 使用 [`#encode()`](https://github.com/YunaiV/skywalking/blob/cc27e35d69d922ba8fa38fbe4e8cc4704960f602/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/ID.java#L83) 方法，编码编号。
* `isValid` 属性，编号是否合法。
    * 使用 [`ID(encodingString)`](https://github.com/YunaiV/skywalking/blob/cc27e35d69d922ba8fa38fbe4e8cc4704960f602/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/ID.java#L56) 构造方法，解析字符串，生成 ID 。

### 2.1.1 GlobalIdGenerator

[`org.skywalking.apm.agent.core.context.ids.GlobalIdGenerator`](https://github.com/YunaiV/skywalking/blob/9db53fd95e1c4c10f0f7a939e4484d7e2102ad3f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/GlobalIdGenerator.java) ，全局编号生成器。

[`#generate()`](https://github.com/YunaiV/skywalking/blob/9db53fd95e1c4c10f0f7a939e4484d7e2102ad3f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/GlobalIdGenerator.java#L62) 方法，生成 ID 对象。代码如下：

* 第 67 行：获得线程对应的 IDContext 对象。
* 第 69 至 73 行：生成 ID 对象。
    * 第 70 行：`ID.part1` 属性，应用编号实例。
    * 第 71 行：`ID.part2` 属性，线程编号。
    * 第 72 行：`ID.part3` 属性，调用 [`IDContext#nextSeq()`](https://github.com/YunaiV/skywalking/blob/9db53fd95e1c4c10f0f7a939e4484d7e2102ad3f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/GlobalIdGenerator.java#L102) 方法，生成带有时间戳的序列号。
* ps ：代码比较易懂，已经添加完成注释。

### 2.1.2 DistributedTraceId

[`org.skywalking.apm.agent.core.context.ids.DistributedTraceId`](https://github.com/YunaiV/skywalking/blob/5fb841b3ae5b78f07d06c6186adf9a8c08295a07/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/DistributedTraceId.java) ，分布式链路追踪编号**抽象类**。

* `id` 属性，全局编号。

DistributedTraceId 有两个实现类：

* [org.skywalking.apm.agent.core.context.ids.NewDistributedTraceId](https://github.com/YunaiV/skywalking/blob/5fb841b3ae5b78f07d06c6186adf9a8c08295a07/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/NewDistributedTraceId.java) ，**新建的**分布式链路追踪编号。当全局链路追踪开始，创建 TraceSegment 对象的过程中，会调用 [`DistributedTraceId()` 构造方法](https://github.com/YunaiV/skywalking/blob/5fb841b3ae5b78f07d06c6186adf9a8c08295a07/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/NewDistributedTraceId.java#L30)，创建 DistributedTraceId 对象。该构造方法内部会调用 `GlobalIdGenerator#generate()` 方法，创建 ID 对象。
* [org.skywalking.apm.agent.core.context.ids.PropagatedTraceId](https://github.com/YunaiV/skywalking/blob/5fb841b3ae5b78f07d06c6186adf9a8c08295a07/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/PropagatedTraceId.java) ，**传播的**分布式链路追踪编号。例如，A 服务调用 B 服务时，A 服务会将 DistributedTraceId 对象带给 B 服务，B 服务会调用 [`PropagatedTraceId(String id)` 构造方法](https://github.com/YunaiV/skywalking/blob/5fb841b3ae5b78f07d06c6186adf9a8c08295a07/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/PropagatedTraceId.java#L30) ，创建 PropagatedTraceId 对象。该构造方法内部会解析 id ，生成 ID 对象。

### 2.1.3 DistributedTraceIds

[`org.skywalking.apm.agent.core.context.ids.DistributedTraceIds`](https://github.com/YunaiV/skywalking/blob/2961e9f539286ef91af1ff1ef7681d0a06f156b0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/DistributedTraceIds.java) ，DistributedTraceId 数组的封装。

* `relatedGlobalTraces` 属性，关联的 DistributedTraceId **链式**数组。
* 

## 2.2 AbstractSpan

layer

tag

Component

## 2.3 TraceSegmentRef

## 2.4 TraceSegment

# 3. Context

