[TOC]


## 问题背景
netty框架是基于异步事件驱动的网络通信框架，具备高性能、高并发支持、低资源占用等优点。netty基于JDK的NIO提供了相对简便的编程，是被使用非常广泛的NIO框架。

我们产品曾使用netty框架开发了文件服务，早期运行比较稳定，从某个版本开始，发现服务有时会报出堆外内存溢出问题，内容比如：
`io.netty.util.internal.OutOfDirectMemoryError: failed to allocate 16777216 byte(s) of direct memory (used: 6425673735, max: 6442450944)`
可以看出当需要分配内存块时发现超出了最大堆外内存限制，报出了`OutOfDirectMemoryError`错误。

## 初步分析
### 找相似问题
netty的`OutOfDirectMemoryError`似乎是使用者较常见的问题，由于NIO编程的复杂性，和堆外内存需手动释放的特点，可以搜到有不少相关问题排错的内容，报错是清一色的`OutOfDirectMemoryError`，原因则各不相同，相同的也就是一定是某处用了堆外内存而未释放，累积起来最终造成溢出。

### 框架升级尝试
netty自身框架更新版本比较多，一些版本的确存在相关问题，并时有更新解决，尝试过几个版本：`4.1.17、4.1.47、4.1.56、4.1.63`等，并不能解决本的问题，目前使用`4.1.56`版相对稳定。

### 堆栈和内存关系分析
报错堆栈中可见服务自身一个处理数据的方法
```
io.netty.util.internal.OutOfDirectMemoryError: failed to allocate 16777216 byte(s) of direct memory (used: 6425673735, max: 6442450944)
	at io.netty.util.internal.PlatformDependent.incrementMemoryCounter(PlatformDependent.java:775) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.util.internal.PlatformDependent.allocateDirectNoCleaner(PlatformDependent.java:730) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.buffer.PoolArena$DirectArena.allocateDirect(PoolArena.java:645) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.buffer.PoolArena$DirectArena.newUnpooledChunk(PoolArena.java:635) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.buffer.PoolArena.allocateHuge(PoolArena.java:215) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.buffer.PoolArena.allocate(PoolArena.java:143) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.buffer.PoolArena.reallocate(PoolArena.java:288) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.buffer.PooledByteBuf.capacity(PooledByteBuf.java:118) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.buffer.AbstractByteBuf.ensureWritable0(AbstractByteBuf.java:307) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.buffer.AbstractByteBuf.ensureWritable(AbstractByteBuf.java:282) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.buffer.AbstractByteBuf.writeBytes(AbstractByteBuf.java:1105) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.buffer.AbstractByteBuf.writeBytes(AbstractByteBuf.java:1098) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.buffer.AbstractByteBuf.writeBytes(AbstractByteBuf.java:1089) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.handler.codec.http.multipart.HttpPostMultipartRequestDecoder.offer(HttpPostMultipartRequestDecoder.java:351) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.handler.codec.http.multipart.HttpPostMultipartRequestDecoder.offer(HttpPostMultipartRequestDecoder.java:52) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.handler.codec.http.multipart.HttpPostRequestDecoder.offer(HttpPostRequestDecoder.java:223) ~[netty-all-4.1.56.Final.jar:4.1.56.Final]
	at com.thunisoft.fileserver.network.HttpFileServerHandler.processData(HttpFileServerHandler.java:1262) ~[cloud-4.jar:1.7.5]
	at com.thunisoft.fileserver.network.HttpFileServerHandler.channelRead(HttpFileServerHandler.java:483) ~[cloud-4.jar:1.7.5]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:379) [netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:365) [netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:357) [netty-all-4.1.56.Final.jar:4.1.56.Final]
	at io.netty.handler.codec.http.cors.CorsHandler.channelRead(CorsHandler.java:95) [netty-all-4.1.56.Final.jar:4.1.56.Final]
	......
```
以上涉及程序业务代码是在方法`processData`，这是处理数据解析和存储的方法，此单元使用到了`HttpPostRequestDecoder`并且有释放方法，虽然怀疑过这个对象是否可正确释放但不容易看出问题和验证。另外考虑堆外内存可能和堆内的对象存在关系，最初尝试过通过dump分析找找蛛丝马迹。而文件服务本身的特点，也给问题增加了一些复杂性，文件服务是通用的存取文件操作，内存中很多内容都是无意义文件数据，由于每次传输文件大小差异巨大从几K到几G，分析相关对象个数和内存也不易看出问题，堆dump分析过程未留，此处略过。

