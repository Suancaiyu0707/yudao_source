# 1. 概述

本文主要分享 **SkyWalking Agent 插件体系**。主要涉及三个流程 ：

* 插件的加载
* 插件的匹配
* 插件的应用

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

TODO

## 2.5 小结

胖友，回过头，在看一下流程图，理解理解。

# 3. 

