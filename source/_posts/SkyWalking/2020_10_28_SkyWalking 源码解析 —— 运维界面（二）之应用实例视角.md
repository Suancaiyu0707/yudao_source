# 1. 概述

本文主要分享**运维界面的第一部分，应用视角**。

> SkyWalking WEBUI ：https://github.com/apache/incubator-skywalking-ui

在我们打开 SkyWalking WEBUI 的 `Instance Override` ( `health/health.html` ) 页时，如下图：

[](http://www.iocoder.cn/images/SkyWalking/2020_10_28/01.png)

* 以应用实例为维度进行展示。
* 红色部分，时间进度条，调用 [「2. AllInstanceLastTimeGetHandler」](#) 接口，获得应用实例最后心跳时间。大多情况下，我们进入该界面，看的是从最后心跳时间开始的应用调用情况。
* 绿色部分，应用列表，初始化以 [ 实例最后心跳时间前一小时 , 实例最后心跳时间 ] 调用 [「3. ApplicationsGetHandler」](#) 接口，获得**范围**数据，而后每 4 秒刷新一次，数据范围向前走 4 秒( 时间进度条是走 2 次 2 秒 )，为 [ 实例最后心跳时间前一小时 + 四秒 , 实例最后心跳时间 + 四秒 ]。 
* 红色部分，【点击任意应用】，应用实例列表，调用 [「4. InstanceHealthGetHandler」](#) 接口，获得**当前时间**数据，而后每 2 秒刷新一次。
* 通过进度条的【**播放/暂停按钮**】可以切换到暂停模式，切换时间范围，查看时间范围为一小时内的应用调用拓扑图。注意，一定要切换到【暂时模式】才可调整时间范围，在【播放模式】下，每次**自动**请求都会重设时间范围。

在我们【点击任意应用实例】，打开 SkyWalking WEBUI 的 `Instance` ( `instance/instance.html` ) 页时，如下图：

[](http://www.iocoder.cn/images/SkyWalking/2020_10_28/02.png)

* 以**单个**应用实例为维度进行展示。
* 橘色部分，应用实例指标，初始化以 [ 打开页面时间前五分钟 , 打开页面时间 ] 调用 [「5. InstanceMetricGetRangeTimeBucketHandler」](#) 接口，获得**范围**数据。
* 红色部分，【点击 auto 开关】，每 1 秒刷新一次，数据范围向前走 1 秒，为 [ 打开页面时间 + 一秒 , 打开页面时间 + 一秒 ]，获得**每秒增量**数据。

> 基情提示：运维界面相关 HTTP 接口，逻辑简单易懂，笔者写的会比较简略一些。 

# 2. AllInstanceLastTimeGetHandler

同 [《SkyWalking 源码分析 —— 运维界面（一）之应用视角》「2. AllInstanceLastTimeGetHandler」](http://www.iocoder.cn/SkyWalking/ui-1-application/?self) 相同。

# 3. ApplicationsGetHandler

[`org.skywalking.apm.collector.ui.jetty.handler.application.ApplicationsGetHandler`](https://github.com/YunaiV/skywalking/blob/a5db282a68747668356a1bc55e9227bd2b7869a0/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/application/ApplicationsGetHandler.java) ，实现 JettyHandler 接口，获得应用列表逻辑处理器。

* [`#pathSpec()`](https://github.com/YunaiV/skywalking/blob/a5db282a68747668356a1bc55e9227bd2b7869a0/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/application/ApplicationsGetHandler.java#L40) ，路径定义，`"applications"` 。
* 响应示例：[](http://www.iocoder.cn/images/SkyWalking/2020_10_28/03.png)
* [`#doGet()`](https://github.com/YunaiV/skywalking/blob/9f2dab1c61b49610eca0fc2634ee7af918ba7d1f/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/time/AllInstanceLastTimeGetHandler.java#L53) 方法，代码如下：
    * 第 73 行：调用 [`ApplicationService#getApplications(startTime, endTime)`](https://github.com/YunaiV/skywalking/blob/a5db282a68747668356a1bc55e9227bd2b7869a0/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/service/ApplicationService.java#L42) 方法，以应用编号为聚合，获得应用实例数量数组。代码如下：
        * 第 44 行：调用 [`NodeComponentEsUIDAO#load(startTime, endTime)`](https://github.com/YunaiV/skywalking/blob/a5db282a68747668356a1bc55e9227bd2b7869a0/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/InstanceEsUIDAO.java#L112) 方法，以应用编号为聚合，获得应用实例数量 JSON 数组。
        * 第 47 至 52 行：设置应用编码。

# 4. InstanceHealthGetHandler

[`org.skywalking.apm.collector.ui.jetty.handler.instancehealth.InstanceHealthGetHandler`](https://github.com/YunaiV/skywalking/blob/68b704ef2395067fdb135262089c5c3d316efee7/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/instancehealth/InstanceHealthGetHandler.java) ，实现 JettyHandler 接口，获得应用的应用实例健康相关信息数组。
 
* [`#pathSpec()`](https://github.com/YunaiV/skywalking/blob/68b704ef2395067fdb135262089c5c3d316efee7/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/instancehealth/InstanceHealthGetHandler.java#L42) ，路径定义，`"/instance/health/applicationId"` 。
* 响应示例：[](http://www.iocoder.cn/images/SkyWalking/2020_10_28/04.png)
* [`#doGet()`](https://github.com/YunaiV/skywalking/blob/68b704ef2395067fdb135262089c5c3d316efee7/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/jetty/handler/instancehealth/InstanceHealthGetHandler.java#L52) 方法，代码如下： 
    * 第 58 至 62 行：解析 `timeBucket` 参数，秒级。
    * 第 65 至 72 行：解析 `applicationIds` 参数，应用编号**数组**。 
    * 第 75 至 79 行：返回字段设置。
    * 第 82 至 85 行：循环应用编号数组，调用 [`InstanceHealthService#getInstances(timeBucket, applicationId)`](https://github.com/YunaiV/skywalking/blob/68b704ef2395067fdb135262089c5c3d316efee7/apm-collector/apm-collector-ui/collector-ui-jetty-provider/src/main/java/org/skywalking/apm/collector/ui/service/InstanceHealthService.java#L76) 方法，获得应用的应用实例健康相关信息数组。代码如下：
        * 第 80 行：获得指定时间内的 **5 秒内**的数组，倒序。为什么？见下文的 InstPerformance 的查询。
        * 第 81 至 82 行：调用 [`InstanceEsUIDAO#getInstances(applicationId, timeBucket)`](https://github.com/YunaiV/skywalking/blob/68b704ef2395067fdb135262089c5c3d316efee7/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/InstanceEsUIDAO.java#L158) 方法，查询查询半小时内有**心跳**的 Instance 数组。
        * 第 90 行：**循环** Instance 数组，逐个查询应用实例的健康相关信息。
        * 第 98 行：调用 [`InstPerformanceEsUIDAO#get(timeBuckets, instanceId)`](https://github.com/YunaiV/skywalking/blob/68b704ef2395067fdb135262089c5c3d316efee7/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/InstPerformanceEsUIDAO.java#L50) 方法，查询应用实例**五秒内**的( `timeBuckets` )的 InstPerformance **累加**数据。
            * 第 100 至 105 行：基于 InstPerformance 数据，设置 `tps` 返回字段。
            * 第 108 至 121 行：基于 InstPerformance 数据，设置 `avg` 和 `healthLevel` 返回数据。
        * 第 124 至 130 行：基于 Instance 数据，设置应用实例是否存活( 两分钟内是否有心跳 )。
        * 第 133 至 135 行：调用 [`GCMetricEsUIDAO#getGCCount(timeBuckets, instanceId)`](https://github.com/YunaiV/skywalking/blob/68b704ef2395067fdb135262089c5c3d316efee7/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/GCMetricEsUIDAO.java#L56) 方法，查询应用实例**五秒内**的( `timeBuckets` )的 GCCount **累加**数据，设置 `ygc` 和 `ogc` 返回字段。

# 5. InstanceMetricGetRangeTimeBucketHandler


