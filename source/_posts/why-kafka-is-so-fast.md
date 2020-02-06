---
title: Kafka高性能原理
date: 2020-02-06 23:00:42
tags:
- Kafka
- 中间件
categories:
- Kafka
- 中间件
---

[Kafka](http://kafka.apache.org)是最热门的消息中间件之一，Benchmark中领先一众对手。道理我都懂，但是Kafka为什么这么快？

<!-- more -->

## 什么是Kafka？

说正事之前，先看下什么是Kafka？官方对它的定义是[A distributed streaming platform](http://kafka.apache.org/intro)，即分布式流处理平台，一般用于系统间数据流管道和实时流处理。

它有这么三个比较关键的能力：

1. 发布订阅，可以当做消息队列用
2. 记录的容错持久化
3. 流处理

我觉得还有第四个，就是他的高性能。

## Kafka线程模型

1(Acceptor) + N(Processor) + M(KafkaRequestHandler)，在Netty，Tomcat，Nginx上面都能看见类似的设计

N = num.networker.threads

M = num.io.threads

一个EndPoint(网卡)对于一个Acceptor，一般来说就一个

![Kafka thread](http://media.kosho.tech/blog/20200206/kafka-thread.png)

## kafka存储模型

### 日志文件

Kafka节点上，一个Partition对应一个磁盘目录，命名为<topic_name>_<partition_id>，分为多个LogSegment，一个LogSegment，一个LogSegment对应磁盘上一个日志文件和一个索引文件，日志文件命名规则为[baseOffset].log，baseOffset是日志文件中第一条消息的offset。

写入时数据直接append到文件末尾，所以不管文件多大，写入总是O(1)的时间复杂度。

![Kafka log](http://media.kosho.tech/blog/20200206/kafka-log-file.png)

### 索引文件

索引是分段和稀疏索引的方式，二分查找定位日志位点，返回低位点。和日志不同，索引文件因为比较小，用mmap的方式操作，速度很快。

![Kafka index](http://media.kosho.tech/blog/20200206/kafka-index.png)

## Kafka高性能原理

说了半天终于到最核心的地方了，也就是Kafka为什么这么快！

### 页缓存

Kafka并不太依赖JVM内存大小，而是主要利用Page Cache，如果使用应用层缓存（JVM堆内存），会增加GC负担，增加停顿时间和延迟，创建对象的开销也会比较高。

读取操作可以直接在Page Cache上进行，如果消费和生产速度相当，甚至不需要通过物理磁盘直接交换数据，这是Kafka高吞吐量的一个重要原因。

这么做还有一个优势，如果Kafka重启，JVM内的Cache会失效，Page Cache依然可用。

![Page cache](http://media.kosho.tech/blog/20200206/page-cache.png)

### 零拷贝

Kafka中存在大量的网络数据持久化到磁盘（Producer到Broker）和磁盘文件通过网络发送（Broker到Consumer）的过程。这一过程的性能直接影响Kafka的整体吞吐量。

#### 传统模式下的四次拷贝与四次上下文切换

以将磁盘文件通过网络发送为例。传统模式下，一般使用如下伪代码所示的方法先将文件数据读入内存，然后通过Socket将内存中的数据发送出去。

```
buffer = File.read
Socket.send(buffer)
```

这一过程实际上发生了四次数据拷贝。首先通过系统调用将文件数据读入到内核态Buffer（DMA拷贝），然后应用程序将内存态Buffer数据读入到用户态Buffer（CPU拷贝），接着用户程序通过Socket发送数据时将用户态Buffer数据拷贝到内核态Buffer（CPU拷贝），最后通过DMA拷贝将数据拷贝到NIC Buffer。同时，还伴随着四次上下文切换。

[![四次拷贝 四次上下文切换](http://media.kosho.tech/blog/20200206/zero-copy1.png)](http://www.jasongj.com/img/kafka/KafkaColumn6/BIO.png)

#### sendfile和transferTo实现零拷贝

Linux 2.4+内核通过`sendfile`系统调用，提供了零拷贝。数据通过DMA拷贝到内核态Buffer后，直接通过DMA拷贝到NIC Buffer，无需CPU拷贝。这也是零拷贝这一说法的来源。除了减少数据拷贝外，因为整个读文件-网络发送由一个`sendfile`调用完成，整个过程只有两次上下文切换，因此大大提高了性能。

[![零拷贝 两次上下文切换](http://media.kosho.tech/blog/20200206/zero-copy2.png)](http://www.jasongj.com/img/kafka/KafkaColumn6/NIO.png)

具体实现上，Kafka的数据传输通过TransportLayer来完成，其子类`PlaintextTransportLayer`通过[Java NIO](http://www.jasongj.com/java/nio_reactor/)的FileChannel的`transferTo`和`transferFrom`方法实现零拷贝。

```java
@Overridepublic long transferFrom(FileChannel fileChannel, long position, long count) throws IOException {
    return fileChannel.transferTo(position, count, socketChannel);
}
```

**注：** `transferTo`和`transferFrom`并不保证一定能使用零拷贝。实际上是否能使用零拷贝与操作系统相关，如果操作系统提供`sendfile`这样的零拷贝系统调用，则这两个方法会通过这样的系统调用充分利用零拷贝的优势，否则并不能通过这两个方法本身实现零拷贝。

### 磁盘顺序写入

即使是普通的机械磁盘，顺序访问速率也接近了内存的随机访问速率。

Kafka的每条消息都是append的，不会从中间写入和删除消息，保证了磁盘的顺序访问。

即使是顺序读写，过于频繁的大量小IO操作一样会造成磁盘的瓶颈，此时又变成了随机读写。Kafka的策略是把消息集合在一起，批量发送，尽可能减少对磁盘的访问。所以，Kafka的Topic和Partition数量不宜过多，可以看阿里云的这个测试结果，超过10000w以后Kafka性能会急剧下降。

![Disk sequence access](http://media.kosho.tech/blog/20200206/disk-seq-access.webp.jpg)

### 全异步

Kafka基本上是没有阻塞操作的，调用发送方法会立即返回，等待buffer满了以后交给轮询线程，发送和接收消息，复制数据也是都是通过NetworkClient封装的poll方式。

### 批量操作

结合磁盘顺序写入，批量无疑是非常有必要（如果用的时候每发送一条消息都调用future.get等待，性能至少下降2个数量级）。写入的时候放到RecordAccumulator进行聚合，批量压缩，还有批量刷盘等...

### 精妙的组件设计

Kafka里有不少精妙的设计，比如说DelayedOperationPurgatory和多层TimingWheel，保证了延迟任务的高性能，Quartz和Netty也有类似的东西。