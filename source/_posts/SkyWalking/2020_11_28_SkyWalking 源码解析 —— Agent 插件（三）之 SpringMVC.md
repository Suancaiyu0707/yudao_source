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

[`org.skywalking.apm.plugin.spring.mvc.v4.define.InvocableHandlerInstrumentation`](https://github.com/YunaiV/skywalking/blob/abed55324ba1d9b870992dedb97a9405eac38b4d/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-4.x-plugin/src/main/java/org/skywalking/apm/plugin/spring/mvc/v4/define/InvocableHandlerInstrumentation.java) ，实现 AbstractSpring4Instrumentation 抽象类，定义了方法切面，代码如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_28/07.png)

* 拦截 [`InvocableHandlerMethod#invokeForRequest(NativeWebRequest, ModelAndViewContainer, Object... providedArgs)`](https://github.com/spring-projects/spring-framework/blob/master/spring-web/src/main/java/org/springframework/web/method/support/InvocableHandlerMethod.java#L128) 方法，提交给 InvokeForRequestInterceptor 处理。

### 4.3.1 InvokeForRequestInterceptor

[`org.skywalking.apm.plugin.spring.mvc.commons.interceptor.InvokeForRequestInterceptor`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/java/org/skywalking/apm/plugin/dubbo/DubboInterceptor.java) ，实现 InstanceMethodsAroundInterceptor 接口，InvocableHandlerInstrumentation 的拦截器。代码如下：

* [`#beforeMethod(...)`](https://github.com/YunaiV/skywalking/blob/9cf741d99ea11f75e6759a8a7c7b58b1b245ce01/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/InvokeForRequestInterceptor.java#L37) 方法，代码如下：
    * 第 42 行：调用 `EnhancedInstance#setSkyWalkingDynamicField(value)` 方法，设置 NativeWebRequest 到 ServletInvocableHandlerMethod 的私有变量( SkyWalking 自动生成 )。
        * `objInst` 类型为 ServletInvocableHandlerMethod 类( 继承 InvocableHandlerMethod 类 )，每次请求都会创建新的该对象，因此设置 NativeWebRequest 对象，线程安全。
        * `allArguments[0]` 类型为 NativeWebRequest 类。

## 4.4 HandlerMethodInstrumentation

[`org.skywalking.apm.plugin.spring.mvc.v4.define.HandlerMethodInstrumentation`](https://github.com/YunaiV/skywalking/blob/d24f0538f84f033f27eafc9eda887493c60dafe8/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-4.x-plugin/src/main/java/org/skywalking/apm/plugin/spring/mvc/v4/define/HandlerMethodInstrumentation.java) ，实现 AbstractSpring4Instrumentation 抽象类，定义了方法切面，代码如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_28/07.png)

* 拦截 [`HandlerMethod#getBean()`](https://github.com/spring-projects/spring-framework/blob/master/spring-web/src/main/java/org/springframework/web/method/HandlerMethod.java#L195) 方法，提交给 GetBeanInterceptor 处理。
* **注意**，上面我们看到的 ServletInvocableHandlerMethod 继承的 InvocableHandlerMethod 类，继承了 HandlerMethod.java 类。（绕口令）。

### 4.4.1 GetBeanInterceptor

[`org.skywalking.apm.plugin.spring.mvc.commons.interceptor.GetBeanInterceptor`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/java/org/skywalking/apm/plugin/dubbo/DubboInterceptor.java) ，实现 InstanceMethodsAroundInterceptor 接口，InvocableHandlerInstrumentation 的拦截器。代码如下：

* [`#afterMethod(...)`](https://github.com/YunaiV/skywalking/blob/d24f0538f84f033f27eafc9eda887493c60dafe8/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/GetBeanInterceptor.java#L42) 方法，代码如下：
    * 第 44 至 48 行： 调用 `EnhancedInstance#setSkyWalkingDynamicField(value)` 方法，将 NativeRequest 设置到 Controller 的 EnhanceRequireObjectCache 的 `nativeWebRequest` 属性中。其中，NativeRequest 来自 InvokeForRequestInterceptor 拦截设置，而  EnhanceRequireObjectCache 来自 ControllerConstructorInterceptor 拦截设置。
        * **注意**，`nativeWebRequest` 属性，当前 Request 对象。**因为一个 Controller 对应一个EnhanceRequireObjectCache 对象，`nativeWebRequest` 对象被多线程共享时会冲突，在 SkyWalking 5.x 版本会修改成 ThreadLocal 属性，解决并发问题**。 

## 4.5 ControllerInstrumentation

[`org.skywalking.apm.plugin.spring.mvc.v4.define.ControllerInstrumentation`](https://github.com/YunaiV/skywalking/blob/1e948e06ef1228ff6fcc38d2cef9483938419f75/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-4.x-plugin/src/main/java/org/skywalking/apm/plugin/spring/mvc/v4/define/ControllerInstrumentation.java) ，实现 AbstractControllerInstrumentation 抽象类，定义了方法切面，代码如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_28/08.png)

* 拦截 `@Controller` 注解的 Controller 类。

### 4.5.1 AbstractMethodInteceptor

[`org.skywalking.apm.plugin.spring.mvc.commons.interceptor.AbstractMethodInteceptor`](https://github.com/YunaiV/skywalking/blob/1e948e06ef1228ff6fcc38d2cef9483938419f75/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/AbstractMethodInteceptor.java) ，实现 InstanceMethodsAroundInterceptor 接口，AbstractControllerInstrumentation 的拦截器的**抽象基类**。代码如下：

* [`#getRequestURL(Method)`](https://github.com/YunaiV/skywalking/blob/1e948e06ef1228ff6fcc38d2cef9483938419f75/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/AbstractMethodInteceptor.java#L43) **抽象方法**，获得方法对应的请求路径。
* 总体逻辑和 Tomcat 的 TomcatInvokeInterceptor 基本类似。
* [`#beforeMethod(...)`](https://github.com/YunaiV/skywalking/blob/1e948e06ef1228ff6fcc38d2cef9483938419f75/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/AbstractMethodInteceptor.java#L46) 方法，创建 EntrySpan 对象。代码如下：
    * 第 49 至 55 行：获得请求地址。首先，从 EnhanceRequireObjectCache 缓存中获取；其次，调用 `#getRequestURL(Method)` 方法，从类+方法的注解获取，并缓存。
   * 第 58 至 64 行：解析 ContextCarrier 对象，用于跨进程的链路追踪。在 [《SkyWalking 源码分析 —— Agent 收集 Trace 数据》「 3.2.3 ContextCarrier 」](http://www.iocoder.cn/SkyWalking/agent-collect-trace/?self) 有详细解析。
    * 第 67 行：调用 `ContextManager#createEntrySpan(operationName, contextCarrier)` 方法，创建 EntrySpan 对象。
        * 注意，大多数情况下，我们部署基于 SpringMVC 框架在 Tomcat 下，那 Tomcat 的 TomcatInvokeInterceptor 也会创建 EntrySpan 对象，而 AbstractMethodInteceptor 也会创建 EntrySpan 对象，会不会重复创建？在 [《SkyWalking 源码分析 —— Agent 收集 Trace 数据》](http://www.iocoder.cn/SkyWalking/agent-collect-trace/?self) 有答案哟。
    * 第 70 至 71 行：设置 EntrySpan 对象的 `url` / `http.method` 标签键值对。
    * 第 74 行：设置 EntrySpan 对象的组件类型。
    * 第 77 行：设置 EntrySpan 对象的分层。
* [`#afterMethod(...)`](https://github.com/YunaiV/skywalking/blob/1e948e06ef1228ff6fcc38d2cef9483938419f75/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/AbstractMethodInteceptor.java#L81) 方法，完成 EntrySpan 对象。
    * 第 86 至 92 行：当返回状态码大于等于 400 时，标记 EntrySpan 发生异常，并设置 `status_code` 标签键值对。
    * 第 95 行：调用 `ContextManager#stopSpan()` 方法，完成 EntrySpan 对象。
* [`#handleMethodException(...)`](https://github.com/YunaiV/skywalking/blob/1e948e06ef1228ff6fcc38d2cef9483938419f75/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/AbstractMethodInteceptor.java#L100) 方法，处理异常。代码如下：
    * 第 104 行：调用 `AbstractSpan#errorOccurred()` 方法，标记 EntrySpan 对象发生异常。
    * 第 106 行：调用 `AbstractSpan#log(Throwable)` 方法，记录异常日志到 EntrySpan 对象。

### 4.5.2 RequestMappingMethodInterceptor

[`org.skywalking.apm.plugin.spring.mvc.commons.interceptor.RequestMappingMethodInterceptor`](https://github.com/YunaiV/skywalking/blob/1e948e06ef1228ff6fcc38d2cef9483938419f75/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/RequestMappingMethodInterceptor.java) ，实现 AbstractMethodInteceptor 抽象类，实现了 [`#getRequestURL(Method)`](https://github.com/YunaiV/skywalking/blob/1e948e06ef1228ff6fcc38d2cef9483938419f75/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/RequestMappingMethodInterceptor.java#L34) 方法，生成 `@RequestMapping` 注解方法的请求路径。

### 4.5.3 RestMappingMethodInterceptor

[`org.skywalking.apm.plugin.spring.mvc.commons.interceptor.RestMappingMethodInterceptor`](https://github.com/YunaiV/skywalking/blob/1e948e06ef1228ff6fcc38d2cef9483938419f75/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/RestMappingMethodInterceptor.java) ，实现 AbstractMethodInteceptor 抽象类，实现了 [`#getRequestURL(Method)`](https://github.com/YunaiV/skywalking/blob/1e948e06ef1228ff6fcc38d2cef9483938419f75/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-commons/src/main/java/org/skywalking/apm/plugin/spring/mvc/commons/interceptor/RestMappingMethodInterceptor.java#L36) 方法，生成 `@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping` / `@PatchMapping` 注解方法的请求路径。

## 4.6 RestControllerInstrumentation

类似 [「 4.5 ControllerInstrumentation 」](#) 。 

[`org.skywalking.apm.plugin.spring.mvc.v4.define.RestControllerInstrumentation`](https://github.com/YunaiV/skywalking/blob/1e948e06ef1228ff6fcc38d2cef9483938419f75/apm-sniffer/apm-sdk-plugin/spring-plugins/mvc-annotation-4.x-plugin/src/main/java/org/skywalking/apm/plugin/spring/mvc/v4/define/RestControllerInstrumentation) ，实现 AbstractControllerInstrumentation 抽象类，定义了方法切面，代码如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_11_28/09.png)

* 拦截 `@RestController` 注解的 Controller 类。

# 5. mvc-annotation-3.x-plugin

类似 [「 4. mvc-annotation-4.x-plugin 」](#) 。 

考虑到 Spring MVC 5.x 都出了，本小节就暂不解析了。

[](http://www.iocoder.cn/images/SkyWalking/2020_11_28/10.png)

# 666. 彩蛋

Spring 的体系，真的是博大精深！被 `core-patch` 部分卡了好久，虽然现在还是有点模糊。

[](http://www.iocoder.cn/images/SkyWalking/2020_11_28/11.png)


