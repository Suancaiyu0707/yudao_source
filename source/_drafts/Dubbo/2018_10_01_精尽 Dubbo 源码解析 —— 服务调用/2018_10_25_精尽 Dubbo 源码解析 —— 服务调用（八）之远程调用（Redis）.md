title: 精尽 Dubbo 源码分析 —— 服务调用（八）之远程调用（Redis）
date: 2018-10-25
tags:
categories: Dubbo
permalink: Dubbo/rpc-redis

-------

摘要: 原创出处 http://www.iocoder.cn/Dubbo/rpc-redis/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Dubbo/rpc-redis/)
- [2. RedisProtocol](http://www.iocoder.cn/Dubbo/rpc-redis/)
  - [2.1 export](http://www.iocoder.cn/Dubbo/rpc-redis/)
  - [2.2 refer](http://www.iocoder.cn/Dubbo/rpc-redis/)
- [666. 彩蛋](http://www.iocoder.cn/Dubbo/rpc-redis/)

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

本文，我们分享 `redis://` 协议的远程调用，主要分成**两个个部分**：

* ~~服务暴露~~
* 服务引用
* 服务调用

对应项目为 `dubbo-rpc-redis` 。

对应文档为 [《Dubbo 用户指南 —— redis://》](https://dubbo.gitbooks.io/dubbo-user-book/references/protocol/redis.html) 。定义如下：

> 基于 Redis 实现的 RPC 协议。

简单的说，通过 Dubbo Service 的调用方式，**透明化**对 Redis 的访问。  
这样，如果未来希望，修改缓存的解决方案，不用修改代码，而只要修改 Dubbo Service 的配置。  
就好像，Java JDBC API 有 MySQL JDBC 、Oracle JDBC 等多种实现，只需要修改对应的 JDBC 驱动实现类，就可以连接上不同的数据库。

另外，Dubbo 提供 `memcached://` 协议，和 `redis://` 对等，差别点在前者使用 Memcached ，后者使用 Redis 。

# 2. RedisProtocol

[`com.alibaba.dubbo.rpc.protocol.redis.RedisProtocol`](https://github.com/YunaiV/dubbo/blob/master/dubbo-rpc/dubbo-rpc-redis/src/main/java/com/alibaba/dubbo/rpc/protocol/redis/RedisProtocol.java) ，实现 AbstractProtocol 抽象类，`redis://` 协议实现类。

## 2.1 export

```Java
@Override
public <T> Exporter<T> export(final Invoker<T> invoker) throws RpcException {
    throw new UnsupportedOperationException("Unsupported export redis service. url: " + invoker.getUrl());
}
```

实际访问的就是 Redis Server 实例，因此无需进行 Dubbo 服务暴露。客户端配置引用方式如下：

> 在客户端使用，注册中心读取：  
> `<dubbo:reference id="store" interface="java.util.Map" group="member" />`
> 
> 或者，点对点直连：  
> `<dubbo:reference id="store" interface="java.util.Map" url="redis://10.20.153.10:6379"`

## 2.2 refer

```Java
  1: @Override
  2: public <T> Invoker<T> refer(final Class<T> type, final URL url) throws RpcException {
  3:     try {
  4:         // 创建 GenericObjectPoolConfig 对象，设置配置
  5:         GenericObjectPoolConfig config = new GenericObjectPoolConfig();
  6:         config.setTestOnBorrow(url.getParameter("test.on.borrow", true));
  7:         config.setTestOnReturn(url.getParameter("test.on.return", false));
  8:         config.setTestWhileIdle(url.getParameter("test.while.idle", false));
  9:         if (url.getParameter("max.idle", 0) > 0)
 10:             config.setMaxIdle(url.getParameter("max.idle", 0));
 11:         if (url.getParameter("min.idle", 0) > 0)
 12:             config.setMinIdle(url.getParameter("min.idle", 0));
 13:         if (url.getParameter("max.active", 0) > 0)
 14:             config.setMaxTotal(url.getParameter("max.active", 0));
 15:         if (url.getParameter("max.total", 0) > 0)
 16:             config.setMaxTotal(url.getParameter("max.total", 0));
 17:         if (url.getParameter("max.wait", 0) > 0)
 18:             config.setMaxWaitMillis(url.getParameter("max.wait", 0));
 19:         if (url.getParameter("num.tests.per.eviction.run", 0) > 0)
 20:             config.setNumTestsPerEvictionRun(url.getParameter("num.tests.per.eviction.run", 0));
 21:         if (url.getParameter("time.between.eviction.runs.millis", 0) > 0)
 22:             config.setTimeBetweenEvictionRunsMillis(url.getParameter("time.between.eviction.runs.millis", 0));
 23:         if (url.getParameter("min.evictable.idle.time.millis", 0) > 0)
 24:             config.setMinEvictableIdleTimeMillis(url.getParameter("min.evictable.idle.time.millis", 0));
 25:         // 创建 JedisPool 对象
 26:         final JedisPool jedisPool = new JedisPool(config, url.getHost(), url.getPort(DEFAULT_PORT),
 27:                 url.getParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT));
 28: 
 29:         // 处理方法名的映射
 30:         final int expiry = url.getParameter("expiry", 0);
 31:         final String get = url.getParameter("get", "get");
 32:         final String set = url.getParameter("set", Map.class.equals(type) ? "put" : "set");
 33:         final String delete = url.getParameter("delete", Map.class.equals(type) ? "remove" : "delete");
 34: 
 35:         // 创建 Invoker 对象
 36:         return new AbstractInvoker<T>(type, url) {
 37: 
 38:             @Override
 39:             protected Result doInvoke(Invocation invocation) {
 40:                 Jedis resource = null;
 41:                 try {
 42:                     // 获得 Redis Resource
 43:                     resource = jedisPool.getResource();
 44:                     // Redis get 指令
 45:                     if (get.equals(invocation.getMethodName())) {
 46:                         if (invocation.getArguments().length != 1) {
 47:                             throw new IllegalArgumentException("The redis get method arguments mismatch, must only one arguments. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url);
 48:                         }
 49:                         // 获得值
 50:                         byte[] value = resource.get(String.valueOf(invocation.getArguments()[0]).getBytes());
 51:                         if (value == null) {
 52:                             return new RpcResult();
 53:                         }
 54:                         // 反序列化
 55:                         ObjectInput oin = getSerialization(url).deserialize(url, new ByteArrayInputStream(value));
 56:                         // 返回结果
 57:                         return new RpcResult(oin.readObject());
 58:                     // Redis set/put 指令
 59:                     } else if (set.equals(invocation.getMethodName())) {
 60:                         if (invocation.getArguments().length != 2) {
 61:                             throw new IllegalArgumentException("The redis set method arguments mismatch, must be two arguments. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url);
 62:                         }
 63:                         // 序列化
 64:                         byte[] key = String.valueOf(invocation.getArguments()[0]).getBytes();
 65:                         ByteArrayOutputStream output = new ByteArrayOutputStream();
 66:                         ObjectOutput value = getSerialization(url).serialize(url, output);
 67:                         value.writeObject(invocation.getArguments()[1]);
 68:                         // 设置值
 69:                         resource.set(key, output.toByteArray());
 70:                         if (expiry > 1000) {
 71:                             resource.expire(key, expiry / 1000);
 72:                         }
 73:                         // 返回结果
 74:                         return new RpcResult();
 75:                     } else if (delete.equals(invocation.getMethodName())) {
 76:                         if (invocation.getArguments().length != 1) {
 77:                             throw new IllegalArgumentException("The redis delete method arguments mismatch, must only one arguments. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url);
 78:                         }
 79:                         // 删除值
 80:                         resource.del(String.valueOf(invocation.getArguments()[0]).getBytes());
 81:                         // 返回结果
 82:                         return new RpcResult();
 83:                     } else {
 84:                         throw new UnsupportedOperationException("Unsupported method " + invocation.getMethodName() + " in redis service.");
 85:                     }
 86:                 } catch (Throwable t) {
 87:                     RpcException re = new RpcException("Failed to invoke redis service method. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url + ", cause: " + t.getMessage(), t);
 88:                     if (t instanceof TimeoutException || t instanceof SocketTimeoutException) {
 89:                         re.setCode(RpcException.TIMEOUT_EXCEPTION);
 90:                     } else if (t instanceof JedisConnectionException || t instanceof IOException) {
 91:                         re.setCode(RpcException.NETWORK_EXCEPTION);
 92:                     } else if (t instanceof JedisDataException) {
 93:                         re.setCode(RpcException.SERIALIZATION_EXCEPTION);
 94:                     }
 95:                     throw re;
 96:                 } finally {
 97:                     // 归还 Redis Resource
 98:                     if (resource != null) {
 99:                         try {
100:                             jedisPool.returnResource(resource);
101:                         } catch (Throwable t) {
102:                             logger.warn("returnResource error: " + t.getMessage(), t);
103:                         }
104:                     }
105:                 }
106:             }
107: 
108:             @Override
109:             public void destroy() {
110:                 // 标记销毁
111:                 super.destroy();
112:                 // 销毁 Redis Pool
113:                 try {
114:                     jedisPool.destroy();
115:                 } catch (Throwable e) {
116:                     logger.warn(e.getMessage(), e);
117:                 }
118:             }
119: 
120:         };
121:     } catch (Throwable t) {
122:         throw new RpcException("Failed to refer redis service. interface: " + type.getName() + ", url: " + url + ", cause: " + t.getMessage(), t);
123:     }
124: }
```

* 使用 Jedis 访问 Redis Server 。
* 第 4 至 24 行：创建 GenericObjectPoolConfig 对象，从 Dubbo URL 中读取相关配置。此时，我们可以看到，为什么 Dubbo 的配置类中，有 `arguments` 属性了。可以使用它，实现不同 Protocol 协议的自定义属性。
* 第 25 至 27 行：创建 JedisPool 对象。
* 第 29 至 33 行：处理方法名的映射。

    > 如果方法名和 redis 的标准方法名不相同，则需要配置映射关系：  
    > `<dubbo:reference id="cache" interface="com.foo.CacheService" url="memcached://10.20.153.10:11211" p:set="putFoo" p:get="getFoo" p:delete="removeFoo" />`
    
    * 当对应的服务接口是 `java.util.Map` 时，对应的 Redis 数据结构为 **Map** 。
    
* 第 35 至 120 行：创建 Invoker 对象。

### 2.2.1 doInvoke

* 第 43 行：获得 Redis Resource 对象。
* 第 44 至 57 行：Redis **get** 指令。
* 第 58 至 74 行：Redis **set/put** 指令。
* 第 75 至 83 行：Redis **delete/remove** 指令。
* 第 84 至 85 行：目前其他命令，暂时不支持。
* 第 86 至 95 行：翻译异常成 Dubbo 错误码。
* 第 97 至 105 行：归还 Redis Resource 对象。

### 2.2.2 destroy

* 第 111 行：调用 `super#destroy()` 方法，标记销毁。
* 第 112 至 117 行：调用 `JedisPool#destroy()` 方法，销毁 Redis Pool 。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

没有彩蛋~


