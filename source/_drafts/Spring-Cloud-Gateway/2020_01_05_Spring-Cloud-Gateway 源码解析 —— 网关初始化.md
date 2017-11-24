# 1. 概述

本文主要分享 **Spring Cloud Gateway 启动初始化的过程**。

在初始化的过程中，涉及到的组件会较多，本文不会细说，留到后面每篇文章针对每个组件详细述说。

在官方提供的实例项目 `spring-cloud-gateway-sample` ，我们看到 GatewaySampleApplication 上有 `@EnableAutoConfiguration` 注解。因为该项目导入了 `spring-cloud-gateway-core` 依赖库，它会扫描 Spring Cloud Gateway 的配置。

在 [`org.springframework.cloud.gateway.config`](https://github.com/YunaiV/spring-cloud-gateway/tree/f552f51fc42db9ed88f783dc5f1291a22b34dcbc/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/config) 包下，我们可以看到**四个** Configuration 类 ：

* GatewayAutoConfiguration
* GatewayClassPathWarningAutoConfiguration
* GatewayLoadBalancerClientAutoConfiguration
* GatewayRedisAutoConfiguration

它们的初始化顺序如下图 ：

[](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_05/01.png)

-------

**推荐 Spring Cloud 书籍**：

* 请支持正版。下载盗版，**等于主动编写低级 BUG** 。
* 程序猿DD —— [《Spring Cloud微服务实战》](https://union-click.jd.com/jdc?d=505Twi)
* 周立 —— [《Spring Cloud与Docker微服务架构实战》](https://union-click.jd.com/jdc?d=k3sAaK)
* 两书齐买，京东包邮。

# 2. GatewayClassPathWarningAutoConfiguration

Spring Cloud Gateway 2.x 基于 Spring WebFlux 实现，GatewayClassPathWarningAutoConfiguration 用于检查项目是否**正确**导入 `spring-boot-starter-webflux` 依赖，而不是错误**导入** `spring-boot-starter-web` 依赖。

点击链接 [链接]() 查看 GatewayClassPathWarningAutoConfiguration 的代码实现。

# 3. GatewayLoadBalancerClientAutoConfiguration

# 4. GatewayRedisAutoConfiguration

# 5. GatewayAutoConfiguration


