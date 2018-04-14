title: 精尽 Dubbo 源码分析 —— 过滤器（五）之 TimeoutFilter
date: 2018-11-16
tags:
categories: Dubbo
permalink: Dubbo/filter-timeout-filter

-------

摘要: 原创出处 http://www.iocoder.cn/Dubbo/filter-timeout-filter/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Dubbo/filter-timeout-filter/)
- [2. TimeoutFilter](http://www.iocoder.cn/Dubbo/filter-timeout-filter/)
- [666. 彩蛋](http://www.iocoder.cn/Dubbo/filter-timeout-filter/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

本文分享过滤器 TimeoutFilter ，用于服务**提供者**中。

# 2. TimeoutFilter

`com.alibaba.dubbo.rpc.filter.TimeoutFilter` ，实现 Filter 接口，超时过滤器。如果服务调用**超时**，记录**告警**日志，**不干涉**服务的运行。代码如下：

```Java
  1: @Activate(group = Constants.PROVIDER)
  2: public class TimeoutFilter implements Filter {
  3: 
  4:     private static final Logger logger = LoggerFactory.getLogger(TimeoutFilter.class);
  5: 
  6:     @Override
  7:     public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
  8:         long start = System.currentTimeMillis();
  9:         // 服务调用
 10:         Result result = invoker.invoke(invocation);
 11:         // 计算调用时长
 12:         long elapsed = System.currentTimeMillis() - start;
 13:         // 超过时长，打印告警日志
 14:         if (invoker.getUrl() != null
 15:                 && elapsed > invoker.getUrl().getMethodParameter(invocation.getMethodName(), "timeout", Integer.MAX_VALUE)) {
 16:             if (logger.isWarnEnabled()) {
 17:                 logger.warn("invoke time out. method: " + invocation.getMethodName()
 18:                         + " arguments: " + Arrays.toString(invocation.getArguments()) + " , url is "
 19:                         + invoker.getUrl() + ", invoke elapsed " + elapsed + " ms.");
 20:             }
 21:         }
 22:         return result;
 23:     }
 24: 
 25: }
```

* 第 10 行：调用 `Invoker#invoke(invocation)` 方法，服务调用。
* 第 12 行：计算调用时长。
* 第 13 至 21 行：超过时长，打印**告警**日志。注意，此处的 `"timeout"` 取得的是服务**提供者**的配置，不同于服务**消费者**的配置。
* 第 22 行：返回调用结果。
* 🙂 再注意，在服务**提供者**，执行服务调用时，即使**超过了超时时间**，也不会取消执行。虽然，服务**消费者**，已经结束调用，返回调用超时。

# 666. 彩蛋

水更一篇。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)


