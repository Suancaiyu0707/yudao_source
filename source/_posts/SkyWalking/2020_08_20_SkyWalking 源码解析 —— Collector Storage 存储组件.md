# 1. 概述

本文主要分享 **SkyWalking Collector Storage 存储组件**。顾名思义，负责将调用链路、应用、应用实例等等信息存储到存储器，例如，ES 、H2 。

> 友情提示：建议先阅读 [《SkyWalking 源码分析 —— Collector 初始化》](http://www.iocoder.cn/SkyWalking/collector-init/?self) ，以了解 Collector 组件体系。

> FROM https://github.com/apache/incubating-skywalking  
> [](http://www.iocoder.cn/images/SkyWalking/2020_08_20/01.jpeg)

下面我们来看看整体的项目结构，如下图所示 ：

[](http://www.iocoder.cn/images/SkyWalking/2020_08_20/02.png)

* `apm-collector-core` 的 `data` 和 `define` **包** ：数据的抽象。 
* `collector-storage-define` ：定义存储组件接口。
* `collector-storage-h2-provider` ：基于 H2 的 存储组件实现。**该实现是单机版，建议仅用于 SkyWalking 快速上手，生产环境不建议使用**。
* `collector-storage-es-provider` ：基于 Elasticsearch 的集群管理实现。**生产环境推荐使用**。

下面，我们从**接口到实现**的顺序进行分享。

# 2. apm-collector-core

`apm-collector-core` 的 `data` 和 `define` **包**，如下图所示：[](http://www.iocoder.cn/images/SkyWalking/2020_08_20/03.png)

我们对类进行梳理分类，如下图：[](http://www.iocoder.cn/images/SkyWalking/2020_08_20/04.png)

* Table ：Data 和 TableDefine 之间的桥梁，每个 Table 定义了该表的**表名**，**字段名们**。
* TableDefine ：Table 的详细定义，包括**表名**，**字段定义**( ColumnDefine )们。在下文中，[StorageInstaller](https://github.com/apache/incubator-skywalking/blob/15328202b8b7df89a609885d9110361ff29ce668/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/apache/skywalking/apm/collector/storage/StorageInstaller.java) 会基于 TableDefine 初始化表的相关信息。
* Data ：数据，包括**一条**数据的数据值们和数据字段( Column )们。在下文中，[Dao](https://github.com/apache/incubator-skywalking/blob/15328202b8b7df89a609885d9110361ff29ce668/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/apache/skywalking/apm/collector/storage/base/dao/DAO.java) 会存储 Data 到存储器中。另外，在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 中，我们也会看到对 Data 的流式处理**通用**封装。

## 2.1 Table

[`org.skywalking.apm.collector.core.data.CommonTable`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/CommonTable.java) ，通用表。

* [`TABLE_TYPE`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/CommonTable.java#L33) **静态**属性，表类型。目前只有 ES 存储组件使用到，下文详细解析。
* [`COLUMN_`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/CommonTable.java#L37) 前缀的**静态**属性，通用的字段名。

在 `collector-storage-define` 的 [`table`](https://github.com/YunaiV/skywalking/tree/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table) **包**下，我们可以看到所有 Table 类，以 `"Table"` 结尾。每个 Table 的表名，在每个实现类里，例如 [ApplicationTable](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/ApplicationTable.java) 。

## 2.2 TableDefine

`org.skywalking.apm.collector.core.data.TableDefine` ，表定义**抽象类**。

* [`name`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/TableDefine.java#L34) 属性，表名。
* [`columnDefines`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/TableDefine.java#L38) 属性，ColumnDefine数组。
* [`#initialize()`](https://github.com/YunaiV/skywalking/blob/578ea4f66f11bdfe5dcda25f574a1ed57ca47d24/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/TableDefine.java#L48) **抽象**方法，初始化表定义。例如：[ApplicationEsTableDefine](https://github.com/YunaiV/skywalking/blob/578ea4f66f11bdfe5dcda25f574a1ed57ca47d24/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/define/ApplicationEsTableDefine.java#L38) 。

不同的存储组件实现，有不同的 TableDefine 实现类，如下图：[](http://www.iocoder.cn/images/SkyWalking/2020_08_20/05.png)

* ElasticSearchTableDefine ：基于 Elasticsearch 的表定义**抽象类**，在 `collector-storage-es-provider` 的 [`define`](https://github.com/YunaiV/skywalking/tree/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/define) **包**下，我们可以看到**所有** ES 的 TableDefine 类。

* H2TableDefine ：基于 H2 的表定义**抽象类**，在 `collector-storage-h2-provider` 的 [`define`](https://github.com/YunaiV/skywalking/tree/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-storage/collector-storage-h2-provider/src/main/java/org/skywalking/apm/collector/storage/h2/define) **包**下，我们可以看到**所有** H2 的 TableDefine 类。

### 2.2.1 ColumnDefine

 [`org.skywalking.apm.collector.core.data.ColumnDefine`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/ColumnDefine.java) ，字段定义**抽象类**。

* [`name`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/ColumnDefine.java#L31) 属性，字段名。
* [`type`](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/ColumnDefine.java#L35) 属性，字段类型。

### 2.2.2 Loader

涉及到的类如下图所示：[](http://www.iocoder.cn/images/SkyWalking/2020_08_20/06.png)

[`org.skywalking.apm.collector.core.data.StorageDefineLoader`](https://github.com/YunaiV/skywalking/blob/0aa5e6a49c1f29b43824ebabf6bb7d76b80e3eb7/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/StorageDefineLoader.java) ，调用 [`org.skywalking.apm.collector.core.define.DefinitionLoader`](https://github.com/YunaiV/skywalking/blob/0aa5e6a49c1f29b43824ebabf6bb7d76b80e3eb7/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/define/DefinitionLoader.java) ，从 [`org.skywalking.apm.collector.core.data.StorageDefinitionFile`](https://github.com/YunaiV/skywalking/blob/0aa5e6a49c1f29b43824ebabf6bb7d76b80e3eb7/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/StorageDefinitionFile.java) 中，加载 TableDefine 实现类数组。

另外，在 `collector-storage-es-provider` 和 `collector-storage-h2-provider` 里都有 `storage.define` 文件，如下图：[](http://www.iocoder.cn/images/SkyWalking/2020_08_20/07.png)

* StorageDefinitionFile 声明了读取该文件。
* **注意**，DefinitionLoader 在加载时，两个文件都会被读取，最终在 `StorageInstaller#defineFilter(List<TableDefine>)` 方法，进行过滤。

代码比较简单，中文注释已加，胖友自己阅读理解下。

## 2.3 Data

`org.skywalking.apm.collector.core.data.Data`  ，数据**抽象类**。

* [`dataXXX`]() **前缀**的属性，字段值们。
    * [`dataStrings`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Data.java#L30) 属性的第一位，是 **ID** 属性。参见 [**构造方法**](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Data.java#L51)的【第 51 行】 或者 [`#setId(id)`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Data.java#L142) 方法。
* [`xxxColumns`]() **后缀**的属性，字段( Column )们。
* 通过上述两种属性 + 自身类，可以确定一条数据记录的表、字段类型、字段名、字段值。
* **继承** [`org.skywalking.apm.collector.core.data.EndOfBatchQueueMessage`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/EndOfBatchQueueMessage.java) ，带是否消息批处理的最后一条标记的**消息抽象类**，[`endOfBatch`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/EndOfBatchQueueMessage.java#L31) 属性，在 [TODO 【4006】]() 详细解析。
    * **继承** [`org.skywalking.apm.collector.core.data.AbstractHashMessage`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/AbstractHashMessage.java) ，带哈希码的**消息抽象类**，[`hashCode`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/AbstractHashMessage.java#L38) 属性，在 [TODO 【4005】]() 详细解析。
* [`#mergeData(Data)`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Data.java#L154) 方法，合并传入的数据到自身。该方法被 [`AggregationWorker#aggregate(message)`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/impl/AggregationWorker.java#L94) 调用，在 [TODO 【4006】]() 详细解析。

在 `collector-storage-define` 的 [`table`](https://github.com/YunaiV/skywalking/tree/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table) **包**下，我们可以看到所有 Data 类，**非** `"Table"` 结尾，例如 [Application](https://github.com/YunaiV/skywalking/blob/beebd8f8f419ca0b25dc086c71a9b1c580a083d4/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/Application.java) 。

### 2.3.1 Column

[`org.skywalking.apm.collector.core.data.Column`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Column.java) ，字段。

* [`name`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Column.java#L31) 属性，字段名。
* [`operation`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Column.java#L35) 属性，操作( Operation )。

### 2.3.2 Operation

[`org.skywalking.apm.collector.core.data.Operation`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Operation.java) ，操作**接口**。用于两个值之间的操作，例如，相加等等。目前实现类有：

* [AddOperation](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/operator/AddOperation.java) ：值相加操作。
* [CoverOperation](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/operator/CoverOperation.java) ：值覆盖操作，即以新值为返回。
* [NonOperation](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/operator/NonOperation.java) ：空操作，即以老值为返回。

# 3. collector-storage-define

`collector-cluster-define` ：定义存储组件接口。项目结构如下 ：

[](http://www.iocoder.cn/images/SkyWalking/2020_08_20/08.png)

## 3.1 StorageModule

`org.skywalking.apm.collector.storage.StorageModule` ，实现 [Module](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java) 抽象类，集群管理 Module 。

[`#name()`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageModule.java#L67) **实现**方法，返回模块名为 `"storage"` 。

[`#services()`](https://github.com/YunaiV/skywalking/blob/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageModule.java#L71) **实现**方法，返回 Service 类名：在 [org.skywalking.apm.collector.storage.dao](https://github.com/YunaiV/skywalking/tree/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/dao) **包**下的所有类。

## 3.2 table 包

在 [`org.skywalking.apm.collector.storage.table`](https://github.com/YunaiV/skywalking/tree/445bf9da669784b28d24f2e31576d3b0673c2852/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table) 包下，定义了存储模块所有的 Table 和 Data 实现类。

## 3.3 StorageInstaller

`org.skywalking.apm.collector.storage.StorageInstaller` ，存储安装器**抽象类**，基于 TableDefine ，初始化存储组件的表。

* [`#defineFilter(List<TableDefine>)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageInstaller.java#L71) **抽象**方法，过滤 TableDefine 数组中，非自身需要的。例如说，ElasticSearchStorageInstaller 过滤后，只保留 ElasticSearchTableDefine 对象。
* [`#isExists(Client, TableDefine)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageInstaller.java#L81) **抽象**方法，判断表是否存在。
* [`#deleteTable(Client, TableDefine)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageInstaller.java#L81) **抽象**方法，删除表。
* [`#createTable(Client, TableDefine)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageInstaller.java#L81) **抽象**方法，创建表。
* [`#install(Client)`](https://github.com/YunaiV/skywalking/blob/2b700457124e7d4f788343d8bcd9a03d2e273aca/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/StorageInstaller.java#L39) 方法，基于 TableDefine ，初始化存储组件的表。
    * 该方法会被 StorageModuleH2Provider 或 StorageModuleEsProvider 启动时调用。

## 3.4 dao 包

在 `collector-storage-define` 项目结构图，我们看到一共有**两**个 `bao` 包：

* `org.skywalking.apm.collector.storage.base.dao` ，**系统**的 DAO 接口。
* `org.skywalking.apm.collector.storage.dao` ，**业务**的 DAO 接口。
    * **继承**系统的 DAO 接口。
    * 被 `collector-storage-xxx-provider` 的 `dao` 包**实现**。

[](http://www.iocoder.cn/images/SkyWalking/2020_08_20/09.png)

### 3.4.1 

# 4. collector-storage-es-provider

# 5. collector-storage-h2-provider


