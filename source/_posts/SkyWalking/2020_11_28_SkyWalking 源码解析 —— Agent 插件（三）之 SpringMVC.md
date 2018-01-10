title: SkyWalking 源码分析 —— Agent 插件（三）之 SpringMVC
date: 2020-11-28
tags:
categories: SkyWalking
permalink: SkyWalking/agent-plugin-spring-mvc

-------

# 1. 概述

# 2. core-patch

[`core-patch`](https://github.com/YunaiV/skywalking/tree/e4bfdfad2540adbc85b4437359b9a5183f05f403/apm-sniffer/apm-sdk-plugin/spring-plugins/core-patch) 模块，给 Spring 打补丁，解决因为 Agent 对类的增强操作导致的冲突。

> 打脸提示：笔者对 Spring 的一些( 大部分 )机制了解的较浅薄，所以本小节更多的是粘贴代码 + 相关 Issue 。

## 2.1 AopProxyFactoryInstrumentation

原因和目的，参见 [Issue#581](https://github.com/apache/incubator-skywalking/issues/581#issuecomment-343386006) 。

SkyWalking Agent 在增强类的构造方法或者实例方法时，会自动实现 [EnhancedInstance](https://github.com/YunaiV/skywalking/blob/ea8b4e879092b39070215b1a2d194e6df12f0ef8/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/interceptor/enhance/EnhancedInstance.java) 接口，导致 Spring 的 `DefaultAopProxyFactory#hasNoUserSuppliedProxyInterfaces(AdvisedSupport)` 返回 `false` 错误，实际应该返回 `true` 。

```
// DefaultAopProxyFactory.java
/**
 * Determine whether the supplied {@link AdvisedSupport} has only the
 * {@link org.springframework.aop.SpringProxy} interface specified
 * (or no proxy interfaces specified at all).
 */
private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
	Class<?>[] ifcs = config.getProxiedInterfaces();
	return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
}
```

-------

[`org.skywalking.apm.plugin.tomcat78x.define.TomcatInstrumentation`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/tomcat-7.x-8.x-plugin/src/main/java/org/skywalking/apm/plugin/tomcat78x/define/TomcatInstrumentation.java) ，实现 ClassInstanceMethodsEnhancePluginDefine 抽象类，定义了方法切面，代码如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_28/02.png)

-------

[`org.skywalking.apm.plugin.spring.patch.CreateAopProxyInterceptor`](https://github.com/YunaiV/skywalking/blob/9ee7c4766ae202bf0b33df49210386acb3ef5f34/apm-sniffer/apm-sdk-plugin/spring-plugins/core-patch/src/main/java/org/skywalking/apm/plugin/spring/patch/CreateAopProxyInterceptor.java) ，实现 InstanceMethodsAroundInterceptor 接口，ClassInstanceMethodsEnhancePluginDefine 的拦截器。代码如下：

* [`#afterMethod(...)`](https://github.com/YunaiV/skywalking/blob/9ee7c4766ae202bf0b33df49210386acb3ef5f34/apm-sniffer/apm-sdk-plugin/spring-plugins/core-patch/src/main/java/org/skywalking/apm/plugin/spring/patch/CreateAopProxyInterceptor.java#L43) 方法，代码如下：
    * 第 47 行：若目标类实现了 EnhancedInstance 接口，返回 `true` 。
    * 第 50 行：否则，返回原有结果 `ret` 。 

## 2.2 AutowiredAnnotationProcessorInstrumentation

原因和目的，参见 [Issue#622](https://github.com/apache/incubator-skywalking/issues/622) 和 [Issue#624](https://github.com/apache/incubator-skywalking/pull/624)。

Spring 的 [`AutowiredAnnotationBeanPostProcessor#determineCandidateConstructors(Class, String)`](https://github.com/spring-projects/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.java#L239) 方法，返回有三种情况：

1. 带有 `@Autowired` 参数的构造方法
2. **仅有一个带参数的构造方法**
3. 不带参数的构造方法

因为 SkyWalking 增强机制会生成**一个私有构造方法**，导致所有被增强的类原先满足第二种情况的，Spring 选择了第三种情况，导致报构造方法不存在。

通过 AutowiredAnnotationProcessorInterceptor ，会**过滤掉私有构造方法**，从而解决冲突问题。

-------

[`org.skywalking.apm.plugin.spring.patch.define.AutowiredAnnotationProcessorInstrumentation`](https://github.com/YunaiV/skywalking/blob/684a837f39f9ca8fb66a84bea9b28678bc632382/apm-sniffer/apm-sdk-plugin/spring-plugins/core-patch/src/main/java/org/skywalking/apm/plugin/spring/patch/define/AutowiredAnnotationProcessorInstrumentation.java) ，实现 ClassInstanceMethodsEnhancePluginDefine 抽象类，定义了方法切面，代码如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_28/03.png)

-------

[`org.skywalking.apm.plugin.spring.patch.CreateAopProxyInterceptor`](https://github.com/YunaiV/skywalking/blob/684a837f39f9ca8fb66a84bea9b28678bc632382/apm-sniffer/apm-sdk-plugin/spring-plugins/core-patch/src/main/java/org/skywalking/apm/plugin/spring/patch/AutowiredAnnotationProcessorInterceptor.java) ，实现 InstanceMethodsAroundInterceptor 和 InstanceConstructorInterceptor接口，AutowiredAnnotationProcessorInstrumentation 的拦截器。代码如下：

* [`#onConstruct(...)`](https://github.com/YunaiV/skywalking/blob/684a837f39f9ca8fb66a84bea9b28678bc632382/apm-sniffer/apm-sdk-plugin/spring-plugins/core-patch/src/main/java/org/skywalking/apm/plugin/spring/patch/AutowiredAnnotationProcessorInterceptor.java#L113) 方法，创建类与构造方法的映射。代码如下：
    * 第 115 行：创建类与构造方法的映射 `candidateConstructorsCache` ，**用于缓存**。
    * 第 117 行：设置到私有变量( SkyWalking 增强生成 )。 
* [`#afterMethod(...)`](https://github.com/YunaiV/skywalking/blob/684a837f39f9ca8fb66a84bea9b28678bc632382/apm-sniffer/apm-sdk-plugin/spring-plugins/core-patch/src/main/java/org/skywalking/apm/plugin/spring/patch/AutowiredAnnotationProcessorInterceptor.java#L49) 方法，处理自动实现   EnhancedInstance 接口的类，和 Spring 的 `AutowiredAnnotationBeanPostProcessor#determineCandidateConstructors(Class, String)` 的冲突。代码如下：
    * 第 54 行：若 `beanClass` 实现了 EnhancedInstance 接口。
    * 第 56 至 57 行：从 `candidateConstructorsCache` 缓存中获得构造方法。
    * 第 58 行：缓存中不存在对应的构造方法，遍历 `beanClass` 的类的构造方法，缓存并返回。
        * ----- `ret == null` 原本方法没找到构造方法，存在冲突 -----
        * 第 80 至 86 行：获得构造方法集合，排除私有构造方法。**为什么排除私有构造方法**？因为 SkyWalking 与 Spring 的冲突，就是因为 SkyWalking 自动生成的私有构造方法，所以需要排除。
        * 第 89 至 90 行：【冲突点】**让第二种情况，依然走第二种**。
        * 第 91 至 94 行：选择第一个构造方法。
        * ----- `ret != null` 原本方法就找到构造方法，不存在冲突 ----- 
        * 第 97 行：**使用原本方法就找到构造方法**。
        * ----- all -----
        * 第 100 行：缓存构造方法到 `candidateConstructorsCache` 中。 
    * 第 103 行：返回结果。

ps：这块略复杂，如果笔者未解释清晰，那是因为我菜。

# 3. mvc-annotation-commons

[`mvc-annotation-commons`](https://github.com/YunaiV/skywalking/tree/e4bfdfad2540adbc85b4437359b9a5183f05f403/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons) 模块，提供公用代码，提供给 `mvc-annotation-4.x-plugin` 和 `mvc-annotation-3.x-plugin` 使用。

## 3.1 PathMappingCache

[`org.skywalking.apm.plugin.spring.mvc.commons.PathMappingCache`](https://github.com/YunaiV/skywalking/blob/ecc6e2aad35769204beba993554692c1cdd2010d/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/PathMappingCache.java) ，缓存 Controller 的所有请求路径，一个 Controller 对象一个 PathMappingCache 对象。代码如下：

* `classPath` 属性，类的请求路径。
* `methodPathMapping` 属性，方法对象与请求路径的映射。
* [`#addPathMapping(method, methodPath)`](https://github.com/YunaiV/skywalking/blob/ecc6e2aad35769204beba993554692c1cdd2010d/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/PathMappingCache.java#L49) 方法，**添加**方法对应的请求路径到映射。
* [`#findPathMapping(method)`]() 方法，从映射中，**查询**方法对应的请求路径。

## 3.2 EnhanceRequireObjectCache

[`org.skywalking.apm.plugin.spring.mvc.commons.EnhanceRequireObjectCache`](https://github.com/YunaiV/skywalking/blob/ecc6e2aad35769204beba993554692c1cdd2010d/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/EnhanceRequireObjectCache.java) ，在 PathMappingCache 的基础上，增加 `nativeWebRequest` 属性。**实际上，一个 Controller 对象一个 EnhanceRequireObjectCache 对象**。代码如下：

* `pathMappingCache` 属性，[「 3.2 PathMappingCache 」](#) 对象。
* `nativeWebRequest` 属性，当前 Request 对象。**因为一个 Controller 对应一个EnhanceRequireObjectCache 对象，`nativeWebRequest` 对象被多线程共享时会冲突，在 SkyWalking 5.x 版本会修改成 ThreadLocal 属性，解决并发问题**。

## 3.3 Constants

[`org.skywalking.apm.plugin.spring.mvc.commons.Constants`](https://github.com/YunaiV/skywalking/blob/ecc6e2aad35769204beba993554692c1cdd2010d/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/Constants.java) ，枚举 [`org.skywalking.apm.plugin.spring.mvc.commons.interceptor`](https://github.com/YunaiV/skywalking/blob/ecc6e2aad35769204beba993554692c1cdd2010d/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/AbstractMethodInteceptor.java) 包下的拦截器类名。

## 3.4 拦截器

[`org.skywalking.apm.plugin.spring.mvc.commons.interceptor`](https://github.com/YunaiV/skywalking/blob/ecc6e2aad35769204beba993554692c1cdd2010d/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/AbstractMethodInteceptor.java) 包下共有**四种**拦截器，如下图：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_28/04.png)

结合 [「 4. mvc-annotation-4.x-plugin 」](#) ，我们一起分享。

# 4. mvc-annotation-4.x-plugin

本小节涉及到的类如下图：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_28/05.png)

我们整理如下：


| Instrumentation | Interceptor |  |
| --- | --- | --- |
| AbstractControllerInstrumentation | ControllerConstructorInterceptor |  |
| InvocableHandlerInstrumentation | InvokeForRequestInterceptor |  |
| HandlerMethodInstrumentation | GetBeanInterceptor |  |
| ControllerInstrumentation | RequestMappingMethodInterceptor |  |
| RestControllerInstrumentation | RestMappingMethodInterceptor |  |

## 4.1 AbstractSpring4Instrumentation

[`org.skywalking.apm.plugin.spring.mvc.v4.define.AbstractSpring4Instrumentation`](https://github.com/YunaiV/skywalking/blob/abed55324ba1d9b870992dedb97a9405eac38b4d/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-4.x-plugin/src/main/java/org/skywalking/apm/plugin/spring/mvc/v4/define/AbstractSpring4Instrumentation.java) ，实现 ClassInstanceMethodsEnhancePluginDefine 抽象类，**所有 Spring MVC 4.x 的 Instrumentation 的抽象基类**。通过定义 [`#witnessClasses()`](https://github.com/YunaiV/skywalking/blob/abed55324ba1d9b870992dedb97a9405eac38b4d/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-4.x-plugin/src/main/java/org/skywalking/apm/plugin/spring/mvc/v4/define/AbstractSpring4Instrumentation.java#L33) 方法，声明 Spring MVC 4.x 的插件生效，需要项目里包括 `org.springframework.web.servlet.tags.ArgumentTag` 类。

通过这样的方式，区分 Spring MVC 4.x 和 3.x 的插件。ArgumentTag 在 Spring MVC 3.x 是不存在的。

`#witnessClasses()` 的相关方法，在 [《SkyWalking 源码分析 —— Agent 插件体系》](http://www.iocoder.cn/SkyWalking/agent-plugin-system/) 有详细解析。

## 4.2 AbstractControllerInstrumentation

[`org.skywalking.apm.plugin.spring.mvc.v4.define.AbstractControllerInstrumentation`](https://github.com/YunaiV/skywalking/blob/cc0f60e120ee5c51a6c063e41cbe8af78e7e7a92/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-4.x-plugin/src/main/java/org/skywalking/apm/plugin/spring/mvc/v4/define/AbstractControllerInstrumentation.java) ，实现 AbstractSpring4Instrumentation 抽象类，定义了方法切面，代码如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_28/06.png)

分成两部分：

* ConstructorInterceptPoint 部分，拦截所有构造方法给 [「 4.2.1 ControllerConstructorInterceptor 」](#) 处理。
* InstanceMethodsInterceptPoint 部分，根据不同的 Mapping 方法注解，拦截提交给 [todo]() 或者 [todo]() 处理。 

AbstractControllerInstrumentation 是一个**抽象基类**，有 [「 4.5 ControllerInstrumentation 」](#) ，[「 4.6 RestControllerInstrumentation 」](#) 两个子类，实现 [`#getEnhanceAnnotations()`](https://github.com/YunaiV/skywalking/blob/cc0f60e120ee5c51a6c063e41cbe8af78e7e7a92/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-4.x-plugin/src/main/java/org/skywalking/apm/plugin/spring/mvc/v4/define/AbstractControllerInstrumentation.java#L115) 抽象方法，返回**不同**的类注解，从而拦截不同的类。

### 4.2.1 ControllerConstructorInterceptor

[`org.skywalking.apm.plugin.spring.mvc.v4.ControllerConstructorInterceptor`](https://github.com/YunaiV/skywalking/blob/cc0f60e120ee5c51a6c063e41cbe8af78e7e7a92/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-4.x-plugin/src/main/java/org/skywalking/apm/plugin/spring/mvc/v4/ControllerConstructorInterceptor.java) ，实现 InstanceConstructorInterceptor 接口，Abstract**Controller**Instrumentation 的拦截器。代码如下：

* [`#onConstruct(...)`](https://github.com/YunaiV/skywalking/blob/cc0f60e120ee5c51a6c063e41cbe8af78e7e7a92/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-4.x-plugin/src/main/java/org/skywalking/apm/plugin/spring/mvc/v4/ControllerConstructorInterceptor.java#L43) 方法，代码如下：
    * 第 45 至 53 行：解析类的请求路径。
    * 第 55 至 56 行：创建 EnhanceRequireObjectCache 缓存对象。
    * 第 59 行：调用 `EnhancedInstance#setSkyWalkingDynamicField(value)` 方法，设置到 Controller 的私有变量( SkyWalking 自动生成 )。即，Controller : EnhanceRequireObjectCache = `1 : 1` 。

## 4.3 InvocableHandlerInstrumentation

### 4.3.1 InvokeForRequestInterceptor

## 4.4 HandlerMethodInstrumentation

### 4.4.1 GetBeanInterceptor

## 4.5 ControllerInstrumentation

### 4.5.1 AbstractMethodInteceptor

### 4.5.2 RequestMappingMethodInterceptor

### 4.5.3 RestMappingMethodInterceptor

## 4.6 RestControllerInstrumentation

# 5. mvc-annotation-3.x-plugin

# 666. 彩蛋


