# 1. 概述

本文接 [《精尽 Dubbo 源码分析 —— NIO 服务器（四）之 Exchange 层》](http://www.iocoder.cn/Dubbo/remoting-api-exchange//?self) 一文，分享 `dubbo-remoting-api` 模块， `buffer` 包，**Buffer 层**。

Buffer 在 NIO 框架中，扮演非常重要的角色，基本每个库都提供了自己的 Buffer 实现，例如：

* Java NIO 的 `java.nio.ByteBuffer`
* Mina 的 `org.apache.mina.core.buffer.IoBuffer`
* Netty4 的 `io.netty.buffer.ByteBuf`

在 `dubbo-remoting-api` 的 `buffer` 包中，一方面定义了 ChannelBuffer 和 ChannelBufferFactory 的接口，同时提供了多种默认的实现。整体类图如下：

[类图](http://www.iocoder.cn/images/Dubbo/2018_12_13/01.png)

* 其中，红框部分，是 Netty3 和 Netty4 ，实现的自定义的 ChannelBuffer 和 ChannelBufferFactory 类。

# 2. ChannelBuffer

[`com.alibaba.dubbo.remoting.buffer.ChannelBuffer`](https://github.com/YunaiV/dubbo/blob/master/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/buffer/ChannelBuffer.java) ，实现 Comparable 接口，**通道 Buffer** 接口。

ChannelBuffer 在接口方法的定义上，主要参考了 Netty 的 ByteBuf  进行设计，所以接口和注释基本一致，本文就不一个一个细讲过去，胖友可以看：

* 英文：[《Netty4.1 ByteBuf API》](https://netty.io/4.1/api/io/netty/buffer/ByteBuf.html)
* 中文：[《深入研究Netty框架之ByteBuf功能原理及源码分析》](https://my.oschina.net/7001/blog/742236)

独有的接口方法 `#factory()` 方法，用于逻辑中，需要创建 ChannelBuffer 的情况。

* 代码如下：

    ```Java
    /**
     * Returns the factory which creates a {@link ChannelBuffer} whose type and
     * default {@link java.nio.ByteOrder} are same with this buffer.
     */
    ChannelBufferFactory factory()
    ```

* 调用方如下：[调用方](http://www.iocoder.cn/images/Dubbo/2018_12_13/02.png)

## 2.1 AbstractChannelBuffer

[`com.alibaba.dubbo.remoting.buffer.AbstractChannelBuffer`](https://github.com/YunaiV/dubbo/blob/master/dubbo-remoting/dubbo-remoting-api/src/main/java/com/alibaba/dubbo/remoting/buffer/AbstractChannelBuffer.java) ，实现 ChannelBuffer 接口，通道 Buffer **抽象类**。

**构造方法**

```Java
/**
 * 读取位置
 */
private int readerIndex;
/**
 * 写入位置
 */
private int writerIndex;
/**
 * 标记的读取位置
 */
private int markedReaderIndex;
/**
 * 标记的写入位置
 */
private int markedWriterIndex;
```

**实现方法**

在 AbstractChannelBuffer 实现的方法，都是**重载**的方法，真正**实质**的方法，需要子类来实现。以 `#getBytes(...)` 方法，举例子：

```Java
@Override
public void getBytes(int index, ChannelBuffer dst, int length) {
    if (length > dst.writableBytes()) {
        throw new IndexOutOfBoundsException();
    }
    getBytes(index, dst, dst.writerIndex(), length);
    dst.writerIndex(dst.writerIndex() + length);
}
```

* 方法中调用的 `#getBytes(index, ds, dstIndex, length)` 方法，并未实现。🙂 **为啥呢**？**实质**的方法，涉及到字节数组的**实现形式**。

如下是所有**未实现**的方法：[未实现方法](http://www.iocoder.cn/images/Dubbo/2018_12_13/03.png)


