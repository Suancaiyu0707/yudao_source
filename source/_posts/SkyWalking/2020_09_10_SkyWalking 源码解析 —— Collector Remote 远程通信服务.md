title: SkyWalking 源码分析 —— Collector Remote 远程通信服务
date: 2020-09-10
tags:
categories: SkyWalking
permalink: SkyWalking/collector-remote-module

-------

# 1. 概述

本文主要分享 **SkyWalking Collector Remote 远程通信服务**。该服务用于 Collector 集群内部通信。

[](http://www.iocoder.cn/images/SkyWalking/2020_09_10/04.png)

目前集群内部通信的目的，跨节点的流式处理。Remote Module **应用**在 SkyWalking 架构图如下位置( **红框** ) ：

> FROM https://github.com/apache/incubating-skywalking  
> [](http://www.iocoder.cn/images/SkyWalking/2020_09_10/01.jpeg)

下面我们来看看整体的项目结构，如下图所示 ：

[](http://www.iocoder.cn/images/SkyWalking/2020_09_10/02.png)

* `collector-remote-define` ：定义远程通信接口。
* `collector-remote-grpc-provider` ：基于 Kafka 的远程通信实现。*目前暂未完成*。
* `collector-remote-grpc-provider` ：基于 [Google gRPC](https://grpc.io/) 的远程通信实现。**生产环境目前使用**

下面，我们从**接口到实现**的顺序进行分享。

# 2. collector-remote-define

`collector-remote-define` ：定义远程通信接口。项目结构如下 ：

[](http://www.iocoder.cn/images/SkyWalking/2020_09_10/03.png)

整体流程如下图：

[](http://www.iocoder.cn/images/SkyWalking/2020_09_10/05.png)

我们按照整个流程的处理顺序，逐个解析涉及到的类与接口。

## 2.1 RemoteModule

`org.skywalking.apm.collector.remote.RemoteModule` ，实现 [Module](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java) 抽象类，远程通信 Module 。

[`#name()`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/RemoteModule.java#L34) **实现**方法，返回模块名为 `"remote"` 。

[`#services()`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/RemoteModule.java#L38) **实现**方法，返回 Service 类名：RemoteSenderService 、RemoteDataRegisterService 。

## 2.2 RemoteSenderService

`org.skywalking.apm.collector.remote.service.RemoteSenderService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，远程发送服务**接口**，定义了 [`#send(graphId, nodeId, data, selector)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteSenderService.java#L40) **接口**方法，调用 RemoteClient ，发送数据。

* `graphId` 方法参数，Graph 编号。通过 `graphId` ，可以查找到对应的 Graph 对象。
    * Graph 在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「2. apm-collector-core/graph」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。
* `nodeId` 方法参数，Worker 编号。通过 `workerId` ，可以查找在 Graph 对象中的 Worker 对象，从而 Graph 中的流式处理。
    * Worker 在 [《SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）》「3. apm-collector-stream」](http://www.iocoder.cn/SkyWalking/collector-streaming-first/?self) 有详细解析。
* `data` 方法参数，Data 数据对象。例如，流式处理的具体数据对象。
    * Data 在 [《SkyWalking 源码分析 —— Collector Storage 存储组件》「2. apm-collector-core」](http://www.iocoder.cn/SkyWalking/collector-storage-module/) 有详细解析。
* `selector` 方法参数，[`org.skywalking.apm.collector.remote.service.Selector`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/Selector.java) 选择器对象。根据 Selector 对象，使用对应的**负载均衡**策略，选择集群内的 Collector 节点，发送数据。
* [RemoteSenderService.Mode](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteSenderService.java#L45) 返回值，发送模式分成 `Remote` 和 `Local` 两种方式。前者，发送数据到远程的 Collector 节点；后者，发送数据到本地，即本地处理，参见 [`RemoteWorkerRef#in(message)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-stream/src/main/java/org/skywalking/apm/collector/stream/worker/base/RemoteWorkerRef.java#L58) 方法。

## 2.3 RemoteClientService

`org.skywalking.apm.collector.remote.service.RemoteClientService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，远程客户端服务**接口**，定义了 [`#create(host, port, channelSize, bufferSize)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteClientService.java#L39) **接口**方法，创建 RemoteClient 对象。

## 2.4 RemoteClient

`org.skywalking.apm.collector.remote.service.RemoteClient` ，继承 `java.lang.Comparable` 接口，远程客户端**接口**。定义了如下接口方法：

* [`#push(graphId, nodeId, data, selector)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteClient.java#L42) **接口**方法，发送数据。
* [`#getAddress()`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteClient.java#L33) **接口**方法，返回客户端连接的远程 Collector 地址。
* [`#equals(address)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteClient.java#L44) **接口**方法，判断 RemoteClient 是否连接了指定的地址。

## 2.5 CommonRemoteDataRegisterService

在说 CommonRemoteDataRegisterService 之前，首先来说下 CommonRemoteDataRegisterService 的意图。

在上文中，我们可以看到发送给 Collector 是 Data 对象，而 [Data](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/data/Data.java) 是数据的**抽象类**，在具体反序列化 Data 对象之前，程序是无法得知它是 Data 的哪个实现对象。这个时候，我们可以给 Data 对象的每个实现类，生成一个对应的**数据协议编号**。

* 在发送数据之前，序列化 Data 对象时，增加该 Data 对应的协议编号，一起发送。
* 在接收数据之后，反序列化数据时，根据协议编号，创建 Data 对应的实现类对象。

[`org.skywalking.apm.collector.remote.service.CommonRemoteDataRegisterService`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java) ，通用远程数据注册服务。

* [`id`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java#L40) 属性，数据协议自增编号。
* [`dataClassMapping`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java#L44) 属性，数据类型( Class<? extends Data> )与**数据协议编号**的映射。
* [`dataInstanceCreatorMapping`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java#L48) 属性，**数据协议编号**与数据对象创建器( RemoteDataInstanceCreator )的映射。

### 2.5.1 RemoteDataRegisterService

`org.skywalking.apm.collector.remote.service.RemoteDataRegisterService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，远程客户端服务**接口**，定义了 [`#register(Class<? extends Data>, RemoteDataInstanceCreator)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteDataRegisterService.java#L37) **接口**方法，注册数据类型对应的远程数据创建器( [`RemoteDataRegisterService.RemoteDataInstanceCreator`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteDataRegisterService.java#L39) )对象。

CommonRemoteDataRegisterService 实现了 RemoteDataRegisterService 接口，[`#register(Class<? extends Data>, RemoteDataInstanceCreator)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java#L56) **实现**方法。

另外，[AgentStreamRemoteDataRegister](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/AgentStreamRemoteDataRegister.java#L42) 会调用 [`RemoteDataRegisterService#register(Class<? extends Data>, RemoteDataInstanceCreator)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteDataRegisterService.java#L37) 方法，注册每个数据类型的 RemoteDataInstanceCreator 对象。注意，例如 `Application::new` 是 RemoteDataInstanceCreator 的**匿名实现类**。

### 2.5.2 RemoteDataIDGetter

`org.skywalking.apm.collector.remote.service.RemoteDataIDGetter` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，远程数据协议编号获取器**接口**，定义了 [`#getRemoteDataId(Class<? extends Data>)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteDataIDGetter.java#L37) **接口**方法，根据数据类型获取数据协议编号。

CommonRemoteDataRegisterService 实现了 RemoteDataIDGetter 接口，[`#getRemoteDataId(Class<? extends Data>)`](https://github.com/YunaiV/skywalking/blob/5e2eb23f33136c979e5056dbe32e880b130d0901/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java#L67) **实现**方法。

### 2.5.3 RemoteDataInstanceCreatorGetter

`org.skywalking.apm.collector.remote.service.RemoteDataInstanceCreatorGetter` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，远程数据创建器的获取器**接口**，定义了 [`#getInstanceCreator(remoteDataId`](https://github.com/YunaiV/skywalking/blob/f70019926200139f9fe235ade9aaf7724ab72c8b/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteDataInstanceCreatorGetter.java#L35) **接口**方法，根据数据协议编号获得远程数据创建器( RemoteDataInstanceCreator )。

CommonRemoteDataRegisterService 实现了 RemoteDataInstanceCreatorGetter 接口，[`#getInstanceCreator(remoteDataId)`](https://github.com/YunaiV/skywalking/blob/f70019926200139f9fe235ade9aaf7724ab72c8b/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/CommonRemoteDataRegisterService.java#L75) **实现**方法。

## 2.6 RemoteSerializeService

`org.skywalking.apm.collector.remote.service.RemoteSerializeService` ，远程通信序列化服务**接口**，定义了 [`#serialize(Data)`](https://github.com/YunaiV/skywalking/blob/f70019926200139f9fe235ade9aaf7724ab72c8b/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteSerializeService.java#L36) **接口**方法，序列化数据，生成 Builder 对象。

## 2.7 RemoteSerializeService

`org.skywalking.apm.collector.remote.service.RemoteDeserializeService` ，远程通信序反列化服务**接口**，定义了 [`#deserialize(RemoteData, Data)`](https://github.com/YunaiV/skywalking/blob/f70019926200139f9fe235ade9aaf7724ab72c8b/apm-collector/apm-collector-remote/collector-remote-define/src/main/java/org/skywalking/apm/collector/remote/service/RemoteDeserializeService.java#L36) **接口**方法，反序列化传输数据。

# 3. collector-remote-grpc-provider

## 3.1 RemoteModuleGRPCProvider


## 3.2 GRPCRemoteSenderService

### 3.2.1 selector

### 3.2.2 注册发现

RemoteModuleGRPCRegistration

## 3.3 GRPCRemoteClientService

## 3.4 GRPCRemoteClient

## 3.5 RemoteCommonServiceHandler

## 3.6 GRPCRemoteSerializeService

## 3.7 GRPCRemoteDeserializeService

# 4. collector-remote-grpc-provider

# 666. 彩蛋

