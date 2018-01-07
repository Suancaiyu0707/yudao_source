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

# 3. SegmentTopGetHandler

# 4. TraceStackGetHandler

# 5. SpanGetHandler

# 6. 彩蛋

