title: 精尽 Dubbo 源码分析 —— 过滤器（八）之 TokenFilter
date: 2018-11-19
tags:
categories: Dubbo
permalink: Dubbo/filter-token-filter

-------

摘要: 原创出处 http://www.iocoder.cn/Dubbo/filter-token-filter/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Dubbo/filter-token-filter/)
- [2.【服务消费者】随机 Token](http://www.iocoder.cn/Dubbo/filter-token-filter/)
- [3.【服务消费者】接收 Token](http://www.iocoder.cn/Dubbo/filter-token-filter/)
- [3.【服务消费者】发送 Token](http://www.iocoder.cn/Dubbo/filter-token-filter/)
- [4.【服务提供者】认证 Token](http://www.iocoder.cn/Dubbo/filter-token-filter/)
- [666. 彩蛋](http://www.iocoder.cn/Dubbo/filter-token-filter/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。


# 1. 概述

本文分享 TokenFilter 过滤器，用于服务**提供者**中，提供 **令牌验证** 的功能。在 [《Dubbo 用户指南 —— 令牌验证》](https://dubbo.gitbooks.io/dubbo-user-book/demos/token-authorization.html) 定义如下：

> 通过令牌验证在**注册中心**控制权限，以决定要不要下发令牌给消费者，可以防止消费者绕过注册中心访问提供者。
> 
> 另外通过注册中心可灵活改变授权方式，而不需修改或升级提供者。
> 
> ![认证流程](http://www.iocoder.cn/images/Dubbo/2018_11_19/01.png)

* 官方文档写的很全，胖友请点击链接看完哈。

So ，可能和大多数胖友（包括我），一开始理解和期望的不太一样 🙂 。

# 2.【服务消费者】随机 Token

在 ServiceConfig 的 `#doExportUrlsFor1Protocol(protocolConfig, registryURLs)` 方法中，随机生成 Token ：

```Java
// token ，参见《令牌校验》https://dubbo.gitbooks.io/dubbo-user-book/demos/token-authorization.html
if (!ConfigUtils.isEmpty(token)) {
    if (ConfigUtils.isDefault(token)) { // true || default 时，UUID 随机生成
        map.put("token", UUID.randomUUID().toString());
    } else {
        map.put("token", token);
    }
}
```

# 3.【服务消费者】接收 Token

服务**消费者**，从注册中心，获取服务提供者的 **URL** ，从而获得该服务着的 Token 。  
所以，即使服务提供者随机生成 Token ，消费者一样可以拿到。

# 3.【服务消费者】发送 Token

RpcInvocation 在创建时，“**自动**”带上 Token ，如下图所示：

![RpcInvocation](http://www.iocoder.cn/images/Dubbo/2018_11_19/02.png)

# 4.【服务提供者】认证 Token

`com.alibaba.dubbo.rpc.filter.TokenFilter` ，实现 Filter 接口，**令牌验证** Filter 实现类。代码如下：

```Java
@Activate(group = Constants.PROVIDER, value = Constants.TOKEN_KEY)
public class TokenFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation inv) throws RpcException {
        // 获得服务提供者配置的 Token 值
        String token = invoker.getUrl().getParameter(Constants.TOKEN_KEY);
        if (ConfigUtils.isNotEmpty(token)) {
            // 从隐式参数中，获得 Token 值。
            Class<?> serviceType = invoker.getInterface();
            Map<String, String> attachments = inv.getAttachments();
            String remoteToken = attachments == null ? null : attachments.get(Constants.TOKEN_KEY);
            // 对比，若不一致，抛出 RpcException 异常
            if (!token.equals(remoteToken)) {
                throw new RpcException("Invalid token! Forbid invoke remote service " + serviceType + " method " + inv.getMethodName() + "() from consumer " + RpcContext.getContext().getRemoteHost() + " to provider " + RpcContext.getContext().getLocalHost());
            }
        }
        // 服务调用
        return invoker.invoke(inv);
    }

}
```

# 666. 彩蛋

再来一发水文。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

