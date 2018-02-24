title: Elastic-Job 源码分析 —— 为什么阅读 Elastic-Job 源码？
date: 2017-09-01
tags:
categories: Elastic-Job-Lite
permalink: Elastic-Job/why-read-Elastic-Job-source-code

-------

摘要: 原创出处 http://www.iocoder.cn/Elastic-Job/why-read-Elastic-Job-source-code/ 「芋道源码」欢迎转载，保留摘要，谢谢！

  - [为什么阅读 Elastic-Job 源码？](http://www.iocoder.cn/Elastic-Job/why-read-Elastic-Job-source-code/)
  - [使用公司](http://www.iocoder.cn/Elastic-Job/why-read-Elastic-Job-source-code/)
  - [步骤/功能](http://www.iocoder.cn/Elastic-Job/why-read-Elastic-Job-source-code/)
  - [Elastic-Job-Cloud 不考虑写的内容](http://www.iocoder.cn/Elastic-Job/why-read-Elastic-Job-source-code/)
  - [XXL-JOB](http://www.iocoder.cn/Elastic-Job/why-read-Elastic-Job-source-code/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------


## 为什么阅读 Elastic-Job 源码？

1. 之前断断续续读过 Quartz 源码，团队里也对 Quartz 做过一些封装管理，很多 Quartz 二次封装开源项目，想了解 Elastic-Job 做了哪些功能，是怎么实现的
2. Quartz 多节点通过数据库锁实现任务抢占，Elastic-Job 基于什么策略实现任务调度与分配
3. 任务分片如何实现
4. Elastic-Job-Cloud 如何实现任务动态扩容和缩容
5. 任务超时如何处理？任务假死怎么判断？

## 使用公司

## 步骤/功能

* [x] 分布式调度协调
* [x] 弹性扩容缩容
* [x] 失效转移
* [x] 错过执行作业重触发
* [x] 作业分片策略
* [x] 作业唯一节点执行
* [x] 自诊断并修复分布式不稳定造成的问题
* [x] 支持并行调度
* [x] 支持作业生命周期操作
* [x] 丰富的作业类型
* [ ] Spring整合以及命名空间提供
* [x] 运维平台
* [x] 事件追踪
* [x] DUMP 作业运行信息
* [x] 作业监听器
* [ ] 基于 Docker 的进程隔离（TBD）
* [x] 高可用


## Elastic-Job-Cloud 不考虑写的内容

* 《Elastic-Job-Cloud 源码解析 —— 注册中心》和《Elastic-Job-Lite 源码解析 —— 注册中心》 一致
* 《Elastic-Job-Cloud 源码解析 —— 作业数据存储》和《Elastic-Job-Lite 源码解析 —— 作业数据存储》 一致
* 《Elastic-Job-Cloud 源码解析 —— 注册中心监听器》和《Elastic-Job-Lite 源码解析 —— 注册中心监听器》 一致
* 《Elastic-Job-Cloud 源码解析 —— 注册中心监听器》和《Elastic-Job-Lite 源码解析 —— 注册中心监听器》 一致
* 《Elastic-Job-Cloud 源码解析 —— 作业事件追踪》和《Elastic-Job-Lite 源码解析 —— 作业事件追踪》 一致
* 《Elastic-Job-Cloud 源码解析 —— 作业分片策略》在《Elastic-Job-Cloud 源码解析 —— 作业调度（一）》包含
* 《Elastic-Job-Cloud 源码解析 —— 作业监听器》该功能暂不支持
* 《Elastic-Job-Cloud 源码解析 —— 作业监控服务》该功能暂不支持
* 《Elastic-Job-Cloud 源码解析 —— 运维平台》融合在每一张里，**统计相关暂时未解析**
* 《Elastic-Job-Cloud 源码解析 —— 诊断自修复》《Elastic-Job-Cloud 源码解析 —— 高可用》包含

## XXL-JOB 

基于 V1.8，会逐渐和 Elastic-Job 功能做对比

* [ ] 1、简单：支持通过Web页面对任务进行CRUD操作，操作简单，一分钟上手；
* [ ] 2、动态：支持动态修改任务状态、暂停/恢复任务，以及终止运行中任务，即时生效；
* [ ] 3、调度中心HA（中心式）：调度采用中心式设计，“调度中心”基于集群Quartz实现，可保证调度中心HA；
* [ ] 4、执行器HA（分布式）：任务分布式执行，任务"执行器"支持集群部署，可保证任务执行HA；
* [ ] 5、任务Failover：执行器集群部署时，任务路由策略选择"故障转移"情况下调度失败时将会平滑切换执行器进行Failover；
* [ ] 6、一致性：“调度中心”通过DB锁保证集群分布式调度的一致性, 一次任务调度只会触发一次执行；
* [ ] 7、自定义任务参数：支持在线配置调度任务入参，即时生效；
* [ ] 8、调度线程池：调度系统多线程触发调度运行，确保调度精确执行，不被堵塞；
* [ ] 9、弹性扩容缩容：一旦有新执行器机器上线或者下线，下次调度时将会重新分配任务；
* [ ] 10、邮件报警：任务失败时支持邮件报警，支持配置多邮件地址群发报警邮件；
* [ ] 11、状态监控：支持实时监控任务进度；
* [ ] 12、Rolling执行日志：支持在线查看调度结果，并且支持以Rolling方式实时查看执行器输出的完整的执行日志；
* [ ] 13、GLUE：提供Web IDE，支持在线开发任务逻辑代码，动态发布，实时编译生效，省略部署上线的过程。支持30个版本的历史版本回溯。
* [ ] 14、数据加密：调度中心和执行器之间的通讯进行数据加密，提升调度信息安全性；
* [ ] 15、任务依赖：支持配置子任务依赖，当父任务执行结束且执行成功后将会主动触发一次子任务的执行, 多个子任务用逗号分隔；
* [ ] 16、推送maven中央仓库: 将会把最新稳定版推送到maven中央仓库, 方便用户接入和使用;
* [ ] 17、任务注册: 执行器会周期性自动注册任务, 调度中心将会自动发现注册的任务并触发执行。同时，也支持手动录入执行器地址；
* [ ] 18、路由策略：执行器集群部署时提供丰富的路由策略，包括：第一个、最后一个、轮询、随机、一致性HASH、最不经常使用、最近最久未使用、故障转移、忙碌转移等；
* [ ] 19、运行报表：支持实时查看运行数据，如任务数量、调度次数、执行器数量等；以及调度报表，如调度日期分布图，调度成功分布图等；
* [ ] 20、脚本任务：支持以GLUE模式开发和运行脚本任务，包括Shell、Python等类型脚本;
* [ ] 21、阻塞处理策略：调度过于密集执行器来不及处理时的处理策略，策略包括：单机串行（默认）、丢弃后续调度、覆盖之前调度；
* [ ] 22、失败处理策略；调度失败时的处理策略，策略包括：失败告警（默认）、失败重试；
* [ ] 23、分片广播任务：执行器集群部署时，任务路由策略选择"分片广播"情况下，一次任务调度将会广播触发对应集群中所有执行器执行一次任务，同时传递分片参数；可根据分片参数开发分片任务；
* [ ] 24、动态分片：分片广播任务以执行器为维度进行分片，支持动态扩容执行器集群从而动态增加分片数量，协同进行业务处理；在进行大数据量业务操作时可显著提升任务处理能力和速度。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

