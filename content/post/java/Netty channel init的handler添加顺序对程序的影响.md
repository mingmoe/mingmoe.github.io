+++
title = "Netty Channel Init的handler添加顺序对程序的影响"
date = 2021-10-23T15:07:52+08:00
draft = false
archives = 2021
categories = "java"
tags = ["netty","java"]
+++

<!--TODO-->
今天在研究netty的时候，碰到了一个channel init的问题:
```java
public final class NettyChannelInit extends ChannelInitializer<NioSocketChannel> {
    @Override
    protected void initChannel(NioSocketChannel  socketChannel) throws Exception {
        var pipeline = socketChannel.pipeline();
        pipeline
                // 添加一个拆包器 (input)
                .addLast("packet length parser",new LengthFieldBasedFrameDecoder(
                        Integer.MAX_VALUE,
                        0,
                        4,
                        0,
                        // 丢弃长度
                        4
                ))
                // 添加一个包分类器 (input)
                .addLast("packet classifier",new PacketClassifier())
                // 添加一些实际处理包的handle (input && output)
                .addLast("packet ping handle",new PingPacketHandle())
                // 添加一个输出解码器 (output)
                .addLast("output encoder",new PacketEncoder())
                ;
    }
}
```
这段代码的依据 input-process-output 的handler逻辑顺序（即解码输入handler-逻辑handler-编码输出handler）来添加handler。这将会错误的工作。
<!--more-->

上面的逻辑是:
```
      Input
           |
           |
           |
+----------+-------------+  Channel Pipeline
|          |             |
| +--------v-----------+ |
| |                    |<+---First
| |                    | |
| | Handler            | |
| |                    | |
| +--------+-----------+ |
|          |             |
| +--------v-----------+ |
| |                    | |
| |                    | |
| | Handler            | |
| |                    | |
| +--------+-----------+ |
|          |             |
| +--------v-----------+ |
| |                    | |
| | Handler            | |
| |                    | |
| +--------+-----------+ |
|          |             |
| +--------v-----------+ |
| |                    | |
| | Handler            |<+---Last
| +--------+-----------+ |
|          |             |
+----------+-------------+
           v
        Output
```

实际中，这段代码无法正确处理请求——特别是对于输出。

[官方文档](https://netty.io/4.1/api/io/netty/channel/ChannelPipeline.html)给了一个小小的例子: 
```
  +---------------------------------------------------+---------------+
  |                           ChannelPipeline         |               |
  |                                                  \|/              |
  |    +---------------------+            +-----------+----------+    |
  |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  |               |
  |               |                                  \|/              |
  |    +----------+----------+            +-----------+----------+    |
  |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  .               |
  |               .                                   .               |
  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
  |        [ method call]                       [method call]         |
  |               .                                   .               |
  |               .                                  \|/              |
  |    +----------+----------+            +-----------+----------+    |
  |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  |               |
  |               |                                  \|/              |
  |    +----------+----------+            +-----------+----------+    |
  |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  |               |
  +---------------+-----------------------------------+---------------+
                  |                                  \|/
  +---------------+-----------------------------------+---------------+
  |               |                                   |               |
  |       [ Socket.read() ]                    [ Socket.write() ]     |
  |                                                                   |
  |  Netty Internal I/O Threads (Transport Implementation)            |
  +-------------------------------------------------------------------+
```
这个例子可能太过于不直观了。

而且官方标注了一些关于添加handler的例子:
```java
 pipeline.addLast("decoder", new MyProtocolDecoder());
 pipeline.addLast("encoder", new MyProtocolEncoder());
 pipeline.addLast(group, "handler", new MyBusinessLogicHandler());

 // 另一个例子
 ChannelPipeline p = ...;
 pipeline.addLast("1", new InboundHandlerA());
 pipeline.addLast("2", new InboundHandlerB());
 pipeline.addLast("3", new OutboundHandlerA());
 pipeline.addLast("4", new OutboundHandlerB());
 pipeline.addLast("5", new InboundOutboundHandlerX());
```
可以见到，官方给出的handler添加顺序应该是: input-output-process 

并且给了一段文字（对于上面代码的后一个例子）:
```
 - 3 and 4 don't implement ChannelInboundHandler, and therefore the actual evaluation order of an inbound event will be: 1, 2, and 5.
 - 1 and 2 don't implement ChannelOutboundHandler, and therefore the actual evaluation order of a outbound event will be: 5, 4, and 3.
 - If 5 implements both ChannelInboundHandler and ChannelOutboundHandler, the evaluation order of an inbound and a outbound event could be 125 and 543 respectively.
```

现在，我们可以得出结论了：

 - 对于入站数据(input)，将会从ChannelPipeline的头部开始调用handler，跳过没有实现ChannelInboundHandler的handler
 - 对于出站数据(output)，将会从ChannelPipeline的尾部开始调用handler，跳过没有实现ChannelOutboundHandler的handler


由此，可以绘出一张简单的示例图:
```
   Input         Output    Channel Pipeline
+----+-------------^---+
|    |             |   |<---First
| +--v-------------+-+ |
| |                  | |
| |   Handler        | |
| |                  | |
| +--+-------------^-+ |
|    |             |   |
| +--v-------------+-+ |
| |                  | |
| |   Handler        | |
| |                  | |
| +--+-------------^-+ |
|    |             |   |
| +--v-------------+-+ |
| |                  | |
| |   Handler        | |
| |                  | |
| |                  | |
| +--+-------------^-+ |
|    |             |   |
| +--v-------------+-+ |
| |                  | |
| |   Handler        | |
| |                  | |
| +--+---------------+ |
|    |             ^   |
|    |             |   |<---Last
+----+-------------+---+
     |             |
     +-------------+

     如果handler未实现ChannelInboundHandler;ChannelOutboundHandler;将会跳过Input;Output阶段。
```

将开头的源代码修正，结果将会是:
```java
var pipeline = socketChannel.pipeline();
        // 添加一个解码器来解析数据长度
        pipeline
                // 添加一个拆包器 (input)
                .addLast("packet length parser",new LengthFieldBasedFrameDecoder(
                        Integer.MAX_VALUE,
                        0,
                        4,
                        0,
                        // 丢弃长度
                        4
                ))
                // 添加一个包分类器 (input)
                .addLast("packet classifier",new PacketClassifier())
                // 添加一个输出解码器 (output)
                .addLast("output encoder",new PacketEncoder())
                // 添加一些实际处理包的handle(input && output)
                .addLast("packet ping handle",new PingPacketHandle())
                ;
```
对于数据，将经过:
```
    input                   output
    |                       |
    ↓                       ↑
    packet length parser    output encoder
    ↓                       ↑
    packet classifier       |
    ↓                       ↑
    → -------------------→  packet ping handle
```

well done

<!--TODO-->