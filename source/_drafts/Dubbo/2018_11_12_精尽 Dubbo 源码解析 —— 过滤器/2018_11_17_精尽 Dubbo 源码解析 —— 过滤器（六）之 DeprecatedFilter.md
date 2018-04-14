title: 精尽 Dubbo 源码分析 —— 过滤器（六）之 DeprecatedFilter
date: 2018-11-17
tags:
categories: Dubbo
permalink: Dubbo/filter-deprecated-filter

-------

摘要: 原创出处 http://www.iocoder.cn/Dubbo/filter-deprecated-filter/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Dubbo/filter-deprecated-filter/)
- [2. DeprecatedFilter](http://www.iocoder.cn/Dubbo/filter-deprecated-filter/)
- [666. 彩蛋](http://www.iocoder.cn/Dubbo/filter-deprecated-filter/)

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

本文分享过滤器 DeprecatedFilter ，用于服务**消费者**中，通过 `<dubbo: service />` 或 `<dubbo:reference />` 或 `<dubbo:method />` 的 `"deprecated"` 配置项为 `true` 来开启。

# 2. DeprecatedFilter

`com.alibaba.dubbo.rpc.filter.DeprecatedFilter` ，实现 Filter 接口，废弃调用的过滤器实现类。当调用废弃的服务方法时，打印错误日志提醒。代码如下：

```Java
  1: @Activate(group = Constants.CONSUMER, value = Constants.DEPRECATED_KEY)
  2: public class DeprecatedFilter implements Filter {
  3: 
  4:     private static final Logger LOGGER = LoggerFactory.getLogger(DeprecatedFilter.class);
  5: 
  6:     /**
  7:      * 已经打印日志的方法集合
  8:      */
  9:     private static final Set<String> logged = new ConcurrentHashSet<String>();
 10: 
 11:     @Override
 12:     public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
 13:         // 获得方法名
 14:         String key = invoker.getInterface().getName() + "." + invocation.getMethodName();
 15:         // 打印告警日志
 16:         if (!logged.contains(key)) {
 17:             logged.add(key);
 18:             if (invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.DEPRECATED_KEY, false)) {
 19:                 LOGGER.error("The service method " + invoker.getInterface().getName() + "." + getMethodSignature(invocation) + " is DEPRECATED! Declare from " + invoker.getUrl());
 20:             }
 21:         }
 22:         return invoker.invoke(invocation);
 23:     }
 24: 
 25:     
 26:     // 省略 getMethodSignature 方法
 27: 
 28: }
```

* `logged` **静态**属性，已经打印日志的方法集合。
* 第 14 行：获得方法名。
* 第 16 至 21 行：打印告警日志。一个服务的方法，**有且仅有**打印一次。例如：

    ```
    [14/04/18 11:51:35:035 CST] main ERROR filter.DeprecatedFilter:  [DUBBO] The service method com.alibaba.dubbo.demo.DemoService.say01(String) is DEPRECATED! Declare from dubbo://192.168.3.17:20880/com.alibaba.dubbo.demo.DemoService?accesslog=true&anyhost=true&application=demo-consumer&callbacks=1000&check=false&client=netty4&default.delay=-1&default.retries=0&delay=-1&deprecated=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello,callbackParam,say03,say04,say01,bye,say02&payload=1000&pid=16820&qos.port=33333&register.ip=192.168.3.17&remote.timestamp=1523720843597&say01.deprecated=true&sayHello.async=true&server=netty4&service.filter=demo&side=consumer&timeout=100000&timestamp=1523721049491, dubbo version: 2.0.0, current host: 192.168.3.17
    ```
    * 注意，【第 18 行】会根据方法在判断。因为，一个服务里，可能只有**部分**方法废弃。

* 第 22 行：调用 `Invoker#invoke(invocation)` 方法，服务调用。

# 3. DeprecatedInvokerListener

功能**类似**，在 [《精尽 Dubbo 源码分析 —— 服务引用（一）之本地引用（Injvm）》「5.2 DeprecatedInvokerListener」](http://www.iocoder.cn/Dubbo/reference-refer-local/?self) 中，已经有详细解析。

# 666. 彩蛋

再水更一篇。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)




