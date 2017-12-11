title: SkyWalking 源码分析 —— Agent 插件体系
date: 2020-07-10
tags:
categories: SkyWalking
permalink: SkyWalking/agent-plugin-system

-------

# 1. 概述

本文主要分享 **SkyWalking Agent 插件体系**。主要涉及三个流程 ：

* 插件的加载
* 插件的匹配
* 插件的拦截

可能看起来有点抽象，不太容易理解。淡定，我们每个小章节进行解析。

本文涉及到的类主要在 [`org.skywalking.apm.agent.core.plugin`](https://github.com/YunaiV/skywalking/tree/3de8a6c15d07aa3b2c3b4e732e6654fc87c4e70e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin) 包里，如下图所示 ：

[](http://www.iocoder.cn/images/SkyWalking/2020_07_10/01.png)

每个流程会涉及到较多的类，我们会贯穿着解析代码实现。

# 2. 插件的加载

在 [《SkyWalking 源码分析 —— Agent 初始化》](http://www.iocoder.cn/SkyWalking/agent-init/?self) 一文中，Agent 初始化时，调用 `PluginBootstrap#loadPlugins()` 方法，加载所有的插件。整体流程如下图 ：

[](http://www.iocoder.cn/images/SkyWalking/2020_07_10/03.png)

[`PluginBootstrap#loadPlugins()`](https://github.com/YunaiV/skywalking/blob/130f0a5a3438663b393e53ba2cca02a8d13c258a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginBootstrap.java#L45) 方法，代码如下 ：

* 第 47 行 ：调用 `AgentClassLoader#initDefaultLoader()` 方法，初始化 AgentClassLoader 。在本文 [「2.1 AgentClassLoader」](#) 详细解析。
* 第 50 至 56 行 ：获得插件**定义路径**数组。在本文 [「2.2 PluginResourcesResolver」](#) 详细解析。
* 第 59 至 66 行 ：获得插件**定义**( [`org.skywalking.apm.agent.core.plugin.PluginDefine`](https://github.com/OpenSkywalking/skywalking/blob/b16d23c1484bec941367d6b36fa932b8ace40971/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginDefine.java) )数组。在本文 [「2.3 PluginCfg」](#) 详细解析。
* 第 69 至 82 行 ：创建**类增强插件定义**( [`org.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine`](https://github.com/OpenSkywalking/skywalking/blob/b16d23c1484bec941367d6b36fa932b8ace40971/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/AbstractClassEnhancePluginDefine.java) )对象数组。不同插件通过实现 AbstractClassEnhancePluginDefine **抽象类**，定义不同框架的**切面**，**记录调用链路**。在本文 [「2.4 AbstractClassEnhancePluginDefine」](#) 简单解析。

## 2.1 AgentClassLoader

`org.skywalking.apm.agent.core.plugin.loader.AgentClassLoader` ，继承 `java.lang.ClassLoader` ，Agent 类加载器。

**为什么实现自定义的 ClassLoader** ？应用**透明**接入 SkyWalking ，不会**显示**导入 SkyWalking 的插件依赖。通过实现自定义的 ClassLoader ，从插件 Jar 中查找相关类。例如说，从 `apm-dubbo-plugin-3.2.6-2017.jar` 查找 `org.skywalking.apm.plugin.dubbo.DubboInstrumentation` 。

-------

AgentClassLoader **构造方法**，代码如下 ：

```Java
public class AgentClassLoader extends ClassLoader {

    /**
     * The default class loader for the agent.
     */
    private static AgentClassLoader DEFAULT_LOADER;

    /**
     * classpath
     */
    private List<File> classpath;
    /**
     * Jar 数组
     */
    private List<Jar> allJars;
    /**
     * Jar 读取时的锁
     */
    private ReentrantLock jarScanLock = new ReentrantLock();

    public AgentClassLoader(ClassLoader parent) throws AgentPackageNotFoundException {
        super(parent);
        File agentDictionary = AgentPackagePath.getPath();
        classpath = new LinkedList<File>();
        classpath.add(new File(agentDictionary, "plugins"));
        classpath.add(new File(agentDictionary, "activations"));
    }
}
```

* `DEFAULT_LOADER` **静态**属性，默认单例。通过 [`#getDefault()`](https://github.com/YunaiV/skywalking/blob/3de8a6c15d07aa3b2c3b4e732e6654fc87c4e70e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/loader/AgentClassLoader.java#L56) 方法，可以获取到它。
* `classpath` 属性，Java 类所在的目录。在构造方法中，我们可以看到 `${AGENT_PACKAGE_PATH}/plugins` / `${AGENT_PACKAGE_PATH}/activations` 添加到 `classpath` 。在 [`#getAllJars()`](https://github.com/YunaiV/skywalking/blob/3de8a6c15d07aa3b2c3b4e732e6654fc87c4e70e/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/loader/AgentClassLoader.java#L163)  方法中，加载该目录下的 Jar 中的 Class 文件。
* `allJars` 属性，Jar 数组。
* `jarScanLock` 属性，Jar 读取时的**锁**。

-------

`#initDefaultLoader()` **静态**方法，初始化**默认**的 AgentClassLoader ，代码如下 ：

```Java
public static AgentClassLoader initDefaultLoader() throws AgentPackageNotFoundException {
    DEFAULT_LOADER = new AgentClassLoader(PluginBootstrap.class.getClassLoader());
    return getDefault();
}
```

* 使用 `org.skywalking.apm.agent.core.plugin.PluginBootstrap` 的类加载器作为 AgentClassLoader 的**父类加载器**。

-------

如下方法已经添加相关中文注释，胖友请自行阅读理解 ：

* [`#findResource(name)`](https://github.com/YunaiV/skywalking/blob/778093d38a0a820b90092c2ed77a08e3393169eb/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/loader/AgentClassLoader.java#L132)
* [`#findResources(String name)`](https://github.com/YunaiV/skywalking/blob/778093d38a0a820b90092c2ed77a08e3393169eb/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/loader/AgentClassLoader.java#L150)
* [`#getAllJars()`](https://github.com/YunaiV/skywalking/blob/778093d38a0a820b90092c2ed77a08e3393169eb/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/loader/AgentClassLoader.java#L182)

在 ClassLoader 加载资源( 例如，类 )，会调用 `#findResource(name)` / `#findResources(name)` 方法。

## 2.2 PluginResourcesResolver

`org.skywalking.apm.agent.core.plugin.PluginResourcesResolver` ，插件资源解析器，读取所有插件的定义文件。插件定义文件必须以 `skywalking-plugin.def` **命名**，例如 ：[](http://www.iocoder.cn/images/SkyWalking/2020_07_10/02.png)

[`#getResources()`](https://github.com/YunaiV/skywalking/blob/d4a6ba291419ab90379a3d1c423b747f682f857f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginResourcesResolver.java#L45) 方法，获得插件定义路径数组，代码如下 ：

* 第 50 行 ：使用 AgentClassLoader 获得所有 `skywalking-plugin.def` 的路径。

## 2.3 PluginCfg

`org.skywalking.apm.agent.core.plugin.PluginCfg` ，插件定义配置，读取 `skywalking-plugin.def` 文件，生成插件定义( [`org.skywalking.apm.agent.core.plugin.PluginDefinie`](https://github.com/YunaiV/skywalking/blob/43241fff19e17f19b918c96ffd588787f8f05519/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginDefine.java#L27) )数组。

[`#load(InputStream)`](https://github.com/YunaiV/skywalking/blob/43241fff19e17f19b918c96ffd588787f8f05519/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginCfg.java#L55) 方法，读取 `skywalking-plugin.def` 文件，添加到 `pluginClassList` 。如下是 `apm-springmvc-annotation-4.x-plugin-3.2.6-2017.jar` 插件的定义文件 ：

```
spring-mvc-annotation-4.x=org.skywalking.apm.plugin.spring.mvc.v4.define.ControllerInstrumentation
spring-mvc-annotation-4.x=org.skywalking.apm.plugin.spring.mvc.v4.define.RestControllerInstrumentation
spring-mvc-annotation-4.x=org.skywalking.apm.plugin.spring.mvc.v4.define.HandlerMethodInstrumentation
spring-mvc-annotation-4.x=org.skywalking.apm.plugin.spring.mvc.v4.define.InvocableHandlerInstrumentation
```

## 2.4 AbstractClassEnhancePluginDefine

`org.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine` ，类增强插件定义**抽象基类**。不同插件通过实现 AbstractClassEnhancePluginDefine **抽象类**，定义不同框架的**切面**，**记录调用链路**。以 Spring 插件为例子，如下是相关类图 ：[](http://www.iocoder.cn/images/SkyWalking/2020_07_05/06.png)

PluginDefine 对象的 `defineClass` 属性，即对应不同插件对AbstractClassEnhancePluginDefine 的**实现类**。所以在 [`PluginBootstrap#loadPlugins()`](https://github.com/YunaiV/skywalking/blob/130f0a5a3438663b393e53ba2cca02a8d13c258a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginBootstrap.java#L45) 方法的【**第 74 行**】，我们看到通过该属性，创建创建**类增强插件定义**对象。

TODO 详细入口

## 2.5 小结

胖友，回过头，在看一下流程图，理解理解。

# 3. 插件的匹配

在 [《SkyWalking 源码分析 —— Agent 初始化》](http://www.iocoder.cn/SkyWalking/agent-init/?self) 一文，我们提到，SkyWalking Agent 基于 **JavaAgent** 机制，实现应用**透明**接入 SkyWalking 。下面笔者默认胖友已经对 JavaAgent 机制已经有一定的了解。如果胖友暂时不了解，建议先阅读如下文章 ：

* [《Instrumentation 新功能》](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html)
* [《JVM源码分析之javaagent原理完全解读》](http://www.infoq.com/cn/articles/javaagent-illustrated)

> 友情提示 ：建议自己手撸一个简单的 JavaAgent ，更容易理解 SkyWalking Agent 。 
>  
> 笔者练手的 JavaAgent 项目地址 ：https://github.com/YunaiV/learning/tree/master/javaagent01

通过 JavaAgent 机制，我们可以在 `#premain(String, Instrumentation)` 方法里，调用 `Instrumentation#addTransformer(ClassFileTransformer)` 方法，向 Instrumentation 注册 [`java.lang.instrument.ClassFileTransformer`](https://docs.oracle.com/javase/7/docs/api/java/lang/instrument/ClassFileTransformer.html) 对象，可以修改 Java 类的二进制，从而**动态**修改 Java 类的代码实现。

如果胖友使用过 AOP 实现切面记录日志，那么就很容易理解，SkyWalking 通过这样的方式，使用不同框架定义**方法切面**，从而在在切面**记录调用链路**。

-------

直接修改 Java 类的二进制，是非常繁杂的。因此，SkyWalking 引入了 [`byte-buddy`](https://github.com/raphw/byte-buddy) 。

> `byte-buddy` 是一个代码生成和操作库，用于在 Java 应用程序
运行时创建和修改 Java 类，而徐无需编译器的帮助。
>   
> 除了参与 Java 类库一起提供代码生成工具外，`byte-buddy` 允许创建任意类，并不限于实现用于创建运行时代理的接口。  
> 
> 此外，`byte-buddy` 提供了一个方便的 API ，用于 Java Agent 或在构建过程中更改类。

下面笔者默认胖友已经对 `byte-buddy` 有一定的了解。如果胖友暂不了解，建议先阅读如下文章 ：

* [《Java字节码3-使用ByteBuddy实现一个Java-Agent》](http://www.jianshu.com/p/fe1448bf7d31)
* [《Easily Create Java Agents with Byte Buddy》](https://www.infoq.com/articles/Easily-Create-Java-Agents-with-ByteBuddy)
* [《skywalking源码分析之javaAgent工具ByteBuddy的应用》](http://www.kailing.pub/article/index/arcid/178.html) 搜索 "BYTE BUDDY应用" 部分

> 友情提示 ：建议自己简单使用下 `byte-buddy` ，更容易理解 SkyWalking Agent 。 
>  
> 笔者练手的 `byte-buddy` 项目地址 ：https://github.com/YunaiV/learning/tree/master/bytebuddy

-------

下面，让我们打开 [`SkyWalkingAgent#premain(String, Instrumentation)`](https://github.com/YunaiV/skywalking/blob/b68162306b4db7adfd4a2c2891a205b7085f38f0/apm-sniffer/apm-agent/src/main/java/org/skywalking/apm/agent/SkyWalkingAgent.java#L79) 方法，从【第 79 行】代码开始看 ：

* 第 79 至 104 行 ：创建 [`net.bytebuddy.agent.builder.AgentBuilder`](https://github.com/raphw/byte-buddy/blob/188366ace6e16ec167a00b144c9048d78495165f/byte-buddy-dep/src/main/java/net/bytebuddy/agent/builder/AgentBuilder.java) 对象，并设置相关属性。
    * AgentBuilder ，提供便利的 API ，创建 Java Agent 。
    * 第 79 行 ：调用 `AgentBuilder#type(ElementMatcher)` 方法，实现 [`net.bytebuddy.matcher.ElementMatcher`](https://github.com/raphw/byte-buddy/blob/188366ace6e16ec167a00b144c9048d78495165f/byte-buddy-dep/src/main/java/net/bytebuddy/matcher/ElementMatcher.java#L13) 接口，设置需要拦截的类。`PluginFinder#buildMatch()` 方法，在本文 [「3.3 PluginFinder」](#) 详细解析。
    * 第 79 至 104 行 ：调用 `AgentBuilder#transform(Transformer)` 方法，设置 Java 类的修改逻辑。
        * 第 84 行 ：调用 `PluginFinder#find(TypeDescription, ClassLoader)` 方法，获得**匹配**的 AbstractClassEnhancePluginDefine 数组。因为在【**第 79 行**】的代码，设置了**所有**插件需要拦截的类，所以此处需要匹配**该类对应**的 AbstractClassEnhancePluginDefine 数组。`PluginFinder#find(TypeDescription, ClassLoader)` 方法，在本文 [「3.3 PluginFinder」](#) 详细解析。
        * 第 85 行 ：判断匹配的 AbstractClassEnhancePluginDefine 数组大于零。从目前的代码看下来，此处属于**防御性编程**，在【**第 79 行**】的代码保证一定能匹配到 AbstractClassEnhancePluginDefine 。
        * 第 86 至 100 行 ：循环匹配到 AbstractClassEnhancePluginDefine 数组，调用 `AbstractClassEnhancePluginDefine#define(...)` 方法，设置 [`net.bytebuddy.dynamic.DynamicType.Builder`](https://github.com/raphw/byte-buddy/blob/188366ace6e16ec167a00b144c9048d78495165f/byte-buddy-dep/src/main/java/net/bytebuddy/dynamic/DynamicType.java) 对象。通过该对象，定义**如何拦截**需要修改的 Java 类。在 `AbstractClassEnhancePluginDefine#define(...)` 方法的内部，会调用 [`net.bytebuddy.dynamic.DynamicType.ImplementationDefinition#intercept(Implementation)`](https://github.com/raphw/byte-buddy/blob/188366ace6e16ec167a00b144c9048d78495165f/byte-buddy-dep/src/main/java/net/bytebuddy/dynamic/DynamicType.java#L1512) 方法，本文 [「4. 插件的拦截」](#) 也会详细解析。
        * 第 91 行 ：为什么会出现返回为**空**的情况呢？同一个框架在不同的**大**版本，使用的方式相同，但是实现的代码却不尽相同。举个例子，SpringMVC 3 和 SpringMVC 4 ，**都**有 `@RequestMapping` 注解定义 URL ，所以【**第 84 行**】会匹配到 `AbstractSpring3Instrumentation` / `AbstractSpring4Instrumentation` **两个**。当应用使用的是 Spring MVC 4 时，调用 `AbstractSpring3Instrumentation#define(...)` 方法会返回空，而调用 `AbstractSpring4Instrumentation#define(...)` 方法会有返回值。这是如何实现的呢？本文 [「4. 插件的拦截」](#) 也会详细解析。
* 第 105 至 134 行 ：调用 [`AgentBuilder#with(Listener)`](https://github.com/raphw/byte-buddy/blob/188366ace6e16ec167a00b144c9048d78495165f/byte-buddy-dep/src/main/java/net/bytebuddy/agent/builder/AgentBuilder.java#L114) 方法，添加监听器。
    * `#onTransformation(...)` 方法，当 Java 类的修改**成功**，进行调用。
    * `#onError(...)` 方法，当 Java 类的修改**失败**，进行调用。InstrumentDebuggingClass 在本文 [「3.1 InstrumentDebuggingClass」](#) 详细解析。
* 第 135 行 ：调用 [`AgentBuilder#installOn(Instrumentation)`](https://github.com/raphw/byte-buddy/blob/master/byte-buddy-dep/src/main/java/net/bytebuddy/agent/builder/AgentBuilder.java#L620) 方法，根据**上面** AgentBuilder 设置的属性，创建 [`net.bytebuddy.agent.builder.ResettableClassFileTransformer`](https://github.com/raphw/byte-buddy/blob/188366ace6e16ec167a00b144c9048d78495165f/byte-buddy-dep/src/main/java/net/bytebuddy/agent/builder/ResettableClassFileTransformer.java) 对象，配置到 Instrumentation 对象上。在 `AgentBuilder#installOn(Instrumentation)` 方法的内部，会调用 `Instrumentation#addTransformer(ClassFileTransformer)` 方法。

-------

😈 这个方法信息量比较大，笔者对 `byte-buddy` 不是很熟悉，花费了较多时间梳理与理解。建议，如果胖友此处不是理解的很清晰，可以阅读完全文，在回过头再捋一捋这块的代码实现。

## 3.1 InstrumentDebuggingClass

`org.skywalking.apm.agent.InstrumentDebuggingClass` ，Instrument 调试类，用于将被 JavaAgent 修改的**所有**类存储到 `${JAVA_AGENT_PACKAGE}/debugger` 目录下。需要配置 `agent.is_open_debugging_class = true` ，效果如下图 ：[](http://www.iocoder.cn/images/SkyWalking/2020_07_10/04.png)

代码比较简单，胖友点击 [InstrumentDebuggingClass](https://github.com/YunaiV/skywalking/blob/b68162306b4db7adfd4a2c2891a205b7085f38f0/apm-sniffer/apm-agent/src/main/java/org/skywalking/apm/agent/InstrumentDebuggingClass.java) 理解。

## 3.2 ClassMatch

在分享本节相关内容之前，我们先来看下 `bytebuddy` 的 [`net.bytebuddy.matcher`](https://github.com/raphw/byte-buddy/tree/188366ace6e16ec167a00b144c9048d78495165f/byte-buddy-dep/src/main/java/net/bytebuddy/matcher) 模块。该模块提供了各种灵活的匹配方法。那么 SkyWalking 为什么实现自己的 [`org.skywalking.apm.agent.core.plugin.match`](https://github.com/YunaiV/skywalking/tree/43241fff19e17f19b918c96ffd588787f8f05519/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/match) 模块？笔者认为，仅定位于**类级别的匹配**，更常用而又精简的 API 。

-------

`org.skywalking.apm.agent.core.plugin.match.ClassMatch` ，类匹配**接口**。目前子类如下 ：

[](http://www.iocoder.cn/images/SkyWalking/2020_07_10/05.png)

* [NameMatch](https://github.com/YunaiV/skywalking/blob/b68162306b4db7adfd4a2c2891a205b7085f38f0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/match/NameMatch.java#L28) ：基于**完整的类名**进行匹配，例如：`"com.alibaba.dubbo.monitor.support.MonitorFilter"` 。
* [IndirectMatch](https://github.com/YunaiV/skywalking/blob/b68162306b4db7adfd4a2c2891a205b7085f38f0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/match/IndirectMatch.java) ：**间接**匹配**接口**。相比 NameMatch 来说，确实比较 "委婉" 🙂 。
    * [ClassAnnotationMatch](https://github.com/YunaiV/skywalking/blob/b68162306b4db7adfd4a2c2891a205b7085f38f0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/match/ClassAnnotationMatch.java) ：基于**类注解**进行匹配，可设置**同时**匹配多个。例如：`"@RequestMapping"`。
    * [HierarchyMatch](https://github.com/YunaiV/skywalking/blob/b68162306b4db7adfd4a2c2891a205b7085f38f0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/match/HierarchyMatch.java) ：基于**父类 / 接口**进行匹配，可设置**同时**匹配多个。
    * [MethodAnnotationMatch](https://github.com/YunaiV/skywalking/blob/b68162306b4db7adfd4a2c2891a205b7085f38f0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/match/MethodAnnotationMatch.java) ：基于**方法注解**进行匹配，可设置**同时**匹配多个。目前项目里主要用于匹配方法上的  [`org.skywalking.apm.toolkit.trace.@Trace`](https://github.com/OpenSkywalking/skywalking/blob/8d9820322bdfc956d9d4f0d04f55ce985926cfae/apm-application-toolkit/apm-toolkit-trace/src/main/java/org/apache/skywalking/apm/toolkit/trace/Trace.java) 注解。

每个类已经添加详细的代码注释，胖友喜欢哪个点哪个哟。

## 3.3 PluginFinder

`org.skywalking.apm.agent.core.plugin.PluginFinder` ，插件发现者。其提供 [`#find(...)`](https://github.com/YunaiV/skywalking/blob/09c654af33081e56547cb8b3b9e0c8525ddce32f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginFinder.java#L80) 方法，获得**类增强插件定义**( [`org.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine`](https://github.com/OpenSkywalking/skywalking/blob/b16d23c1484bec941367d6b36fa932b8ace40971/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/AbstractClassEnhancePluginDefine.java) )对象。

PluginFinder **[构造方法](https://github.com/YunaiV/skywalking/blob/09c654af33081e56547cb8b3b9e0c8525ddce32f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginFinder.java#L56)**，代码如下 ：

* 第 57 至 77 行 ：循环 AbstractClassEnhancePluginDefine 对象数组，添加到 `nameMatchDefine` / `signatureMatchDefine` 属性，方便 `#find(...)` 方法查找 AbstractClassEnhancePluginDefine 对象。
    * 第 65 至 72 行 ：处理 NameMatch 为匹配的 AbstractClassEnhancePluginDefine 对象，添加到 `nameMatchDefine`  属性。
    * 第 74 至 76 行 ：处理**非** NameMatch 为匹配的 AbstractClassEnhancePluginDefine 对象，添加到 `signatureMatchDefine` 属性。

-------

[`#find(...)`](https://github.com/YunaiV/skywalking/blob/b68162306b4db7adfd4a2c2891a205b7085f38f0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginFinder.java#L89) 方法，获得**类增强插件定义**( [`org.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine`](https://github.com/OpenSkywalking/skywalking/blob/b16d23c1484bec941367d6b36fa932b8ace40971/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/AbstractClassEnhancePluginDefine.java) )对象，代码如下 ：

* 第 92 至 96 行 ：以 `nameMatchDefine` 属性来匹配 AbstractClassEnhancePluginDefine 对象。
* 第 98 至 104 行 ：以 `signatureMatchDefine` 属性来匹配 AbstractClassEnhancePluginDefine 对象。在这个过程中，会调用 `IndirectMatch#isMatch(TypeDescription)` 方法，进行匹配。

-------

[`#buildMatch()`](https://github.com/YunaiV/skywalking/blob/b68162306b4db7adfd4a2c2891a205b7085f38f0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/PluginFinder.java#L116) 方法，获得全部插件的类匹配，多个插件的类匹配条件以 `or` 分隔，代码如下 ：

* 第 117 至 123 行 ：以 `nameMatchDefine` 属性来匹配。
* 第 124 至 132 行 ：以 `signatureMatchDefine` 属性来匹配。
* **实际上，该方法和 `#find(...)` 方法逻辑是一致的**。

# 4. 插件的拦截

在上文中，我们已经提到，SkyWalking 通过 JavaAgent 机制，对需要拦截的类的方法，使用 `byte-buddy` **动态**修改 Java 类的二进制，从而进行方法切面拦截，记录调用链路。

看具体的代码实现之前，想一下**拦截**会涉及到哪些元素 ：

* 拦截切面 InterceptPoint
* 拦截器 Interceptor
* 拦截类的定义 Define ：一个类有哪些拦截切面及对应的拦截器

下面，我们来看看本小节会涉及到的类。如图所示：

[](http://www.iocoder.cn/images/SkyWalking/2020_07_10/06.png)

看起来类比想象的多？梳理之，结果如图 ：

[](http://www.iocoder.cn/images/SkyWalking/2020_07_10/07.png)

* 根据方法类型的不同，使用不同 ClassEnhancePluginDefine 的实现类。其中，构造方法和静态方法使用相同的实现类。
* 相比上面提到**拦截**会涉及到的元素，多了一个 **Inter** ？如下是官方的说明 ：

    > In this class, it provide a bridge between `byte-buddy` and `sky-walking` plugin.

## 4.1 ClassEnhancePluginDefine 

整体类图如下：

[](http://www.iocoder.cn/images/SkyWalking/2020_07_10/08.png)

* AbstractClassEnhancePluginDefine ：SkyWalking 类增强插件定义**抽象基类**。
* ClassEnhancePluginDefine ：SkyWalking 类增强插件定义**抽象类**。
* 从 UML 图中的方法，我们可以看出，AbstractClassEnhancePluginDefine 注重在**定义**( Define )，ClassEnhancePluginDefine 注重在**增强**( Enhance )。

整体流程如下 ：

[](http://www.iocoder.cn/images/SkyWalking/2020_07_10/09.png)

OK ，下面我们开始看看代码是如何实现的。

### 4.1.1 AbstractClassEnhancePluginDefine

`org.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine` ，SkyWalking 类增强插件定义**抽象基类**。它注重在**定义**( Define )的抽象与实现。

[`#enhanceClass()`](https://github.com/YunaiV/skywalking/blob/30683167c79c71ad088666c587a05c6d9f0daf3f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/AbstractClassEnhancePluginDefine.java#L93) **抽象**方法，定义了类匹配( ClassMatch ) 。

[`#witnessClasses()`](https://github.com/YunaiV/skywalking/blob/30683167c79c71ad088666c587a05c6d9f0daf3f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/AbstractClassEnhancePluginDefine.java#L105) 方法，见证类列表。当且仅当应用存在见证类列表，插件才生效。**什么意思**？让我们看看这种情况：一个**类库**存在两个发布的版本( 如 `1.0` 和 `2.0` )，其中包括**相同**的目标类，但不同的方法或不同的方法参数列表。所以我们需要根据库的不同版本使用插件的不同版本。然而版本显然不是一个选项，这时需要使用见证类列表，判断出当前引用类库的发布版本。

* 举个实际的例子，SpringMVC 3 和 SpringMVC 4 ，**都**有 `@RequestMapping` 注解定义 URL 。
    * 通过判断存在 `org.springframework.web.servlet.view.xslt.AbstractXsltView` 类，应用使用 SpringMVC 3 ，使用 `apm-springmvc-annotation-3.x-plugin.jar` 。
    * 通过判断存在 `org.springframework.web.servlet.tags.ArgumentTag` 类，应用使用 SpringMVC 4 ，使用 `apm-springmvc-annotation-4.x-plugin.jar` 。
* **另外**，该方法返回**空数组**。即默认情况，插件生效，无需见证类列表。

-------

[`#define(...)`](https://github.com/YunaiV/skywalking/blob/30683167c79c71ad088666c587a05c6d9f0daf3f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/AbstractClassEnhancePluginDefine.java#L46) 方法，设置 [`net.bytebuddy.dynamic.DynamicType.Builder`](https://github.com/raphw/byte-buddy/blob/188366ace6e16ec167a00b144c9048d78495165f/byte-buddy-dep/src/main/java/net/bytebuddy/dynamic/DynamicType.java) 对象。通过该对象，定义**如何拦截**需要修改的目标 Java 类(方法的 `transformClassName` 参数)。代码如下 ：

* 第 57 至 70 行 ：判断见证类列表是否都存在。若不存在，则插件不生效。
    * [`org.skywalking.apm.agent.core.plugin.WitnessClassFinder`](https://github.com/YunaiV/skywalking/blob/30683167c79c71ad088666c587a05c6d9f0daf3f/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/plugin/WitnessClassFinder.java) ，已经添加完整注释，胖友点击查看。
* 第 72 至 76 行 ：调用 `#enhance(...)` **抽象**方法，使用拦截器增强目标类。

### 4.1.2 ClassEnhancePluginDefine



### 4.1.3

## 4.2 InterceptPoint

| InterceptPoint | 方法类型 | 方法匹配 | 拦截器 | `#isOverrideArgs()` |
| --- | --- | --- | --- | --- |
| StaticMethodsInterceptPoint | 静态方法 | `#getMethodsMatcher()` | `#getMethodsInterceptor()` | 有 |
| ConstructorInterceptPoint | 构造方法 | `#getConstructorMatcher()` |   `#getConstructorInterceptor()` | 无 |
| InstanceMethodsInterceptPoint | 实例方法 | `#getMethodsMatcher()` |   `#getMethodsInterceptor()` | 有 |

XXXInterceptPoint **接口**，对应一个 `net.bytebuddy.matcher.ElementMatcher`  和一个拦截器。

代码比较简单，胖友自己查看。


