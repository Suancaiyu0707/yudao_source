title: 精尽 Dubbo 源码分析 —— 服务调用（九）之远程调用（Memcached）
date: 2018-10-28
tags:
categories: Dubbo
permalink: Dubbo/rpc-memcached

-------

摘要: 原创出处 http://www.iocoder.cn/Dubbo/rpc-memcached/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Dubbo/rpc-memcached/)
- [2. MemcachedProtocol](http://www.iocoder.cn/Dubbo/rpc-memcached/)
  - [2.1 export](http://www.iocoder.cn/Dubbo/rpc-memcached/)
  - [2.2 refer](http://www.iocoder.cn/Dubbo/rpc-memcached/)
- [666. 彩蛋](http://www.iocoder.cn/Dubbo/rpc-memcached/)

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

本文接 [《精尽 Dubbo 源码分析 —— 服务调用（八）之远程调用（Redis）》](http://www.iocoder.cn/Dubbo/rpc-redis/?self) ，我们分享 `memcached://` 协议的远程调用，主要分成**两个个部分**：

* ~~服务暴露~~
* 服务引用
* 服务调用

对应项目为 `dubbo-rpc-memcached` 。

对应文档为 [《Dubbo 用户指南 —— memcached://》](https://dubbo.gitbooks.io/dubbo-user-book/references/protocol/memcached.html) 。定义如下：

> 基于 Memcached 实现的 RPC 协议。
 
# 2. MemcachedProtocol

[`com.alibaba.dubbo.rpc.protocol.memcached.MemcachedProtocol`](https://github.com/YunaiV/dubbo/blob/master/dubbo-rpc/dubbo-rpc-memcached/src/main/java/com/alibaba/dubbo/rpc/protocol/memcached/MemcachedProtocol.java) ，实现 AbstractProtocol 抽象类，`memcached://` 协议实现类。

## 2.1 export

```Java
@Override
public <T> Exporter<T> export(final Invoker<T> invoker) throws RpcException {
    throw new UnsupportedOperationException("Unsupported export redis service. url: " + invoker.getUrl());
}
```

实际访问的就是 Memcached Server 实例，因此无需进行 Dubbo 服务暴露。客户端配置引用方式如下：

> 在客户端使用，注册中心读取：  
> `<dubbo:reference id="store" interface="java.util.Map" group="member" />`
> 
> 或者，点对点直连：  
> `<dubbo:reference id="store" interface="java.util.Map" url="memcached://10.20.153.10:11211"`

## 2.2 refer

```Java
  1: @Override
  2: public <T> Invoker<T> refer(final Class<T> type, final URL url) throws RpcException {
  3:     try {
  4:         // 创建 MemcachedClient 对象
  5:         String address = url.getAddress();
  6:         String backup = url.getParameter(Constants.BACKUP_KEY);
  7:         if (backup != null && backup.length() > 0) {
  8:             address += "," + backup;
  9:         }
 10:         MemcachedClientBuilder builder = new XMemcachedClientBuilder(AddrUtil.getAddresses(address));
 11:         final MemcachedClient memcachedClient = builder.build();
 12: 
 13:         // 处理方法名的映射
 14:         final int expiry = url.getParameter("expiry", 0);
 15:         final String get = url.getParameter("get", "get");
 16:         final String set = url.getParameter("set", Map.class.equals(type) ? "put" : "set");
 17:         final String delete = url.getParameter("delete", Map.class.equals(type) ? "remove" : "delete");
 18:         return new AbstractInvoker<T>(type, url) {
 19: 
 20:             @Override
 21:             protected Result doInvoke(Invocation invocation) throws Throwable {
 22:                 try {
 23:                     // Memcached get 指令
 24:                     if (get.equals(invocation.getMethodName())) {
 25:                         if (invocation.getArguments().length != 1) {
 26:                             throw new IllegalArgumentException("The memcached get method arguments mismatch, must only one arguments. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url);
 27:                         }
 28:                         return new RpcResult(memcachedClient.get(String.valueOf(invocation.getArguments()[0])));
 29:                     // Memcached set 指令
 30:                     } else if (set.equals(invocation.getMethodName())) {
 31:                         if (invocation.getArguments().length != 2) {
 32:                             throw new IllegalArgumentException("The memcached set method arguments mismatch, must be two arguments. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url);
 33:                         }
 34:                         memcachedClient.set(String.valueOf(invocation.getArguments()[0]), expiry, invocation.getArguments()[1]);
 35:                         return new RpcResult();
 36:                     // Memcached delele 指令
 37:                     } else if (delete.equals(invocation.getMethodName())) {
 38:                         if (invocation.getArguments().length != 1) {
 39:                             throw new IllegalArgumentException("The memcached delete method arguments mismatch, must only one arguments. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url);
 40:                         }
 41:                         memcachedClient.delete(String.valueOf(invocation.getArguments()[0]));
 42:                         return new RpcResult();
 43:                     // 不支持的指令，抛出异常
 44:                     } else {
 45:                         throw new UnsupportedOperationException("Unsupported method " + invocation.getMethodName() + " in memcached service.");
 46:                     }
 47:                 } catch (Throwable t) {
 48:                     RpcException re = new RpcException("Failed to invoke memcached service method. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url + ", cause: " + t.getMessage(), t);
 49:                     if (t instanceof TimeoutException || t instanceof SocketTimeoutException) {
 50:                         re.setCode(RpcException.TIMEOUT_EXCEPTION);
 51:                     } else if (t instanceof MemcachedException || t instanceof IOException) {
 52:                         re.setCode(RpcException.NETWORK_EXCEPTION);
 53:                     }
 54:                     throw re;
 55:                 }
 56:             }
 57: 
 58:             @Override
 59:             public void destroy() {
 60:                 // 标记销毁
 61:                 super.destroy();
 62:                 // 关闭 MemcachedClient
 63:                 try {
 64:                     memcachedClient.shutdown();
 65:                 } catch (Throwable e) {
 66:                     logger.warn(e.getMessage(), e);
 67:                 }
 68:             }
 69: 
 70:         };
 71:     } catch (Throwable t) {
 72:         throw new RpcException("Failed to refer memcached service. interface: " + type.getName() + ", url: " + url + ", cause: " + t.getMessage(), t);
 73:     }
 74: }
```

* 第 4 至 11 行：创建 MemcachedClient 对象。
* 第 13 至 17 行：处理方法名的映射。此处有个问题，Memcached 不存在 Map 数据结构，因此不存在 put 和 remove 指令。
* 第 18 至 73 行：创建 Invoker 对象。

### 2.2.1 doInvoke

* 第 23 至 28 行：Memcached **get** 指令。
* 第 29 至 35 行：Memcached **set** 指令。
* 第 36 至 42 行：Memcached **delete** 指令。
* 第 43 至 46 行：目前其他命令，暂时不支持。
* 第 47 至 55 行：翻译异常成 Dubbo 错误码。

### 2.2.2 destroy

* 第 61 行：调用 `super#destroy()` 方法，标记销毁。
* 第 63 至 67 行：调用 `Memcached#shutdown()` 方法，关闭 MemcachedClient 。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

没有彩蛋~



