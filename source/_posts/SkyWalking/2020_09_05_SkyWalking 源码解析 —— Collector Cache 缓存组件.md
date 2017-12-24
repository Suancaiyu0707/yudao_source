# 1. 概述

本文主要分享 **SkyWalking Collector Cache Module**，缓存组件。该组件用于缓存 Application 、Instance 、ServiceName 等**常用**且**不变**的数据，以提升性能。

> 友情提示：本文内容较为简单，胖友可快速阅读。

Cache Module 在 SkyWalking 架构图处于如下位置( **红框** ) ：

> FROM https://github.com/apache/incubating-skywalking  
> ![](http://www.iocoder.cn/images/SkyWalking/2020_09_05/01.jpeg)

下面我们来看看整体的项目结构，如下图所示 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_09_05/02.png)

* `collector-cache-define` ：定义缓存组件接口。
* `collector-cache-guava-provider` ：基于 [Google Guava](https://github.com/google/guava) 的缓存组件实现。

下面，我们从**接口到实现**的顺序进行分享。

# 2. collector-cache-define

`collector-cache-define` ：定义队列组件接口。项目结构如下 ：

![](http://www.iocoder.cn/images/SkyWalking/2020_09_05/03.png)

## 2.1 CacheModule

`org.skywalking.apm.collector.cache.CacheModule` ，实现 [Module](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java) 抽象类，缓存 Module 。

[`#name()`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-cache/collector-cache-define/src/main/java/org/skywalking/apm/collector/cache/CacheModule.java#L36) **实现**方法，返回模块名为 `"cache"` 。

[`#services()`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-cache/collector-cache-define/src/main/java/org/skywalking/apm/collector/cache/CacheModule.java#L40) **实现**方法，返回 Service 类名：ApplicationCacheService 、InstanceCacheService 、ServiceIdCacheService 、ServiceNameCacheService 。

## 2.2 ApplicationCacheService

[`org.skywalking.apm.collector.cache.service.ApplicationCacheService`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-cache/collector-cache-define/src/main/java/org/skywalking/apm/collector/cache/service/ApplicationCacheService.java) ，应用数据缓存服务**接口**。

* Table ：[`org.skywalking.apm.collector.storage.table.register.ApplicationTable`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/ApplicationTable.java)
* Data ：[`org.skywalking.apm.collector.storage.table.register.Application`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/Application.java)

## 2.3 InstanceCacheService

[`org.skywalking.apm.collector.cache.service.InstanceCacheService`](https://github.com/YunaiV/skywalking/blob/a1667a9ecf4eecf77cc2390f0af709c3b1bb7e4b/apm-collector/apm-collector-cache/collector-cache-define/src/main/java/org/skywalking/apm/collector/cache/service/InstanceCacheService.java) ，应用实例数据缓存服务**接口**。

* Table ：[`org.skywalking.apm.collector.storage.table.register.InstanceTable`](https://github.com/YunaiV/skywalking/blob/a1667a9ecf4eecf77cc2390f0af709c3b1bb7e4b/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/InstanceTable.java)
* Data ：[`org.skywalking.apm.collector.storage.table.register.Instance`](https://github.com/YunaiV/skywalking/blob/a1667a9ecf4eecf77cc2390f0af709c3b1bb7e4b/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/Instance.java)

## 2.4 ServiceNameCacheService

[`org.skywalking.apm.collector.cache.service.ServiceNameCacheService`](https://github.com/YunaiV/skywalking/blob/a859f4751203d73f10f30bb7c9cf2adfdecf955c/apm-collector/apm-collector-cache/collector-cache-define/src/main/java/org/skywalking/apm/collector/cache/service/ServiceNameCacheService.java) ，服务名数据缓存服务**接口**。  
[`org.skywalking.apm.collector.cache.service.ServiceIdCacheService`](https://github.com/YunaiV/skywalking/blob/a859f4751203d73f10f30bb7c9cf2adfdecf955c/apm-collector/apm-collector-cache/collector-cache-define/src/main/java/org/skywalking/apm/collector/cache/service/ServiceIdCacheService.java) ，服务编号数据缓存服务**接口**。

* Table ：[`org.skywalking.apm.collector.storage.table.register.ServiceNameTable`](https://github.com/YunaiV/skywalking/blob/a859f4751203d73f10f30bb7c9cf2adfdecf955c/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/ServiceNameTable.java)
* Data ：[`org.skywalking.apm.collector.storage.table.register.ServiceName`](https://github.com/YunaiV/skywalking/blob/a859f4751203d73f10f30bb7c9cf2adfdecf955c/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/ServiceName.java)

# 3. collector-cache-guava-provider

`collector-cache-guava-provider` ，基于 [Google Guava](https://github.com/google/guava) 的缓存组件实现。

项目结构如下 ：![](http://www.iocoder.cn/images/SkyWalking/2020_09_05/04.png)

**默认配置**，在 [`application-default.yml`](https://github.com/YunaiV/skywalking/blob/8b7205313e60e84d50579261992042c8b581492f/apm-collector/apm-collector-core/src/main/resources/application-default.yml#L5) **已经**配置如下：

``` JSON
cache:
  guava:
```

## 3.1 CacheModuleGuavaProvider

`org.skywalking.apm.collector.cache.guava.CacheModuleGuavaProvider` ，实现 [ModuleProvider](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java) **抽象类**，基于 Guava 的缓存组件服务提供者。

[`#name()`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-cache/collector-cache-guava-provider/src/main/java/org/skywalking/apm/collector/cache/guava/CacheModuleGuavaProvider.java#L44) **实现**方法，返回组件服务提供者名为 `"guava"` 。

[`module()`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-cache/collector-cache-guava-provider/src/main/java/org/skywalking/apm/collector/cache/guava/CacheModuleGuavaProvider.java#L48) **实现**方法，返回组件类为 CacheModule 。

[`#requiredModules()`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-cache/collector-cache-guava-provider/src/main/java/org/skywalking/apm/collector/cache/guava/CacheModuleGuavaProvider.java#L66) **实现**方法，返回依赖组件为空。

-------

[`#prepare(Properties)`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-cache/collector-cache-guava-provider/src/main/java/org/skywalking/apm/collector/cache/guava/CacheModuleGuavaProvider.java#L52) **实现**方法，执行准备阶段逻辑。

* 第 44 行 ：创建 ApplicationCacheGuavaService 、InstanceCacheGuavaService 、ServiceIdCacheGuavaService 、ServiceNameCacheGuavaService 对象，并调用 `#registerServiceImplementation()` **父类**方法，注册到 `services` 。

[`#start()`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-cache/collector-cache-guava-provider/src/main/java/org/skywalking/apm/collector/cache/guava/CacheModuleGuavaProvider.java#L59) **实现**方法，方法为空。

[`#notifyAfterCompleted()`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-cache/collector-cache-guava-provider/src/main/java/org/skywalking/apm/collector/cache/guava/CacheModuleGuavaProvider.java#L62) **实现**方法，方法为空。

## 3.2 ApplicationCacheGuavaService

[`org.skywalking.apm.collector.cache.guava.service.ApplicationCacheGuavaService`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-cache/collector-cache-guava-provider/src/main/java/org/skywalking/apm/collector/cache/guava/service/ApplicationCacheGuavaService.java) ，实现 ApplicationCacheService 接口，基于 Guava 的应用数据缓存服务**实现类**。

* EsDAO ：[`org.skywalking.apm.collector.storage.es.dao.ApplicationEsCacheDAO`](https://github.com/YunaiV/skywalking/blob/ed68f92bf1f5ac397c0bb0a5cc23fa4f3e3c32d1/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/ApplicationEsCacheDAO.java)

## 3.3 InstanceCacheGuavaService

[`org.skywalking.apm.collector.cache.guava.service.InstanceCacheGuavaService`](https://github.com/YunaiV/skywalking/blob/a1667a9ecf4eecf77cc2390f0af709c3b1bb7e4b/apm-collector/apm-collector-cache/collector-cache-guava-provider/src/main/java/org/skywalking/apm/collector/cache/guava/service/InstanceCacheGuavaService.java) ，实现 InstanceCacheService 接口，基于 Guava 的应用实例数据缓存服务**实现类**。

* EsDAO ：[`org.skywalking.apm.collector.storage.es.dao.InstanceEsCacheDAO`](https://github.com/YunaiV/skywalking/blob/a1667a9ecf4eecf77cc2390f0af709c3b1bb7e4b/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/InstanceEsCacheDAO.java)

## 3.4 ServiceNameCacheGuavaService

[`org.skywalking.apm.collector.cache.guava.service.ServiceNameCacheGuavaService`](https://github.com/YunaiV/skywalking/blob/a859f4751203d73f10f30bb7c9cf2adfdecf955c/apm-collector/apm-collector-cache/collector-cache-guava-provider/src/main/java/org/skywalking/apm/collector/cache/guava/service/ServiceIdCacheGuavaService.java) ，实现 ServiceNameCacheService 接口，基于 Guava 的服务名数据缓存服务**实现类**。  
[`org.skywalking.apm.collector.cache.guava.service.ServiceIdCacheGuavaService`](https://github.com/YunaiV/skywalking/blob/a859f4751203d73f10f30bb7c9cf2adfdecf955c/apm-collector/apm-collector-cache/collector-cache-guava-provider/src/main/java/org/skywalking/apm/collector/cache/guava/service/ServiceNameCacheGuavaService.java) ，实现 ServiceNameCacheService 接口，基于 Guava 的服务编号数据缓存服务**实现类**。

* EsDAO ：[`org.skywalking.apm.collector.storage.es.dao.ServiceNameEsCacheDAO`](https://github.com/YunaiV/skywalking/blob/a859f4751203d73f10f30bb7c9cf2adfdecf955c/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/ServiceNameEsCacheDAO.java)

# 666. 彩蛋

真的是一篇水文，都写的无从下手，只好罗列了相关的类。见谅。

![](http://www.iocoder.cn/images/SkyWalking/2020_09_05/05.png)


