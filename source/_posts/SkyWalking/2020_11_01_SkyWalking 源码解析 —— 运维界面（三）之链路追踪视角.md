# 1. 概述

本文主要分享**运维界面的第三部分，链路追踪视角**。

> SkyWalking WEBUI ：https://github.com/apache/incubator-skywalking-ui

在我们打开 SkyWalking WEBUI 的 `Trace Stack` ( `trace/trace.html` ) 页时，如下图：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_01/01.png)

* 以应用实例为维度进行展示。
* 红色部分，应用筛选器，调用 [「2. ApplicationsGetHandler」](#) 接口，获得应用列表。
* 紫色部分，TraceSegment 分页列表，调用 [「3. SegmentTopGetHandler」](#) 接口，获得数据。
* 蓝色部分，【点击单条 TraceSegment】，**一次完整**的分布式链路追踪记录详情，调用 [「5. TraceStackGetHandler」](#) 接口，获得数据。
* 黄色部分，【点击单个 Span】，单条 Span 记录详情，调用 [「5. SpanGetHandler」](#) 接口，获得数据。

> 基情提示：运维界面相关 HTTP 接口，逻辑简单易懂，笔者写的会比较简略一些。 

# 2. ApplicationsGetHandler

同 [《SkyWalking 源码分析 —— 运维界面（二）之应用实例视角》「3. ApplicationsGetHandler」](http://www.iocoder.cn/SkyWalking/ui-2-instance/?self) 相同。

# 3. SegmentTopGetHandler

[`org.skywalking.apm.collector.ui.jetty.handler.SegmentTopGetHandler`](https://github.com/YunaiV/skywalking/blob/826af725e7477b5d8d49a479a5cbbdee021c8306/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/SegmentTopGetHandler.java) ，实现 JettyHandler 接口，获得 TraceSegment 分页列表的逻辑处理器。

* [`#pathSpec()`](https://github.com/YunaiV/skywalking/blob/826af725e7477b5d8d49a479a5cbbdee021c8306/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/SegmentTopGetHandler.java#L42) ，路径定义，`"/segment/top"` 。
* 响应示例：[](http://www.iocoder.cn/images/SkyWalking/2020_11_01/03.png)
* [`#doGet()`](https://github.com/YunaiV/skywalking/blob/826af725e7477b5d8d49a479a5cbbdee021c8306/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/SegmentTopGetHandler.java#L52) 方法，代码如下：
    * 第 62 至 73 行：解析 `startTime` 和 `endTime` 参数。
    * 第 75 至 87 行：解析 `from` 和 `limit` **分页**参数。
    * 第 90 至 97 行：解析 `minCost` 和 `maxCost` 参数。
    * 第 100 至 103 行：解析 `globalTraceId` 参数。
    * 第 106 至 109 行：解析 `operationName` 参数。
    * 第 112 至 117 行：解析 `applicationId` 参数。
    * 第 120 至 132 行：解析 `error` 参数。
    * 第 135 至 141 行：解析 `sort` 排序**键**参数，倒序。
    * 第 73 行：调用 [`SegmentTopService#load(...)`](https://github.com/YunaiV/skywalking/blob/826af725e7477b5d8d49a479a5cbbdee021c8306/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/service/SegmentTopService.java#L53) 方法，，获得 TraceSegment 分页列表。代码如下：
        * 第 59 至 62 行：若存在 `globalTraceId` 参数，调用 [`GlobalTraceEsUIDAO#getSegmentIds(globalTraceId)`](https://github.com/YunaiV/skywalking/blob/826af725e7477b5d8d49a479a5cbbdee021c8306/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/GlobalTraceEsUIDAO.java#L65) 方法，查询对应的 TraceSegment 编号数组。**为什么需要这么查询呢**？在下面，我们会看到，实际查询的是 SegmentCost 表，该表不存在 `global_trace_id` ，所以需要查询到 `segment_id` 数组，作为查询条件。
        * 第 65 行：调用 [`SegmentCostEsUIDAO#loadTop(...)`](https://github.com/YunaiV/skywalking/blob/826af725e7477b5d8d49a479a5cbbdee021c8306/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/SegmentCostEsUIDAO.java#L50) 方法，查询 **SegmentCost** 数组。
        * 第 68 至 77 行：循环 **SegmentCost** 数组，调用 [`GlobalTraceEsUIDAO#getGlobalTraceId(segmentId)`](https://github.com/YunaiV/skywalking/blob/826af725e7477b5d8d49a479a5cbbdee021c8306/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/GlobalTraceEsUIDAO.java#L46) 方法，获得每个 SegmentCost 对应的 `global_trace_id` 属性，并设置返回。

# 4. TraceStackGetHandler

# 5. SpanGetHandler

[`org.skywalking.apm.collector.ui.jetty.handler.SpanGetHandler`](https://github.com/YunaiV/skywalking/blob/c3f55e55593158e065b9589855ca90e819558765/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/SpanGetHandler.java) ，实现 JettyHandler 接口，获得 TraceSegment 单个 Span 详细的逻辑处理器。

* [`#pathSpec()`](https://github.com/YunaiV/skywalking/blob/c3f55e55593158e065b9589855ca90e819558765/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/SpanGetHandler.java#L40) ，路径定义，`"/span/spanId"` 。
* 响应示例：[](http://www.iocoder.cn/images/SkyWalking/2020_11_01/02.png)
* [`#doGet()`](https://github.com/YunaiV/skywalking/blob/c3f55e55593158e065b9589855ca90e819558765/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/SpanGetHandler.java#L50) 方法，代码如下：
    * 第 52 行：解析 `segmentId` 参数。
    * 第 55 至 62 行：解析 `spanId` 参数。
    * 第 73 行：调用 [`SpanService#load(segmentId, spanId)`](https://github.com/YunaiV/skywalking/blob/c3f55e55593158e065b9589855ca90e819558765/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/service/SpanService.java#L54) 方法，获得 TraceSegment 单个 Span 详细。代码如下：
        * 第 44 行：调用 [`SegmentEsUIDAO#load(segmentId)`](https://github.com/YunaiV/skywalking/blob/c3f55e55593158e065b9589855ca90e819558765/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/SegmentEsUIDAO.java#L45) 方法，获得 TraceSegment 。
        * 第 58 至 139 行：循环获得的 TraceSegment 的 Span 数组，找到对应的 Span 记录，设置后返回。

# 6. 彩蛋

