title: SkyWalking 源码分析 —— Collector Streaming Computing 流式处理（一）
date: 2020-08-15
tags:
categories: SkyWalking
permalink: SkyWalking/collector-streaming-first

-------

# 1. 概述

本文主要分享 **Collector Streaming 流式处理**。主要包含如下部分：

* `apm-collector-core` 模块的 `graph` 包，提供**最精简**、**单节点**的流式处理的封装。如下图所示：[](http://www.iocoder.cn/images/SkyWalking/2020_08_15/01.png)

* `apm-collector-stream` 模块，在 `graph` 包的基础上，提供**异步**、**跨节点**等等的流式处理的封装。如下图所示：[](http://www.iocoder.cn/images/SkyWalking/2020_08_15/02.png)

> 免打脸大保健：笔者对流式处理非常不了解，本文可能是一本正经的胡说八道。考虑到笔者是靠脸吃饭（颜值我只服我红雷哥），所以读者老爷请爱护下笔者。

Collector Streaming 在 SkyWalking 架构图处于如下位置( **红框** ) ：

> FROM https://github.com/apache/incubating-skywalking  
> [](http://www.iocoder.cn/images/SkyWalking/2020_08_15/03.jpeg)

OK，下面来一本正经的代码走起！

# 2. apm-collector-core/graph

整体类图如下：[](http://www.iocoder.cn/images/SkyWalking/2020_08_15/04.png)

看起来略复杂，不要方，我们先来看一个流式大数据处理框架 Apache Storm 的说明：

> FROM [《流式大数据处理的三种框架：Storm，Spark和Samza》](http://www.csdn.net/article/2015-03-09/2824135)  
> 在 [Storm](https://storm.apache.org/) 中，先要设计一个用于实时计算的图状结构，我们称之为拓扑（topology）。这个拓扑将会被提交给集群，由集群中的主控节点（master node）分发代码，将任务分配给工作节点（worker node）执行。

* Graph ：定义了一个**数据**在**各个** Node 的处理拓扑图。
* WayToNode ：提交**数据**给 Node 的**方式**。
* Node ：节点，包含一个 NodeProcessor 和 一个 Next 。 
    * NodeProcessor ：Node 处理器，处理**数据**。
    * Next ：包含 WayToNode 数组，即 Node 提交**数据**给 Next 的 Node **数组**的**方式**。

整体交互流程如下：[](http://www.iocoder.cn/images/SkyWalking/2020_08_15/05.png)

* **粉色**箭头：当数据进来时，提交给 Grpah 。按照定义的拓扑图，使用 NodeWay 提交给 Node ，NodeProcessor 进行处理。
* **蓝色**箭头：当 NodeProcessor 处理完成后，Next **逐个**使用 NodeWay **数组**提交给下面的 Node ，继续处理。
    * ps ：**注意**，这块流程，根据不同的 NodeProcessor 的实现类会有不同，**蓝色**箭头的过程，只是**其中的一种**，下面会详细解析。

整体顺序图如下：[](http://www.iocoder.cn/images/SkyWalking/2020_08_15/06.png)

* DirectWay 是 WayToNode **接口**的一种实现，正如其名，**直接**提交数据给 Node 。在 [「3. apm-collector-stream」](#) 会看到其他实现，例如提交到其他服务器节点的 Node，从而实现跨服务器节点的流式处理。
* AbstractWorker 在 `apm-collector-stream` 模块，是 NodeProcessor **接口**的一种实现，处理提交给 Node 的数据。在 `#onWork(message)` **抽象**方法里，子类可以实现该方法，根据自身需求，是否调用 `#onNext(message)` 方法，Next 逐个使用 NodeWay 数组提交给下面的 Node ，继续处理。

-------

下面，我们来详细分别看看如下逻辑的详细代码实现：

* Graph 创建
* Graph 启动

## 2.1 Graph 创建

创建 Graph 的顺序图如下：[](http://www.iocoder.cn/images/SkyWalking/2020_08_15/07.png)

* 第一步，调用 `GraphManager#createIfAbsent(graphId, input)` 方法( `input` 参数没用 )，创建一个 Graph 对象。
* 第二步，调用 `Graph#addNode(WayToNode)` 方法，创建该 Graph 的**首个** Node 对象。
* 第三步，调用 `Node#addNext(WayToNode)` 方法，创建该 Node 的下一个 Node 对象。

如下是 `collector-agent-stream-provider` 模块，`TraceStreamGraph#createServiceReferenceGraph()` 方法的代码：

```Java
public void createServiceReferenceGraph() {
    QueueCreatorService<ServiceReference> queueCreatorService = moduleManager.find(QueueModule.NAME).getService(QueueCreatorService.class);
    RemoteSenderService remoteSenderService = moduleManager.find(RemoteModule.NAME).getService(RemoteSenderService.class);

    Graph<ServiceReference> graph = GraphManager.INSTANCE.createIfAbsent(SERVICE_REFERENCE_GRAPH_ID, ServiceReference.class);
    graph.addNode(new ServiceReferenceAggregationWorker.Factory(moduleManager, queueCreatorService).create(workerCreateListener))
        .addNext(new ServiceReferenceRemoteWorker.Factory(moduleManager, remoteSenderService, SERVICE_REFERENCE_GRAPH_ID).create(workerCreateListener))
        .addNext(new ServiceReferencePersistenceWorker.Factory(moduleManager, queueCreatorService).create(workerCreateListener));
}
```

让我们来看看每个方法的具体代码实现。

-------

**第一步**

[`GraphManager#createIfAbsent(graphId, input)`](todo) 方法，创建一个 Graph 对象，并添加到 Graph 映射。代码如下：

* `INSTANCE` 属性，单例。
* `allGraphs` 属性，Graph 映射。其中映射的 KEY 为**每个** Graph 全局唯一编号。在 [JvmMetricStreamGraph](todo) 、[RegisterStreamGraph](todo) 、[TraceStreamGraph](todo) 类中，枚举了实际使用的 Graph 编号们。
* 第 50 至 58 行：当 Graph 映射里不存在指定 Graph 编号时，创建 Graph 对象，并返回。

-------

**第二步**

[`Graph#addNode(WayToNode)`](todo) 方法，创建该 Graph 的**首个** Node 对象。代码如下：

* `id` 属性，Graph 编号。
* `entryWay`，**首个**提交数据给 Node 的方式。
* 第 58 行 ：将方法参数 `entryWay` 赋值给 `this.entryWay` 属性。在下分享的 `Graph#start(input)` 方法里，我们会看到这是 Graph 启动的入口，**首个**提交给 Node 的方式。
* 第 60 至 62 行 ：调用 `WayToNode#buildDestination(Graph)` 方法，创建 Node 对象，并**返回该 Node** 。在上文中，我们已经说过创建的 Node 对象，为该 Graph 的**首个** Node 。

[`WayToNode#buildDestination()`](todo) 方法，创建该 **WayToNode** 的 Node 对象。代码如下：

* `destination` 属性，目标 Node 。即该 WayToNode 提交**数据**到的 Node 。
* `destinationHandler` 属性，目标 Node 的处理器。见 [`#out(INPUT)`](todo) 方法。
* 第 42 行：创建 Node 对象。
    * 目前，`destinationHandler` 属性，除了用于创建 Node 对象，无其他用途。

[Node `构造方法`](todo) 方法，代码如下：

* `nodeProcessor` 属性，节点处理器。
* `next` 属性，包含 WayToNode 数组，即 Node 提交数据给 Next 的 Node 数组的方式。
* 第 44 行：调用 `Graph#checkForNewNode(Node)` 方法，校验 Node 的 NodeProcessor 在其 Graph 里，**编号唯一**。

[`Graph#checkForNewNode(Node)`](todo) 方法，校验 Node 的 NodeProcessor 在 Graph 里，**编号唯一**，代码如下：

* `nodeIndex` 属性，处理器编号与 Node 的映射。其中映射的 KEY 为 `NodeProcessor#id()` 。 
* 第 72 至 78 行：校验 Node 的 NodeProcessor 在 Graph 里，**编号唯一**。

-------

**第三步**

[`Node#addNext(WayToNode)`](todo) 方法，创建该 Node 的下一个 Node 对象。代码如下：

* 第 54 行：调用 `WayToNode#buildDestination(Graph)` 方法，创建该 Node 的下面的 Node 对象。
* 第 56 行：添加创建的 Node 对象到 `next` 属性。
* 第 58 行：返回创建的 Node 对象。

## 2.2 Graph 启动

创建 Graph 的顺序图如下：[](http://www.iocoder.cn/images/SkyWalking/2020_08_15/08.png)


| 数据流向 | FROM | TO | 逻辑 |
| --- | --- | --- | --- |
| 第一步 | Graph | WayToNode |  |
| 第二步 | WayToNode | Node |  |
| 第三步 | Node | NodeProcessor |  |
| 第四步 | NodeProcessor | Next | 根据具体实现，若到 Next ，重复第二步 |

[](http://www.iocoder.cn/images/SkyWalking/2020_08_15/09.png)

-------

**第一步**

[`Graph#start(input)`](todo) 方法，启动 Graph ，处理数据。代码如下：

* 第 49 行：调用 `WayToNode#in(input)` 方法，输入数据给 WayToNode 。

[`WayToNode#in(input)`](todo) **抽象**方法，以 [`DirectWay#in(input)`](todo) **实现**方法举例子，代码如下：

* 第 30 行：调用 `super#out(input)` 方法，**直接**输出数据，调用 `Node#execute(input)` 方法，提交数据给 Node ，进行处理。

-------

**第二步**

[`Node#execute`](todo) 方法，调用 `NodeProcessor#process(input, next)` 方法，处理数据。

-------

**第三步**

[`NodeProcessor#process(input, next)`](todo) **接口**方法，以 [`AbstractWorker#process(input, next)`](todo) **实现**方法举例子，代码如下：

* 第 56 行：将方法参数 `next` 赋值给 `this.next` 属性。`this.next` 属性，用于封装的 `#onNext(OUTPUT)` 方法，提交数据给当前 Node 的 Next ( 下面的 Node 们 )继续处理数据。
* 第 59 行：调用 [`#onWork`](todo) **抽象**方法，处理数据。当 AbstractWorker **抽象类**的实现类需要继续讲数据提交给 Next 时，需要在 `#onWork` 方法里，调用 `#onNext(OUTPUT)` 方法，例如 [`ApplicationRegisterRemoteWorker#onWork(Application)`](todo) 。

-------

**第四步**

[`Next#execute(Input)`](todo) 方法，**循环** WayToNode 数组，输入数据给 WayNode ，相当于"**重回**"【第二步】。

# 3. apm-collector-stream

