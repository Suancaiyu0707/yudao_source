title: SkyWalking 源码分析 —— 运维界面（四）之操作视角
date: 2020-11-05
tags:
categories: SkyWalking
permalink: SkyWalking/ui-4-operation

-------



# 1. 概述

本文主要分享**运维界面的第四部分，操作视角**。

> SkyWalking WEBUI ：https://github.com/apache/incubator-skywalking-ui

在我们打开 SkyWalking WEBUI 的 `Service Tree` ( `service/serviceTree.html` ) 页时，如下图：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_05/01.png)

* 以操作为维度进行展示。
* 黄色部分，时间进度条，调用 [「2. AllInstanceLastTimeGetHandler」](#) 接口，获得应用实例最后心跳时间。大多情况下，我们进入该界面，看的是从最后心跳时间开始的操作情况。
* 红色部分，应用筛选器，调用 [「3. ApplicationsGetHandler」](#) 接口，获得应用列表。
* 紫色部分，TraceSegment 分页列表，调用 [「3. SegmentTopGetHandler」](#) 接口，获得数据。
* 蓝色部分，【点击单个操作】，单条 Span 记录详情，调用 [「5. SpanGetHandler」](#) 接口，获得数据。

> 基情提示：运维界面相关 HTTP 接口，逻辑简单易懂，笔者写的会比较简略一些。 

# 2. AllInstanceLastTimeGetHandler

同 [《SkyWalking 源码分析 —— 运维界面（一）之应用视角》「2. AllInstanceLastTimeGetHandler」](http://www.iocoder.cn/SkyWalking/ui-1-application/?self) 相同。

# 3. ApplicationsGetHandler

同 [《SkyWalking 源码分析 —— 运维界面（二）之应用实例视角》「3. ApplicationsGetHandler」](http://www.iocoder.cn/SkyWalking/ui-2-instance/?self) 相同。

# 4. SegmentTopGetHandler

# 5. SpanGetHandler



# 6. 彩蛋

水更第四发！

[](http://www.iocoder.cn/images/SkyWalking/2020_10_25/05.png)

胖友，分享一波朋友圈可好？

