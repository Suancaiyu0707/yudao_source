title: 精尽 Dubbo 源码分析 —— 过滤器（九）之 TpsLimitFilter
date: 2018-11-20
tags:
categories: Dubbo
permalink: Dubbo/filter-limit-filter

-------

摘要: 原创出处 http://www.iocoder.cn/Dubbo/filter-limit-filter/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Dubbo/filter-limit-filter/)
- [2. TpsLimitFilter](http://www.iocoder.cn/Dubbo/filter-limit-filter/)
- [3. TPSLimiter](http://www.iocoder.cn/Dubbo/filter-limit-filter/)
  - [3.1 DefaultTPSLimiter](http://www.iocoder.cn/Dubbo/filter-limit-filter/)
  - [3.2 StatItem](http://www.iocoder.cn/Dubbo/filter-limit-filter/)
- [666. 彩蛋](http://www.iocoder.cn/Dubbo/filter-limit-filter/)

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

本文分享 TpsLimitFilter 过滤器，用于服务**提供者**中，提供 **限流** 的功能。

**配置方式**

① 通过 `<dubbo:parameter key="tps" value="" />` 配置项，添加到 `<dubbo:service />` 或 `<dubbo:provider />` 或 `<dubbo:protocol />` 中开启，例如：

```Java
<dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoServiceImpl" protocol="injvm" >
    <dubbo:parameter key="tps" value="100" />
</dubbo:service>
```

② 通过 `<dubbo:parameter key="tps.interval" value="" />` 配置项，设置 TPS **周期**。

**注意**

笔者阅读的 Dubbo 版本，目前暂未配置 TpsLimitFilter 到 Dubbo SPI 文件里，所以我们需要添加到 `com.alibaba.dubbo.rpc.Filter` 中，例如：

```
tps=com.alibaba.dubbo.rpc.filter.TpsLimitFilter
```

# 2. TpsLimitFilter

`com.alibaba.dubbo.rpc.filter.TpsLimitFilter` ，实现 Filter 接口，TPS 限流过滤器实现类。代码如下：

```Java
  1: @Activate(group = Constants.PROVIDER, value = Constants.TPS_LIMIT_RATE_KEY)
  2: public class TpsLimitFilter implements Filter {
  3: 
  4:     private final TPSLimiter tpsLimiter = new DefaultTPSLimiter();
  5: 
  6:     @Override
  7:     public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
  8:         if (!tpsLimiter.isAllowable(invoker.getUrl(), invocation)) {
  9:             throw new RpcException(
 10:                     new StringBuilder(64)
 11:                             .append("Failed to invoke service ")
 12:                             .append(invoker.getInterface().getName())
 13:                             .append(".")
 14:                             .append(invocation.getMethodName())
 15:                             .append(" because exceed max service tps.")
 16:                             .toString());
 17:         }
 18:         // 服务调用
 19:         return invoker.invoke(invocation);
 20:     }
 21: 
 22: }
```

* 第 8 至 17 行：调用 `TPSLimiter#isAllowable(url, invocation)` 方法，根据 tps 限流规则判断是否限制此次调用。若是，抛出 RpcException 异常。目前使用 TPSLimiter 作为限流器的实现类。
* 第 19 行：调用 `Invoker#invoke(invocation)` 方法，服务调用。

# 3. TPSLimiter

`com.alibaba.dubbo.rpc.filter.tps.TPSLimiter` ，TPS 限制器接口。代码如下：

```Java
public interface TPSLimiter {

    /**
     * judge if the current invocation is allowed by TPS rule
     *
     * 根据 tps 限流规则判断是否限制此次调用.
     *
     * @param url        url
     * @param invocation invocation
     * @return true allow the current invocation, otherwise, return false
     */
    boolean isAllowable(URL url, Invocation invocation);

}
```

## 3.1 DefaultTPSLimiter

`com.alibaba.dubbo.rpc.filter.tps.DefaultTPSLimiter` ，实现 TPSLimiter 接口，**默认** TPS 限制器实现类，**以服务为维度**。代码如下：

```Java
  1: public class DefaultTPSLimiter implements TPSLimiter {
  2: 
  3:     /**
  4:      * StatItem 集合
  5:      *
  6:      * key：服务名
  7:      */
  8:     private final ConcurrentMap<String, StatItem> stats = new ConcurrentHashMap<String, StatItem>();
  9: 
 10:     @Override
 11:     public boolean isAllowable(URL url, Invocation invocation) {
 12:         // 获得 TPS 大小配置项
 13:         int rate = url.getParameter(Constants.TPS_LIMIT_RATE_KEY, -1);
 14:         // 获得 TPS 周期配置项，默认 60 秒
 15:         long interval = url.getParameter(Constants.TPS_LIMIT_INTERVAL_KEY, Constants.DEFAULT_TPS_LIMIT_INTERVAL);
 16:         String serviceKey = url.getServiceKey();
 17:         // 要限流
 18:         if (rate > 0) {
 19:             // 获得 StatItem 对象
 20:             StatItem statItem = stats.get(serviceKey);
 21:             // 不存在，则进行创建
 22:             if (statItem == null) {
 23:                 stats.putIfAbsent(serviceKey, new StatItem(serviceKey, rate, interval));
 24:                 statItem = stats.get(serviceKey);
 25:             }
 26:             // 根据 TPS 限流规则判断是否限制此次调用.
 27:             return statItem.isAllowable(url, invocation);
 28:         // 不限流
 29:         } else {
 30:             // 移除 StatItem
 31:             StatItem statItem = stats.get(serviceKey);
 32:             if (statItem != null) {
 33:                 stats.remove(serviceKey);
 34:             }
 35:             // 返回通过
 36:             return true;
 37:         }
 38:     }
 39: 
 40: }
```

* `stats` 属性，StatItem 集合，Key 为 服务名，**即以服务为维度**。
* 第 13 行：获得 TPS 大小配置项 `"tps"`。
* 第 15 行：获得 TPS 周期配置项 `"tps.interval"`，默认 60 * 1000 毫秒。
* 第 17 至 27 行：若要限流，调用 `StatItem#isAllowable(url, invocation)` 方法，根据 TPS 限流规则判断是否限制此次调用。
* 第 28 至 37 行：若不限流，移除 StatItem 对象。

## 3.2 StatItem

`com.alibaba.dubbo.rpc.filter.tps.StatItem` ，统计项。

### 3.2.1 构造方法

```Java
/**
 * 统计名，目前使用服务名
 */
private String name;
/**
 * 周期
 */
private long interval;
/**
 * 限制大小
 */
private int rate;
/**
 * 最后重置时间
 */
private long lastResetTime;
/**
 * 当前周期，剩余种子数
 */
private AtomicInteger token;

StatItem(String name, int rate, long interval) {
    this.name = name;
    this.rate = rate;
    this.interval = interval;
    this.lastResetTime = System.currentTimeMillis();
    this.token = new AtomicInteger(rate);
}
```

### 3.2.2 isAllowable

```Java
public boolean isAllowable(URL url, Invocation invocation) {
    // 若到达下一个周期，恢复可用种子数，设置最后重置时间。
    long now = System.currentTimeMillis();
    if (now > lastResetTime + interval) {
        token.set(rate); // 回复可用种子数
        lastResetTime = now; // 最后重置时间
    }

    // CAS ，直到或得到一个种子，或者没有足够种子
    int value = token.get();
    boolean flag = false;
    while (value > 0 && !flag) {
        flag = token.compareAndSet(value, value - 1);
        value = token.get();
    }

    // 是否成功
    return flag;
}
```

# 666. 彩蛋

实际在服务的限流时，更推荐使用 **令牌桶算法** ，在 [《Eureka 源码解析 —— 基于令牌桶算法的 RateLimiter》](http://www.iocoder.cn/Eureka/rate-limiter/?self) 中，我们有详细分享。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

