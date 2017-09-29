title: Eureka 源码解析 —— Eureka-Client 初始化（二）之 EurekaClientConfig
date: 2018-04-23
tags:
categories: Eureka
permalink: Eureka/eureka-client-init-second

---

摘要: 原创出处 http://www.iocoder.cn/Eureka/eureka-client-init-second/ 「芋道源码」欢迎转载，保留摘要，谢谢！



---

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文接[《Eureka 源码解析 —— Eureka-Client 初始化（一）之 EurekaInstanceConfig》](http://www.iocoder.cn/Eureka/eureka-client-init-second/?self)，主要分享 **Eureka-Client 自身初始化的过程**的第二部分 —— **EurekaClientConfig**，不包含 Eureka-Client 向 Eureka-Server 的注册过程( 🙂后面会另外文章分享 )。

Eureka-Client 自身初始化过程中，涉及到主要对象如下图：

![](http://www.iocoder.cn/images/Eureka/2018_04_15/01.png)

1. **创建** EurekaInstanceConfig对象
1. 使用 EurekaInstanceConfig对象 **创建** InstanceInfo对象
1. 使用 EurekaInstanceConfig对象 + InstanceInfo对象 **创建** ApplicationInfoManager对象
1. **创建** EurekaClientConfig对象
1. 使用 ApplicationInfoManager对象 + EurekaClientConfig对象 **创建** EurekaClient对象

考虑到整个初始化的过程中涉及的配置特别多，拆分成三篇文章：

1. （一）[EurekaInstanceConfig]((http://www.iocoder.cn/Eureka/eureka-client-init-first/))
2. **【本文】**（二）EurekaClientConfig
3. （三）[EurekaClient](http://www.iocoder.cn/Eureka/eureka-client-init-third/)

下面我们来看看每个**类**的实现。

# 2. EurekaClientConfig

`com.netflix.discovery.EurekaClientConfig`，**Eureka-Client** 配置**接口**。

## 2.1 类关系图

EurekaClientConfig 整体类关系如下图：

[](../../../images/Eureka/2018_04_22/04.png)

* 本文只解析**红圈**部分类。
* EurekaArchaius2ClientConfig 基于 [Netflix Archaius 2.x](https://github.com/Netflix/archaius) 实现，目前还在开发中，因此暂不解析。

## 2.2 配置属性

## 2.3 DefaultEurekaClientConfig

# 3. EurekaTransportConfig

# 666. 彩蛋

涉及到配置，内容初看起来会比较多，慢慢理解后，就会变得很“啰嗦”，请保持耐心。

胖友，分享一个朋友圈可好。