## 基于netty原理的深入分析

综上分析试错过程，对于netty的此类问题，还是得从其自身机制并结合程序代码来分析。

### Netty线程模型
netty的线程模型基于主从多reactor模型，引用一个图如下：

![](https://bed.cdpt.pro/ibed/2021/07/10/Cej4JNR9k.png)



* 服务启动先创建`bossGroup`和`workerGroup`两个`NioEventLoopGroup`线程池，用于接收和处理请求
* 构建一个`ServerBootstrap`来初始化启动内容，包括了绑定端口，注册到以上2个`NioEventLoopGroup`线程池，并注册自己的`NioServerSocketChannel`来处理tcp通道请求
* boss和worker两个线程池类型是`NioEventLoopGroup`，而`NioEventLoopGroup`管理一组`NioEventLoop`
* `NioEventLoop`封装了`Selector`，可处理各种IO操作如`accept、connect、read、write`等，处理请求数据的`channel`会注册到`NioEventLoop`上来接收处理相关事件
* `Channel`是通信的主体，用于做各种传输数据的读写处理，如`FileChannel、SocketChannel、ServerSocketChannel`等，程序需注册自己的`ChannelHandler`来处理自己的数据。

### ChannelHandler的注册

文件服务注册了处理HTTP和HTTPS请求的Handler，继承`ChannelDuplexHandler`来完成通过HTTP和HTTPS文件收发的功能。

Channel基本的生命周期：

* channelUnregistered，未注册到Eventloop，不会处理IO请求
* channelRegistered，注册到EventLoop，可以处理IO请求
* channelActive，变为活跃状态，连接到远程主机后触发，可以收发数据了
* channelInactive，处理非活跃状态，和远端断开连接时触发

ChannelHandler基本的生命周期和IO事件：
* handlerAdded，当 ChannelHandler 添加到 ChannelPipeline 调用
* handlerRemoved，当 ChannelHandler 从 ChannelPipeline 移除时调用
* exceptionCaught，当 ChannelPipeline 执行抛出异常时调用
* channelRead，读取和处理传入的数据
* channelReadComplete，读取传入数据完成（实际仅限于一次socket传输完成）

文件服务的文件传输处理Handler为`HttpFileServerHandler`，主要有如下逻辑：
```java
public class HttpFileServerHandler extends ChannelDuplexHandler {
    private HttpPostRequestDecoder decoder;
    // ...
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg)
        throws Exception {
            if (msg instanceof HttpRequest) { // 头信息
               //... 创建decoder
            } else if (msg instanceof HttpContent) { // 数据部分
                if (isFileUpload()) {
                    // 处理数据解析和保存
                    processData(ctx, (HttpContent) msg);
                }
            }
            //...
    }
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        //... 释放decoder
        release();
    }  
    protected final void release() {
        try {
            if (decoder != null) {
                decoder.destroy();
                decoder = null;
            }
        } catch (Exception e) {
            logger.warn("destroy http post decoder", e);
        } finally {
            decoder = null;
        }
        //...
    }
    protected void processData(ChannelHandlerContext ctx, HttpContent content) {
        decoder.offer(content);
        // ...
    }
}
```

大致如下逻辑：

* 在检测到HttpRequst时创建decoder
* 在有数据可读时使用decoder读取数据并处理
* 在channel断开连接时释放decoder

一般情况这个逻辑可以正常工作，但既然出现了内存不释放则说明这个逻辑存在问题，需找出这个问题。还是需要想办法复现问题，触发这个控制逻辑潜在的问题。

## 直面堆外内存

从堆外内存的申请与释放的角度来分析

netty的报错堆栈可见出自`io.netty.util.internal.PlatformDependent#incrementMemoryCounter`

```java
    private static void incrementMemoryCounter(int capacity) {
        if (DIRECT_MEMORY_COUNTER != null) {
            long newUsedMemory = DIRECT_MEMORY_COUNTER.addAndGet(capacity);
            if (newUsedMemory > DIRECT_MEMORY_LIMIT) {
                DIRECT_MEMORY_COUNTER.addAndGet(-capacity);
                throw new OutOfDirectMemoryError("failed to allocate " + capacity
                        + " byte(s) of direct memory (used: " + (newUsedMemory - capacity)
                        + ", max: " + DIRECT_MEMORY_LIMIT + ')');
            }
        }
    }
```

该单元使用一个变量计数管理堆外内存的大小：

* 在`incrementMemoryCounter`中申请内存并增加计数

* 在`decrementMemoryCounter`释放内存做减少计数

也就是说，一定是有调用了前者而未调用后者释放的情况，首要问题是知道什么操作会导致内存不释放。
### 尝试复现
通过小内存独立反复测试来尝试复现问题。思路为：

1. 先用一个定时任务一直检查堆外内存，由于看到堆外内存计数使用了静态变量，可方便直接反射获得，这里是在检测到和上次不一样就打印，为了尽量减少一些打印，：

   ```java
   private static long lastDirectMemoryCounter = 0L;
   private static Field directMemoryField;
   
   // 服务启动反射字段并创建定时任务
   directMemoryField = ReflectionUtils.findField(PlatformDependent.class, "DIRECT_MEMORY_COUNTER");
   ReflectionUtils.makeAccessible(directMemoryField);
   
   ScheduledExecutorService scheduleTask = Executors.newScheduledThreadPool(1);
   scheduleTask.scheduleWithFixedDelay(()->{
     AtomicLong result= (AtomicLong) ReflectionUtils.getField(directMemoryField, PlatformDependent.class);
     long value = result.get();
     if (value != lastDirectMemoryCounter) {
       logger.info("PlatformDependent.DIRECT_MEMORY_COUNTER change to {}", value);
       lastDirectMemoryCounter = value;
     }
   }, 1000, 100, TimeUnit.MILLISECONDS);
   ```

2. 调试环境启动文件服务，设定较小的对外内存`-XX:MaxDirectMemorySize=16m`，为了更容易短时间复现问题

3. 在创建和释放`HttpPostRequestDecoder`时增加日志，在channel主要事件上增加一些日志

4. 通过各种方式发文件，包括客户端程序单独发文件、群发文件，客户端接收文件、web页面收发文件等

一般的请求，可见`PlatformDependent.DIRECT_MEMORY_COUNTER`数字涨高但可在传输完成后回落为最初值，直到发现一个web页面https文件传输。

### 解决释放顺序问题
![](https://bed.cdpt.pro/ibed/2021/07/10/Cej4J4Jiz.jpg)

图左为正常情况，内存可回收；图右为非正常情况，最后内存不再变动也就是未回收。

图右的一个影响因素是出现了https证书未知错误`certificate_unknown`，随后触发了`channelInactive`，其中会释放`decoder`，乍一看也是可以释放`decoder`为什么内存还未释放？



经反复确认终于发现，因为：出现错误并触发`channelInactive`时，`decoder`实际还未创建，此时调用`release`其中会检查`decoder != null`才做释放，所以看似有释放实则未释放任何东西，而后续通道还会有数据处理，处理完数据没有再触发`channelInactive`也就没有释放`decoder`。

这里也是侧面证明了前面释放的逻辑不够：**`channelInactive`后还可能有`channelRead`事件触发！**

这里想到的其他方式检测数据传输完毕，在`processData`中实际原来已经判断过最后的http传输片段：

```java
    protected void processData(ChannelHandlerContext ctx, HttpContent content) {
        decoder.offer(content);
        if (!LastHttpContent.class.isInstance(content)) {
            return;
        }
        // ...处理文件传输完成后的状态、记录信息等
        // 最后增加decoder的释放操作
        if (decoder != null)  {
            release();
        }
    }
```

经测试可解决上述问题，在上述情况下堆外内存也可正常回落。

### 进一步观察和分析

#### 仍有未释放情况

经过上面的处理，虽然解决了一种情况的内存未释放问题，但不能保证所有情况。既然确定到这个`decoder`可能未释放，那么继续进一步的处理，就是对`decoder`的创建释放详细的做监控，通过增加日志方式来详细的记录相关内容。

* 记录创建decoder、释放decoder的详细日志，仅在`decoder != null`的情况记录释放日志。由于文件有唯一的fid，在创建、释放decoder时都打印fid参数。

* 后续将创建、释放decoder日志收集出来，找到对应关系，如果有未释放的情况，根据fid看请求相关日志。

日志举例：
```
handleUpload sid=hyxx_1eb58b5_63_3d8ef6591, fid=60e7c39d9ca53e465dcb195e
free decoder, sid=hyxx_1eb58b5_63_3d8ef6591, fid=60e7c39d9ca53e465dcb195e, cause=processLastHttpContent
```
可以用正则分析日志，再转化为sql，创建表将数据入库，再以关联查询的方式检查出同一fid未做free decoder操作的记录，这样起码可以知道是否还有`decoder`未释放的情况。
![](https://bed.cdpt.pro/ibed/2021/07/10/Cej4IDJ0z.jpg)

经过一段时间日志收集，确实发现了有未释放decoder的情况，而`PlatformDependent.DIRECT_MEMORY_COUNTER`也仍然有一直增长直到溢出的问题。

#### 使用心跳检测机制
通过日志看到未释放时并未伴随前面的SSL异常，说明不是同一个问题。

前面的逻辑是在文件传输完或`channelInactive`时做对象释放，那么可从另一方面考虑，如果文件传输因网络或程序问题中断，是不是就无法正常释放。

经过验证，**在传输时断网（笔记本wifi关闭），或杀掉程序，都并不触发`channelInactive`，甚至也不触发`exceptionCaught`，通道既不关闭也不异常**，且文件未传输完毕，这种情况就不会释放对象。

一个解决办法是增加通道的读写心跳检测，长时间无读写操作则关闭通道，关闭通道时释放decoder对象。

```java
// 在ChannelPipeline上注册两个handler
// 一个是IdleStateHandler检测心跳，也就是读写事件空闲的时间，达到一定时长即认为超时
// 另一个是处理心跳超时事件，关闭channel
ch.pipeline().addLast("idleStateHandler", new IdleStateHandler(0, 0, heartbeatTimeout, TimeUnit.SECONDS));
ch.pipeline().addLast(new ServerHeartBeatHandler());

// handler内容
public class ServerHeartBeatHandler extends ChannelInboundHandlerAdapter {
    static final Logger logger = LoggerFactory.getLogger(ServerHeartBeatHandler.class);

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent idleEvent = (IdleStateEvent) evt;
            if (idleEvent.state() == IdleState.ALL_IDLE) { // 一段时间无读写
                logger.warn("channel {} close because all heart beat timeout", ctx.channel().id());
                ctx.channel().close();
            }
        }
        super.userEventTriggered(ctx, evt);
    }
}
```

修改后主动验证客户端断网或杀进程的情况，可以等到一段时间后的心跳超时事件，对资源做释放。线上经过修改后，继续观察日志一段时间，未发现创建decoder后不执行free的情况，证明此方法可解决异常中断引起的不释放问题。

## 总结
* netty的堆外内存问题可通过关注`PlatformDependent.DIRECT_MEMORY_COUNTER`变化来跟踪问题，使用独立小内存环境反复操作来复现，实际排查时还可以覆盖类`PlatformDependent`并增减堆外内存方法里加日志详细看请求与释放情况
* 问题的原因往往不只一个，解决完问题的一个原因后，还需持续跟踪表现情况，可针对性的做记录和对记录结果进行统计分析
* 基于事件驱动的编程时，实际都需要对事件持一定怀疑态度来看，事件未必是在一般认为的时机触发，有时要考虑事件顺序不符合预期，甚至不触发的情况。


## 参考阅读
* [Netty堆外内存泄露排查盛宴](https://tech.meituan.com/2018/10/18/netty-direct-memory-screening.html)
* [Netty整体架构](https://blog.csdn.net/u013857458/article/details/82527722)
* [Netty 系列之 Netty 线程模型](https://www.infoq.cn/article/netty-threading-model)
* [Netty 实战精髓篇](https://www.w3cschool.cn/essential_netty_in_action)