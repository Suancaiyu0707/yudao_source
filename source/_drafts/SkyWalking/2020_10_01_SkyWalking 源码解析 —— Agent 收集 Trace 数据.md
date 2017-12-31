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

[`#append(DistributedTraceId)`](https://github.com/YunaiV/skywalking/blob/ad259ad680df86296036910ede262765ffb44e5e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/DistributedTraceIds.java#L50) 方法，添加分布式链路追踪编号( DistributedTraceId )。代码如下：

* 第 51 至 54 行：移除**首个** NewDistributedTraceId 对象。为什么呢？在 [「2.4 TraceSegment」](#) 的构造方法中，会默认创建 NewDistributedTraceId 对象。在跨线程、或者跨进程的情况下时，创建的 TraceSegment 对象，需要指向父 Segment 的 DistributedTraceId ，所以需要移除默认创建的。
* 第 56 至 58 行：添加 DistributedTraceId 对象到数组。

## 2.2 AbstractSpan

[`org.skywalking.apm.agent.core.context.trace.AbstractSpan`](https://github.com/YunaiV/skywalking/blob/96fd1f0aacb995f725c446b1cfcdc3124058e6a6/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/AbstractSpan.java) ，Span **接口**( 不是抽象类 )，定义了 Span 通用属性的接口方法：

* `#getSpanId()` 方法，获得 Span 编号。一个整数，在 TraceSegment 内**唯一**，从 0 开始自增，在创建 Span 对象时生成。
* `#setOperationName(operationName)` 方法，设置操作名。
    * 操作名，定义如下：[](http://www.iocoder.cn/images/SkyWalking/2020_10_01/01.png)
    * `#setOperationId(operationId)` 方法，设置操作编号。考虑到操作名是字符串，Agent 发送给 Collector 占用流量较大。因此，Agent 会将操作注册到 Collector ，生成操作编号。在 [《SkyWalking 源码分析 —— Agent DictionaryManager 字典管理》](http://www.iocoder.cn/SkyWalking/agent-dictionary/?self) 有详细解析。
* `#setComponent(Component)` 方法，设置 [`org.skywalking.apm.network.trace.component.Component`](https://github.com/YunaiV/skywalking/blob/a51e197a78f82400edae5c33b523ba1cb5224b8f/apm-network/src/main/java/org/skywalking/apm/network/trace/component/Component.java) ，例如：MongoDB / SpringMVC / Tomcat 等等。目前，官方在 [`org.skywalking.apm.network.trace.component.ComponentsDefine`](https://github.com/YunaiV/skywalking/blob/a51e197a78f82400edae5c33b523ba1cb5224b8f/apm-network/src/main/java/org/skywalking/apm/network/trace/component/ComponentsDefine.java) 定义了目前已经支持的 Component 。
    * `#setComponent(componentName)` 方法，直接设置 Component 名字。大多数情况下，我们不使用该方法。

        > Only use this method in explicit instrumentation, like opentracing-skywalking-bridge. 
        > It it higher recommend don't use this for performance consideration.

* `#setLayer(SpanLayer)` 方法，设置 [`org.skywalking.apm.agent.core.context.trace.SpanLayer`](https://github.com/YunaiV/skywalking/blob/a51e197a78f82400edae5c33b523ba1cb5224b8f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/NoopSpan.java) 。目前有，DB 、RPC_FRAMEWORK 、HTTP 、MQ ，未来会增加 CACHE 。

* `#tag(key, value)` 方法，设置键值对的标签。可以调用多次，构成 Span 的标签集合。在 [「2.2.1 Tag」](#) 详细解析。
* 日志相关
    * `#log(timestampMicroseconds, fields)` 方法，记录一条通用日志，包含 `fields` 键值对集合。
    * `#log(Throwable)` 方法，记录一条异常日志，包含异常信息。 
* `#errorOccurred()` 方法，标记发生异常。大多数情况下，配置 `#log(Throwable)` 方法一起使用。
* `#start()` 方法，开始 Span 。一般情况的实现，设置开始时间。
* `#isEntry()` 方法，是否是入口 Span ，在 [「2.2.2.1 EntrySpan」](#) 详细解析。
* `#isExit()` 方法，是否是出口 Span ，在 [「2.2.2.2 ExitSpan」](#) 详细解析。

### 2.2.1 Tag

#### 2.2.1.1 AbstractTag

[`org.skywalking.apm.agent.core.context.tag.AbstractTag<T>`](https://github.com/YunaiV/skywalking/blob/e0c449745dfabe847b2e918d5352381f191a4469/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/tag/AbstractTag.java) ，标签**抽象类**。注意，这个类的用途是将标签属性设置到 Span 上，或者说，它是设置 Span 的标签的**工具类**。代码如下：

* `key` 属性，标签的键。
* `#set(AbstractSpan span, T tagValue)` **抽象**方法，设置 Span 的标签键 `key` 的值为 `tagValue` 。

#### 2.2.1.2 StringTag

[`org.skywalking.apm.agent.core.context.tag.StringTag`](https://github.com/YunaiV/skywalking/blob/e0c449745dfabe847b2e918d5352381f191a4469/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/tag/StringTag.java) ，值类型为 String 的标签**实现类**。

* `#set(AbstractSpan span, String tagValue)` **实现**方法，设置 Span 的标签键 `key` 的值为 `tagValue` 。

#### 2.2.1.3 Tags

[`org.skywalking.apm.agent.core.context.tag.Tags`](https://github.com/YunaiV/skywalking/blob/e0c449745dfabe847b2e918d5352381f191a4469/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/tag/Tags.java) ，**常用** Tag **枚举类**，内部定义了**多个** HTTP 、DB 相关的 StringTag 的静态变量。

在 [《opentracing-specification-zh —— 语义惯例》](https://github.com/opentracing-contrib/opentracing-specification-zh/blob/master/semantic_conventions.md#%E6%A0%87%E5%87%86%E7%9A%84span-tag-%E5%92%8C-log-field) 里，定义了标准的 Span Tag 。

### 2.2.2 AbstractSpan 实现类

AbstractSpan 实现类如下图：[](http://www.iocoder.cn/images/SkyWalking/2020_10_01/03.png)

* 左半边的 Span 实现类：**有**具体操作的 Span 。
* 右半边的 Span 实现类：**无**具体操作的 Span ，和左半边的 Span 实现类**相对**，用于不需要收集 Span 的场景。

抛开右半边的 Span 实现类的特殊处理，Span 只有三种实现类：

* EntrySpan ：入口 Span
* LocalSpan ：本地 Span
* ExitSpan ：出口 Span

下面，我们分小节逐步分享。

#### 2.2.2.1 AbstractTracingSpan

[`org.skywalking.apm.agent.core.context.trace.AbstractTracingSpan`](https://github.com/YunaiV/skywalking/blob/11b66b8d36943d6492f51c676b455f29c9c0abc6/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/AbstractTracingSpan.java) ，实现 AbstractSpan 接口，链路追踪 Span **抽象类**。

在创建 AbstractTracingSpan 时，会传入 `spanId` , `parentSpanId` , `operationName` / `operationId` 参数。参见构造方法：

* `#AbstractTracingSpan(spanId, parentSpanId, operationName)`
* `#AbstractTracingSpan(spanId, parentSpanId, operationId)`

-------

大部分是 setting / getting 方法，或者类似方法，已经添加注释，胖友自己阅读。

[`#finish(TraceSegment)`](https://github.com/YunaiV/skywalking/blob/11b66b8d36943d6492f51c676b455f29c9c0abc6/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/AbstractTracingSpan.java#L126) 方法，完成( 结束 ) Span ，将当前 Span ( 自己 )添加到 TraceSegment 。为什么会调用该方法，在 TODO 详细解析。

#### 2.2.2.2 StackBasedTracingSpan
 [`org.skywalking.apm.agent.core.context.trace.StackBasedTracingSpan`](https://github.com/YunaiV/skywalking/blob/c1e513b4581443e7ca720f4e9c91ad97cc6f0de1/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/StackBasedTracingSpan.java) ，实现 AbstractTracingSpan 抽象类，基于**栈**的链路追踪 Span 抽象类。这种 Span 能够被多次调用 `#start(...)` 和 `#finish(...)` 方法，在类似堆栈的调用中。在 [「2.2.2.2.1 EntrySpan」](#) 中详细举例子。代码如下：
 
* `stackDepth` 属，**栈**深度。
* [`#finish(TraceSegment)`](https://github.com/YunaiV/skywalking/blob/c1e513b4581443e7ca720f4e9c91ad97cc6f0de1/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/StackBasedTracingSpan.java#L52) **实现**方法，完成( 结束 ) Span ，将当前 Span ( 自己 )添加到 TraceSegment 。**当且仅当 `stackDepth == 0` 时，添加成功**。代码如下：
    * 第 53 至 73 行：栈深度为零，出栈成功。调用 `super#finish(TraceSegment)` 方法，完成( 结束 ) Span ，将当前 Span ( 自己 )添加到 TraceSegment 。
        * 第 55 至 72 行：当操作编号为空时，尝试使用操作名获得操作编号并设置。用于**减少** Agent 发送 Collector 数据的网络流量。
    * 第 74 至 76 行：栈深度非零，出栈失败。

##### 2.2.2.2.1 EntrySpan

**重点**

[`org.skywalking.apm.agent.core.context.trace.EntrySpan`](https://github.com/YunaiV/skywalking/blob/d36f6a47a208720f4caac9d9a8b7263bd36f2187/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/EntrySpan.java) ，实现 StackBasedTracingSpan 抽象类，**入口** Span ，用于服务提供者( Service Provider ) ，例如 Tomcat 。

EntrySpan 是 TraceSegment 的第一个 Span ，这也是为什么称为"**入口**" Span 的原因。

**那么为什么 EntrySpan 继承 StackBasedTracingSpan** ？

例如，我们常用的 SprintBoot 场景下，Agent 会在 SkyWalking 插件在 Tomcat 定义的方法切面，创建 EntrySpan 对象，也会在 SkyWalking 插件在 SpringMVC 定义的方法切面，创建 EntrySpan 对象。那岂不是出现**两个** EntrySpan ，一个 TraceSegment 出现了两个入口 Span ？

答案是当然不会！Agent 只会在第一个方法切面，生成 EntrySpan 对象，第二个方法切面，栈深度 **+ 1**。这也是上面我们看到的 `#finish(TraceSegment)` 方法，只在栈深度为零时，出栈成功。通过这样的方式，保持一个 TraceSegment 有且仅有一个 EntrySpan 对象。

当然，多个 TraceSegment 会有多个 EntrySpan 对象 ，例如【服务 A】远程调用【服务 B】。

另外，虽然 EntrySpan 在第一个服务提供者创建，EntrySpan 代表的是最后一个服务提供者，例如，上面的例子，EntrySpan 代表的是 Spring MVC 的方法切面。所以，`startTime` 和 `endTime` 以第一个为准，`componentId` 、`componentName` 、`layer` 、`logs` 、`tags` 、`operationName` 、`operationId` 等等以最后一个为准。并且，一般情况下，最后一个服务提供者的信息也会**更加详细**。

**ps**：如上内容信息量较大，胖友可以对照着实现方法，在理解理解。HOHO ，良心笔者当然也是加了注释的。

如下是一个 EntrySpan 在 SkyWalking 展示的例子：[](http://www.iocoder.cn/images/SkyWalking/2020_10_01/04.png)

##### 2.2.2.2.2 ExitSpan

**重点**

[`org.skywalking.apm.agent.core.context.trace.ExitSpan`](https://github.com/YunaiV/skywalking/blob/958830d8db481b5b8a70498a09bc18eb7c721737/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/ExitSpan.java) ，继承 StackBasedTracingSpan 抽象类，**出口** Span ，用于服务消费者( Service Consumer ) ，例如 HttpClient 、MongoDBClient 。

-------

ExitSpan 实现 [`org.skywalking.apm.agent.core.context.trace.WithPeerInfo`](https://github.com/YunaiV/skywalking/blob/958830d8db481b5b8a70498a09bc18eb7c721737/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/WithPeerInfo.java) 接口，代码如下：

* `peer` 属性，节点地址。
* `peerId` 属性，节点编号。

如下是一个 ExitSpan 在 SkyWalking 展示的例子：[](http://www.iocoder.cn/images/SkyWalking/2020_10_01/05.png)

-------

**那么为什么 ExitSpan 继承 StackBasedTracingSpan** ？

例如，我们可能在使用的 Dubbox 场景下，【Dubbox 服务 A】使用 HTTP 调用【Dubbox 服务 B】时，实际过程是，【Dubbox 服务 A】=》【HttpClient】=》【Dubbox 服务 B】。Agent 会在【Dubbox 服务 A】创建 ExitSpan 对象，也会在 【HttpClient】创建 ExitSpan 对象。那岂不是**一次出口**，出现**两个** ExitSpan ？

答案是当然不会！Agent 只会在【Dubbox 服务 A】，生成 EntrySpan 对象，第二个方法切面，栈深度 **+ 1**。这也是上面我们看到的 `#finish(TraceSegment)` 方法，只在栈深度为零时，出栈成功。通过这样的方式，保持**一次出口**有且仅有一个 ExitSpan 对象。

当然，一个 TraceSegment 会有多个 ExitSpan 对象 ，例如【服务 A】远程调用【服务 B】，然后【服务 A】再次远程调用【服务 B】，或者然后【服务 A】远程调用【服务 C】。

另外，虽然 ExitSpan 在第一个消费者创建，ExitSpan 代表的也是第一个服务提消费者，例如，上面的例子，ExitSpan 代表的是【Dubbox 服务 A】。

**ps**：如上内容信息量较大，胖友可以对照着实现方法，在理解理解。HOHO ，良心笔者当然也是加了注释的。

#### 2.2.2.3 LocalSpan 

[`org.skywalking.apm.agent.core.context.trace.LocalSpan`](https://github.com/YunaiV/skywalking/blob/96fd1f0aacb995f725c446b1cfcdc3124058e6a6/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/LocalSpan.java) ，继承 AbstractTracingSpan 抽象类，本地 Span ，用于一个普通方法的链路追踪，例如本地方法。

如下是一个 EntrySpan 在 SkyWalking 展示的例子：[](http://www.iocoder.cn/images/SkyWalking/2020_10_01/06.png)

#### 2.2.2.4 NoopSpan

[`org.skywalking.apm.agent.core.context.trace.NoopSpan`](https://github.com/YunaiV/skywalking/blob/f00d2f405ca23e89778febeb4ada7b389858f258/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/NoopSpan.java) ，实现 AbstractSpan 接口，**无操作**的 Span 。配置 IgnoredTracerContext 一起使用，在 IgnoredTracerContext 声明[单例](https://github.com/YunaiV/skywalking/blob/f00d2f405ca23e89778febeb4ada7b389858f258/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/IgnoredTracerContext.java#L37) ，以减少不收集 Span 时的对象创建，达到减少内存使用和 GC 时间。

##### 2.2.2.3.1 NoopExitSpan

[`org.skywalking.apm.agent.core.context.trace.NoopExitSpan`](https://github.com/YunaiV/skywalking/blob/f00d2f405ca23e89778febeb4ada7b389858f258/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/ExitSpan.java) ，实现 [`org.skywalking.apm.agent.core.context.trace.WithPeerInfo`](https://github.com/YunaiV/skywalking/blob/958830d8db481b5b8a70498a09bc18eb7c721737/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/WithPeerInfo.java) 接口，继承 StackBasedTracingSpan 抽象类，**出口** Span ，无操作的**出口** Span 。和 ExitSpan **相对**，不记录服务消费者的出口 Span 。

## 2.3 TraceSegmentRef

[`org.skywalking.apm.agent.core.context.trace.TraceSegmentRef`](https://github.com/YunaiV/skywalking/blob/49dc81a8bcaad1879b3a3be9917944b0b8b5a7a4/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/TraceSegmentRef.java) ，TraceSegment 指向，通过 `traceSegmentId` 和 `spanId` 属性，指向父级 TraceSegment 的指定 Span 。

* `type` 属性，指向类型( [SegmentRefType](https://github.com/YunaiV/skywalking/blob/49dc81a8bcaad1879b3a3be9917944b0b8b5a7a4/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/TraceSegmentRef.java#L206) ) 。不同的指向类型，使用不同的构造方法。
    * `CROSS_PROCESS` ，跨进程，例如远程调用，对应构造方法 [#TraceSegmentRef(ContextCarrier)](https://github.com/YunaiV/skywalking/blob/49dc81a8bcaad1879b3a3be9917944b0b8b5a7a4/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/TraceSegmentRef.java#L97) 。 
    * `CROSS_THREAD` ，跨线程，例如异步线程任务，对应构造方法 [#TraceSegmentRef(ContextSnapshot)](https://github.com/YunaiV/skywalking/blob/49dc81a8bcaad1879b3a3be9917944b0b8b5a7a4/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/TraceSegmentRef.java#L123) 。
    * 构造方法的代码，在 [「3. Context」](#) 中，伴随着调用过程，一起解析。

* `traceSegmentId` 属性，**父** TraceSegment 编号。**重要**
* `spanId` 属性，**父** Span 编号。**重要**
* `peerId` 属性，todo
* `peerHost` 属性，todo
* `entryApplicationInstanceId` 属性，**入口**应用实例编号。例如，在一个分布式链路 `A->B->C` 中，此字段为 A 应用的实例编号。
* `parentApplicationInstanceId` 属性，**父**应用实例编号。
* `entryOperationName` 属性，**入口**操作名。
* `entryOperationId` 属性，**入口**操作编号。
* `parentOperationName` 属性，**父**操作名。
* `parentOperationId` 属性，**父**操作编号。

## 2.4 TraceSegment

在看完了 TraceSegment 的各个元素，我们来看看 TraceSegment 内部实现的方法。

[TraceSegment 构造方法](https://github.com/YunaiV/skywalking/blob/ad259ad680df86296036910ede262765ffb44e5e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/TraceSegment.java#L79)，代码如下：

* 第 80 行：调用 `GlobalIdGenerator#generate()` 方法，生成 ID 对象，赋值给 `traceSegmentId` 。
* 第 81 行：创建 `spans` 数组。
    * [`#archive(AbstractTracingSpan)`](https://github.com/YunaiV/skywalking/blob/ad259ad680df86296036910ede262765ffb44e5e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/TraceSegment.java#L114) 方法，被 `AbstractSpan#finish(TraceSegment)` 方法调用，添加到 `spans` 数组。
* 第 83 至 84 行：创建 DistributedTraceIds 对象，并添加 NewDistributedTraceId 到它。
    * **注意**，当 TraceSegment 是一次分布式链路追踪的**首条**记录，创建的 NewDistributedTraceId 对象，即为分布式链路追踪的**全局编号**。
    * [`#relatedGlobalTraces(DistributedTraceId)`](https://github.com/YunaiV/skywalking/blob/ad259ad680df86296036910ede262765ffb44e5e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/TraceSegment.java#L104) 方法，添加 DistributedTraceId 对象。被 `TracingContext#continued(ContextSnapshot)` 或者 `TracingContext#extract(ContextCarrier)` 方法调用，在 [「3. Context」](#) 详细解析。

[`#ref(TraceSegmentRef)`](https://github.com/YunaiV/skywalking/blob/ad259ad680df86296036910ede262765ffb44e5e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/TraceSegment.java#L92) 方法，添加 TraceSegmentRef 对象，到 `refs` 属性，即**指向**父 Segment 。

# 3. Context

在 [「2. Trace」](#) 中，我们看了 Trace 的数据结构，本小节，我们一起来看看 Context 是怎么收集 Trace 数据的。

## 3.1 ContextManager

ContextListener




    public TraceSegmentRef(ContextCarrier carrier) {

    public TraceSegmentRef(ContextSnapshot snapshot) {


